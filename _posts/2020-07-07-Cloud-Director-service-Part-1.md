g---
layout: post
title:  "Cloud Director service - Part 1 : VMware.CDS.Community PowerShell module"
date:   2020-07-07 20:00:00 +0200
author: Adrian Begg
categories: [Cloud Director]
tags: [vcpp,CDS,CloudDirector]
---
{% include wordcount.html %}
Today I want to share a quick post on a PowerShell module I published to facilitate code based deployments of VMware Cloud Director instances using VMWare's recently announced Cloud Director service. I was lucky as an external partner to be involved during the Early Access period of the product development and an early requirement for testing was to develop a repeatable, consistent approach to deploying instances to allow quick stand up and tear down of the environment and lay the foundations for operationalizing a future offering. As my Go skills are currently lacking (and I dislike Python) I opted to hack together a module for PowerShell to cover the basic functions.

# What is Cloud Director service ?
Cloud Director service is a SaaS for VMware Cloud Director running on AWS's Global Infrastructure which is currently in Early Access. It allows VMWare Cloud on AWS (VMC) SDDCs to be consumed as Provider VDCs by a Cloud Director installation also running on AWS's Global Infrastructure managed by VMware. It allows VMware Cloud Service Providers to spin up VMware Cloud Director (and Provider VDCs backed by VMC) quickly, wherever (hopefully eventually) VMC is available. Have a customer requirement to have a presence in us-east-2 but don't have a datacenter ? There are a heap of other use cases but agile geo-expansion and being able to offer VMC to smaller customers (that might not have the workload density to move to a full VMC SDDC) are the top of my list.

Pretty cool stuff in my opinion, with the right tooling you can easily stand up everything you need for a fully configured, customized and ready for customer consumption Cloud Director instance easily within a single working day anywhere the VMC and CDS service are available in a consistent, automated manner.

In order to use the service your Organization will need to be a VMWare MSP and have the service enabled on your organization. For more information on the process (and the product itself) please see [VMware Cloud Director service microsite](https://cloud.vmware.com/cloud-provider-hub/cloud-director-service).

## Getting Started
In the below example I will walk through creating a new instance in the us-west-2 Early Access environment (the only available region at the time of writing), associating a VMC SDDC. This assumes that you have already setup a VMC SDDC which will be used as the Resource vCenter for the instance.

Step 1. Install the module either from GitLab or directly from the PowerShell Gallery:
```
Install-Module -Name VMware.CDS.Community -Scope CurrentUser
Import-Module VMware.CDS.Community
```

Step 2. Logon to VMware Cloud Services and under the Provider Internal Organization that will host the resources generate an API token (My Account > API Tokens > Generate Token) with the VMware Cloud Director role "Cloud Director Administrator"
![alt text](/assets/vcds-pwsh-1.png "Generate an API Token"){:style="float: right;margin-right: 20px;margin-top: 0px;"}

Step 3. Connect to the VMware Cloud Service Hub using the Connect-VCDService cmdlet and the API token from Step 2. - this will establish a session and store the session within a Global variable "$VCDService"
```
Connect-VCDService -CSPAPIToken "Your token goes here"
```
Step 4. Create a set of HashTables which will be used to configure the initial instance deployment.

```
# The CDS Environment to deploy the instance.
[Hashtable] $CloudDirectorEnvironment = @{
    Name = "Early Access Environment"
    Location = "us-west-2"
}

# The Template options
[Hashtable] $CloudDirectorTemplate = @{
    Name = "VCloud Director 10.1.0"
}

# The Cloud Director instance options
[Hashtable] $CloudDirectorInstance = @{
    Name = "CloudDirector-TestInstance-01"
    AdministratorPassword = "P@ssword!123"
}
```
Step 5. Create the new instance (takes about 10 - 15 minutes) to complete.
```
# Get the Environment and Template Parameters
$CDSEnvironment = Get-VCDSEnvironments @CloudDirectorEnvironment
$CloudDirectorTemplate.Add("EnvironmentId",$CDSEnvironment.id)
$CloudDirectorInstance.Add("EnvironmentId",$CDSEnvironment.id)

$Template = Get-VCDSTemplates @CloudDirectorTemplate
$CloudDirectorInstance.Add("TemplateId",$Template.id)

# Make a call to create a new Cloud Director instance and return the CDS Task object
$TaskInstanceCreate = New-VCDSInstance @CloudDirectorInstance

# Watch the Task and Wait for it to complete
if(!(Watch-VCDSTaskCompleted -Task $TaskInstanceCreate -Timeout 1800)){
    throw "An error occurred creating the instance $($InstanceConfig.Instance.InstanceName) please check the console and try the operation again."
}
```
You should also see the new instance in the Cloud Director service portal and the workflows executing. The process typically takes around 15 minutes, the instances actually come online within minutes but there is a delay waiting for the public DNS records to propagate.
![alt text](/assets/vcds-pwsh-2.png "New Cloud Director instance workflow"){:style="float: right;margin-right: 20px;margin-top: 0px;"}

Step 6. After the instance creation task complete the next step is to Associate a VMC SDDC with the instance which will be used as the Resource vCenter hosting the Cloud Director workloads. This can be any SDDC within the 150ms RT latency boundary of the Cloud Director service (e.g. us-west-2, us-west-1, us-east-1, us-east-2, eu-west-2). You will need to provide the VMC Organization UUID hosting the SDDC, the SDDC Name and an API token with access to the Organization
```
# Get the Instance properties of the Cloud Director instance created in previous step
$Instance = Get-VCDSInstances -Name $CloudDirectorInstance.Name -EnvironmentId $CloudDirectorInstance.EnvironmentId

# The SDDC Configuration
[Hashtable] $SDDCInstance = @{
    VMCAPIToken = "SDDC Organisation API token goes here"
    VMCOrganisationUUID = "SDDC Organisation UUID goes here"
    SDDCName = "SDDC Name goes here"
    EnvironmentId = $CloudDirectorInstance.EnvironmentId
    InstanceId = $Instance.id
}

# Register the SDDC and Disconnect
$RegistrationTask = Register-VCDSSDDC @SDDCInstance

Disconnect-VCDService
```

## Functional Coverage
The following cmdlets are available in the current release.
Session:
* Connect-VCDService : Establishes a new connection to the VMware Cloud Director service using an API Token from the VMware Console Services Portal
* Disconnect-VCDService : This cmdlet removes the currently connected VMware Cloud Director service connection.

Environment
* Get-VCDSEnvironments : Returns a collection of Cloud Director Service environments for the default CSP environment on the currently available under the currently connected VMware Console Services Portal account.
* Get-VCDSTemplates : Returns the available templates for the provided Cloud Director Service environment.
* Get-VCDSInstances : Returns the Cloud Director Service instances currently running under the currently connected VMware Console Services Portal account.
* New-VCDSInstance : Creates a new instance of Cloud Director Service under the currently connected VMware Console Services Portal account.
* Remove-VCDSInstance : Deletes an instance of Cloud Director Service under the currently connected VMware Console Services Portal account.

Operations:
* New-VCDSSupportBundle : Generates a Cloud Director support bundle
* Register-VCDSSDDC : Associate an VMC SDDC with a VMware Cloud Director service instance.
* Set-VCDSDomain : Configures a Custom DNS name and X.509 SSL certificates for a Cloud Director service instance and Console Proxy endpoints.

Administration:
* Get-VCDSTasks : Returns a collection of Tasks from the connected Cloud Director Service environment.
* Watch-VCDSTaskCompleted : A helper function to monitor a running task and returns True when the task completes.

All of the cmdlets in the module should have well described PowerShell help available. For detailed help including examples please use `Get-help <cmdlet> -Detailed` (e.g. `Get-help New-VCDSInstance -Detailed`).

In terms of the future I will attempt to maintain the module as needed however community contributions are very welcome. I personally hope that VMware continues to extend the VMware Cloud terraform provider and adds support for Cloud Director service as this is definitely where all of my future investment in IaC based deployments is heading.

I expect that the VMware Cloud Director service will be announced for General Availability soon; I hope that this module may be useful for people interested in the product. Any questions or feedback please reach out to me on Twitter (@AdrianBegg) or on the VCPP/vExpert or VMWare Code Public Slack.