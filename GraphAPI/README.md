Jitendra: B2C integrates with Graph SDK code. 

# Azure AD B2C user account management with .NET Core and Microsoft Graph

This .NET Core console application demonstrates the use of the Microsoft Graph API to perform user account management operations (create, read, update, delete) within an Azure AD B2C directory. Also shown is a technique for the bulk import of users from a JSON file. Bulk import is useful in migration scenarios like moving your users from a legacy identity provider to Azure AD B2C.

The code in this sample backs the [Manage Azure AD B2C user accounts with Microsoft Graph](https://docs.microsoft.com/azure/active-directory-b2c/manage-user-accounts-graph-api) article on docs.microsoft.com.

## Contents

| File/folder          | Description                                                   |
|:---------------------|:--------------------------------------------------------------|
| `./data`             | Example user data in JSON format.                             |
| `./src`              | Sample source code (*.proj, *.cs, etc.).                      |
| `.gitignore`         | Defines the Visual Studio resources to ignore at commit time. |
| `CODE_OF_CONDUCT.md` | Information about the Microsoft Open Source Code of Conduct.  |
| `LICENSE`            | The license for the sample.                                   |
| `README.md`          | This README file.                                             |
| `SECURITY.md`        | Guidelines for reporting security issues found in the sample. |

## Prerequisites

* [Visual Studio](https://visualstudio.microsoft.com/) or [Visual Studio Code](https://code.visualstudio.com/) for debugging or file editing
* [.NET Core SDK](https://dotnet.microsoft.com/) 3.1+
* [Azure AD B2C tenant](https://docs.microsoft.com/azure/active-directory-b2c/tutorial-create-tenant) with one or more user accounts in the directory
* [Management app registered](https://docs.microsoft.com/azure/active-directory-b2c/microsoft-graph-get-started) in your B2C tenant

## Setup

1. Clone the repo or download and extract the [ZIP archive](https://github.com/Azure-Samples/ms-identity-dotnetcore-b2c-account-management/archive/master.zip)
2. Modify `./src/appsettings.json` with values appropriate for your environment:
    - Azure AD B2C **tenant ID**
    - Registered application's **Application (client) ID**
    - Registered application's **Client secret**
3. Build the application with `dotnet build`:

    ```console
    azureuser@machine:~/ms-identity-dotnetcore-b2c-account-management$ cd src
    azureuser@machine:~/ms-identity-dotnetcore-b2c-account-management/src$ dotnet build
    Microsoft (R) Build Engine version 16.4.0+e901037fe for .NET Core
    Copyright (C) Microsoft Corporation. All rights reserved.

      Restore completed in 431.62 ms for /home/azureuser/ms-identity-dotnetcore-b2c-account-management/src/b2c-ms-graph.csproj.
      b2c-ms-graph -> /home/azureuser/ms-identity-dotnetcore-b2c-account-management/src/bin/Debug/netcoreapp3.0/b2c-ms-graph.dll

    Build succeeded.
        0 Warning(s)
        0 Error(s)

    Time Elapsed 00:00:02.62
    ```
4. Add 2 custom attributes to your B2C instance in order to run all the sample operations with custom attributes involved.
   Attributes to add:
    - FavouriteSeason (string)
    - LovesPets (boolean)

## Running the sample

Execute the sample with `dotnet b2c-ms-graph.dll`, select the operation you'd like to perform, then press ENTER.

For example, get a user by object ID (command `2`), then exit the application with `exit`:

```console
azureuser@machine:~/ms-identity-dotnetcore-b2c-account-management/src$ dotnet bin/Debug/netcoreapp3.0/b2c-ms-graph.dll

Command  Description
====================
[1]      Get all users (one page)
[2]      Get user by object ID
[3]      Get user by sign-in name
[4]      Delete user by object ID
[5]      Update user password
[6]      Create users (bulk import)
[7]      Create user with custom attributes and show result
[8]      Get all users (one page) with custom attributes
[help]   Show available commands
[exit]   Exit the program
-------------------------
Enter command, then press ENTER: 2

```

## Key concepts

The application uses the [OAuth 2.0 client credentials grant](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) flow to get an access token for calling the Microsoft Graph API. In the client credentials grant flow, the application non-interactively authenticates as itself, as opposed to requiring a user to sign in interactively.


The Microsoft Graph Client Library for .NET is a wrapper for MSAL.NET, providing helper classes for authenticating with and calling the Microsoft Graph API.

### Creating the GraphServiceClient

After parsing the values in `appsettings.json`, a [GraphServiceClient][GraphServiceClient] (the primary utility for working with Graph resources) is instantiated with following object instantiation flow:

[ConfidentialClientApplication][ConfidentialClientApplication] :arrow_right: [ClientCredentialProvider][ClientCredentialProvider] :arrow_right: [GraphServiceClient][GraphServiceClient]

From [`Program.cs`](./src/Program.cs):

```csharp
// Read application settings from appsettings.json (tenant ID, app ID, client secret, etc.)
AppSettings config = AppSettingsFile.ReadFromJsonFile();

// Initialize the client credential auth provider
IConfidentialClientApplication confidentialClientApplication = ConfidentialClientApplicationBuilder
    .Create(config.AppId)
    .WithTenantId(config.TenantId)
    .WithClientSecret(config.AppSecret)
    .Build();
ClientCredentialProvider authProvider = new ClientCredentialProvider(confidentialClientApplication);

// Set up the Microsoft Graph service client with client credentials
GraphServiceClient graphClient = new GraphServiceClient(authProvider);
```

### Graph operations with GraphServiceClient

The initialized *GraphServiceClient* can then be used to perform any operation for which it's been granted permissions by its [app registration](https://docs.microsoft.com/azure/active-directory-b2c/microsoft-graph-get-started).

For example, getting a list of the user accounts in the tenant (from [`UserService.cs`](./src/Services/UserService.cs)):

```csharp
public static async Task ListUsers(AppSettings config, GraphServiceClient graphClient)
{
    Console.WriteLine("Getting list of users...");

    // Get all users (one page)
    var result = await graphClient.Users
        .Request()
        .Select(e => new
        {
            e.DisplayName,
            e.Id,
            e.Identities
        })
        .GetAsync();

    foreach (var user in result.CurrentPage)
    {
        Console.WriteLine(JsonConvert.SerializeObject(user));
    }
}
```
