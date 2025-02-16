---
title: .NET Client Library Samples for Azure DevOps
description: C# samples showing how to integrate with Azure DevOps from apps and services on Windows.
ms.assetid: 9ff78e9c-63f7-45b1-a70d-42aa6a9dbc57
ms.subservice: azure-devops-ecosystem
ms.custom: devx-track-dotnet
ms.topic: conceptual
monikerRange: '<= azure-devops'
ms.author: chcomley
author: chcomley
ms.date: 05/05/2022
---

# C# client library samples 

[!INCLUDE [version-lt-eq-azure-devops](../../../includes/version-lt-eq-azure-devops.md)]

Samples showing how to extend and integrate with Azure DevOps using the [.NET client libraries](../../concepts/dotnet-client-libraries.md).


## Samples in GitHub
There are many samples with instructions on how to run them on our [.NET Sample GitHub Page](https://github.com/microsoft/azure-devops-dotnet-samples). 

## Other samples
REST examples on this page require the following NuGet packages:
* [Microsoft.TeamFoundationServer.Client](https://www.nuget.org/packages/Microsoft.TeamFoundationServer.Client/)
* [Microsoft.VisualStudio.Services.Client](https://www.nuget.org/packages/Microsoft.VisualStudio.Services.Client/)
* [Microsoft.VisualStudio.Services.InteractiveClient](https://www.nuget.org/packages/Microsoft.VisualStudio.Services.InteractiveClient/)

>[!NOTE]
> The Work Item Tracking (WIT) and Test Client OM are scheduled to be deprecated in 2020. For more information, see [Deprecation of WIT and Test Client OM](../../concepts/wit-client-om-deprecation.md).

#### Example: Using a REST-based HTTP client

```cs
// https://www.nuget.org/packages/Microsoft.TeamFoundationServer.Client/
using Microsoft.TeamFoundation.WorkItemTracking.WebApi;
using Microsoft.TeamFoundation.WorkItemTracking.WebApi.Models;

// https://www.nuget.org/packages/Microsoft.VisualStudio.Services.InteractiveClient/
using Microsoft.VisualStudio.Services.Client;

// https://www.nuget.org/packages/Microsoft.VisualStudio.Services.Client/
using Microsoft.VisualStudio.Services.Common; 

/// <summary>
/// This sample creates a new work item query for New Bugs, stores it under 'MyQueries', runs the query, and then sends the results to the console.
/// </summary>
public static void SampleREST()
{
    // Connection object could be created once per application and we use it to get httpclient objects. 
    // Httpclients have been reused between callers and threads.
    // Their lifetime has been managed by connection (we don't have to dispose them).
    // This is more robust then newing up httpclient objects directly.  
    
    // Be sure to send in the full collection uri, i.e. http://myserver:8080/tfs/defaultcollection
    // We are using default VssCredentials which uses NTLM against an Azure DevOps Server.  See additional provided
    // Create a connection with PAT for authentication
    VssConnection connection = new VssConnection(orgUrl, new VssBasicCredential(string.Empty, personalAccessToken));


    // Create instance of WorkItemTrackingHttpClient using VssConnection
    WorkItemTrackingHttpClient witClient = connection.GetClient<WorkItemTrackingHttpClient>();

    // Get 2 levels of query hierarchy items
    List<QueryHierarchyItem> queryHierarchyItems = witClient.GetQueriesAsync(teamProjectName, depth: 2).Result;

    // Search for 'My Queries' folder
    QueryHierarchyItem myQueriesFolder = queryHierarchyItems.FirstOrDefault(qhi => qhi.Name.Equals("My Queries"));
    if (myQueriesFolder != null)
    {
        string queryName = "REST Sample";

        // See if our 'REST Sample' query already exists under 'My Queries' folder.
        QueryHierarchyItem newBugsQuery = null;
        if (myQueriesFolder.Children != null)
        {
            newBugsQuery = myQueriesFolder.Children.FirstOrDefault(qhi => qhi.Name.Equals(queryName));
        }
        if (newBugsQuery == null)
        {
            // if the 'REST Sample' query does not exist, create it.
            newBugsQuery = new QueryHierarchyItem()
            {
                Name = queryName,
                Wiql = "SELECT [System.Id],[System.WorkItemType],[System.Title],[System.AssignedTo],[System.State],[System.Tags] FROM WorkItems WHERE [System.TeamProject] = @project AND [System.WorkItemType] = 'Bug' AND [System.State] = 'New'",
                IsFolder = false
            };
            newBugsQuery = witClient.CreateQueryAsync(newBugsQuery, teamProjectName, myQueriesFolder.Name).Result;
        }

        // run the 'REST Sample' query
        WorkItemQueryResult result = witClient.QueryByIdAsync(newBugsQuery.Id).Result;

        if (result.WorkItems.Any())
        {
            int skip = 0;
            const int batchSize = 100;
            IEnumerable<WorkItemReference> workItemRefs;
            do
            {
                workItemRefs = result.WorkItems.Skip(skip).Take(batchSize);
                if (workItemRefs.Any())
                {
                    // get details for each work item in the batch
                    List<WorkItem> workItems = witClient.GetWorkItemsAsync(workItemRefs.Select(wir => wir.Id)).Result;
                    foreach (WorkItem workItem in workItems)
                    {
                        // write work item to console
                        Console.WriteLine("{0} {1}", workItem.Id, workItem.Fields["System.Title"]);
                    }
                }
                skip += batchSize;
            }
            while (workItemRefs.Count() == batchSize);
        }
        else
        {
            Console.WriteLine("No work items were returned from query.");
        }
    }
}
```

## Authenticating

To change the method of authentication to Azure DevOps Services or Azure DevOps Server, change the VssCredential type passed to VssConnection when creating it.

##### Personal Access Token authentication for REST services
```cs
public static void PersonalAccessTokenRestSample()
{
    // Create instance of VssConnection using Personal Access Token
    VssConnection connection = new VssConnection(orgUrl, new VssBasicCredential(string.Empty, personalAccessToken));
}
```

##### Visual Studio sign-in prompt (Microsoft Account or Azure Active Directory backed) for REST services (.NET Framework only)

Because interactive dialogs aren't supported by the .NET Core version of the clients, this sample applies only to the .NET Framework version of the clients.

```cs
public static void MicrosoftAccountRestSample()
{
    // Create instance of VssConnection using Visual Studio sign-in prompt
    VssConnection connection = new VssConnection(new Uri(collectionUri), new VssClientCredentials());
}
```

##### Azure Active Directory Authentication for REST services
```cs
public static void AADRestSample()
{
    // Create instance of VssConnection using Azure AD Credentials for Azure AD backed account
    VssConnection connection = new VssConnection(new Uri(collectionUri), new VssAadCredential(userName, password));
}
```

##### OAuth Authentication for REST services
Follow this link to learn how to obtain and refresh an OAuth accessToken: https://github.com/microsoft/azure-devops-auth-samples
```cs
public static void OAuthSample()
{
    // Create instance of VssConnection using OAuth Access token
    VssConnection connection = new VssConnection(new Uri(collectionUri), new VssOAuthAccessTokenCredential(accessToken));
}
```
