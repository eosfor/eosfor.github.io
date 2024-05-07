---
title: 'Визуализируем трафик, текущий через Azure Firewall c помощью PowerShell, Jupyter и d3js'
date: 2024-05-02T21:36:03-07:00
draft: true
featuredImage: "article/sankey-d3/header.webp"
---

<script type="module" src="display.js"></script>

И снова здравствуйте. В этот раз посмотрим на то, как можно визуализировать трафик, текущий через Azure Firewall, средствами PowerShell, Jupyter и D3js

<!--more-->

На самом деле все относительно несложно. Нам, как обычно, дужно всего лишь вынуть данные из Azure, подготовить их и "скормить" JavaScript библиотеке. С первым все очень, просто. Шлем запрос в Log Analytics, и все. Он содержит все нужные данные.

```powershell
$query = @"
AZFWApplicationRule | where TimeGenerated >= $dateFilter
"@

$data = Invoke-AzOperationalInsightsQuery -WorkspaceId $WorkspaceID -Query $query -ErrorAction Stop | Select-Object -ExpandProperty Results

$sourceIpGroups = $data | Group-Object SourceIp

$dataSet = 
foreach ($group in $sourceIpGroups) {
    $source = $group.Name
    $targets = $group.Group | Group-Object DestinationIp, Action

    foreach($target in $targets) {
        [PSCustomObject]@{
            source = $source;
            sourceName = $source
            target = $target.Group[0].DestinationIp
            action = $target.Group[0].Action
            value = $target.Count
        }
    }
}
```

Теперь дело за трансформацией. Данные для библиотечки должны быть примерно вот в таком виде. Это просто список вершин, и список ребер, с дополнгительным атрибутом - `value`. Атрибут содержит "вес" ребра, собственно некое значение, показывающее "силу" связи.

```json
{
    "nodes": [
        {
            "name": "Agricultural 'waste'"
        },
        {
            "name": "Bio-conversion"
        },
        {
            "name": "Liquid"
        },
        {
            "name": "Losses"
        }
    ],
    "links": [
        {
            "source": 0,
            "target": 1,
            "value": 124.729
        },
        {
            "source": 1,
            "target": 2,
            "value": 0.597
        },
        {
            "source": 1,
            "target": 3,
            "value": 26.862
        },
        {
            "source": 1,
            "target": 4,
            "value": 280.322
        }
    ]
}
```

Таким образом, нам надо пребразовать наши данные к этому виду:

```powershell
$nodes = ($data.source + $data.target) | Select-Object -Unique

$links = $data | % {
    [pscustomobject]@{
        source = $nodes.IndexOf($_.source)
        target = $nodes.IndexOf($_.target)
        value = $_.value
    }
}

[pscustomobject]@{
    nodes = $nodes | % { [pscustomobject]@{name = $_} }
    links = $links
} | ConvertTo-Json -Depth 10 | out-file "traffic-data/data.json"
```

Тут мы просто объединяем список источников и приемников трафика в единый список вершин, удаляем неуникальные записи. Затем формируем список связей в нужном нам формате, и объединяем эти два списка в единый объект. И экспортируем результат в json.
Ну и наконец, мы можем прочесть это файл в JavaScript и, не мудрствуя лукаво, визуализировать его

```js
const width = 800;
const height = 2000;

var data = await d3f.json("traffic-data/data.json");

// Create the SVG container.
const svg = d3.select('#graph-container')
    .append('svg')
    .attr("width", width)
    .attr("height", height)
    .call(d3.zoom().on("zoom", (event) => {
        svg.attr("transform", event.transform);
    }))
    .append("g");

drawSankey();

function drawSankey() {
    var sankey = d3sa.sankey()
        .nodeAlign(d3sa.sankeyLeft)
        .nodeWidth(20)
        .nodePadding(20)
        .extent([[1, 50], [width - 1, height - 5]]);
    
    var graph = sankey(data)

    const color = d3.scaleOrdinal(d3.schemeSet3);

    // Drawing nodes
    const rect = svg.append("g")
        .attr("stroke", "#000")
        .selectAll("rect")
        .data(graph.nodes)
        .join("rect")
        .attr("x", d => d.x0)
        .attr("y", d => d.y0)
        .attr("height", d => d.y1 - d.y0 >= 3 ? d.y1 - d.y0 : 3)
        .attr("width", d => d.x1 - d.x0)
        .attr("fill", d => color(d.name));
    
    rect.append("title")
        .text(d => `${d.name}\n${d.targetLinks.length > 0 ? d.targetLinks.map(o => o.source.name).join("\n") : ""}`);

    // Creating gradients for links
    const defs = svg.append("defs");
    graph.links.forEach((link, i) => {
        const gradient = defs.append("linearGradient")
            .attr("id", "gradient" + i)
            .attr("gradientUnits", "userSpaceOnUse")
            .attr("x1", link.source.x1)
            .attr("x2", link.target.x0);

        gradient.append("stop")
            .attr("offset", "0%")
            .attr("stop-color", color(link.source.name));

        gradient.append("stop")
            .attr("offset", "100%")
            .attr("stop-color", color(link.target.name));
    });

    // Drawing links with gradient
    svg.append("g")
        .attr("fill", "none")
        .attr("stroke-opacity", 0.5)
        .selectAll("path")
        .data(graph.links)
        .join("path")
        .attr("d", d3sa.sankeyLinkHorizontal())
        .attr("stroke", (d, i) => `url(#gradient${i})`)
        .attr("stroke-width", d => Math.max(1, d.width))
        .append("title")
        .text(d => `${d.source.name} → ${d.target.name}`);

    // Drawing labels for the nodes
    svg.append("g")
        .selectAll("text")
        .data(graph.nodes)
        .join("text")
        .attr("x", d => d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6)
        .attr("y", d => (d.y1 + d.y0) / 2)
        .attr("dy", "0.35em")
        .attr("text-anchor", d => d.x0 < width / 2 ? "start" : "end")
        .attr("font-size", "15px")
        .text(d => d.name);
}
```

Получается примерно следующее:

<div style="display: flex; justify-content: space-between;">
    <div id="graph-container" style="flex-grow: 1;"></div>
</div>

Готовый Jupyter notebook можно найти [тут](https://github.com/eosfor/scripting-notes/blob/main/notebooks/en/traffic-through-AzureFW-d3js.ipynb)

{{< video type="youtube" id="0RDeLdTq4Is" >}}