---
layout:     post
title:      OmniAuth - Looking under the hood
date:       2015-01-02 14:30:00
summary:    In Rails, development logs are super important. However, authenticating a user via an external identification provider requires your server to send and receive HTTP requests that are not typically expressed in your log files. This post demonstrates how to log OmniAuth's request cycle to see the whole process in action.
categories: technical
---

I've written lately about the OAuth2.0 process and implementing it in Rails via OmniAuth. However, to fully understand what takes place on your server it helps to see the entire process in action.

In this post, I'll show you how to override the default behaviour of Figaro, the gem OmniAuth uses for its HTTP requests, in order to see logs of the entire process in action. 

*This post is largely motivated by [this question](https://stackoverflow.com/questions/11489397/what-mechanism-do-omniauth-provide-to-ensure-a-secure-login/) on [StackOverflow](http://www.stackoverflow.com) and my subsequent [response](http://stackoverflow.com/a/27747318/1038034).*

### Question

> Using an OmniAuth login strategy, a non-logged in user is redirected to an identity provider. The identity provider will ensure a user is logged and then redirect the user to a callback url allowing the user to login to the third party site, using the identity provider's authentication.  
> How is it assured that a malicious user does not spoof this callback so that he can gain access to the authenticating user's third party account?  
[source](https://stackoverflow.com/questions/11489397/what-mechanism-do-omniauth-provide-to-ensure-a-secure-login/)

### Answer

Our application takes this Authorization Code from the user, combines it with our Secret Code that we received upon registering our application and exchanges these with the identity provider for an almighty Access Code. Specifically, here are the steps taken to ensure this callback hasn't been spoofed:

  1. The entire process is managed through SSL (ie HTTPS). This makes intercepting any of these codes very difficult. 
  2. Your identity provider validates the identity of the user by supplying the Authorization Code upon successful login and validating this code when received from your application. 
  3. Your identity provider ensures the identity of your application by verifying your application's Secret Code.

Thus, it would be very difficult to intercept/decrypt a code and if you tried to spoof the redirect with a fake Authorization Code your login request would be rejected when your server tries to validate this code with your identification provider. 

### See it in action

OmniAuth manages most of this process behind the scenes. However, the following code should allow you to see everything in your `log/development.log` file:

{% highlight ruby %}
# Add the following gem to your config/routes.rb
gem 'httplog', group: :development
{% endhighlight %}

We'll make use of the httplog gem. This will essentially dump request logs into our log/development.log file. We need to initialize:

{% highlight ruby %}
# Create a new file: config/initializers/httplog.rb
HttpLog.options[:logger] = Rails.logger if Rails.env.development?
{% endhighlight %}

Now launch a new Rails server for you application in the terminal:

{% highlight bash %}
bundle install
rails s
{% endhighlight %}

In a new tab, tail your development logs:

{% highlight bash %}
tail -f log/development.log
{% endhighlight %}

Go ahead and open a browser and login to your application with your chosen identity provider. Open the terminal window tailing your development logs and after the callback request from the user: 

{% highlight bash %}
Started GET "/auth/google_oauth2/callback?state=1 
{% endhighlight %}

You should see something like:

{% highlight bash %}
[httplog] Connecting: accounts.google.com:443
[httplog] Sending: POST http://accounts.google.com:443/o/oauth2/token
[httplog] Data: client_id=123412341234-1234h1234h1234h1234h.apps.googleusercontent.com&client_secret=12341234123412341234&code=123412341234123412341234&grant_type=authorization_code&redirect_uri=https%3A%2F%2Fyourapp.domain.com%2Fauth%2Fgoogle_oauth2%2Fcallback
....
[httplog] Response:
{
  "access_token" : "123412341234123412341234",
  "token_type" : "Bearer",
  "expires_in" : 3599,
  ...
}
{% endhighlight %}


This is your server verifying the Authorization Code and verifying the Access Token. Next, OmniAuth uses this token to get some user information. Further down you should also see another request:

{% highlight bash %}
[httplog] Connecting: www.googleapis.com:443
[httplog] Sending: GET http://www.googleapis.com:443/plus/v1/people/me/openIdConnect
[httplog] Status: 200
[httplog] Response:
{
  "kind": "plus#personOpenIdConnect",
  "gender": "male",
  "sub": "1234123412341234",
  "name": "Mattyc",
  "given_name": "Matt",
  ...  
}
{% endhighlight%}

This represents OmniAuth fetching the information you requested using the Access Token.

A long explanation, but I hope it helps. Be sure to disable or remove the httplog gem before pushing to production!
