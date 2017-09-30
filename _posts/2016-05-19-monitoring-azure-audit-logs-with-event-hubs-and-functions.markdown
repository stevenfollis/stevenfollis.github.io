---
layout: post
title: Monitoring Audit Logs in Slack with Event Hubs + Azure Functions
date: 2016-05-19 15:45:35.000000000 -04:00
---
![Colors](/content/images/2016/05/slack.png)

Keeping tabs on an Azure Subscription can be difficult. What's happening in our environments?  Are operations failing?  These questions are hard enough with a handful of resources, but in a busy subscription containing hundreds of resources we can quickly be searching for needles in a haystack. 

Here's where logs can help.  Azure Audit Logs give us a consistent way of tracking Operations and Events at the Resource, Resource Group, or Subscription level.  

![Azure Audit Logs](/content/images/2016/05/azure-audit-logs.png)

These logs get down in the nitty gritty of our resources and serve as a critical tool in tracking changes to our environments over time.  Logs are kept for 90 days, which can be easily extended by configuring continuous export to an Azure Storage Account.  This makes a great use case for the brand new [Azure Cool Blob Storage](https://azure.microsoft.com/en-us/blog/introducing-azure-cool-storage/) accounts (I *love* that new resource smell!)

Keeping our logs sitting around until the next ice age is great, but what if I want actionable data right now?  For that we can turn to a new feature of Audit Logs: Export to Azure Event Hub. This [new capability](https://azure.microsoft.com/en-us/blog/new-features-for-azure-diagnostics-and-azure-audit-logs/) allows us to pipe log data into an Event Hub, which is optimized for high levels of data ingestion.  Once in the hub, we can interact with that data in a variety of ways.

For example, we could use Azure Stream Analytics to analyze the data pouring into the hub by taking advantage of its SQL-esque syntax and windowing support.  Or we could get our big data on by processing Event Hub data with [Storm](https://azure.microsoft.com/en-us/documentation/articles/hdinsight-storm-develop-java-event-hub-topology/).  We could roll a [Worker Role](https://blogs.msdn.microsoft.com/kaevans/2015/02/24/scaling-azure-event-hubs-processing-with-worker-roles/) with the lovable EventProcessorHost.

Those are all great options, but sound like a lot of work.  **All I want to do is pop an announcement into Slack whenever there's an event that I need to take a look at my subscription**.  Valuing speed, I decided to take the new Azure Functions out for a spin.

<iframe src="https://channel9.msdn.com/Shows/Cloud+Cover/Episode-205-Azure-Functions-with-Chris-Anderson/player" width="960" height="540" allowFullScreen frameBorder="0"></iframe>
(Shout out to my ~~boy~~ colleague [@romitgirdhar](https://twitter.com/romitgirdhar) whose cube used to be near mine in Charlotte)

Every time I touch Functions I smile like a kid that just got handed a puppy.  As a developer, Functions abstract away as much boilerplate, table stakes code as possible, and allows me to focus on code that I actually care about.  It is delightful, and [has support](https://azure.microsoft.com/en-us/documentation/articles/functions-overview/) for C#, Node.js, Python, F#, PHP, batch, bash, Java, ~~and a partridge in a pear tree~~.

## Configure Slack
Let's start by getting Slack ready to go.  We'll need to do three things:

* Create a new channel that will be dedicated to our alerts
* Setup a web hook integration that will pipe messages into our channel
* Connect the web hook to our channel

Inside of Slack, create a new channel named "auditlogs" by clicking the "+" icon next to "Channels" on the left hand nav.  From there, select the gear icon at the top and select "Add an app or integration"

![Slack start](/content/images/2016/05/slack-start.png)

This will pop a browser window, where you can search for "Incoming WebHooks".  This integration sets up an endpoint inside of Slack.  Whenever an HTTP post hits that endpoint carrying a message, the message will be inserted into a Slack channel.  Web Hooks are a terrific way to integrate disparate systems in a straightforward manner.  

![Slack Integrations](/content/images/2016/05/slack-integrations.png)

Follow the wizard to stand up a web hook integration for the "auditlogs" channel, being sure to make note of the "Webhook URL".  We will need that shortly.

![Webhook Settings](/content/images/2016/05/slack-settings.png)

Taking the URL and the sample from the "Sending Messages" section's documentation, let's test that endpoint in [Postman](http://www.getpostman.com/) to see webhook functionality in action.  

![Postman test](/content/images/2016/05/postman.png)

![Postman Success](/content/images/2016/05/postman-success.png)

When we send a simple HTTP POST to the endpoint, the object we pass it beautifully appears in our channel. Now that have a Slack integration ready to accept POSTs, let's fire up Azure.

## Azure
First off, we need a Service Bus Namespace.  This will serve as a container for our Event Hub.  From the Azure Portal, hit the "+" sign in the top right, and under "Hybrid Integration" select "Service Bus".

![Azure Portal Service Bus](/content/images/2016/05/azure-portal-sb.png)

Currently, the Service Bus functionality has not yet landed in the new portal, so we find ourselves quickly sitting in the Classic Portal.  What we are really after is the Service Bus Namespace, so do not worry too much about the Queue name (we're about to delete it). 

![Service Bus Creation](/content/images/2016/05/sb-creation.png)

After the Service Bus is provisioned, click into the new resource and delete the queue.

![Service Bus Home](/content/images/2016/05/sb-home.png)
![Delete Queue](/content/images/2016/05/deletequeue.png)

Back in the Ibiza Portal, let's hook up the Service Bus to our Audit Logs.  Select the flyout from the left nav and search for "Audit" to quickly locate the link for the Audit Log blad. Click it to have the blade flyout.

![Portal Search](/content/images/2016/05/portal-search.png)

Next, click the "Export" button from the top nav to show the export options.

![Export button](/content/images/2016/05/export-button.png)

Click on the section for "Azure Event Hub" and select our Service Bus.  **This export functionality will automatically setup an Event Hub for us**, hence why we only configured a Service Bus Namespace without an actual Event Hub earlier.

![EH Export](/content/images/2016/05/export-eh.png)

*However*, after clicking "OK" the "Export Audit Logs" blade won't let us save. It wants us to configure a storage account first.  I'm not sure if this is a UI bug (it says right at the top this is "Preview") or intended behavior, but it creates a dead end.

![greyed out options](/content/images/2016/05/greyedout.png)

I went ahead and created a new storage account (Standard_LRS), and piped it into this configuration area. Also worth mentioning that I selected all the regions available in the previous selection drop down.

![Option Available](/content/images/2016/05/greyedin.png)

With the export functionality wired up, we now need a Function to handle our logic.  Provision a new Function in the portal with the little green "+" sign in the top left, putting it in the same Resource Group as the Storage account we created previously.

![Provision Function](/content/images/2016/05/function-app.png)
![RG Resources](/content/images/2016/05/rg-resources.png)

Opening up our Function, let's start with "New" in the top left. Selecting "Node.js" and "Core" from the dropdowns, click the tile for "EventHubTrigger - Node".  This template will get us started with an input binding for Event Hubs. Input and output bindings mean we don't have to write all the code to hook into and transfer data between a data source - in this case, an Event Hub.  It's magic.

![New Function Settings](/content/images/2016/05/new-function.png)

Scrolling down we can configure the input binding. Remember when I said the Audit Log Export feature creates an event hub for us? It's named "insights-operational-logs".  Fill that in, and click "Select" next to the Event Hub Connection box.

![Binding Config](/content/images/2016/05/binding-config.png)

Here's where we connect back to our Event Hub. 

![SB Config](/content/images/2016/05/connection-setup.png)

Only problem is that we haven't yet setup a Shared Access Policy.  We need a SAP to get a connection string.  Sorry to keep jumping between portals, but open Classic Portal in a new tab.  From there, open our Event Hub and click the "Configure" tab. Here we can create a new Shared Access Policy with "Listen" Permissions.  This permission level allows our Function to read items off of the Event Hub. Click save, then head to the Dashboard tab on the top nav.

![EH Policy Setup](/content/images/2016/05/eh-policy.png)

On the dashboard of our Event Hub, click "Connection Strings" on the bottom nav bar and copy the String for the policy name you just created.

![EH String](/content/images/2016/05/eh-string.png)

Back in the new portal's tab, use the connection string to complete the "Add a Service Bus Connection" blade.  Before pasting, remove the `;EntityPath=insights-operational-logs` from the end of the string.

`Endpoint=sb://auditlogprocessor-ns.servicebus.windows.net/;SharedAccessKeyName=FunctionPolicy;SharedAccessKey=Fc1et/idbtoprwO5K1MLaFgegXTuXpdR/eUW2Dm3swc=;EntityPath=insights-operational-logs`

becomes

`Endpoint=sb://auditlogprocessor-ns.servicebus.windows.net/;SharedAccessKeyName=FunctionPolicy;SharedAccessKey=Fc1et/idbtoprwO5K1MLaFgegXTuXpdR/eUW2Dm3swc=`

The entity is handled in "Event Hub Name" box in the function binding configuration.  Click on and then "Create" the function.

![SB Connection](/content/images/2016/05/sbconn.png)

![Function Setup Finished](/content/images/2016/05/function-setup-finish.png)

We now have a Function App ready for us. You may see a flurry of activity as your Function comes online and processes the logs that have been piped from the Audit Logs into the Event Hub.  These logs have been sitting in the Event Hub just waiting for a consumer (our Function) to come pick them up and process them.

![Console catchup](/content/images/2016/05/logs-catchup.png)

Now for our custom code.  We want to do a variety of things

* Loop through each received log entry, as the Event Hub sends us an array of entries
* Discard any log entries that are Informational in nature. There are **tons** of these generated by the Azure service, and they do not typically require immediate attention
* Build a response packet of formatted strings
* Send response to the Slack Web Hook endpoint

For these activities, we use the following code:

```
"use strict";

var request = require('request');

module.exports = function (context, myEventHubTrigger) {

    context.log('======================================================');
    context.log('Node.js eventhub trigger function processed work item');

    // Grab logs from the returned records object
    var logs = myEventHubTrigger.records;

    // Prepare a response
    // Multiple logs may be received, but we'll batch into one response 
    var alert = [];

    // Loop through the logs
    var counter = 0;
    logs.forEach(function (log) {

        // Only process logs that are non-informational or a new RG
        // This cuts down on chatter everytime Azure ties its shoes
        if (log.level !== 'Information' || log.operationName === 'MICROSOFT.RESOURCES/SUBSCRIPTIONS/RESOURCEGROUPS/WRITE') {

            // Build out an alert's markup
            alert.push(buildAlert(log));

        }

        // Iterate counter and check progress
        counter++;
        if (counter === logs.length) {

            context.log('Finished looping through logs');

            // Finished iterating, send batched alert
            sendAlert(context, alert);

        }

    });

};

function buildAlert(log) {

    // String build via an array
    // Using ES6 Template Literals 
    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
    var logContent = [
        `time:\t ${log.time}`,
        `tenantId:\t ${log.identity.claims["http://schemas.microsoft.com/identity/claims/tenantid"]}`,
        `operationName:\t ${log.operationName}`,
        `category:\t ${log.category}`,
        `resultType:\t ${log.resultType}`,
        `resultSignature:\t ${log.resultSignature}`,
        `identity:\t ${log.identity.claims.name} (${log.identity.claims["http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"]})`,
        `location:\t ${log.location}`,
        `resourceId: ${log.resourceId}`
    ];

    // Determine color for alert
    var color;
    switch (log.level) {
        case 'Error':
            color = 'danger';
            break;
        case 'Warning':
            color = 'warning';
            break;
        case 'Information':
            color = '#439FE0';
            break;
    }

    // Create object to return
    var formattedLog = {
        "fallback": logContent.join('\n'),
        "color": color,
        "fields": [
            {
                "title": log.level,
                "value": logContent.join('\n'),
                "short": false
            }
        ]
    };

    // Return
    return formattedLog;

}

function sendAlert(context, alert) {

    // Send alert to Slack webhook
    request.post(process.env.SLACK_WEBHOOK_URL, {
        json: {
            "username": "audit-bot",
            "icon_url": "https://pbs.twimg.com/profile_images/546024114272468993/W9gT7hZo.png",
            "attachments": alert
        }
    }, function (error) {

        if (error) {
            context.log('Error posting the message');
            context.done(error);
        }
        else {
            context.log('Successfully posted the message');
            context.done();
        }

    });

}
```

Our code needs a few additional last pieces to function.  In `sendAlert()` we're using an environmental variable rather than hardcoding the URL directly into our code.  This makes the function more flexible. 

To set that environmental variable, click "Function App Settings" in the top right corner of the Function designer and select "Go to App Service Settings".

![App Settings Blade](/content/images/2016/05/function-app-settings.png)

Oh hello there, App Service Blade! You look familiar! From here, we can head to Settings -> Application Settings to add in an environmental variable named `SLACK_WEBHOOK_URL`, just like any other Azure Web App.

![Function App Settings](/content/images/2016/05/settings.png)

Also note that there's an App Setting called "EventHubConnection".  That should also look familiar, as it's the connection we previously configured in the Function's binding configuration. Save the Application S

One more comment on the code from above is that we have a dependency on a NPM library named ["Request"](https://github.com/request/request).  We need to go add this library into our Function, or else the code will fail. On the main App Service Blade, click "Browse" on the top nav to pop a new tab navigating to our Function app's URL.

Spoiler: it's going to be anti-climactic.  We don't have pages to serve back, so you all you'll see is a lovely white screen.  Instead, let's head over to the Kudu add `.scm` into the URL structure right before `.azurewebsites.net`. Our original URL of `http://auditlogfunction.azurewebsites.net/` becomes `http://auditlogfunction.scm.azurewebsites.net/` and voila! Kudu Time&#0153;.

![Kudu Home](/content/images/2016/05/kudu.png)

Click the "Debug Console" from the top nav and select PowerShell.  From here we can navigate to our Function app by selecting site -> wwwroot -> AuditLogProcessor (our function name).  In this directory we can see `index.js` containing the code we pasted in earlier, and `function.json` containing the bindings for our Function.

Click into the console, type in `npm install request` and hit enter. The package will be installed and our app is ready to rock.

![Console](/content/images/2016/05/kudu-console.png)

We could have provided an entire `package.json` chalked full of NPM dependencies, but either way we get a lovely `node_modules` folder.

![NPM modules](/content/images/2016/05/finished.png)

## Wrap Up
We now have an Azure Function that pulls messages off of an Event Hub, which is receiving a steady stream of data from the Azure Audit Logs.  The Function then hits a Slack WebHook to alert me that I need to take action.  Slack even gives us pretty colors to use in distinguishing Informational logs from Errors or Warnings.

![Colors](/content/images/2016/05/slack.png)

Enjoy!

(Oh and that was a ton of steps. Take the easy way and [download this ARM Template](https://github.com/stevenfollis/Samples/tree/master/AuditLogFunction) that deploys all the above infrastructure AND does a Web Deploy of the Function code. I won't judge.)
