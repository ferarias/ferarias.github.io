---
layout: post
title:  "Scripting VSTS with PowerShell"
date:   2017-06-21 15:20:00 +0200
categories: build myget vsts
---
## Introduction

I have found myself repeating a lot of boilerplate tasks in Visual Studio Team Services, involving the creation of build/release definitions. I have been thinking for a long time that I should automate the process but I didn't find the time. A colleague at my job developed a simple script to create variable groups, so I thought I could start from there... 

The process led me into learning a little of [PowerShell](https://www.pluralsight.com/courses/powershell-first-day), the [VSTS REST API](https://www.visualstudio.com/en-us/docs/integrate/api/overview) and some [hacks](https://developercommunity.visualstudio.com/content/problem/71289/vsts-api-conditions-for-running-a-task.html) you'll see in a moment. So, here you have a tutorial about accessing the VSTS REST API from a PowerShell script.

## Pre-requisites

I assume you have some knowledge about Visual Studio Team Services and build/release definitions, so you can understand what I'm talking about. It it's not the case, you're in the wrong place. You will need a Personal Access Token (PAT) for connecting to the API. If you haven’t one, [create it.](https://www.visualstudio.com/en-gb/docs/integrate/get-started/auth/overview) 

You will need PowerShell installed. If you are using Windows 10, it will be already there. Fire up PowerShell ISE (press the Windows key and start writing "ise"). Create a new script. Give it a descriptive name such as "create-great-builddefinitions.ps1" and save it.

## Connecting to the API

Let's create a basic skeleton of a script that allows us to connect to the API. 

{% highlight powershell %}
[CmdletBinding(SupportsShouldProcess=$true)]
Param(    
    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string]
    $PAT='INSERT A DEFAULT PAT HERE'
)

# Authorization header
$headers = @{Authorization = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$PAT"))}

# Get projects
$projectsApiUrl = 'https://acme.visualstudio.com/DefaultCollection/_apis/projects?api-version=1.0'
$projects = Invoke-RestMethod -Headers $headers $projectsApiUrl 
$projects.value | Select id,name | Write-Host
{% endhighlight %} 

That's it! Save the script and run it (press F5). You'll get something like the following:

> @{id=00000000-7b45-4fca-81ab-00abcdefffff; name=Horses} @{id=00000000-2770-406f-8c36-00abcdefffff; name=Cars} @{id=00000000-5601-4b66-aba8-00abcdefffff; name=Runtime} @{id=00000000-a748-4411-b228-00abcdefffff; name=Properties} @{id=00000000-ad90-43f2-80f3-00abcdefffff; name=Syndicate}

## Explanation

Let's analyze a little bit the script. We are accepting a parameter named PAT. If you don't provide it, the script will use the default you put in the script. Obviously this is for convenience, but if you want you can remove the default value and make your script ask for the PAT everytime you use it. Once we have a PAT, we use it to set an Authorization header for the HTTP request. As you may know, the format is similar to this:

> `Authorization = 'Basic XXXXXXXX'`

where _XXXXXXXX_ is a Base-64 string representation of the string "user:password". 

Since we are using a PAT, the user part is an empty string, hence the ":$PAT". Now that we have a header to authenticate, let's perform the actual request. You can find the API call to obtain projects from [here](https://www.visualstudio.com/en-us/docs/integrate/api/tfs/projects). 

{% highlight powershell %}
$projectsApiUrl = 
  'https://acmd.visualstudio.com/DefaultCollection/_apis/projects?api-version=1.0'
{% endhighlight %} 

Now we use PowerShell's Invoke-RestMethod and pass the authentication headers: 

{% highlight powershell %}
$projects = 
  Invoke-RestMethod -Headers $headers $projectsApiUrl
{% endhighlight %} 

Finally, we pass the results through a pipeline that selects (pretty-prints) the results: 

{% highlight powershell %}
$projects.value | Select id,name | Write-Host
{% endhighlight %}

That was simple, wasn't it? Now you know how to obtain all your team projects. Now let's proceed to something more useful. I recommend you read this before proceeding to the next section: [Get started with the REST APIs](https://www.visualstudio.com/en-us/docs/integrate/get-started/rest/basics)

## Obtain project and GIT repository by name

To process a build definition you will nee the following values:

* Team Project 
* Repository (I assume you are using GIT repos) 

To get these values, the easiest way is to pass them by parameter. But we are going to make an easier experience by allowing the user to select the project from a list. To do so, we need the projects list (as seen in the previous section) and the repositories list for the project. So, if the user does not provide any of these by parameter, we will present a list and allow the user to choose. 

{% highlight powershell %}
# Get project
$projectsApiUrl = 'https://acme.visualstudio.com/DefaultCollection/_apis/projects?api-version=1.0'
$projects = Invoke-RestMethod -Headers $headers $projectsApiUrl 
if([string]::IsNullOrWhiteSpace($projectName)) {
    $projects.value | Select name | Write-Host
    $projectName = Read-Host -Prompt "Choose a project name"
}
$projectId = ($projects.value | Where name -EQ $projectName).id
Write-Host "Project Id is: $projectId"
{% endhighlight %}

What we are doing is here is check if the user provided a projectName. If so, we just get the id. Otherwise we show the list and ask the user to choose one. Now we have the project, let's do the same with the repo: 

{% highlight powershell %}
# Get repo
$reposApiUrl = "https://acme.visualstudio.com/DefaultCollection/$projectId/_apis/git/repositories?api-version=1.0"
$repos = Invoke-RestMethod -Headers $headers $reposApiUrl 
if([string]::IsNullOrWhiteSpace($repoName)) {
    $repos.value | Select name | Write-Host
    $repoName = Read-Host -Prompt "Choose a repo name"
}
$repoId = ($repos.value | Where name -EQ $repoName).id
Write-Host "Repo Id is: $repoId"
{% endhighlight %}

Notice we built the URL replacing the project id with the value obtained before. So, the repo URL is 
{% highlight powershell %}
$repoUrl = 
  "https://acme.visualstudio.com/DefaultCollection/_git/$repoName";
{% endhighlight %}


Now we can finally build the URL to access the build definition creation: 

{% highlight powershell %}
$createBuildDefinitionApiUrl = 
  "https://acme.visualstudio.com/DefaultCollection/$projectId/_apis/build/definitions?api-version=2.0"
{% endhighlight %}

## Building the Build Definition object

I know there must be better methods to do this, but since I'm not a PowerShell guru, I'll stick to an easy solution. If you take a look at the [documentation for creating build definitions](https://www.visualstudio.com/en-us/docs/integrate/api/build/definitions#create-a-build-definition), you'll discover that you need to pass a JSON object in the body of a POST request, sent as application/json. The format of the JSON is quite self-descriptive. ![](http://fernando.ariasmarques.com/wp-content/uploads/2017/06/2017-06-21_14h48_07-300x122.png) 

I will build a dynamic object that resembles that structure and after that I will finally convert to JSON using the `ConvertTo-Json` command. See below the main structure of the script: 

{% highlight powershell %}
# We build an object that contains the required values. After we will convert it to JSON to send it to the REST Api
$buildDef = @{
    name = $buildDefinitionName;
    type = 'build';
    # [...]
}

$payload = $buildDef | ConvertTo-Json -Depth 32

$result = 
   Invoke-RestMethod -Uri $createBuildDefinitionApiUrl 
                     -Method Post 
                     -ContentType "application/json" 
                     -Headers $headers 
                     -Body $payload

Write-Host $result

{% endhighlight %}


So, the only thing we need now is to build the actual build definition object:

It has the following structure:

{% highlight powershell %}
$buildDef = @{
    name = $buildDefinitionName;
    type = 'build';
    buildNumberFormat = '$(date:yyyyMMdd)$(rev:-r)';
    quality = 'definition';
    queue = @{ id = 1 };
    build = @(
        @{},
        @{},
        [...]
    );
    repository = @{ };
    options = @(
        @{ }
    );
    variables = @{ };
    triggers = @(
        @{ }
    );
    demands = @(    )
}
{% endhighlight %}

Note: if you are unfamiliar to PowerShell, you just need to know that @{ key1 = value1; key2 = value2; ... keyN = valueN } is an object and @( value1, value2,... valueN ) is an array.

The properties are quite easy to understand. Some notes, though...

* The queue with the number 1 is the Default queue. I will show in another post how you can use a different queue that you choose.
* The build property contains all the steps (the tasks of the process)
* The repository property is a reference to the GIT repository or repositories that contain the source code.
* The variables allow you to set some special behaviors of the build such as multi-configuration
* triggers specify under which circumstances the build is launched
* demands are tags that the build agents must have so that builds can be sent to them

Find below a full object so that you can see a real example. This build definition is a .Net core build that performs the following steps: dotnet restore, build, test, pack and publish.

{% highlight powershell %}
$buildDef = @{
    name = $buildDefinitionName;
    type = 'build';
    buildNumberFormat = '$(date:yyyyMMdd)$(rev:-r)';
    quality = 'definition';
    queue = @{ id = $queueId };
    build = @(
        @{
            task = $dotNetTask;
            enabled = $true;
            continueOnError = $false;
            alwaysRun = $false;
            displayName = 'dotnet restore';
            timeoutInMinutes = 0;
            inputs = @{
                command = 'restore';
                publishWebProjects = "true";
                projects = '**/*.csproj';
                arguments = '';
                zipAfterPublish = "true"
            }
        },
        @{
            task = $dotNetTask;
            enabled = $true;
            continueOnError = $false;
            alwaysRun = $false;
            displayName = 'dotnet build';
            timeoutInMinutes = 0;
            inputs = @{
                command = 'build';
                publishWebProjects = "true";
                projects = '**/*.csproj';
                arguments = '--configuration $(BuildConfiguration)';
                zipAfterPublish = "true"
            }
        },
        @{
            task = $dotNetTask;
            enabled = $true;
            continueOnError = $false;
            alwaysRun = $false;
            displayName = 'dotnet test';
            timeoutInMinutes = 0;
            inputs = @{
                command = 'test';
                publishWebProjects = "true";
                projects = 'test/**/*.csproj';
                arguments = '--configuration $(BuildConfiguration) --no-build';
                zipAfterPublish = "true"
            }
        },
        @{
            task = $dotNetTask;
            enabled = $true;
            continueOnError = $false;
            alwaysRun = $false;
            displayName = 'dotnet pack';
            timeoutInMinutes = 0;
            inputs = @{
                command = 'pack';
                publishWebProjects = "true";
                projects = 'src/**/*.csproj';
                arguments = '--output $(build.artifactstagingdirectory)\packages\ --include-symbols';
                zipAfterPublish = "true"
            }
        },
        @{
            task = $publishTask;
            enabled = $true;
            continueOnError = $false;
            alwaysRun = $true;
            displayName = 'Publish Artifact: GeneratedNuGetPackages';
            timeoutInMinutes = 0;
            condition = 'succeeded()';
            inputs = @{
                PathtoPublish = '$(build.artifactstagingdirectory)\packages';
                ArtifactName = 'GeneratedNuGetPackages';
                ArtifactType = 'Container';
                TargetPath = ''
            }
        }
    );
    repository = @{
        id = $repoId;
        name = $repoName;
        type = 'TfsGit';
        localPath = '$(sys.sourceFolder)/$repoName';
        url = $repoUrl;
        defaultBranch = 'refs/heads/master';
        clean = $false

    };
    options = @(
        @{
            enabled = $true;
            definition = @{ id = '7c555368-ca64-4199-add6-9ebaf0b0137d' };
            inputs = @{
                multipliers = '["BuildConfiguration"]';
                parallel = "true";
                additionalFields = "{}";
                continueOnError = "true"
            }
        }
    );
    variables = @{
        BuildConfiguration = @{ value = 'debug,release'; allowOverride = $true};
        BuildPlatform = @{ value = 'any cpu'; allowOverride = $true};
    };
    triggers = @(
        @{
            branchFilters = @("+refs/heads/master");
            pathFilers = @();
            batchChanges = $false;
            maxConcurrentBuildsPerBranch = 1;
            pollingInterval = 0;
            triggerType = "continuousIntegration";
        }
    );
    demands = @(
        "MSBuild_14.0",
        "VisualStudio_15.0"
    )
}
{% endhighlight %}
