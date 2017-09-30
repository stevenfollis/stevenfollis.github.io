---
layout: post
title: Using CSS Regions in Visual Studio
date: 2015-04-23 07:54:13.000000000 -04:00
---
Short tip today regarding CSS in Visual Studio that is far from universally known.  Many folks use comments to keep sections of CSS code organized:

![Regular CSS Comments](/content/images/2015/04/Screenshot_042315_080415_AM.jpg)

But our file is still super long.  If we want to adjust our footer, we have to scroll through numerous sections that we don't care about. 

This is where CSS Regions can help out!  Let's amend the comments a bit, changing `/*` over to `/*#region` and at the end of a block adding a `/*#endregion*/`:

![CSS with Regions Added](/content/images/2015/04/Screenshot_042315_080807_AM.jpg)

Those pink arrows are pointing out some new expand/collapse icons.  We can now easily collapse chunks of our code, allowing us to easily pinpoint where on the page the footer specific styles are located:

![Collapsed code](/content/images/2015/04/Screenshot_042315_081126_AM.jpg)

This is an extremely simple example, but on a long CSS file with 10+ sections of code using regions can save loads of time and headache.  Also for new team members it's great to open up a CSS file to nicely collapsed sections, allowing them to at a glance ramp up on the different chunk of an application. 

Prefer keyboard shortcuts?  CSS Regions has you covered!  Expand or Collapse All with `CTRL` + `M` + `L`.  For an individual section, place your cursor in that section and use `CTRL` + `M` + `M`.

Very simple technique, but one that makes working with CSS in Visual Studio even easier.  Thanks!
