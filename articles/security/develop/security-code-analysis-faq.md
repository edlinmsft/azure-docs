---
title: Microsoft Azure Security Code Analysis documentation FAQs
description: This article contains FAQs about the Security Code Analysis Extension
author: vharindra
manager: sukhans
ms.author: terrylan
ms.date: 07/31/2019
ms.topic: article
ms.service: security
services: azure

ms.assetid: 521180dc-2cc9-43f1-ae87-2701de7ca6b8
ms.devlang: na
ms.tgt_pltfrm: na
ms.workload: na
---

# Frequently asked questions
Got questions? Check out the FAQs below for more information.

## General FAQs

### Can I install the extension on my TFS (not Azure DevOps) server? 

No, the extension isn't available for download and install for TFS.

### Do I have to run Microsoft Security Code Analysis with my build? 

Yes and no. Depending on the type of analysis tool, the source code itself may be the only thing that's required or the output of the build may be required. For example, since Credential Scanner analyzes files within the folder structure of the code repository, you could run the Credential Scanner and Publish Security Analysis logs build tasks in a standalone build to retrieve results.
For other tools that analyze post build artifacts, like BinSkim, the build will be required first.

### Can I break my build when results are found? 
Yes, you can introduce a build break when any tool reports an issue, a finding, in its log file. Just add the Post-Analysis build task and check the checkbox for any tool for which you would like to break the build. You can choose to break the build when any tool reports errors, or warnings and errors, in the UI of the Post-Analysis task.

### How are the command line arguments different in Azure DevOps than they are in the standalone desktop tools? 

For the most part, the Azure DevOps build tasks are direct wrappers around the command line arguments of the security tools. Anything you would normally pass to the tool on the command line from your desktop, you can pass to the Arguments input of the build task.
Here is a list of noticeable differences:
 - The tool will be executing from the source folder of the agent $(Build.SourcesDirectory) or %BUILD_SOURCESDIRECTORY%. Example: C:\agent\_work\1\s 
 - Paths in the Arguments can be relative to the root of the source directory listed above or absolute either by running an on-prem agent with known deployment locations of local resources or using Azure DevOps Build Variables
 - Tools will automatically provide an output file path or folder if an output path is provided, it will be removed and replaced with a path to our well-known logs location on the build agent
 - Some additional command line parameters are sanitized and removed on some tools, such as the addition or removal of options to ensure that no GUI is launched.

### Can I run a build task (for example, Credential Scanner) across multiple repositories in an Azure DevOps Build? 

No, running the secure development tools against multiple repositories in a single pipeline is currently not supported.

###  The output file I specified is not being created / I can’t find the output file I specified 

The build tasks currently sanitize user input and update the location of the output file generated to a common location on the build agent. For more information on this location, see the following questions.

### Where are the output files generated by the tools saved? 

The build tasks automatically add output paths to the following well-known location on the build agent $(Agent.BuildDirectory)\_sdt\logs. By standardizing on this location, we can guarantee that other teams producing code analysis logs or consuming them will have access.

### Can I queue a build to run these tasks on a Hosted Build Agent? 

Yes, all tasks and tools in the extension can be executed on a hosted build agent.

>[!NOTE]
> The Anti-Malware build task requires a build agent with Windows Defender enabled, which is true on "Hosted VS2017" or later build agents. (It will not run on the legacy/VS2015 "Hosted" agent.)
Signatures cannot be updated on these agents, but the signature should always be relatively current, less than 3 hours old.
>

### Can I run these build tasks as part of a Release Pipeline (as opposed to a Build pipeline)? 
In most cases, Yes. 
However, tasks that publish artifacts are not supported by Azure DevOps to be run from within Release Pipelines: "The only category of tasks not expected to work with Release are the ones that publish artifacts. This is because, as of now, we don’t have support for publishing artifacts within Release".
This prevents the "Publish Security Analysis Logs" task from running successfully from a release pipeline; it will fail, with a descriptive error message.

### From where do the build tasks download the tools? 
The build tasks a) download NuGet packages for the tools from the following [Azure DevOps Package Management feed](https://securitytools.pkgs.visualstudio.com/_packaging/SecureDevelopmentTools/nuget/v3/index.json)
or using the Node Package Manager, which must be pre-installed on the build agent (example: "npm install tslint").

### What effect will installing the extension have on my Azure DevOps Organization? 

Upon installing, the security build tasks provided by the extension will become available for use by all users in your organization. When creating or editing an Azure pipeline, these tasks will be available to add from the build task collection list. Otherwise, installing the extension in your Azure DevOps organization has no impact. It does not modify any account or project settings or pipelines.

### Will installing the extension modify my existing Azure Pipelines? 

No. Installing the extension will make the security build tasks available to add to your Azure Pipelines. Users are still required to add or update build definitions to integrate the tools into your build process.

## Task specific FAQs

FAQs specific to build tasks will be listed in this section.

### Credential Scanner FAQs

#### What are common suppressions scenarios and examples? 
Two of the most common suppression scenarios are detailed below:
##### Suppress all occurrences of a given secret within the specified path 
The hash key of the secret from the Credential Scanner output file is required as shown in the sample below
   
        {
            "tool": "Credential Scanner",
            "suppressions": [
            {
                "hash": "CLgYxl2FcQE8XZgha9/UbKLTkJkUh3Vakkxh2CAdhtY=",
                "_justification": "Secret used by MSDN sample, it is fake."
            }
          ]
        }

>[!WARNING]
> The hash key is generated by a portion of the matching value or file content. Any source code revision could change the hash key and disable the suppression rule. 

##### To suppress all secrets in a specified file (or to suppress the secrets file itself) 
The file expression could be a file name or any postfix portion of the full file path/name. Wildcards are not supported. 

**Example** 

File to be suppressed: [InputPath]\src\JS\lib\angular.js 

Valid Suppression Rules: 
- [InputPath]\src\JS\lib\angular.js -- suppress the file in the specified path
- \src\JS\lib\angular.js
- \JS\lib\angular.js
- \lib\angular.js
- angular.js -- suppress any file with the same name

        {
            "tool": "Credential Scanner",
            "suppressions": [
            {
                "file": "\\files\\AdditonalSearcher.xml", 
                "_justification": "Additional CredScan searcher specific to my team"
            },
            {
                "file": "\\files\\unittest.pfx", 
                "_justification": "Legitimate UT certificate file with private key"
            }
          ]
        }      

>[!WARNING] 
> All future secrets added to the file will also get suppressed automatically. 

#### What are recommended Secrets management guidelines? 
While detecting hard coded secrets in a timely manner and mitigating the risks is helpful, it is even better if one could prevent secrets from getting checked in altogether. In this regard, Microsoft has released CredScan Code Analyzer as part of [Microsoft DevLabs extension](https://marketplace.visualstudio.com/items?itemName=VSIDEDevOpsMSFT.ContinuousDeliveryToolsforVisualStudio) for Visual Studio. While in early preview, it provides developers an inline experience for detecting potential secrets in their code, giving them the opportunity to fix those issues in real-time. For more information, please refer to [this](https://devblogs.microsoft.com/visualstudio/managing-secrets-securely-in-the-cloud/) blog on Managing Secrets Securely in the Cloud. 
Below are few additional resources to help you manage secrets and access sensitive information from within your applications in a secure manner: 
 - [Azure Key Vault](https://docs.microsoft.com/azure/key-vault/)
 - [Azure Active Directory](https://docs.microsoft.com/azure/sql-database/sql-database-aad-authentication)
 - [Azure AD Managed Service Identity](https://azure.microsoft.com/blog/keep-credentials-out-of-code-introducing-azure-ad-managed-service-identity/)
 - [Managed Service Identity (MSI) for Azure resources](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
 - [Azure Managed Service Identity](https://docs.microsoft.com/azure/app-service/overview-managed-identity)
 - [AppAuthentication Library](https://docs.microsoft.com/azure/key-vault/service-to-service-authentication)

#### Can I write my own custom searchers?

Credential Scanner relies on a set of content searchers commonly defined in the **buildsearchers.xml** file. The file contains an array of XML serialized objects that represent a ContentSearcher object. The program is distributed with a set of searchers that have been well tested but it does allow you to implement your own custom searchers too. 

A content searcher is defined as follows: 

- **Name** – The descriptive searcher name to be used in Credential Scanner output file. It is recommended to use camel case naming convention for searcher names. 
- **RuleId** – The stable opaque ID of the searcher. 
    - Credential Scanner default searchers are assigned with RuleIds like CSCAN0010, CSCAN0020, CSCAN0030, etc. The last digit is reserved for potential searcher regex group merging or division.
    - RuleId for customized searchers should have its own namespace in the format of: CSCAN-{Namespace}0010, CSCAN-{Namespace}0020, CSCAN-{Namespace}0030, etc.
    - The fully qualified searcher name is the combination of the RuleId and the searcher name. Example CSCAN0010.KeyStoreFiles, CSCAN0020.Base64EncodedCertificate, etc.
- **ResourceMatchPattern** – Regex of file extensions to check against searcher
- **ContentSearchPatterns** – Array of strings containing Regex statements to match. If no search patterns are defined, all files matching the resource match pattern will be returned.
- **ContentSearchFilters** – Array of strings containing Regex statements to filter searcher specific false positives.
- **Matchdetails** – A descriptive message and/or mitigation instructions to be added for each match of the searcher.
- **Recommendation** – Provides the suggestions field content for a match using PREfast report format.
- **Severity** – An integer to reflect the severity of the issue (Highest = 1).
![Credential Scanner Setup](./media/security-tools/6-credscan-customsearchers.png)

### Roslyn Analyzers FAQs

#### What are the most common errors when using the Roslyn Analyzers task?

**Error: The project was restored using Microsoft.NETCore.App version x.x.x, but with current settings, version y.y.y would be used instead. To resolve this issue, make sure the same settings are used for restore and for subsequent operations such as build or publish. Typically this issue can occur if the RuntimeIdentifier property is set during build or publish but not during restore:**

Roslyn analyzers run as part of compilation, so the source tree on the build machine needs to be in a buildable state. A step (probably "dotnet.exe publish") between your main build and Roslyn analyzers may have put the source tree in an unbuildable state. Perhaps duplicating the step that does a Nuget Restore, just before the Roslyn Analyzers step, will put the source tree back in a buildable state.

**"csc.exe" exited with error code 1 -- An instance of analyzer AAAA cannot be created from C:\BBBB.dll : Could not load file or assembly 'Microsoft.CodeAnalysis, Version=X.X.X.X, Culture=neutral, PublicKeyToken=31bf3856ad364e35' or one of its dependencies. The system cannot find the file specified.**

Ensure your compiler supports Roslyn analyzers. "csc.exe /version" should report at least v2.6.x. In some cases, individual .csproj files can override the build machine's Visual Studio installation, by referencing a package from Microsoft.Net.Compilers. If using a specific version of the compiler was unintended, remove references to Microsoft.Net.Compilers. Otherwise, make sure the referenced package is also at least v2.6.x. Try to get the error log, which you can find in the /errorlog: parameter from the csc.exe command line (found in the Roslyn build task's log). It may look something like: /errorlog:F:\ts-services-123\_work\456\s\Some\Project\Code\Code.csproj.sarif

**The C# compiler is not recent enough (it must be >= 2.6)**

The latest versions of the C# compiler are released here: https://www.nuget.org/packages/Microsoft.Net.Compilers. To get the installed version you are using run `C:\>csc.exe /version` from command prompt. Ensure that you do not have any reference to a Microsoft.Net.Compilers NuGet package that is < v2.6.

**MSBuild/VSBuild Logs Not Found**

Because of how the task works, this task needs to query Azure DevOps for the MSBuild log from the MSBuild build task. If this task runs immediately after the MSBuild build task, the log will not yet be available; Place other build tasks, including SecDevTools build tasks, like Binskim, Antimalware Scan, and others), between the MSBuild build task and the Roslyn Analyzers build task. 

## Next steps

If you need additional assistance, Microsoft Security Code Analysis Support is available Monday through Friday from 9:00 AM - 5:00 PM Pacific Standard Time

  - Onboarding - Contact your Technical Account Managers to get started. 
  >[!NOTE] 
  >If you don’t already have a paid support relationship with Microsoft or if you have a support offering that doesn’t allow you to purchase services from the Phoenix catalog, please visit our [support services home page](https://www.microsoft.com/enterprise/services/support) for more information.

  - Support - Email our team at [Microsoft Security Code Analysis Support](mailto:mscahelp@microsoft.com?Subject=Microsoft%20Security%20Code%20Analysis%20Support%20Request)