---
layout: post
title:  "Batch modification of build definitions in VSTS"
date:   2016-12-20 16:00:11 +0200
categories: build myget vsts
---
## Overview
In this post I will show you how to write a simple console application in C# that can access your [Visual Studio Team Services API](https://www.visualstudio.com/en-us/docs/integrate/api/overview) to modify your build definitions. 

## Step by step
First, create a new blank Console Application and add the following NuGet packages

*   Microsoft.TeamFoundationServer.Client
*   Microsoft.VisualStudio.Services.Client
*   Microsoft.VisualStudio.Services.InteractiveClient

Let's start by connecting to Visual Studio Team Services. If you don't have a Visual Studio Team Services account, you can [set one up for free](https://www.visualstudio.com/docs/setup-admin/team-services/sign-up-for-visual-studio-team-services). 

You will need a Personal Access Token (PAT) for connecting to the API. If you haven't one, [create a new one](https://www.visualstudio.com/en-gb/docs/integrate/get-started/auth/overview). 

{% highlight csharp %}
using System;
using System.Threading.Tasks;
using Microsoft.VisualStudio.Services.Client;
using Microsoft.VisualStudio.Services.Common;

namespace Toolfactory.Vsts.BuidDefinitionProcessor
{
    internal static class Program
    {
        private static void Main()
        {
            Task.Run(AsyncMain).Wait();
        }

        private static async Task AsyncMain()
        {
            // Create instance of VssConnection using AAD Credentials for AAD backed account
            var vssConnection = new VssConnection(new Uri("https://acme.visualstudio.com/"),
                new VssBasicCredential(string.Empty, "Your PAT"));
        }
    }
}
{% endhighlight }

Now, we will obtain all the projects in your Visual Studio Team Services instance. We have to create a client using the connection and then send a query:

{% highlight csharp %}
using System;
using System.Threading.Tasks;
using Microsoft.TeamFoundation.Core.WebApi;
using Microsoft.VisualStudio.Services.Client;
using Microsoft.VisualStudio.Services.Common;

namespace Toolfactory.Vsts.BuidDefinitionProcessor
{
    internal static class Program
    {
        [...]

        private static async Task AsyncMain()
        {
            var vssConnection = new VssConnection(new Uri("https://acme.visualstudio.com/"),
                new VssBasicCredential(string.Empty, "Your PAT"));
            using (var projectClient = await vssConnection.GetClientAsync<ProjectHttpClient>())
            
            {
                var projects = await projectClient.GetProjects();
                foreach (var teamProjectReference in projects)
                {
                    Console.WriteLine($"{teamProjectReference.Name}");
                }
            }
        }
    }
}
{% endhighlight }

Now we will traverse all projects looking for build definitions. Notice that buildClient.GetDefinitionsAsync() gets build reference items. These items don't contain all the information of the build definition, just some basic data. To obtain the full build definition you have to invoke buildClient.GetDefinitionAsync() passing the definitionReference.Id.

{% highlight csharp %}
using System;
using System.Threading.Tasks;
using Microsoft.TeamFoundation.Build.WebApi;
using Microsoft.TeamFoundation.Core.WebApi;
using Microsoft.VisualStudio.Services.Client;
using Microsoft.VisualStudio.Services.Common;

namespace Toolfactory.Vsts.BuidDefinitionProcessor
{
    internal static class Program
    {
        [...]

        private static async Task AsyncMain()
        {
            var vssConnection = new VssConnection(new Uri("https://acme.visualstudio.com/"),
                new VssBasicCredential(string.Empty, "Your PAT"));
            using (var projectClient = await vssConnection.GetClientAsync<ProjectHttpClient>())
            using (var buildClient = vssConnection.GetClient<BuildHttpClient>())
            {
                var projects = await projectClient.GetProjects();
                foreach (var teamProjectReference in projects)
                {
                    Console.WriteLine($"{teamProjectReference.Name}");
                    var definitionReferences = await buildClient.GetDefinitionsAsync(teamProjectReference.Id, type: DefinitionType.Build);
                    foreach (var definitionReference in definitionReferences)
                    {
                        var definition = await buildClient.GetDefinitionAsync(definitionId: definitionReference.Id, project: teamProjectReference.Id) as BuildDefinition;
                        if (definition == null) continue;

                        Console.WriteLine($"\t{definitionReference.Name} ({definitionReference.Id}) - {definition.Steps.Count} steps");                       
                    }
                }
            }
        }
    }
}
{% endhighlight }

Now that we have the build definitions, we can process them in any way we want. Once we have modified the build definition, we save the changes by sending a request to the API with the method

{% highlight csharp %}
await buildClient.UpdateDefinitionAsync(definition, 
  definitionId: definitionReference.Id,
  project: teamProjectReference.Id);
{% endhighlight }

For example, the following code adds two variables to the build definition that can be used for setting MyGet credentials:

{% highlight csharp %}
[...]
foreach (var definitionReference in definitionReferences)
{
  var definition = await buildClient.GetDefinitionAsync(definitionId: definitionReference.Id, project: teamProjectReference.Id) as BuildDefinition;
  if (definition == null) continue;

  Console.WriteLine($"\t{definitionReference.Name} ({definitionReference.Id}) - {definition.Steps.Count} steps");
  if (!definition.Variables.ContainsKey("MyGetUsername"))
  {
      definition.Variables.Add("MyGetUsername", new BuildDefinitionVariable() { Value = "mygetuser" });
  }
  if (!definition.Variables.ContainsKey("MyGetPassword"))
  {
      definition.Variables.Add("MyGetPassword", new BuildDefinitionVariable() { Value = "supersecretpassword", IsSecret = true });
  }

  await buildClient.UpdateDefinitionAsync(definition, definitionId: definitionReference.Id, project: teamProjectReference.Id);
}
{% endhighlight }

The following code reorders steps so that an specific step is always in the first position.

{% highlight csharp %}
[...]
// we have to know the step we want to reorder. Either look for it in other definitions
// or provide the task definition id.
BuildDefinitionStep myCustomBuildStep = GetCustomStep(projectClient, buildClient);
[...]

var definition =
          await
              buildClient.GetDefinitionAsync(definitionId: definitionReference.Id,
                  project: teamProjectReference.Id) as BuildDefinition;
      if (definition == null) continue;

      Console.WriteLine(
          $"\t{definitionReference.Name} ({definitionReference.Id}) - {definition.Steps.Count} steps");

      // find the step into build definition's steps. if it's not present, add it (at the end)
      var currentAddFeedStep =
          definition.Steps.FirstOrDefault(
              s => s.TaskDefinition.Id == myCustomBuildStep.TaskDefinition.Id);
      if (currentAddFeedStep == null)
      {
          currentAddFeedStep = new BuildDefinitionStep
          {
              TaskDefinition = myCustomBuildStep.TaskDefinition
          };
          foreach (var stepInput in myCustomBuildStep.Inputs)
          {
              currentAddFeedStep.Inputs.Add(stepInput);
          }
          definition.Steps.Add(currentAddFeedStep);
      }

      // be sure you enable the step
      currentAddFeedStep.Enabled = true;

      // create a new list of steps, adding the desired step at the beginning
      var orderedSteps = new List<BuildDefinitionStep>(definition.Steps.Count) {currentAddFeedStep};

      // add the rest of steps
      orderedSteps.AddRange(
          definition.Steps.Where(s => s.TaskDefinition.Id != myCustomBuildStep.TaskDefinition.Id));

      // replace current list of steps with the reordered one
      definition.Steps.Clear();
      definition.Steps.AddRange(orderedSteps);

      // save changes
      await
          buildClient.UpdateDefinitionAsync(definition, definitionId: definitionReference.Id,
              project: teamProjectReference.Id);
{% endhighlight }

Enjoy!


See also: [https://www.visualstudio.com/en-gb/docs/integrate/extensions/overview](https://www.visualstudio.com/en-gb/docs/integrate/extensions/overview)