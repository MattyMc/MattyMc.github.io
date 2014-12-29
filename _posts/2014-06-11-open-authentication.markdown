---
layout:     post
title:      Goodfellas, and a social explanation of 3rd Party Verification 
date:       2014-12-27 5:30:00
summary:    Learning how 3rd Party Verification works (ie what the <b>Sign-in with Google</b> button does behind the scenes) should not have been as difficult as I found it to be. This post explains <i>why</i> OAuth works the way it does for those new to the idea. 
categories: technical
---

Imagine for a moment that you’re running one of those high stakes, 1950s mobster-style backroom card games - such as the one from the movie Goodfellas (you’re Ray Liotta in this analogy). Your good friend, think Joe Peschi, brings with him a new guy to the game and although you want to grow the number of players you’re a little weary of new people. How would you know this New Guy is “cool”?

Well, if you have both your buddy Joe and the New Guy standing in front of you at the same time, this problem is easily solved. “Hey Joe, is this the New Guy? Is he cool?” Joe: “Ya, I know him. He’s cool”. Problem solved. 

To properly frame the problem that is 3rd Party Verification, now imagine that the new guy shows up *without* Joe - the only person who knows New Guy. How would the conversation go? Perhaps something like:  
New Guy: “I’d like to come join the game. Don’t worry, I’m cool. I know Joe.”  
You: “How do I know that you know Joe?”  
New Guy: “Well just give him a call and ask him if he knows me.”

We give our good friend Joe a call:
“Ya, I definitely know New Guy. But how do I know that the person standing in front of you is actually New Guy?” 

### Road bump #1

Hmm. I know that Joe knows New Guy, but since Joe can’t see him we have no way of knowing whether the person in front of me is a pretender. So what do we do? Let’s send New Guy to Joe.

So we do just that. New Guy leaves, goes to see Joe and comes back. So I call Joe and say “Joe, did New Guy come to see you?” Joe says “Yes.” And here’s where our next problem comes into play: How do I know my New Guy is the same guy Joe just saw? 

### Road bump #2

Well, to solve this let’s have Joe give new guy a secret passphrase before sending him back. This way, Joe can verify New Guy's identify, give him a passphrase, and when New Guy returns we could call Joe ourselves and verify the passphrase (effectively verifying that the person Joe verified is the same person we're talking to).

So, here’s how the whole thing could work:
  
  + New Guy arrives and tells me he’s cool because he knows Joe. I send New Guy over to see Joe.
  + New Guy arrives at Joe’s. Joe verifies that New Guy is indeed the person he knows and gives New Guy a passphrase, *Miss Lippy’s Car is Green* for example, and sends him back to me.
  + When New Guy gets back, I give Joe a call: “Hey Joe, it’s me. Did you give New Guy the passphrase *Miss Lippy’s Car is Green*?” Joe verifies he did.

Whew. Must be tough to have been a mobster in 1950. 

### Beyond the analogy

Although described through analogy, this problem is nearly identical to that which an OAuth 2.0 authentication process between your application, a user and a 3rd party service such as Google attempts to solve. Namely:

  - Your application needs to verify a user
  - Google knows the user, and can verify the user
  - You need to verify with Google that the person visiting your application is the same user Google knows

Moreover, the process is nearly the same:

  - You register your application with Google - this is the internet’s way of becoming “friends”. Google gives you a secret code so that it always knows it’s you and you exchange phone numbers (in this case redirect URIs). 
  - When a user wants to login to your application, you redirect him/her over to Google. Once they login with their Google login/password, Google redirects the user back to you using the phone number you gave Google and with a secret pass code (an Authorization Code in web-speak).
  - Once the user comes back to you, you give Google a call. Just to be sure everyone’s legit, you give Google back the Authorization Code, this verifies the user, and you tell Google the Secret Code you both made up when you became BFFs back in Step 1. 
  - Provided everything’s cool, Google sends you back an Access Code that will allow your application to gain access to some of this user’s information, assuming they gave you permission. 

### Final thoughts

The entire process of 3rd party verification was a little overwhelming for me at first. Most explanations I read introduced many new terms (Authorization Codes, Access Tokens, Refresh Tokens, Request Codes, etc), many new roles (Resource Server, Client, Resource Owner, etc), and combined with the technical implementation details I struggled to articulate the problem actually being solved, and as a result I struggled reason my way through it. 

My hope is that by abstracting these details you find the central problem solved by OAuth 2.0 more easy to define:  How do we know if a user is who they say they are, if being vouched for by a 3rd party?

Through the understanding of this central problem and the challenges that arise in it’s implementation I believe you will find learning the terminology and technical implementation of OAuth 2.0 much easier. I certainly did. 

If you’re done talking about mobsters and ready to dive in and start building, here’re a few places to get started:

  + [OAuth 2.0 Simplified, by Rohit Ghatol](http://www.slideshare.net/rohitsghatol/oauth-20-simplified) - A nice, succinct explanation of OAuth 2.0
  + [Using OAuth 2.0 to Access Google APIs](https://developers.google.com/accounts/docs/OAuth2) - An detailed explanation of using OAuth 2.0 with Google 
  + [OmniAuth](https://github.com/intridea/omniauth) - A great gem for implementing OAuth 2.0 in Rails