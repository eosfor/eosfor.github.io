---
title: 'Визуализируем Azure Networking при помощи D3js'
date: 2024-04-29T16:26:08-07:00
draft: true
---

<script type="module" src="display.js"></script>

<style>
#side-card {
    width: 20%;
    overflow-y: auto;
    background: #fff; /* Background color */
    padding: 10px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.1); /* Shadow for raised effect */
    border-radius: 5px; /* Optional: adds rounded corners */
}

#node-list {
    display: grid;
    grid-template-columns: repeat(2, 1fr); /* Creates two columns */
    gap: 5px; /* Space between items */
    list-style: none; /* Removes default list styling */
    padding: 0;
}

#node-list li {
    background: #f8f8f8; /* Light background for each item */
    padding: 5px;
    border-radius: 3px; /* Rounded corners for list items */
    cursor: pointer; /* Indicates interactivity */
}

#filter-input {
    margin-bottom: 10px; /* Spacing between input and list */
    padding: 8px;
    width: calc(100% - 16px); /* Full width taking padding into account */
    box-sizing: border-box; /* Includes padding and border in width */
    border-radius: 5px; /* Rounded corners for input */
    border: 1px solid #ccc; /* Subtle border for the input */
}

#node-list li:hover {
    background-color: lightgray;  // Highlight list item on hover
    cursor: pointer;
}

circle {
    transition: all 0.3s ease;  // Smooth transition for changes in size and color
}
</style>


Изображение стоит тысячи слов. Когда вы работаете со сложной сетевой инфраструктурой, было бы замечательно иметь вид сверху. В этой статье я хочу обсудить, как это можно достичь с помощью PowerShell, Jupyter notebooks и [d3js](https://d3js.org/)
<!--more-->

Мы уже знакомы с ядром .NET Interactive и Jupyter notebooks. Теперь мы хотим представить новый инструмент - d3js. Вкратце, это мощная визуализационная платформа для JavaScript. Среди прочего, она может помочь визуализировать графические и сетевые структуры. По сути, нам нужно построить собственный граф зависимостей между различными элементами сети, а затем преобразовать эти данные таким образом, чтобы их могла использовать d3js.

Первое, что мы сделаем, - это использование предварительной версии модуля [PSQuickGraph](https://www.powershellgallery.com/packages/PSQuickGraph/1.1), который позволяет динамически генерировать графические структуры.

```powershell
Install-Module -Name PSQuickGraph -AllowPrerelease -RequiredVersion "2.0.2-alpha"
Import-Module PSQuickGraph -RequiredVersion "2.0.2"
```

Теперь мы можем инициализировать граф и собрать наши сетевые элементы из Azure.

```powershell
$g = New-Graph

# pull necessary data
$vnets = Get-AzVirtualNetwork
$nics = Get-AzNetworkInterface
```

Далее нам нужно обработать наши данные и построить из них граф. Мы фактически сканируем наши массивы `$vnets` и `$nics`, добавляем их элементы в виде вершин и соединяем их направленными ребрами, когда они связаны: VNETs включают Subnets, а NICs прикреплены к Subnets.

```powershell
# add vnets and peerings to the graph
$vnets | ForEach-Object {
    $currentVnet = $_
    $vnetVertex = [PSGraph.Model.PSVertex]::new($currentVnet.Id, $currentVnet)
    Add-Vertex -Graph $g -Vertex $vnetVertex
    $currentVnet.Subnets | % {
        $currentSubnet = $_
        $subnetVertex = [PSGraph.Model.PSVertex]::new($currentSubnet.Id, $currentSubnet)
        Add-Edge -Graph $g -From $vnetVertex -To $subnetVertex
    }
}

foreach ($v in $g.Vertices){
    foreach($p in $v.OriginalObject.VirtualNetworkPeerings) {
        foreach ($rvn in $p.RemoteVirtualNetwork) {
            $targetVertex = $g.Vertices.Where({$_.Label -eq $rvn.id})[0]
            Add-Edge -From $v -To $targetVertex -Graph $g
        }
    }
}

# add NICs to the graph
$nics | ForEach-Object {
    $vnetID = $_.IpConfigurations[0].Subnet.Id
    $targetVertex = $g.Vertices.Where({$_.Label -eq $vnetID})[0]
    Add-Edge -Graph $g -From ([PSGraph.Model.PSVertex]::new($_.name, $_)) -To $targetVertex
}

```

На этом этапе мы хотим использовать немного магии JS и D3, чтобы визуализировать граф. Часть JS добавляет несколько приятных функций пользовательского интерфейса, таких как список элементов, чтобы мы могли искать определенный узел в графе или выделять узел и другие прямо связанные узлы при клике.

This is how it looks like:

<div style="display: flex; flex-direction: column; width: 80%; margin: auto;">
    <div id="graph-container" style="width: 100%; height: 800px;"></div>
    <div id="side-card" style="width: 100%;">
        <input type="text" id="filter-input" placeholder="Filter nodes...">
        <ul id="node-list"></ul>
        <div id="pagination"></div>
    </div>
</div>

Вы можете увидеть все это в готовом к использованию [jupyter notebook](https://github.com/eosfor/scripting-notes/blob/main/notebooks/en/vnet-topology-visualization-d3js.ipynb). 

{{< video type="youtube" id="qnWar8mPbfg" >}}