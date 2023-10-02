---
title: 'Search Azure Objects'
date: 2015-11-02 19:35:54.000000000+03:00
draft: true
tags: ['Powershell', 'Azure']
---

In our environment we have more than one subscription. Even more than five. And this amount grows constantly. What we do with all of this is we support VMs and stuff there. One of the issues is that users usually do not know name of Azure Service which is used to host their VMs, and it so happens that they do not know even their Subscription Name or Subscription ID. To bring up and control their VMs and environments the use some “middleware” which hides all of this info from them. So they usually come to us and say: “here you are a list of VMs we have problems with, please have a look and fix”. Sometimes such list contains not only VMs but storage accounts along with VMs etc. Unfortunately classic azure cmdlets does not provide an option to search for objects in the cloud by their name. On the other hand new Azure Management portal does. It can find objects by name. So I decided to do a little hack end use this API in order to simplify our live.

## Solution

I have to admit that I’m not a developer – I’m an infrastructure guy. I’m quite far from all of that REST stuff so my definitions might be not really accurate. I’ll try to describe some steps I took to find and use this “hidden” API, as far as I did not find any documentation regarding it. Actually, it was pretty easy to find the actual  REST call using [Fiddler](https://www.telerik.com/fiddler). If it does not wok for you now, as it was working couple of weeks ago, you can use Message Analyzer for the same. They both do the trick. So I reproduced the search query in new portal in browser and captured the REST call. It was a call to URI **https://management.azure.com/api/invoke**. The interesting thing was a **x-ms-path-query**’ header containing the following:

```console
/resources?api-version=2014-04-01-preview;$filter=(subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID') and ((substringof('SomeName', name) or substringof('SomeName', resourcegroup)))
```

I was trying to google for pieces of it but did not find anything, except that it looked like a subset of [Azure AD Graph API](https://msdn.microsoft.com/en-us/library/azure/dn727074.aspx). Well, that was something, so next step was just to try to call it via PowerShell. But how to authenticate over Azure and put correct "authenticator" into HTTP request? [This](http://blogs.technet.com/b/keithmayer/archive/2014/12/30/authenticating-to-the-azure-service-management-rest-api-using-azure-active-directory-via-powershell-list-azure-administrators.aspx) helped me a lot.

After I got the Auth Header I was able to run a REST query captured at the very beginning. REST call returns a set of JSON objects, i parse them for later use and they look like below

```console
PS C:\>Get-AzureObject -Name ACDB2-AG -RawOutput -All
SubscriptionID : 35882c3a-d01e-479c-a46e-67903c9cae39
type : Microsoft.ClassicCompute/virtualMachines
location : eastus
ObjectName : VMNAME-1
ResourceGroup : ResourceGroup1
SubscriptionID : 49a40aa2-5cb7-40ca-abd9-9de732c93a0d
type : Microsoft.ClassicCompute/virtualMachines
location : eastus
ObjectName : VMNAME-1
ResourceGroup : ResourceGroup2
```

So whole solution works as follows:

- Runs a query as is against all registered subscriptions and names provided
- Parses the results and removes all that is not required according to parameters, for example, remove all results except VMs
- For each object from step 3 uses classic cmdlets to receive actual objects using Subscription, ObjectName and resource group returned by the original search query


```powershell
<#
# Load ADAL Assemblies
$adal = "${env:ProgramFiles(x86)}\Microsoft SDKs\Azure\PowerShell\ServiceManagement\Azure\Services\Microsoft.IdentityModel.Clients.ActiveDirectory.dll"
$adalforms = "${env:ProgramFiles(x86)}\Microsoft SDKs\Azure\PowerShell\ServiceManagement\Azure\Services\Microsoft.IdentityModel.Clients.ActiveDirectory.WindowsForms.dll"
[System.Reflection.Assembly]::LoadFrom($adal)
[System.Reflection.Assembly]::LoadFrom($adalforms)
#>

function Get-AzureObject {
[CmdletBinding()]
param(
    [Parameter(Mandatory=$true)]
    [string[]]$Name,
    [Parameter(Mandatory = $false)]
    [string]$SubscriptionName,
    $apiVersion = '2014-04-01-preview',
    [Parameter()]
    [switch]$VMOnly,
    [switch]$ServiceOnly,
    [switch]$StorageOnly,
    [switch]$All,
    [Parameter(Mandatory=$false, HelpMessage = 'Returns raw data, works faster')]
    [switch]$RawOutput,
    $ADTenant = "yourADTenantNameHere.onmicrosoft.com",
    $authHeader
)
begin{
    if (! $PSBoundParameters["authHeader"]) {$authHeader = Get-AzureAuthHeader -ADTenant $ADTenant}

    ## hashtable with resource types and actions for each type
    $typesToFilter = @{'Microsoft.ClassicCompute/virtualMachines' = {param($id, $rg, $n) Get-AzureVM -SubscriptionName (Get-AzureSubscription -SubscriptionId $id).SubscriptionName -ServiceName $rg -Name $n};
                       'Microsoft.ClassicCompute/domainNames' = {param($id, $rg, $n) Get-AzureService -SubscriptionName (Get-AzureSubscription -SubscriptionId $id).SubscriptionName -ServiceName $rg};
                       'microsoft.classicstorage/storageaccounts' = {param($id, $rg, $n) getStorageAccount $id $rg $n}}
    
    ## query string to include all subscriptions registered by using Add-AzureAccount (subscriptions part of the query)
    $subscrFilterString = ?: {$PSBoundParameters['SubscriptionName']} {generateFilterStringForSubscription -SubscriptionName $SubscriptionName} {generateFilterStringForSubscription} 

    $headers = @{"x-ms-version"="$headerDate";
                "Authorization" = $authHeader;
                'Accept' = 'application/json'}

    
    # API method
    $method = "GET"

    #defaultFilter
    $foundFilterString = @() #
}
process{
    ## by default function runs a query for all objects and after that it filters out the
    ## resulting set by just removing unnecessary stuff
    ## REST filter

    ## prepare set of filters to remove unnecessary stuff afterwards
    if ($VMOnly.IsPresent){
       $foundFilterString +=  "(`$_.type -eq 'Microsoft.ClassicCompute/virtualMachines')"
    }

    elseif ($ServiceOnly.IsPresent){
       $foundFilterString +=  "(`$_.type -eq 'Microsoft.ClassicCompute/domainNames')"
    }

    elseif ($StorageOnly.IsPresent){
       $foundFilterString +=  "(`$_.type -eq 'microsoft.classicstorage/storageaccounts')"
    }
    else {
        $foundFilterString = $typesToFilter.Keys | % {"(`$_.type -eq '$_')"}
    }

    ## build filter string for REST Query call
    $objectFilter = ($Name | %{ ("substringof('$_',name)", "substringof('$_',resourcegroup)") -join " or " }) -join " or "
    
    ## query header (name part of the filter)
    $headers.'x-ms-path-query' = "/resources?api-version=$apiVersion&`$filter=($subscrFilterString) and ($objectFilter)"

    # generate the API URI
    $URI = "https://management.azure.com/api/invoke"

    # execute the Azure REST API
    $list = Invoke-RestMethod -Uri $URI -Method $method -Headers $headers -ErrorAction stop
    
    ## parse received objects
    $objectsFound = 
        $list.value | %{
            $element = $_
            $r = [regex]::Match($element.id, "/subscriptions/(?<SubscriptionID>.+)/resourceGroups/(?<ResourceGroup>.+?)/.+/(?<ObjectName>.+)$")
            new-object psobject -Property @{SubscriptionID = $r.Groups["SubscriptionID"]; ResourceGroup = $r.Groups["ResourceGroup"]; ObjectName = $element.name; type = $element.type; location = $element.location}
        }  
    
    ## remove unnecessary results
    $resultingFilterStr = ($foundFilterString -join " -or ")
    write-verbose $resultingFilterStr
    
    $foundFilter = [scriptblock]::Create($resultingFilterStr)

    $filteredObjects = $objectsFound | where $foundFilter


    if (! $all.IsPresent){
        ## if -All is set return all objects
        $filteredObjects = $filteredObjects | where ObjectName -in $Name
    }
    
    if ($RawOutput.IsPresent) {$filteredObjects; return}

    ## query objects using classic cmdlets based on filtered results
    $filteredObjects | % {& $typesToFilter[$_.type] $_.SubscriptionID $_.ResourceGroup $_.ObjectName}
}
}

function Get-AzureAuthHeader {
[CmdletBinding()]
param($ADTenant = "yourADTenantNameHere.onmicrosoft.com")
    Write-Verbose "Getting auth header"
    # Set well-known client ID for AzurePowerShell
    $clientId = "1950a258-227b-4e31-a9cf-717495945fc2" 
    # Set redirect URI for Azure PowerShell
    $redirectUri = "urn:ietf:wg:oauth:2.0:oob"
    # Set Resource URI to Azure Service Management API
    $resourceAppIdURI = "https://management.core.windows.net/"
    # Set Authority to Azure AD Tenant
    $authority = "https://login.windows.net/$ADTenant"
    # Create Authentication Context tied to Azure AD Tenant
    $authContext = New-Object "Microsoft.IdentityModel.Clients.ActiveDirectory.AuthenticationContext" -ArgumentList $authority
    # Acquire token
    $authResult = $authContext.AcquireToken($resourceAppIdURI, $clientId, $redirectUri, "Auto")
    # API header
    $headerDate = '2014-10-01'
    $authHeader = $authResult.CreateAuthorizationHeader()


    $authHeader
}

function generateFilterStringForName{
[CmdletBinding()]
param($Name, [switch]$VMOnly, [switch]$ServiceOnly, [switch]$StorageOnly)
$nameFilterString = 
    ($Name | % {
        $current = $_
        if ($VMOnly.IsPresent) {"substringof('$current',name)"}
        elseif ($ServiceOnly.IsPresent) {"substringof('$current',resourcegroup)"}
        elseif ($StorageOnly.IsPresent) {"substringof('$current',name)"}
        else {("substringof('$current',name)", "substringof('$current',resourcegroup)", "substringof('$current',name)")}
    }) -join " or "

write-verbose "Names filter`: $nameFilterString"
$nameFilterString
}

function generateFilterStringForSubscription{
[CmdletBinding()]
param($SubscriptionName)
    if ($PSBoundParameters['SubscriptionName']){
        $subscriptions = (Get-AzureSubscription -SubscriptionName $SubscriptionName).SubscriptionId
    }
    else {
        $subscriptions = (Get-AzureSubscription).SubscriptionId
    }

    $subscrFilterString = ($subscriptions | % {"subscriptionId eq '$_'"}) -join ' or '

    write-verbose "Subscriptions filter`: $subscrFilterString"
    $subscrFilterString
}

function getStorageAccount {
param($id, $rg, $n)
    $sub = Get-AzureSubscription -SubscriptionId $id
    $acct = Get-AzureStorageAccount -StorageAccountName $n -SubscriptionName $sub.SubscriptionName 3> $null

    $acct
}
```

I have to say that this piece of code uses some proxy functions that I wrote to hide Subscription context switching. But I'll describe them next time.
