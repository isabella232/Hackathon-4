# CampRubrikHackathon

This lab guide consists of 4 separate separate sections:

1. [Live Mount a database](https://github.com/rubrikinc/Hackathon/tree/main/Singapore#live-mount-a-database)
1. [Getting started with PowerShell ](https://github.com/rubrikinc/Hackathon/tree/main/Singapore#live-mount-a-database#getting-started-with-powershell)
1. [Generate an asBuilt Rubrik cluster report](https://github.com/rubrikinc/Hackathon/tree/main/Singapore#live-mount-a-database#generate-an-asbuilt-rubrik-cluster-report)
1. [Links and references](https://github.com/rubrikinc/Hackathon/tree/main/Singapore#links-and-references)

## Live Mount a database

Live mount allows for near instant recovery of a database. If a database restore/export normally takes hours, then live mounting a database will take a few minutes. Live Mount does a full recovery of a database to either the same SQL Server Instance with a different database name or another SQL Server Instance with the same or different database name. The recovery of the database is much faster, because Rubrik does not need to copy the contents of the backup from the Rubrik Cluster back to the SQL Server. All of the recovery work is done on the Rubrik cluster itself. Then the database files are presented to the SQL Server Instance via a secure SMB3 share that is only accessible by the machine the share is mounted to.

Live Mounting a database is great for a lot of different use cases:

- Object level recovery
- Developer testing
- DevOps Automation
- Reporting databases
- DBA Backup validation testing
- Database migration application smoke test validation.

A key parameter is RecoveryTime. All dates in Rubrik are stored in UTC format. This parameter is expecting a fully qualified date and time in UTC format. example value is 2018-08-01T02:00:00.000Z. 

### Step 1 - Access the Rubrik API playground 

The API Playground is an implementation of the open source Swagger UI which allows you to quickly discover and make live API calls against your Rubrik cluster. Access the playground via ```https://{{node_ip}}/docs/{{v1}}/playground ```

### Step 2 - Authorize the playground

Use your Rubrik cluster credentials to authorize. Use basic authorization for this exercise.

![alt text](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/Authenticate1.png)

![alt text](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/Authenticate2.png)

### Step 3 - Find SQL Server instance id

Use ``` GET /mssql/instance```

![alt text](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/getinstance-id.png)

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/GetInstance2.png)


### Step 4 - Find SQL Server database id

Use ``` GET /mssql/db```

Pass the SQL Server instance id fetched in Step 3 as a value for "**instance_id**" parameter

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/Get%20DB%20Information.png)

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/Get%20DB%20Information2.png)

### Step 5 - Live mount a snapshot on a target host

Use ``` POST /mssql/db/{id}/mount```

_For this exercise we will use the same SQL Server instance to Live mount the database._

This will create an asynchronous request to create a Live Mount SQL server database on a target instance identified by "**targetInstanceId**" value in the config object.

Modify the sample config object to your environment.

``` Javascript
{
  "recoveryPoint": {
    "date": "2021-07-11T11:05:22.788Z" // Recovery point specified in ISO8601 format, such as "2021-07-11T11:05:22.788Z"
  },
  "mountedDatabaseName": "AdventureWorks2019_LiveMount", // Name to assign to the mounted database.
  "targetInstanceId": "MssqlInstance:::05b2b608-b7df-4a85-8ba2-652f6e27361a", //ID of the SQL Server instance to mount the database on. In this exercise we will use the same SQL Server instance 
  "recoveryModel": "FULL" // Recovery model for a SQL Server database -  SIMPLE, FULL, BULK_LOGGED
}
```

From the response body note the request id.

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/LM1.png)

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/LM2.png)


### Step 6 - Find the status of the async request

Use ``` GET /mssql/request/{id} ```

Pass the request id of the async job fetched in Step 5 as a value for "**id**" parameter

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/request1.png)
![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/request2.png)


### Step 7 - Check the Live Mount database in SQL Server Management Studio

Switch to SQL Server Management Studio on your Windows host to view the Live Mounted Database.

![](https://github.com/rubrikinc/Hackathon/blob/main/Singapore/images/SSMS.png)

## Getting started with PowerShell 

### Installing the Rubrik SDK for PowerShell

```
:pushpin: Note:
If you need to install PowerShell in production environments or for home use I would recommend using this link to download the latest version of PowerShell for your OS of your choice: aka.ms/getps6
```

1. Open a Powershell console with the Run as Administrator option.
1. Run `Get-ExecutionPolicy` to validate Execution policy is set to RemoteSigned or Bypass.
    1. If not set correctly Run `Set-ExecutionPolicy` using the parameter RemoteSigned or Bypass.
1. Run `Get-Module -ListAvailable *Rubrik*` to validate that the correct modules are installed.
    1. Run `Install-Module -Name Rubrik -Scope CurrentUser -Force` to download the module from the PowerShell Gallery. Note that the first time you install from the remote repository it may ask you to first trust the repository.
    1. Run `Install-Module -Name AsBuiltReport.Rubrik.CDM -Scope CurrentUser` to download the module from the PowerShell Gallery.

### Connecting to the Rubrik Cluster

To begin, connect to a Rubrik cluster. To keep things simple, we'll do the first command without any supplied parameters.

```
:pushpin: Note:
If you need to install PowerShell in production environments or for home use I would recommend using this link to download the latest version of PowerShell for your OS of your choice: aka.ms/getps6
```

1. Open a PowerShell session
1. Type `Connect-Rubrik` and press enter.

A prompt will appear asking you for a server. Enter the IP address or fully qualified domain name (FQDN) of any node in the cluster. An additional prompt will appear asking for your user credentials. Once entered, you will see details about the newly created connection.

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image1.png)

### Authenticate by providing credential object

It is also possible to store your credentials in a credential object by using the built-in Get-Credential cmdlet. To store your credentials in a variable, the following sample can be used:

``` PowerShell
$Credential = Get-Credential
```

PowerShell will prompt for the UserName and Password and the credentials will be securely stored in the `$Credential` variable. This variable can now be used to authenticate against the Rubrik Cluster in by running the following code:

``` PowerShell
Connect-Rubrik -Server 10.0.2.10 -Credential $Credential
```
### Commands and Help

What if we didn't know that `Connect-Rubrik` exists? To see a list of all available commands, type in `Get-Command -Module Rubrik`. This will display a list of every function available in the module. Note that all commands are in the format of **Verb-RubrikNoun**. This has two benefits:

* Adheres to the Microsoft requirements for PowerShell functions.
* Use of "Rubrik" in the name avoids collisions with anyone else's commands.

For details on a command, use the PowerShell help command `Get-Help`. For example, to get help on the `Connect-Rubrik` function, use the following command:

``` PowerShell
Get-Help Connect-Rubrik
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image2.png)

This will display a description about the command. For details and examples, use the `-Full` parameter on the end.

``` PowerShell
Get-Help Connect-Rubrik -Full
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image3.png)

As this is a lot of help to process, the help function can be used instead of Get-Help, to get scrolling output.

``` PowerShell
help Connect-Rubrik -Full
```

An alternative method is to view the help in your default browser by using the following command:

``` PowerShell
help Get-RubrikVM -Online
```

If you are more interested in a hands-on approach, the following parameter can be used: `-Examples`

```PowerShell
Get-Help New-RubrikSLA -Examples
```

Feel free to try one of the examples and see how easy it is to create an SLA from scratch using the Rubrik PowerShell module

### Gathering Data

Let's get information on the cluster. The use of any command beginning with the word get is safe to use. No data will be modified, so these are good commands to use if this is your first time using PowerShell.

We'll start by looking up the version running on the Rubrik cluster. Enter the command below and press enter:

``` PowerShell
Get-RubrikVersion
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image5.png)

The result is fairly simple: the command will output the cluster's code version. How about something a bit more complex? Try getting all of the SLA Domain details from the cluster. Here's the command:

``` PowerShell
Get-RubrikSLA
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image6.png)

A lot of stuff should be scrolling across the screen. You're seeing details on every SLA Domain known by the cluster at a very detailed level. If you want to see just one SLA Domain, tell the command to limit the results. You can do this by using a parameter. Parameters are ways to control a function. Try it with this example:.

``` PowerShell
Get-RubrikSLA -SLA 'Gold'
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image7.png)

The `-SLA` portion is a parameter and "Gold" is a value for the parameter. This effectively asks the function to limit results to one SLA Domain: Gold. Easy, right?

For a full list of available parameters and examples, use `Get-Help Get-RubrikSLA -Full`. Every Rubrik command has native help available.

### Modifying Data

Not every command will be about gathering data. Sometimes you'll want to modify or delete data, too. The process is nearly the same, although some safeguards have been implemented to protect against errant modifications. Let's start with an simple one.

This example works best if you have a test virtual machine that you are not concerned with changes being made to it. Make sure that virtual machine is visible to the Rubrik cluster. To validate this, use the following command:

``` PowerShell
Get-RubrikVM -VM "your vm name"
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image8.png)

Make sure to replace `"JBrasser-Win"` with the actual name of the virtual machine. If you received data back from Rubrik, you can be sure that this virtual machine is known to the cluster and can be modified.

### Note - Quoting Rules

*The double quotes, or single quotes, are required if your virtual machine has any spaces in the name. It's generally considered a good habit to always use quotes around the name of objects.*

Let's protect this virtual machine with the "Gold" SLA Domain. To do this, use the following command:

``` PowerShell
Get-RubrikVM -VM 'Name' | Protect-RubrikVM -SLA 'Gold'
```

Before the change is made, a prompt will appear asking you to confirm the change.

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image9.png)

This is a safeguard. You can either take the default action of "Yes" by pressing enter, or type "N" if you entered the wrong name or changed your mind. If you want to skip the confirmation check all together, use the `-Confirm:$false` parameter like this:

``` PowerShell
Get-RubrikVM -VM 'Name' | Protect-RubrikVM -SLA 'Gold' -Confirm:$false
```

This will make the change without asking for confirmation. Be careful!

Additionally, it is also possible to either set the confirmation preference for an individual cmdlet or function by changing the default parameters and arguments:

``` PowerShell
$PSDefaultParameterValues = @{"Rubrik\Protect-RubrikVM:Confirm" = $false}
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image10.png)

By setting this, for the duration of your current PowerShell session, `Protect-RubrikVM` will no longer prompt for confirmation.

### Gather data for reporting

If we want to know the status of backups for certain workloads or SLAs we can easily gather this data using the PowerShell module.

``` PowerShell
Get-RubrikVM -SLAID 'Gold'
```

We can use the SLAID parameter of Get-RubrikVM to only gather a list of VMs that are protected under the Gold SLA. If we want to have the number of VMs that are protected under the Gold SLA, we can use the `Measure-Object` cmdlet in PowerShell:

``` PowerShell
Get-RubrikVM -SLAID 'Gold' | Measure-Object
```

![alt text](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/img/image10.png)

If we want to make this output readable, we could also choose to only display either the output or the count-property:

``` PowerShell
Get-RubrikVM -SLAID 'Gold' | Measure-Object | Select-Object -Property Count
Get-RubrikVM -SLAID 'Gold' | Measure-Object | Select-Object -ExpandProperty Count
```

Alternatively we can also display all VMs that are protected by any SLA:

``` PowerShell
Get-RubrikVM | Where-Object {$_.EffectiveSlaDomainName -ne 'Unprotected'}
```

We use the PowerShell `Where-Object` cmdlet to filter out VMs that are not protected by SLAs. Now if we want to take this one step further, we can also make a generate a summary of all the number of VMs assigned to each SLA:

``` PowerShell
Get-RubrikVM | Where-Object {$_.EffectiveSlaDomainName -ne 'Unprotected'} |
Group-Object -Property EffectiveSlaDomainName | Sort-Object -Property Count -Descending
```

## Generate an asBuilt Rubrik cluster report

![](https://github.com/AsBuiltReport/AsBuiltReport.Rubrik.CDM/raw/dev/Samples/Rubrik-Report-1.png)

### :pencil2: Configuration

The Rubrik CDM As Built Report utilises a JSON file to allow configuration of report information, options, detail and healthchecks.

A Rubrik report configuration file can be generated by executing the following command;
```powershell
New-AsBuiltReportConfig -Report Rubrik.CDM -Path <User specified folder> -Name <Optional>
```

Executing this command will copy the default Rubrik report JSON configuration to a user specified folder.

All report settings can then be configured by directly modifying the JSON file.

The following provides information of how to configure each schema within the report's JSON file.


### InfoLevel

The **InfoLevel** sub-schema allows configuration of each section of the report at a granular level. The following sections can be set

There are 4 levels (0/1/3/5) of detail granularity for each section as follows;

| Setting | InfoLevel     | Description                                                                                        |
|---------|---------------|----------------------------------------------------------------------------------------------------|
| 0       | Disabled      | Does not collect or display any information                                                        |
| 1       | Summary       | Provides summarized information for a collection of objects                                        |
| 2       | Unused        | Reserved for future use                                                                            |
| 3       | Detailed      | Provides detailed information for individual objects                                               |
| 4       | Unused        | Reserved for future use                                                                            |
| 5       | Comprehensive | Provides comprehensive information for individual objects, such as advanced configuration settings |

***Note*** While you can specify InfoLevels of 2 and 4, they will simply default to the closest defined level below them. IE 2 becomes 1 and 4 becomes 3.

The table below outlines the default and maximum **InfoLevel** settings for each section.

| Sub-Schema        | Default Setting | Maximum Setting |
|-------------------|-----------------|:---------------:|
| Cluster           | 3               |                 |
| SLADomains        | 3               |                 |
| ProtectedObjects  | 3               |                 |
| SnapshotRetention | 3               |                 |

### Generate a Rubrik AsBuilt Report

  Generate a Rubrik CDM As Built Report for a cluster named 'cluster1.domain.local' using stored credentials.  Export report to HTML & Text formats. Use default report style. Save reports to 'C:\Reports'
   ```powershell
   New-Item C:\Reports -Type Directory
   $Creds = Get-Credential -Message 'Please enter the lab Rubrik CDM credentials'
   New-AsBuiltReport -Target 10.0.2.10 -Credential $Creds -Report Rubrik.CDM -Format Html,Text -OutputFolderPath 'C:\Reports'
   ```

You can view this report by navigating in Windows explorer to the Reports folder and clicking the HTML report once the action is complete.

## Links and references

* [PowerShell Download Link](https://aka.ms/getps6)
* [GUIDE: Rubrik PowerShell - Quick Start Guide](https://github.com/rubrikinc/rubrik-sdk-for-powershell/blob/devel/docs/quick-start.md)
* [VIDEO: Getting Started with the Rubrik SDK for PowerShell](https://www.youtube.com/watch?v=tY6nQLNYRSE)
* [BLOG: Get-Started with the Rubrik PowerShell Module](https://www.rubrik.com/blog/get-started-rubrik-powershell-module/)
* [BLOG: Using Automation to Validate Applications and Services in Rubrik Backups](https://www.rubrik.com/blog/automation-to-validate-in-rubrik-backups/)* [Rubrik CDM API * Documentation](https://github.com/rubrikinc/api-documentation)
* [GitHub - AsBuiltReport - RubrikCDM](https://github.com/AsBuiltReport/AsBuiltReport.Rubrik.CDM)
* [BLOG: Rubrik AsBuiltReport](https://www.rubrik.com/blog/products-solutions/20/4/rubrik-as-built-report-for-easier-automated-documentation)
