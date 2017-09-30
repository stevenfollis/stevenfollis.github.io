---
layout: post
title: Containerizing an Access Database in a Windows Container
date: 2017-09-30 13:08:10.000000000 -04:00
---

# Containerizing an Access Database in a Windows Container

At Docker we are seeing a cavalcade of customers finding value in containers not just for greenfield microservice applications, but also as a key to modernizing legacy applications. One recent conversation involved a question: can a legacy line of business application that uses an Access database be transitioned to a container? Having not touched an Access database in many years, I set out to find an answer. 

## The Database

Needing a sample database that would be representative of an older application, I fired up Access 2017 and created a new database from the Northwind Traders template. This sample database started up as a `.accdb` file, the file format that Access moved to back in 2007. I wanted to lean more oldschool, so attempted an export to the older `.mdb` format. Unfortunately, Access refused the operation, citing the new features in-use that were not backwards compatible to previous formats.

With Northwind appearing to be a dead end, I looked online for a second sample database known to SQL aficionados as AdventureWorks. [Codeplex](https://adventureworksaccess.codeplex.com/) had a copy, and although it downloaded as a `.aacdb`, Access easily exported it as a `.mdb` by using the **Access 2000 Database** option.

![image](/content/images/2017/09/image001.png)

> The .mdb file size is over 100MB, which makes GitHub will barf. To shave off some space, I trimmed several rows out of the `Sales_SalesOrderDetail` table

## The Web Application

With a database secured, it was time to stand up a web application to interface with the data. My goal was to create a proof of concept web application that communicate with the database and displays data via a browser. 

Firing up Visual Studio 2017 Community Edition, I created a solution based on the web application template. Wanting to be as legacy-esque as possible, I set the .NET version to a stately 2.0.

![image](/content/images/2017/09/image002.png)

Once the solution provisioned, I created a `default.aspx` web form and an `App_Data` folder. The `AdventureWorks.mdb` file went into App_Data, and I scaffolded out a simple page in the aspx file by importing the Bootstrap library and adding a heading. 

![image](/content/images/2017/09/image003.png)

Now came the fun part: plumbing the database into the web app. The first path taken invovled opening the Server Explorer and using the connection wizard to setup a connnection to an Access DB. This turns out to use [JET](https://en.wikipedia.org/wiki/Microsoft_Jet_Database_Engine), which was long superceed by ACE. Another downside to using JET was that while it worked, I had to install an Office Distributable package. This package was an `.exe` wizard with a UI, and I was nervous about getting it to run in headless container environment (where's an MSI when you need one?) The customer was also using ODBC today, so I stepped back and re-implemented the connection via an ODBC connector. 

ODBC has several ways of configuration, and the Visual Studio wizard took me down a path of using a system-level DSN. This DSN securely maps the location of the database file and ODBC driver settings into a credential that my application can use. After setting the world's smallest `ConnectionString` in `web.config` I was off to the races. Running locally, I was able to see a table of data pulled from the Access database file.

With my simple application working on my local laptop it was time to container-ize.

## The Container

Creating a container involves crafting a Dockerfile. As a base image, `microsoft/windowsservernano` is a lighter weight base, however legacy applications rely on a myriad of features and capabilities that are simply not possible in nano. For this app, I instead turned to `microsoft/aspnet:4.6.2`, based on the larger and more robust Windows Server Core. 

> The [Visual Studio Tools for Docker](https://docs.microsoft.com/en-us/dotnet/core/docker/visual-studio-tools-for-docker) are terrific for setting a debugging environment within VS for testing containers. However, I find it overly complex for simpler projects, and the heavy use of build time arguments that are necessary for debugging make greatly increases the complexity of moving the project into a CI/CD pipeline. Instead of auto-generating the Dockerfile via the Tools, I instead opted to handcraft my own

With the `FROM` statement ready to go, it was time to configure the inside of the container to match my local environment. This requires a few steps:

1. Configure the ODBC DSN with the local of the `.mdb` file and driver. This was accomplished with a bit of PowerShell, and was far easier than I initally expected.

1. Insert my VS solution code into the pre-determined IIS location of `C:\inetpub\wwwroot` as [instruced](https://hub.docker.com/r/microsoft/aspnet/) by the aspnet image.

However, after adding the steps, building the image, and starting a container, I was met with an ugly error screen. 

StackOverflow informed me that this was likely due to a mismatch in running a 64-bit application with the 32-bit ODBC driver. When I initially created the application, VS by default used the 32-bit version of IIS Express (hence no issue), whereas the aspnet image was defaulting to using 64-bit IIS. Back in VS, I set it to use 64-bit IIS Express, and confirmed that I saw the same error - this issue was realted to 32 vs. 64 bit settings.

The solution was to adjust the IIS Application Pool within the container to run a 32-bit application. This again was a snippet of PowerShell, which enabled the feature. After a quick re-build and re-creation of a container, I was pleased to see the sample application running as expected.

## The Wrap Up

This [incredibly] simple application imparted several nuggets of wisdom related to the containerization of legacy applications. 

First, the "hello world" examples are nice for a quick demonstration, but real-world apps are going to take some finese and adjustments to the base images. Be prepared to track down PowerShell commandlets and overloads that may not be immediately obvious.

Second, there are many ways to skin a cat and those ways have likely evolved between when the app was created and today. There were 3+ methods of talking to an Access database, each with varying pros/cons, histories, and present-day supportability concerns. 

Finally, erring on the side of configuration via PowerShell rather than an MSI or other installing simplifies the creation and maintenance of images. This will not always be an option, but I greatly preferred PS to touching a ~10 year old installer .exe.