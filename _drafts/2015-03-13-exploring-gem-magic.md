---
layout:     post
title:      Exploring Ruby Gems, Bundler
date:       2015-01-13 16:32:18
summary:    There is no shortage of opinions on how to implement Twitter's Bootstrap in a Rails application. After installing the bootstrap-rails gem and being unable to find any of the SASS/SCSS files, I figured it was time to go exploring.
categories: technical
---

### Where the hell...?
I must have flipped two or three times between using [this approach]() which recommended copying the minified CSS files directly into my asset pipeline, and using the recommended [bootstrap-rails gem](https://github.com/twbs/bootstrap-sass) before finally settling on the gem approach.  
As it turns out, I have quite a bit of learning to do on how Bundler works its magic. After installing the gem, I was left wondering:
  * Where the hell are all my SASS files? 
  * Why are they not in the vendor or asset directories? 
  * How the hell do I configure this thing?  

After digging through some of the gem's code on Github, I figured it was time to finally start digging into Bundler and see this machine works.

### Some handy terminal commands
These commands that I first found [here](http://www.sitepoint.com/how-to-customize-twitter-bootstraps-design-in-a-rails-app/) are super-useful for exploring gems: 

`bundle show [gemname]`  
Returns the location of your gem

cd \`bundle show [gemname]\`
Presumably executes the command inside the backtics, returning the directory of the gem and takes you there using the 'change directory' command. *Side note: I actually had to Google what the name of the "\`" symbol was. Turns out it's called the Grave Accent, and there's a whole [wikipedia article](http://en.wikipedia.org/wiki/Grave_accent) dedicated to it.*

