---
layout:     post
title:      Don't Wait for Rails 5 - Leverage WebSockets Now with Firebase + Rails
date:       2016-04-02 09:32:18
summary:    How to cook-up a realtime question and answer application in 8 minutes using Firebase and Rails. Based on a short talk I gave at Ruby Lightning Talks TO on April 5th, 2016.
categories: technical
---

Today we're going to build an application that has two parts. First, we'll have a page where an administrator can create a question - Yes/No, Multiple Choice - and see responses. Next, we'll have a page where respondents will be able to be prompted to answer. We'll also have a few nice features:  

  - When an instructor creates a new question, all users (respondents) will see the new question with no need to refresh the page.
  - We'll keep track of responses in realtime.
  - We can ask a new question without a refresh.

Lastly, we're going to easily implement these realtime features using Firebase, which implements a pub/sub, no-SQL object storage database we can interact with through their Javascript library.

### A Primer: Firebase

Firebase is a service that combines:

  - [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) for a fast (low-latency), bi-directional long running connection between a client and server.
  - A Publish/Subscribe (pub/sub) database.
  - No-SQL object storage.
  - A nice Javascript library (libraries are available [in many languages](https://www.firebase.com/docs/web/libraries/)).

Fly on over to [Firebase.com](https://www.firebase.com), register for free and spin up a new database instance. You'll receive a URL (my url: https://testdatabase12345.firebaseio.com/) and with the [Firebase Javascript API](https://www.firebase.com/docs/web/api/) lets start by instantiating our connect:

{% highlight javascript %}
var myFirebaseRef = new Firebase("https://testdatabase12345.firebaseio.com/");
{% endhighlight %}

Listening for changes and publishing data to our database is also easy.

##### Listening for changes
{% highlight javascript %}
myFirebaseRef.on("value", function(snapshot) {
  alert(snapshot.val());  // Alerts our new object on create
});
{% endhighlight %}

##### Push data to our database
{% highlight javascript %}
myFirebaseRef.set({
  city: "Toronto",
  team: "Blue Jays",
  captain: {
    firstName: "Jose",
    state: "Bautista"
  }
});
{% endhighlight %}

### Building our application

Lets start by spinning up a new instance of Rails, creating a controller with a few actions and checking our code into git:
{% highlight bash %}
rails new question-demo --database=postgresql
cd question-demo
bundle install
rails g controller Questions question answer

git init
git add .
git commit -m "initial commit"
{% endhighlight %}

Next, lets update our routes.rb to the following:
{% highlight base %}
# config/routes.rb
# Make our answer view default:
root 'questions#answer'
get 'question' => 'questions#question'
{% endhighlight %}

Finally, add the `rails_12factor` gem to the production group to help Heroku serve our static assets:  
{% highlight ruby %}
gem 'rails_12factor', group: :production
{% endhighlight %}

Perfect. We can fire up `rails server` and check that our routes are responding correctly.

### Including Firebase

Grab the latest version of Firebase from [Firebase's quickstart guide](https://www.firebase.com/docs/web/quickstart.html). To include it in our application:
{% highlight html %}
<!-- application.html.erb -->
<head>
  <script src="https://cdn.firebase.com/js/client/2.4.2/firebase.js"></script>
  ...
</head>
{% endhighlight %}

##### Bonus: Include Bootstrap

Totally unnecessary, but why not take the 30 seconds to make your app look a little more clean. Again, grab a link to the latest version from [bootstrapcdn.com](https://www.bootstrapcdn.com/), and in your application.html.erb:
{% highlight html %}
<!-- application.html.erb -->
  <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet">
  ...
<head>

<!-- throw 'yield' inside a container div -->
<div class="container">
  <%= yield %>
</div>
{% endhighlight %}

### Setup question view

Very simply, we'll add a form to allow a user to choose a question type, write a question and hit send:  
{% highlight html %}
