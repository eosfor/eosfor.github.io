---
title: 'Visualizing Azure Networking using D3js'
date: 2024-04-29T16:26:08-07:00
draft: true
featuredImage: "article/azure-networking-d3js/header2.webp"
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


A picture worth a thousand words. When you work with a complex networking infrastructure, it would be great to have a bird's-eye view of it. In this article, I want to discuss how this can be achieved using PowerShell, Jupyter notebooks, and [d3js](https://d3js.org/)
<!--more-->

We are already familiar with the .NET Interactive kernel and Jupyter notebooks. Now, we want to introduce a new tool - [d3js](https://d3js.org/). In a few words, it is a powerful visualization framework for JS. Among other features, it can help visualize graph and network structures using a force-aware algorithm. Basically, what we need to do is to build our own graph of dependencies between various networking elements, and then transform this data in such a way that it can be consumed by d3js.

First thing we are going to do is to use the preview version of the [PSQuickGraph](https://www.powershellgallery.com/packages/PSQuickGraph/1.1) module, which lets us generate graph structures dynamically.

```powershell
Install-Module -Name PSQuickGraph -AllowPrerelease -RequiredVersion "2.0.2-alpha"
Import-Module PSQuickGraph -RequiredVersion "2.0.2"
```

Now we can initialize the graph and collect our networking elements from Azure

```powershell
$g = New-Graph

# pull necessary data
$vnets = Get-AzVirtualNetwork
$nics = Get-AzNetworkInterface
```

Next, we need to process our data and build a graph out of it. We basically scan our `$vnets` and `$nics` arrays, add their elements as vertices and connect them with directed edges, when they are related: VNETs include Subnets, and NICs are attached to Subnets.

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

At this point we want to use a bit of JS and D3 magic, to visualize the graph. The JS piece adds a few nice UI features, like a list of elements, so we can search for a specific node in a graph, or highlight a node and other directly connected nodes when clicked.

This is how it looks like:

<div style="display: flex; flex-direction: column; width: 80%; margin: auto;">
    <div id="graph-container" style="width: 100%; height: 800px;"></div>
    <div id="side-card" style="width: 100%;">
        <input type="text" id="filter-input" placeholder="Filter nodes...">
        <ul id="node-list"></ul>
        <div id="pagination"></div>
    </div>
</div>


You can see the whole thing in a ready-to use [jupyter notebook](https://github.com/eosfor/scripting-notes/blob/main/notebooks/en/vnet-topology-visualization-d3js.ipynb).

{{< video type="youtube" id="qnWar8mPbfg" >}}