---
layout: post
title:  "Dynamically adding a MyGet feed to your VSTS build process"
date:   2016-12-19 16:53:56 +0200
categories: vsts
---
# Overview

Some history: when it was launched for the first time, Visual Studio Team Services (formerly Visual Studio Online or Team Foundation Server Online) had no support for hosting your NuGet packages into the same platform as your code. Later on, they added a very basic support in the form of a NuGet Packages feed. It lacked lots of features, but it worked. And it was nicely integrated with VSTS. So we all rushed and adopted this "free" feature. And everything was easy and beautiful. 

But suddenly, one dark night, Microsoft did what we knew it would eventually do. Visual Studio Package Management ended its beta stage and was launched as [General Availability](https://www.visualstudio.com/en-us/articles/news/2016/nov-16-team-services). That means, more or less, that you had to pay to continue using the service. At this point, we had to choose whether to continue using Microsoft's package Management or move to a different solution. 

Setting up an on-premise service was out of question. We wanted something easy to set up that required no manteinance. After all, we don't want to reinvent the wheel. And of course, we wanted it to be free or cheap. Then we stumbled upon MyGet.

## MyGet features

MyGet offers lots of functionalities and costs less than Microsoft's package management solution. Here's a succint list of features:

*   It's fast: one of the problems we had with VSTS Package manager it's that it was slow. Very slow.
*   Symbols: we can publish symbols packages. This solves one of the typical headaches when dealing with the pubishing of symbols, namely debugging our own libraries.
*   Other kind of packages: npm, bower and vsix. I don't use them currently, but I eventually will.
*   Reliable: VSTS Package Manager suffered from fails from time to time. MyGet is well tested and robust,
*   Cheaper: it's not free, but it's ~30% cheaper than Microsoft
*   Allows statistics, quotas, package cleanup, galleries and several other features not present on VSTS.
*   It's nicer: overall, it looks a lot sexier.

## Migration process

Considering what I need, talking to MyGet support and reading the docs, I managed to set up a VSTS environment that had little impact on current developers.

### Set up the new MyGet feed

Create a new MyGet feed with the name of your company

[https://acme.myget.org/F/myfeed/api/v3/index.json](https://acme.myget.org/F/myfeed/api/v3/index.json)

Then, the current VSTS feed is added as an Upstream source. This means that you can add packages from the VSTS feed and the packages downloaded are automatically added to the MyGet feed. [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_12h43_29-1024x471.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_12h43_29.png) To make things simpler, I added all the latest versions of existing packages in the newly created feed, so existing developers can still use the new feed without changes. A new user is created to access the feed from VSTS. We will need the URLs for pushing packages and symbols and the API key. Both can be found in the Feed Details tab [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_15h24_57.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_15h24_57.png)

### Configure releases to publish to MyGet

Once we have the feed created, we need our Release Definitions to publish to it. We intend to publish normal packages and symbols packages. Symbol packages are created by invoking NuGet.exe with the -Symbols parameter. This creates two package files: .nupkg and .symbols.nupkg. So, we need to:

*   Modify nuspec packages to include PDB files
*   Add MyGet endpoints to VSTS
*   Make NuGet pack command to include the -Symbols parameter

#### Add PDB files in .nuspec files

We have to modify the .nuspec files so that PDB files are included in the package. Let's see an example: 

{% highlight xml %}
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
    <metadata>
        <id>Acme.Marketing.Tracking</id>
        <version>1.3.0.0</version>
        <title>ACMEMarketing Tracking</title> 
        [...]
    </metadata> 
</package>
{% endhighlight %}

As you can see, we are only including the assembly files in package. If we also want to include the PDB files, we can change the file to something like this:

{% highlight xml %}
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
    <metadata>
        <id>Acme.Marketing.Tracking</id>
        <version>1.3.0.0</version>
        <title>ACMEMarketing Tracking</title>
         [...]
    </metadata>
</package>
{% endhighlight %}

#### Add new endpoints for MyGet

Next, we add two new endpoints to each Team Project that needs to publish to MyGet, one for normal packages and the other for symbol packages. This is done by browsing to Project Settings → Services and adding generic endpoints. We need the following endpoints:

*   Push packages: `https://acme.myget.org/F/myfeed/api/v2/package`
*   Push symbols: `https://acme.myget.org/F/myfeed/symbols/api/v2/package`

To set the credentials for both endpoints, we simply use an empty string as the username and the API Key as the password.

#### Change Release definition to publish to MyGet

The next step is to change the Release Definitions so that the two packages are published to MyGet. For this purpose, we have created a Group Task that we share among the different definitions. The Task Group has two steps, one to push normal packages and other to push symbol packages. The first step is a NuGet publish task that gets all packages except symbol packages and pushes them to MyGet

*   Path/pattern to nupkg: `$(NuPackagesFolder)/*.nupkg;-:$(NuPackagesFolder)/*.symbols.nupkg`
*   Feed Type: External NuGet feed
*   NuGet Server Endpoint: the feed endpoint created in previous step

The second step is a NuGet publish task that gets all symbol packages and pushes them to MyGet

*   Path/pattern to nupkg: `$(NuPackagesFolder)/*.symbols.nupkg`
*   Feed Type: External NuGet feed
*   NuGet Server Endpoint: the symbols feed endpoint created in previous step

IMPORTANT: in both steps, make sure you change the NuGet.exe version to 3.5\. Otherwise you'll face an error when pushing packages. [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_15h31_10.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_15h31_10.png)

### Configure Build Definitions to consume MyGet feed

This process is simple to understand, but slightly more complex to implement. This is what we need to do

*   Add a build step that injects the MyGet NuGet feed and sets the required credentials in the Build Agent
*   Restore NuGet packages

#### Add MyGet source to Nuget in the Build Agent

This one is the tricky part. We need to add our current MyGet source to the list of available upstream sources in the build agent. And we want it to add it so that feed credentials are not visible at any time during the build process. We have developed a custom task that does excatly this. We wil add it to each build definition (more on how to batch add tasks later). [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h35_26-1024x494.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h35_26.png) This task takes the following parameters:

*   Feed name: the name of the feed. You can use whatever meaningful name here, such as MyGetFeed.
*   Feed URL: the URL of the feed. E.g.: `https://acme.myget.org/F/myfeed/api/v3/index.json`
*   User name: MyGet user name. We will use a variable that we will set. Thus we set this parameter to $(MyGetUsername)
*   User password: MyGet password. We will use a variable that we will set. Thus we set this parameter to $(MyGetPassword)
*   NuGet.exe path: you can specify a custom NuGet.exe executable, if you wish. Otherwise leave empty to use the default NuGet.exe from the agent.

Now, we need to add the variables to set credentials:

*   MyGetUserName: MyGet user name
*   MyGetPassword: MyGet password. mark this variable with the little keylock icon so that it is marked as secret.

Set the correct values for the variables (your MyGet credentials) and you're done. [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h37_08.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h37_08.png)

#### Restore NuGet packages

Now the easy part. Just add a NuGet restore packages task AFTER the previous task. Do not set a specific NuGet.config file, leave empy so NuGet.exe takes the sources added before. [![](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h44_15.png)](http://fernando.ariasmarques.com/wp-content/uploads/2016/12/2016-12-19_16h44_15.png)

## The custom Task

The custom task just removes the source if it exists and then adds it to the list of NuGet sources 

{% highlight powershell %}
[CmdletBinding(DefaultParameterSetName = 'None')]
param
(
    [string] [Parameter(Mandatory = $true)]
    $SourceName,

    [string] [Parameter(Mandatory = $false)]
    $FeedUrl,

    [string] [Parameter(Mandatory = $false)]
    $Username,

    [string] [Parameter(Mandatory = $false)]
    $Password,

    [string] [Parameter(Mandatory = $false)]
    $nuGetPath
)

function GetProjectFiles {
    param( [string] $sourcesDirectory )
    gci -Path $sourcesDirectory -Recurse -Include *.??proj | foreach { gci -Path $_.FullName -Recurse }
}

Write-Verbose "Entering script $MyInvocation.MyCommand.Name"
Write-Verbose "Parameter Values"
foreach($key in $PSBoundParameters.Keys)
{
    Write-Verbose ($key + ' = ' + $PSBoundParameters[$key])
}

Write-Verbose "Importing modules"
import-module "Microsoft.TeamFoundation.DistributedTask.Task.Internal"
import-module "Microsoft.TeamFoundation.DistributedTask.Task.Common"

$buildSourcesDirectory = Get-TaskVariable $distributedTaskContext "build.sourcesdirectory"

$useBuiltinNuGetExe = !$nuGetPath

if($useBuiltinNuGetExe)
{
    $nuGetPath = Get-ToolPath -Name 'NuGet.exe';
}

if (-not $nuGetPath)
{
    throw ("Unable to locate 'nuget.exe'")
}

$initialNuGetExtensionsPath = $env:NUGET_EXTENSIONS_PATH
try
{
    if ($env:NUGET_EXTENSIONS_PATH)
    {
        if($useBuiltinNuGetExe)
        {
            # NuGet.exe extensions only work with a single specific version of nuget.exe. This causes problems
            # whenever we update nuget.exe on the agent.
            $env:NUGET_EXTENSIONS_PATH = $null
            Write-Warning ("The NUGET_EXTENSIONS_PATH environment variable is set, but nuget.exe extensions are not supported when using the built-in NuGet implementation.")   
        }
        else
        {
            Write-Host ("Detected NuGet extensions loader path. Environment variable NUGET_EXTENSIONS_PATH is set to: " + $env:NUGET_EXTENSIONS_PATH)
        }
    }


    $LocalPath = $ENV:BUILD_REPOSITORY_LOCALPATH

    $addSourceCommand = "sources add -name ""$SourceName"" -source $FeedUrl -username ""$Username"" -password ""$Password"""
    $removeSourceCommand = "sources remove -name ""$SourceName"" -username ""$Username"" -password ""$Password"""
    try {
        Write-Output "Removing package Source"
        Invoke-Tool -Path $nugetPath -Arguments "$removeSourceCommand"
    } catch {
        $Error.Clear();
        Write-Output "'$SourceName' not found into sources."
    }

    try {
        Write-Output "Adding package source '$SourceName'"
        Invoke-Tool -Path $nugetPath -Arguments "$addSourceCommand"
    } catch {
        $Error.Clear();
        Write-Output "Can't add source. Skipping it. Probably NuGet restore will fail :-("
    }
}
finally
{
    $env:NUGET_EXTENSIONS_PATH = $initialNuGetExtensionsPath
}

Write-Verbose "Leaving script $MyInvocation.MyCommand.Name"

{% endhighlight %}

That's all! Thanks for reading.
