---
layout:     post
title:      Angular, The Rails Way
date:       2016-01-26 12:32:18
summary:    Rails is famous for its 'convention over configuration' approach to web development. However, with the emergence of single page applications powered by front-end frameworks such as Reach and Angular, these conventions feel increasingly less defined.
categories: technical
---

Rails is famous for its 'convention over configuration' approach to web development. However, with the emergence of single page applications powered by front-end frameworks such as ReactJS and AngularJS, these conventions feel increasingly less defined.  

### Why AngularJS and not ReactJS

Before making this choice, I did my due diligence:

  - I started by working through multiple tutorials with each framework to gain a basic level of familiarity.  
  - I sat down with two big supporters of React, and two of Angular. I explained my project and we discussed the value each framework would add.  
  - I tore down an existing component of my project and rebuilt it using Angular, torn it down again and rebuilt it using React.  

I finally settled on Angular for the following reasons:  

  - I wanted a full MVC solution. React is the 'V', but I wanted the model and controller too.
  - Wasn't a big fan of JSX. The largest limiting factor I experience in my development process is the cost of switching between languages and frameworks. I simply didn't want another layer of abstraction, no matter how simple.  

### My code is a mess

My last six months could be very evenly split in two:  
  - July through October: Getting the 'basics' up and running. Installing Bootstrap, AngularJS, creating a layout I liked, setting up basic AngularJS services/controllers, creating a larger schematic for the parts of my application.
  - October through January: Living in Ruby, running data analysis in Ruby, building what I believe will be the "engine" of my application. 

When it was time to jump back into my Javascript/AngularJS code I was totally lost. I had forgotten what all the key concepts to Angular (modules, services, controllers, load order, scope, etc.) are used for and realized I needed to re-learn it all. Also, I learned my code was a total mess:

####HTML & CSS
  
  - Bootstrap is installed using the `bootstrap-sass` gem, and is thus not located in my `stylesheets` or `vendor` folders.
  - My `application.scss` has all this shit in it. Comments such as `"bootstrap-sprockets" must be imported before "bootstrap" and "bootstrap/variables"` are littered everywhere to prevent me from jumping back in and screwing things up later.
  - 