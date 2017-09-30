---
layout: post
title: Integrating Application Insights with AngularJS Apps via Angulartics
date: 2015-03-10 19:57:07.000000000 -04:00
---
Who is using your web application?  Where are they from?  What devices are they using?  These kinds of  questions have long been essential to understanding your audience and informing decisions with cold hard data.

Traditionally, we'd simply toss a block of Google Analytics into a page template or masterpage and voila! Instant analytics.  Each time a user navigated to a new page, we had pretty graphs and charts showing us behavior data and general attributes.

This changes in the Single Page Application (SPA) world.  Instead of the user requesting and the server serving multiple physical pages, only one page is ever transferred.  All subsequent "pages" are actually application "routes" showing different views.  How do we log behavior when there aren't multiple postbacks to and from the server?  

# Application Insights
"[Application Insights](http://azure.microsoft.com/en-us/services/application-insights)" is a service that has been a part of Visual Studio Online for a while, but recently moved into the [Azure Portal](http://porta.azure.com).  At first glance, it appears very similar to the Google Analytics features that have been used for years.  You get all the familiar data around page visits, sessions, unique users, browser stats, demographics, etc. included in pretty charts.  

However, the real power of App Insights is that it combines client side scripts with a server side component.  This server side component keeps tabs on what's happening under the hood of your app, alerting you to any shenanigans that impact app performance or availability.  In conjuction with the client side script it's a holistic way to track all sorts of facets of your app.  

Today's focus is on the client side, but couldn't resist plugging the rest of the App Insights platform.  Multiple backends are supported, including [NodeJS](https://github.com/Microsoft/AppInsights-node.js), [Ruby](https://github.com/Microsoft/AppInsights-Ruby), [PHP](https://github.com/Microsoft/AppInsights-PHP), [Java](http://azure.microsoft.com/en-us/documentation/articles/app-insights-java-get-started/), [Python](https://github.com/Microsoft/AppInsights-Python), and of course [.NET](http://azure.microsoft.com/en-us/documentation/articles/app-insights-start-monitoring-app-health-usage). 

<iframe src="//channel9.msdn.com/Shows/Edge/App-Insights-Performance-Monitoring/player" width="960" height="540" allowFullScreen frameBorder="0"></iframe>

# The Plan
* Setup a new Application Insights Instance via the Azure Portal
* Install the Apps Insight JavaScript file 
* Install Angulartics core script
* Install the Angulartics-Azure script
* Include both scripts into your App.js file
* Use Angulartics' special directives to log user event behavior

# Configuring App Insights
1. Fire up the [Azure Portal](http://portal.azure.com) and sign on in.  
![Azure Portal Home Page](/content/images/2015/04/Screenshot_040115_120111_PM.jpg)

2. In the bottom left hand corner, hit the "+" sign and select "Developer Services", then select "Application Insights".
![New Flyout](/content/images/2015/04/Screenshot_040115_120821_PM.jpg)

3. Fill out a name, resource group, and subscription, and click OK to finish creating the App Insights instance.

4. If you're not automatically taken to your instance's blade, hit the "Browse" button on the left hand nav bar and locate the name you just created.
![Browse Image](/content/images/2015/04/Screenshot_040115_021659_PM.jpg)

5. Inside this blade is a little cloud icon labeled "Quick Start". Selecting it opens up the Quick Start panel, where you can select "Get code to monitor my web pages."  This also opens up a new blade, where a script tag resides.  It's everything you need to get going, complete with a bow on top. 
![AI Blade](/content/images/2015/04/Screenshot_040115_022154_PM.jpg)

6.  Copy/paste this code right before the closing `</head>` tag on your Angular's HTML shell page.  The "secret sauce" is in the `instrumentationKey` variable, which wires you up to the correct instance of App Insights.  

	`<!--
	To collect end-user usage analytics about your application,
	insert the following script into each page you want to track.
	Place this code immediately before the closing </head> tag,
	and before any other scripts. Your first data will appear
	automatically in just a few seconds.
	-->
	`
    `<script type="text/javascript">
		var appInsights=window.appInsights||function(config){
    		function s(config){t[config]=function(){var i=arguments;t.queue.push(function(){t[config].apply(t,i)})}}var t={config:config},r=document,f=window,e="script",o=r.createElement(e),i,u;for(o.src=config.url||"//az416426.vo.msecnd.net/scripts/a/ai.0.js",r.getElementsByTagName(e)[0].parentNode.appendChild(o),t.cookie=r.cookie,t.queue=[],i=["Event","Exception","Metric","PageView","Trace"];i.length;)s("track"+i.pop());return config.disableExceptionTracking||(i="onerror",s("_"+i),u=f[i],f[i]=function(config,r,f,e,o){var s=u&&u(config,r,f,e,o);return s!==!0&&t["_"+i](config,r,f,e,o),s}),t
		}({
    		instrumentationKey:"00000000-0000-0000-0000-000000000000"
		});`
    
	`window.appInsights=appInsights;
	//appInsights.trackPageView(); /* Will be handled via Angulartics */
	</script>`

You're now broadcasting user data!  Hurray JavaScript.  Notice that near the end of the block, a line has been commented out.  This is because Angulartics will handle that bit here momentarily.

# Installing Angulartics
Now that we have App Insights, how do we get that into an AngularJS single page application?  We *could* roll up our sleeves and write a custom Angular directive to handle the integration, but chances are someone's already done that legwork.  

The [Angulartics](http://luisfarzati.github.io/angulartics/) project aims to be the missing piece for connecting SPA's with analytics providers.  It abstracts the logic necessary to record page views and user interactions.  

At the time of writing, 12 plugins have been made for Angulartics.  Unforunately, Application Insights is not one of them.  I saw this and set about forking the repo to add such functionality, when discovering [a pull request already existed for App Insights](https://github.com/luisfarzati/angulartics/pull/290).  Presumably this will soon be accepted and officially in the repo, but that doesn't stop us from getting functionality today! 

Angulartics' [Getting Started](http://luisfarzati.github.io/angulartics/#/getting-started) section outlines first steps.  Download the Angulartics base file, and include it in your app.js module injections.  The second step is less documented: copy the code from the pull request into a new "angulartics-azure.js" and also add "angulartics.azure" to your app.js file.

Your "angulartics-azure.js" should look like:

	/**
	 * @license Angulartics v0.17.2
	 * (c) 2013 Luis Farzati http://luisfarzati.github.io/angulartics
	 * Microsoft Azure Application Insights plugin contributed by https://github.com/anthonychu
	 * License: MIT
	 */
	(function (angular) {
		'use strict';

		/**
		 * @ngdoc overview
		 * @name angulartics.azure
		 * Enables analytics support for Microsoft Azure Application Insights (http://azure.microsoft.com/en-us/services/application-insights/)
		 */
		angular.module('angulartics.azure', ['angulartics'])
		.config(['$analyticsProvider', function ($analyticsProvider) {

    		$analyticsProvider.registerPageTrack(function (path) {
    			appInsights.trackPageView(path);
    		});

    		/**
			 * Numeric properties are sent as metric (measurement) properties.
			 * Everything else is sent as normal properties.
			 */
    		$analyticsProvider.registerEventTrack(function (eventName, eventProperties) {
    			var properties = {};
    			var measurements = {};

    			angular.forEach(eventProperties, function (value, key) {
    				if (isNumeric(value)) {
    					measurements[key] = parseFloat(value);
    				} else {
    					properties[key] = value;
    				}
    			});

    			appInsights.trackEvent(eventName, properties, measurements);
    		});

		}]);

		function isNumeric(n) {
			return !isNaN(parseFloat(n)) && isFinite(n);
		}
	})(angular);

Hopefully the pull request purgatory that this code is sitting in will be cleared up soon, and we can pull in the file in a more normal method. 

With Angulartics + Angulartics Azure up and running, we should now be receiving page view data back in App Insights.  Click around your app, goign through different routes and check in AI in ~5 minutes to start seeing data:
![AI Data Coming In](/content/images/2015/04/Screenshot_040115_050523_PM.jpg)

AI also shows us entry and exit pages, aka where users first start in our apps and what page they are on when they leave. 

# Configuring Events
We get routing page views out of the gate, but how about more custom events?  Say we have a navigation bar at the top of our page:

	<ul class="nav navbar-nav">
		<li><a href="#work">Work</a></li>
		<li><a href="#capabilties">Capabilities</a></li>
		<li><a href="#about-us">About Us</a></li>
		<li><a href="#contact">Contact</a></li>
	</ul>

Nothing fancy, just a Bootstrap 3 nav component.  We want to track when users click navigation items, to tell where on the page users spend their time.  Angulartics gives us two directives to add:

`analytics-on="click"` describes the type of event.  Typically we'll be looking for user clicks.

`analytics-event="EventName"` gives a name to the event for tracking purposes.  This string is how we'll distinguish events from each other, so make sure it's easy to read.  I like to use a period to separate types of events into categories, for example "Navigation.Work" would describe the event when the user clicks our first link.

Updating our navbar example with these directives gives us:

	<ul class="nav navbar-nav">
		<li><a href="#work" analytics-on="click" analytics-event="Navigation.Work">Work</a></li>
		<li><a href="#capabilties" analytics-on="click" analytics-event="Navigation.Capabilities">Capabilities</a></li>
		<li><a href="#about-ux" analytics-on="click" analytics-event="Navigation.About">About UX</a></li>
		<li><a href="#contact" analytics-on="click" analytics-event="Navigation.Contact">Contact</a></li>
	</ul>

Now, anytime a user clicks a navigation link we can capture the event name.  Looking at the data, perhaps we notice that the About section is getting significantly more traffic than the other pages.  We could then potentially move it up on the page, or refine the content to make sure visitors get the information that they need.

Back in App Insights, we can already see data trickling in!
![Event Data](/content/images/2015/04/Screenshot_040115_052137_PM.jpg)

Be sure to check out [Angulartics' documentation on Event tracking](http://luisfarzati.github.io/angulartics/#/event-tracking) for further details.  

# Conclusion
We've taken an AngularJS application and hooked in Application Insights via the Angulartics plugin.  We can now more accurately track and understand visitors to our application and make better decisions on enhancements, updates, etc.

Hope it's been helpful!

Thanks!
Steven
