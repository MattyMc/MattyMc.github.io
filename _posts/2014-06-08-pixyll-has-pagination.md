---
layout:     post
title:      A minimalist's tutorial on Rails and OmniAuth, powered by TDD
date:       2014-12-30 11:21:29
summary:    A step-by-step tutorial on building an authentication system using Rails, OmniAuth and Google's OAuth 2.0 (easily swappable for Facebook, Twitter or Github's APIs). We're going to setup a nice development environment, follow test-driven development best practices, and end up with a barebones app you can build on.
categories: technical
---

### What this is
This is a step-by-step, test-driven tutorial for web developers in creating a Rails application that will allow users to sign-in with their Google Accounts. Here’s what you’ll end up with:

  * A Rails application that allows users to login/out using their Google credentials
Test coverage somewhere between mediocre and good (would love suggestions in the comments section)
  * Version control using GIT
  * Your application up and running on Heroku
  * All your secret keys/passwords in your app kept secret (so you can push to GitHub, if desired)

### Motivation

I spent the better part of two days implementing a **Sign-in with Google** button. I'm not proud it took me this long, but I really wanted to do it *well*. I'd implemented authentication systems previously with help from gems like [Devise](https://github.com/plataformatec/devise), but admitingly there were a few times where I'd be left scratchy my head from Devise's magic and I definitely let the magician do his thing, if you know what I mean. Not this time. I wanted to have a full understanding of the entire process, which for me meant:

  * Test-Driven (unit tests and functional tests).
  * No Devise. Users will login with their Google Account.
  * A great dev environment. I don't want to be creating users in the console.
  * Minimalist. One model, one controller, one view and three routes.

I'll try to bring together all the [tutorials](https://www.twilio.com/blog/2014/09/gmail-api-oauth-rails.html), [GitHub docs](https://github.com/intridea/omniauth), [RailsCasts](railscasts.com/episodes/235-omniauth-part-1) and [StackOverflow Q&A's](http://stackoverflow.com/questions/10737200/how-to-rescue-omniauthstrategiesoauth2callbackerror) that I used, throw in some TDD and provide explanations where required. You'll end up with a Rails 4 application that will let users register, login and logout - absoultely nothing else. 

Let's get started!

### Make sure you have these

For development, I'll be using: 

  * Ruby 2.1.5 
  * Rails 4.2.0 
  * Git 2.2.1 

You can check your versions in the terminal with: 

{% highlight bash %}
$ ruby -v

$ rails -v

$ git --version
{% endhighlight %}

You'll also want to make sure you've installed and configured the [Heroku Toolbelt](https://toolbelt.heroku.com/) and [PostgreSQL](http://www.postgresql.org/) which we'll use as our database system. You should also have [Rubygems](https://rubygems.org/), [Bundler](bundler.io/) and [RVM](https://rvm.io/) installed.

On a side note, I'm developing using with [Sublime](www.sublimetext.com/) as my editor and strongly prefer [iTerm2](iterm2.com/) over the Mac OX terminal. 

### Let's build!

Let's get going with a new Rails application, using postgres as our database:
*replace 'googlelogin' with your desired application name*

{% highlight bash %}
$ rails new googlelogin --database=postgresql
{% endhighlight %}


Now that we've got our application built, navigate into the new application directory and let's open our Gemfile. 
























