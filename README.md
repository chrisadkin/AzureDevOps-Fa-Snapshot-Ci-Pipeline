# AzureDevOps-Fa-Snapshot-Ci-Pipeline
Azure Pipeline to illsutrate the use of a database refresh in a contunuous integration (CI) workflow.
## Overview

![pipeline](https://user-images.githubusercontent.com/15145995/53749355-1c7a2580-3e9f-11e9-83aa-7cacdae17bda.PNG)

This example Azure Pipeline checks a SQL Server data tools project and solution out of GitHub (the code), builds this into a DACPAC (the artefact), refreshes a pseudo test database from a pseudo production database and then applies the DACPAC to the test database.

## Build Environment Infrastructure Topology

![build-infra](https://user-images.githubusercontent.com/15145995/53748345-14b98180-3e9d-11e9-8436-5b4aa6a24622.PNG)

## Prerequisites

1. A windows server to self-host one or more windows build agents on.
2. An Azure Pipeline build agent for Windows installed according to the following specification:
 - build agent configured as a windows service running under a windows domain account.
 - build agent service account can log into any target instance containing database(s) to be refreshed and
   has the necessary permissions to on and offline those database(s).
3. purestoragedbatools installed on the build server, this in turn will install the dbatools and PureStoragePowerShellSDK
   PowerShell modules.
4. An [Azure DevOps](https://azure.microsoft.com/en-gb/services/devops/) account, a free account will suffice for the purposes of   running this example.
5. The build server requires the ability to communicate with the Azure Pipeline service via https (port 443).
6. The user databases that act as the source and target of the refresh element of the pipeline require that:
 - their datafiles and the transaction log file reside on a single FlashArray volume,
 - their files reside on the same FlashArray.
 
 ## Configuring The Build Agent Server
 
 ### Build Agent Installation and Configuration
 
 1. Create an [Azure DevOps](https://azure.microsoft.com/en-gb/services/devops/) account, a free account will suffice for the purposes of this example.
 
 2. Navigate to the link:

    https://dev.azure.com/<organization name>/_usersSettings/tokens 

    Click on new token, copy the string for the personal access token (PAT), this is required for installing the self-hosted agent.
    
 3. Navigate to the following link on the build agent server from within a web browser:

    https://dev.azure.com/<organization name>/_settings/agentpools?poolId=1&_a=agents
 
    then download the agent by clicking on 'Download':

![image](https://user-images.githubusercontent.com/15145995/52900754-e6c41400-31f1-11e9-8625-ea304734643c.png)

 4. Open a PowerShell session with elevated access rights and create a directory into which the build agent will be installed:

    `PS C:\> mkdir agent ; cd agent`

 5. Install the agent by issuing the following PowerShell command:

    `PS C:\agent> Add-Type -AssemblyName System.IO.Compression.FileSystem[System.IO.Compression.ZipFile]::ExtractToDirectory("$HOME\Downloads\vsts-agent-win-x86-2.146.0.zip", "$PWD")`
    
 6. Configure the build agent by executing the following command:

    `PS C:\agent> .\config.cmd`
    
    the following information will be prompted for:
    
    - `Enter server URL`
       Enter https://dev.azure.com/<organization name>.

    - `Enter personal access token`
       Paste in the access token string obtained from step 2.

    - `Enter agent pool`

    - `Enter agent name (hostname)`
      The default name of the agent is the name of the host the agent runs on, hit enter to accept this as the default.

    - `Enter work folder ? (Press enter for _work)`
      Hit enter to accept the default. 

    - `Enter run agent as service (Y|N)`
      Enter 'Y'.

    - `Enter user account to use for the service`
      Enter the fully qualified name of a domain account.

    - `Enter password for the account`
      Enter the password asscociated with the account entered in the previous step.
      
    If the information requested has been entered correctly, a number of messages will appear, with the last message stating that the 
    agent service has started successfully.
    
 ### msbuild and SQL Server Data Tools Installation
 
 1. Downloasd the command line for nuget (nuget.exe) from this [link](https://dist.nuget.org/win-x86-commandline/v4.7.0/nuget.exe).
 
 2. Add the absolute path of nuget.exe to the PATH variable.
 
 3. Install SQL Server Data Tools:
 
    `nuget.exe install Microsoft.data.tools.msbuild -ExcludeVersion -OutputDirectory "C:\SSDTTools"`
    
 4. Configure the environment for SQL Server Data Tools by running the following three commands:

    `setx PATH "%PATH%;C:\SSDTTools\Microsoft.Data.Tools.Msbuild\lib\net46" /M`
    
    `setx SQLDBExtensionsRefPath C:\SSDTTools\Microsoft.Data.Tools.Msbuild\lib\net46 /M`
    
    `setx SSDTPath C:\SSDTTools\Microsoft.Data.Tools.Msbuild\lib\net46 /M`
    
 5. Open a DOS command shell whilst logged in as the domain account that the Azure DevOps build agent runs under. 
    Check that msbuild and sqlpackage can be found by using the following commands:

    `where msbuild`
    
    `where sqlpackage`
    
    If the tooling is installed correctly, the where commands will return the absolute paths for msbuild and sqlpackage.
 
 ### PureStorageDbaTools Installation
 
 1. Check that the PowerShell gallery is a trusted repository by running the following PowerShell function:

    `Get-PsRepository`
    
    This should return the following output:

    ![image](https://user-images.githubusercontent.com/15145995/52906043-2879ac80-323c-11e9-93b5-438acc7035c0.png)
    
 2. If the PowerShell gallery is not set up as a trusted repository, run the following command:

    `Register-PSRepository -Default -InstallationPolicy Trusted`
    
 3. Install the PureStorageDbaTools module, this will also install the dbatools and PureStoragePowerShellSDK modules:
 
    `Install-Module -Name PureStorageDbaTools`
 
 ### Pipeline Creation and Configuration
 
 1.  Log into GitHub, you will need to create a GitHub account unless you already have one.#
 
 2.  Create a fork of the chrisadkin/AzureDevOps-Fa-Snapshot-Ci-Pipeline repo
 
 3.  Click on Pipeline in the pane on the left hand side of the screen and New pulldown menu, select "Build pipeline"

 4.  At the "Where is your code ?" prompt select GitHub

 5.  Log into GitHub using your GitHub account credentials
 
 6.  Select \<your account name>/AzureDevOps-Fa-Snapshot-Ci-Pipeline

 7.  Hit Run, the Pipeline will fail (expected) because we need to add some parameterised value to it. At the time of
     writing there is no option to save the pipeline without running it first

 8.  Click on the three dots (...) in the top right hand corner of the screen and select "Edit pipeline"
 
 9.  Click on the three dots (...) in the top right hand corner of the screen and select "Pipeline settings"

 10. In the variables section, create the following variables:

 - `agentPool`       name of the agent pool that contains the self hosted windows build agent
 - `pfaUsername`     name of an account for accessing the FlashArray which the source and target databases are stored on
 - `pfaPassword`     password of the account for accessing the FlashArray
 - `refreshDatabase` name of the database to refresh
 - `refreshSource`   SQL Server instance hosting the database which is the source of the refresh
 - `refreshTarget`   SQL Server instance hosting the database which is the target of the refresh
 - `pfaEndpoint`     ip address of the FlashArray management endpoint
