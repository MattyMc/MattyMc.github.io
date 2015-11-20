---
layout:     post
title:      Duplicating objects in Ruby
date:       2015-11-20 16:32:18
summary:    A brief look at the (sometimes) surprising effects of freezing simple and slightly more complex objects in Ruby, and the subsequent effects of manipulating the duplicate's properties.  
categories: technical
---

### Duplicating simple objects
Let's start off with a brief example:
{% highlight ruby %}
test
{% endhighlight %}

After digging through some of the gem's code on Github, I figured it was time to finally start digging into Bundler and see this machine works.

### Some handy terminal commands
These commands that I first found [here](http://www.sitepoint.com/how-to-customize-twitter-bootstraps-design-in-a-rails-app/) are super-useful for exploring gems: 

`bundle show [gemname]`  
Returns the location of your gem

cd \`bundle show [gemname]\`
Presumably executes the command inside the backtics, returning the directory of the gem and takes you there using the 'change directory' command. *Side note: I actually had to Google what the name of the "\`" symbol was. Turns out it's called the Grave Accent, and there's a whole [wikipedia article](http://en.wikipedia.org/wiki/Grave_accent) dedicated to it.*

