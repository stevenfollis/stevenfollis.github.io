---
layout: post
title: Continuous Integration (CI) for Azure Resource Manager (ARM) Templates with
  Visual Studio Team Services (VSTS)
date: 2016-12-12 06:07:34.000000000 -05:00
---
The concept of [Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration) makes developing any piece of software infinitely more efficient and convenient.  At a high level, CI takes a set of changes in a code repository and makes sure the changes don't break stuff. For code that has compilation steps this is better understood, but how do we get the benefits of CI with a declarative JSON document such as an ARM Template?  

This was the question that arose on a recent project that included an ARM Template that needed to be deployed in multiple regions.  Regular changes were being made to the template file that were committed to the repository without any real checks - it took a full deployment to determine if the template would successfully deploy or fail miserably. To smooth out this process, we took the ARM Template and used a [Visual Studio Team Services Release Definition](https://www.visualstudio.com/en-us/docs/release/author-release-definition/more-release-definition) to automate the validation of a template in a myriad of regions.  

We began by authoring a PowerShell script that would take an ARM Template and loop through all regions worldwide to create Resource Groups and deployments.  Doing a full deployment in every region worldwide is expensive, time consuming, and may hit subscriptions limits for resources such as VM cores.  Instead, we can send a template to ARM's [Validate Endpoint](https://docs.microsoft.com/en-us/rest/api/resources/deployments#Deployments_Validate).  This endpoint checks the syntax of a template file without doing an actual deployment, and determines if the file will be accepted by the Resource Manager during a deployment.  Using this endpoint is a cheaper, easier way to get a good idea of a template's deploy-ability, and was a better option than full on deployments. 

Delightfully, the out of the box Azure Resource Group Deployment task within VSTS already had the capability to do a validation only deployment, meaning the in-flight PowerShell script was scrapped. 

![image](/content/images/2016/12/image001.png)

First, we need a new Release Definition. From there I created an **Environment** for each of the 15 Azure Regions that I wanted to test the template against. I'd used 2-5 Environments before, but VSTS had no issue with 15 disparate Environments.  Within each of these Environments two tasks were created:

* The first Azure Resource Group Deployment task takes the specified ARM Template and executes a **Validation Only** deployment. Since a Resource Group is needed, this step automatically generates a RG as part of the task

* The second Azure Resource Group Deployment tasks deletes the Resource Group that is created in the first step

![image](/content/images/2016/12/image002-1.png)

> Note that under **Control Options** we've checked the box for **Continue on error**. This will ensure that even if a template fails validation that the release will continue to the cleanup task. Else our subscription will be littered with empty resource groups

By default the regions will execute serially one after another. This is fine if we're using the convenient Hosted Agent within VSTS to run our builds up in the cloud, however we may have a pool of servers with Build Agents installed. In this case we'd like to execute validations in parallel.  Go ahead and open the **Deployment conditions** for each Environment.

![image](/content/images/2016/12/image003.png)

Next, adjust the Trigger to **After release creation**. This will allow multiple environments to run at the same time, making the process far quicker.

![image](/content/images/2016/12/image004.png)

With our Release Definition completed with an Environment for each Azure Region, create a New Release. VSTS will run the tasks and provide a nice dashboard.

![image](/content/images/2016/12/image005.png)

I can see from the Deployment Status column that there was an issue with Brazil South. At this time the task does not provide verbose feedback on why the template failed to validate, but I now have enough information to focus my debugging efforts.

The Validate endpoint does not catch 100% of issues. We ran into a scenario where a template validated, but a particular VM size was unavailable in the given region. There is no real substitute for a real deployment to ensure a template functions 100% properly, however this approach of the Validate endpoint catches a large number of issues before a deployment even occurs. 
