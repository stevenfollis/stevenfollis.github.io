---
layout: post
title: Node + Express + AzureAD + Yeoman with generator-express-azuread
date: 2017-06-23 09:45:10.000000000 -04:00
comments: true
---
Authorization, while always essential, is rarely fun. An Express app can be up and running in the Azure App Service in seconds, but any real-world application needs to be locked down to a particular set of intended users. In the Node world, authentication is handled via the excellent [PassportJS](http://passportjs.org/) middleware. The AzureAD team maintains an official Passport "strategy" called [passport-azure-ad](https://github.com/AzureAD/passport-azure-ad), making it easy to secure Node applications with Azure Active Directory.

> Need a quick 'n dirty way to secure your web app? Check out the ["EasyAuth"](https://docs.microsoft.com/en-us/azure/app-service/app-service-authentication-overview) feature of App Service for a no-code change way of standing up an authentication barrier in front of your app. Even works with social identify providers such as FB!

Even with Passport it can be difficult to retrofit an existing code base to support authentication. In the AzureAD space it can be particular tricky - are we needing the [v1 or v2](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-compare) endpoint? [OIDC or Bearer](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-flows)? What [permissions](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-scopes) does our app need? 

I routinely find myself piecing together chunks of code from GitHub samples, blog posts, and duct tape to get my app working as expected. To expedite such activities, I threw together a simple [Yeoman](http://yeoman.io/) generator for scaffolding out an Express 4 application with AzureAD baked right in. Currently it works for v1 and v2 endpoints with OIDC, but in the future I'd love to expand to support the Bearer strategy, and broader frameworks (Sails, Loopback, Restify, etc.)

![screenshot](https://github.com/stevenfollis/generator-express-azuread/raw/master/media/generator-express-azuread-demo.gif)

Let's walkthrough how to setup an app using [generator-express-azuread](https://github.com/stevenfollis/generator-express-azuread).

# Create an AzureAD Application
Head over to [https://apps.dev.microsoft.com](https://apps.dev.microsoft.com) and login.  You'll see two sections of application registrations:

* **Converged Applications** is the newer "v2" endpoint that support both Microsoft Accounts and Organizational Accounts. If you are building a new or multi-tenant application, this would be a good place to hangout.
* **AzureAD only applications** is the legacy "v1" endpoint that only support Organizational Accounts. 

> Check the [docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-compare) for a more thorough comparison

For this sample we'll use the AAD v2 endpoint. Click **Add an app** next to Converged Applications. 

![screenshot](/content/images/2017/07/2017-07-07-11_00_30-My-Applications.png)

Enter an application name and click the **Create** button.

![screenshot](/content/images/2017/07/2017-07-07-11_07_29-Register-App---Microsoft-Identity-Platform.png)

Make a note of the Application Id for later, then click **Generate Password** and note the generated password.  These two values will be used in our app during the authentication process.

![screenshot](/content/images/2017/07/2017-07-07-11_08_48-GeneratorSample-Registration.png)

Finally, AzureAD needs to know a white listed URL to forward tokens. Under the Platforms heading, click the **Add Platform** button. Select **Web** and in the **Redirect URLs** section, enter `http://localhost:3000/auth/openid/return`.  This URL corresponds to the route that our application has implemented to handle the return message from AzureAD. It can be anything, but by default the generator has implemented such logic at `/auth/openid/return` (following the lead of the AzureAD team's samples). Feel free to adjust to your heart's content. Be sure to hit the **Save** button at the bottom of the screen, then you're good to go.

![screenshot](/content/images/2017/07/2017-07-07-11_14_23-GeneratorSample-Registration.png)

> The section titled **Microsoft Graph Permissions** allows us to define which services and features our application can access. By default you can see `User.Read` but feel free to configure whichever scope(s) your app requires.

You should now have a ClientID and Password for the application. Next, we'll scaffold out the application.

# Create the application
To create the application we'll be using the ever-popular [Yeoman](http://yeoman.io/) tool. Not familiar with Yeoman? Think of it as `File -> New Project` in Visual Studio, for everything non-MSFT. Yeoman simply generates a folder structure and set of files to begin a new application. Check out [Getting Started](http://yeoman.io/learning/index.html) for setup help. You'll need NodeJS installed, and then run `npm install -g yo` to install Yeoman globally.

Once Yeoman is installed, install [generator-express-azuread](https://github.com/stevenfollis/generator-express-azuread) with `npm install -g generator-express-azuread`.

With Yeoman and the generator ready to roll, open a terminal window and create a new directory.  Then run `yo express-azuread`. 

* Give the application a name
* Select **Version 2** as the specified endpoint
* Paste in the Client ID and Secret
* The generator defaults to the `common` endpoint. This can be used for multi-tenant applications. Use this, or paste in the GUID for your tenant if you'd like to be more specific.
* The redirectUrl should match what was created in the app registration process, by default we used `http://localhost:3000/auth/openid/return`

![screenshot](/content/images/2017/07/2017-07-07-11_28_35-npm.png)

Yeoman then scaffolds out all the files necessary for your application, and runs an `npm install` to hydrate the app with all required dependencies.

# Run the application
From your directory, execute `npm start`.  The application will be running at `http://localhost:3000`. Open a browser and navigate to the URL.  

The home route is public, meaning it does not require authentication. Click the **Login** button in the top right corner to initiate the login process.

![screenshot](/content/images/2017/07/2017-07-07-11_33_44-GeneratorSample.png)

The application redirects you to AzureAD's login page, where you can login with a MSA or Organizational set of credentials.

![screenshot](/content/images/2017/07/2017-07-07-11_35_14-Sign-in-to-your-account.png)

After entering credentials, you will be asked to accept a series of permissions, granting your application rights as defined in the `User.Profile` scope set by default in Application Registration.

![screenshot](/content/images/2017/07/2017-07-07-13_13_17-Authorize-GeneratorSample.png)

Once signed in, AzureAD redirects you back to the application's `redirectUrl` route that we specified earlier. You should now see your name in the top right corner. Click the name to see a dropdown, and then select **Profile**.

![screenshot](/content/images/2017/07/2017-07-07-13_16_12-GeneratorSample.png)

This profile route is the 2nd of two screens implemented by the generator. It queries the [Microsoft Graph](https://developer.microsoft.com/en-us/graph/) for the signed in user, displaying all properties as a table.

![screenshot](/content/images/2017/07/2017-07-07-13_18_12-GeneratorSample.png)

Voilà! We now have an Express application with AzureAD scaffolded out and authenticating users against Azure Active Directory. 

# Wrap up
The generator is not intended as a kitchen sink simple, and has purposely implemented very limited logic. From this starting point the intent is to build out routes and application behavior as applicable for your scenario.  

Hope this saves a bit of time on your next web application project! And as always, please feel free to send pull requests and make the project even better. 

Thanks!

> Be sure to check out all the code snippets at [Azure.com](https://azure.microsoft.com/en-us/resources/samples/?service=active-directory) for more AzureAD goodness.
