---
layout: post
title: Using Git from the command line with Visual Studio Online (VSO) and a Personal
  Access Token
date: 2015-11-16 10:35:35.000000000 -05:00
---
Once upon a time, using Visual Studio Online/TFS meant you were locked into using Team Foundation Version Control (TFVC) for all of your versioning needs. Nowadays you have a choice of centralized TFVC, or decentralized Git repositories (and no, [TFVC is not dead](http://blogs.msdn.com/b/bharry/archive/2014/04/14/is-microsoft-abandoning-tfvc-in-favor-of-git.aspx)).

For those who have taken the Git path, how about that [Visual Studio integration](https://www.visualstudio.com/en-us/get-started/code/share-your-code-in-git-vs)? Wraps all the command line fun in a nice, pretty UI that simplifies pulls and puts. However, you don't yet get a full 100% Git experience in the UI. For those ocassions, and for when you plain prefer using a command line, you can access VSO repositories via a variety of command lines. 

Sounds simple right? Fire up PowerShell/Terminal/CMD/xplat cli/et. al., hammer out a little `git pull`, login with a username and password, and you're good to go:
![Failed PowerShell login](/content/images/2015/11/Screenshot_111615_074736_AM.jpg)

"Authentication failed"? That's not good! What gives? My organization requires two factor authentication for user logins, and the command line has a history of not playing nicely (though [getting better](https://azure.microsoft.com/en-us/blog/azure-cli-supports-microsoft-account-logins/)).

In order to access my repo, I need to setup a "Personal Access Token" via VSO.  Fire up your browser, and from your project's homepage click your name, then "My Profile".
![VSO Homepage](/content/images/2015/11/Untitled_Clipping_111615_075811_AM.jpg)

That will take you to your Profile.  On the tab next to "Profile", click "Security" and then confirm the left hand nav is set to "Personal Access Tokens". 
![Security Pane](/content/images/2015/11/Untitled_Clipping_111615_080141_AM.jpg)

Here you can manage tokens. We'll create a new one for our command line by clicking the "Add" button. Now we're cooking! We can give out token a name, define an expiration date of 90/180/365 days, which account(s) the token applies, and set specific scopes.  I'm keeping things vanilla for mine but feel free to tighten up as needed.
![Token Creation Screen](/content/images/2015/11/Untitled_Clipping_111615_081153_AM.jpg)

Click "Generate Token" and congratulation! You're got your very own personal access token. Copy it down, and let's retry our command line `git pull` a second time:
![Command line success](/content/images/2015/11/Screenshot_111615_081714_AM.jpg)

We've now successfully used the command line with a personal access token to execute a pull request for code sitting in a Visual Studio Online (VSO) Git repository. You can see it even pullled down new files locally!  

With this PAT we're free to use the command line for all of our Git related needs.  Thanks!
