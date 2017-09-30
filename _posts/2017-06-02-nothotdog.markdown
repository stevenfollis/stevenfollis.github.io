---
layout: post
title: '"Not Hotdog" from the Silicon Valley TV show with Cognitive Services + Bot
  Framework'
date: 2017-06-02 18:25:53.000000000 -04:00
---
I'm a huge fan of the television show [Silicon Valley](https://en.wikipedia.org/wiki/Silicon_Valley_(TV_series)) on HBO. It follows the lives of a startup company in California, but also is wicked funny. 

A few weeks ago I was watching S04E04 when a funny app was shown:

> Disclaimer: HBO is OK with potty mouths but potentially NSFW

<iframe width="560" height="315" src="https://www.youtube.com/embed/ACmydtFDTGs" frameborder="0" allowfullscreen></iframe>

Jin Yang's app gave me an idea: how hard would it be to implement this Not Hotdog app on Azure? Spoiler: pretty easy. 

![hotdog](/content/images/2017/06/hotdog.PNG)

Components used:

* [Bot Framework](https://dev.botframework.com/) was faster than building a true mobile application
* MSFT [Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/) has a Computer Vision function that takes an image and returns what's in it (hopefully pre-optimized for hotdog related imagery)

I started with a basic bot. The default conversation route `/` receives all incoming conversations.  If text is received, the bot returns a message instructing the user to upload an image. If an image attachment(s) is uploaded, then Cognitive Services is called to check on what's in the image.

Be sure to set an environment variable to your Cognitive Services key.  The ever-creative `COGNITIVE_SERVICES_KEY` variable is used by the bot to authenticate POST messages to the CS endpoint.

What if you send something other than a hotdog? The bot writes out into the card the description field returned by CS. 

![nothotdog](/content/images/2017/06/nothotdog.PNG)

Happy Hotdogging!

> Code is hanging out on [GitHub](https://github.com/stevenfollis/nothotdog)
