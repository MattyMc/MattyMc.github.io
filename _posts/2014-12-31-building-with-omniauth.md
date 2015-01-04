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

### The Plan

Allowing users to register and login to your application using their Google ID requires your application to have a conversation with Google. Greg Baugues at [Twilio](https://www.twilio.com/blog/2014/09/gmail-api-oauth-rails.html) explains this process from the developers perspective:

  > 1. You set up an app within Google’s developer portal and tell it what APIs you’d like to use and Google gives you a Client ID and Client Secret (basically, a username and password).
  > 2. When a user signs into your app, your app redirects them to Google and passes along the client id/secret.
  > 3. Google asks the user permission to access their GMail data
  > 4. Google calls back to your app and sends a code which you can use, along with your credentials, to retrieve an access token with which you can make authenticated API calls.

*For a more in-depth explanation on why this process takes place, see my previous post: [Goodfellas, and a social explanation of 3rd Party Verification]({% post_url 2014-06-11-open-authentication %})*

Luckily, the OmniAuth gem will manage most of this entire procedure for us. We're going to hit a few more bumps in the road as we go, but we'll see that they're pretty easy to get through. 

Let's get started!

### Create your app!

Let's get going with a new Rails application using postgres as our database:
*replace 'googlelogin' with your desired application name*

{% highlight bash %}
$ rails new googlelogin --database=postgresql
$ cd googlelogin
{% endhighlight %}

We might as well initialize our repository as well:
{% highlight bash %}
$ git init
$ git add .
$ git commit -m "initial commit"
{% endhighlight %}

We're going to make a few modifcations here to get things working with OmniAuth, Google and Heroku. Let's go ahead and do this.

Add the following to your `Gemfile`:
{% highlight ruby %}
# Required by Heroku. This ensures we'll always be using this version of Ruby
ruby "2.1.5"  

# Heroku uses this gem for asset serving (among other things)
gem 'rails_12factor', group: :production 

# Help us manage our user authenication with Google
gem 'omniauth', '~> 1.2.2'   
gem 'omniauth-google-oauth2'

# Helps us parse JSON responses
gem 'json'

# Useful for testing model validations
gem 'shoulda', group: :test

# Creates a private config/application.yml file
gem 'figaro'
{% endhighlight %}

Now we should be almost ready to deploy to Heroku. We'll need to install our gems first, and commit our changes to Git:

{% highlight bash %}
$ bundle install
$ git add .
$ git commit -m "almost ready for heroku"
{% endhighlight %}

Next, let's push to Heroku:

{% highlight bash %}
$ heroku login
$ heroku create
$ git push heroku master
{% endhighlight %}

If we were to open our production application by entering `$ heroku open` in the terminal we'll receive an error since we:

  * Have no routes or views defined in our application
  * Have not yet migrated a database to Heroku

Let's fix this by adding a Sessions controller with an index action and afterwards migrate our database up to Heroku:

{% highlight bash %}
$ rails g controller Sessions index
{% endhighlight %}

In your `config/routes.rb' file, add the following route on line 2:

{% highlight ruby %}
root to:'sessions#index'
{% endhighlight %}

That should do it. Let's commit our changes and migrate our database:
{% highlight bash %}
$ git add .
$ git commit -m "ready for heroku"
$ rake db:create db:migrate
$ heroku run rake db:migrate
$ git push heroku master
$ heroku open
{% endhighlight %}

Hopefully you see the `Sessions#index` message from our `app/views/sessions/index.html.erb` file. First step is complete! 

### Building the User model

Before we build out our User model, what information exactly will we have access to? Looking at the `omniauth-google-oauth2` documentation shows exactly the type of JSON response we can expect from Google:
{% highlight json %}
{
    :provider => "google_oauth2",
    :uid => "123456789",
    :info => {
        :name => "John Doe",
        :email => "john@company_name.com",
        :first_name => "John",
        :last_name => "Doe",
        :image => "https://lh3.googleusercontent.com/url/photo.jpg"
    },
    :credentials => {
        :token => "token",
        :refresh_token => "another_token",
        :expires_at => 1354920555,
        :expires => true
    },
    :extra => {
        :raw_info => {
            :sub => "123456789",
            :email => "user@domain.example.com",
            :email_verified => true,
            :name => "John Doe",
            :given_name => "John",
            :family_name => "Doe",
            :profile => "https://plus.google.com/123456789",
            :picture => "https://lh3.googleusercontent.com/url/photo.jpg",
            :gender => "male",
            :birthday => "0000-06-25",
            :locale => "en",
            :hd => "company_name.com"
        }
    }
}
{% endhighlight %}

There are a few things we're most likely going to want to pull from this response. Specifically:

  * The user ID (:uid)
  * The user's :first_name, :last_name, :email, :image
  * The :token, :refresh_token, :expires_at attributes

We'll leave the rest for now, let's go ahead and generate a new User model with these attributes and migrate the database:

{% highlight bash %}
$ rails g model User uid:string:uniq name:string first_name:string last_name:string email:string:uniq image:string token:string refresh_token:string expires_at:datetime
{% endhighlight %}

You may notice above that for the `uid` and `email` attributes we're enforcing a uniqueness constraint in our database by using `uid:string:uniq` instead of simply `uid:string`. This effectively adds the following to our migration file for us:
{% highlight ruby %}
# db/migrate/somenumber_create_users.rb
add_index :users, :uid, unique: true
add_index :users, :email, unique: true
{% endhighlight %}

Creating uniqueness validations in our model is not enough to ensure uniqueness in production. Derek Prior's provides an excellent explanation in his article, [The Perils of Uniqueness Validations](http://robots.thoughtbot.com/the-perils-of-uniqueness-validations), should you so desire. Unfortunately, this will create a new problem for us. When we created the User model, Rails automatically generated a few fixtures for us to use when we run our tests. You can see them in `test/fixtures/user.yml`:

{% highlight ruby %}
# test/fixtures/user.yml

one:
  uid: MyString
  name: MyString
  first_name: MyString
  last_name: MyString
  email: MyString
  image: MyString
  token: MyString
  refresh_token: MyString
  expires_at: 2014-12-31 10:00:25

two:
  uid: MyString
  name: MyString
  first_name: MyString
  last_name: MyString
  email: MyString
  image: MyString
  token: MyString
  refresh_token: MyString
  expires_at: 2014-12-31 10:00:25
{% endhighlight %}

Beyond this tutorial this code will come in handy. However, you may notice that both fixtures are using `MyString` for both their `uid` and `email` attributes. Since we just enforced a unique constraint on our database, these fixtures will throw an error on every test. For now, lets comment out these fixtures by changing our `test/fixtures/user.yml` to the following:

{% highlight ruby %}
# test/fixtures/user.yml

# one:
#   uid: MyString
#   name: MyString
#   first_name: MyString
#   last_name: MyString
#   email: MyString
#   image: MyString
#   token: MyString
#   refresh_token: MyString
#   expires_at: 2014-12-31 10:00:25

# two:
#   uid: MyString
#   name: MyString
#   first_name: MyString
#   last_name: MyString
#   email: MyString
#   image: MyString
#   token: MyString
#   refresh_token: MyString
#   expires_at: 2014-12-31 10:00:25
{% endhighlight %}

Now we're ready to migrate our database. In our terminal:
{% highlight bash %}
$ rake db:migrate
{% endhighlight %}

#### Validations

After generating our User model, Rails has kindly generated a test file that we'll now make use of. Open up `test/models/user_test.rb` and making use of the Shoulda gem we'll create some failing tests. Add the following at the top of your `UserTest` class:
{% highlight ruby %}
# test/models/user_test.rb

should validate_presence_of :uid
should validate_presence_of :name
should validate_presence_of :first_name
should validate_presence_of :last_name
should validate_presence_of :email
should validate_presence_of :image
should validate_presence_of :token
should validate_presence_of :expires_at
validate_uniqueness_of :uid
validate_uniqueness_of :email
{% endhighlight %}

Running `rake test` or simply `rake` should render your tests many failures. Let's add some validations to our User model to help these pass.

{% highlight ruby %}
# app/models/user.rb

validates :uid, :name, :first_name, :last_name, :email, :token, :expires_at, :image, presence: true
validates :email, :uid, uniqueness: { case_sensitive: false }
{% endhighlight %}

Run our tests again (`rake`) and everything should be passing. Excellent. 

#### User.find_or_create_from_google_callback

Now that we have some validations around our User model, let's build a class helper method so that we can easily find or create a new User from the response we receive from Google's servers. But first, our test (I'll be combining two steps here):

{% highlight ruby %}
# test/models/user_test.rb

# Make sure the User class responds_to our new method
test "User should respond to find_or_create_from_google_callback method" do
  assert User.respond_to?( :find_or_create_from_google_callback), "User.create_from_google_callback should exist" 
end

# We'll be looking up users by their uid's. Lets make sure our app
# behaves appropriately here. Making use of the response Hash:
test "should create a new user from Google's callback params" do
  google_response = {
      :provider => "google_oauth2",
      :uid => "123456789",
      :info => {
          :name => "John Doe",
          :email => "john@company_name.com",
          :first_name => "John",
          :last_name => "Doe",
          :image => "https://lh3.googleusercontent.com/url/photo.jpg"
      },
      :credentials => {
          :token => "token",
          :refresh_token => "another_token",
          :expires_at => 1354920555,
          :expires => true
      },
      :extra => {
          :raw_info => {
              :sub => "123456789",
              :email => "user@domain.example.com",
              :email_verified => true,
              :name => "John Doe",
              :given_name => "John",
              :family_name => "Doe",
              :profile => "https://plus.google.com/123456789",
              :picture => "https://lh3.googleusercontent.com/url/photo.jpg",
              :gender => "male",
              :birthday => "0000-06-25",
              :locale => "en",
              :hd => "company_name.com"
          }
      }
  }
  user_count = User.count

  user = User.find_or_create_from_google_callback google_response
  # Have we created a new record?
  assert_equal (user_count+1), User.count
  assert_equal User, user.class
  assert_equal User.find_by(uid: "123456789"), user
end  

test "should find and return an existing user from Google's callback params" do
  google_response = {
      :provider => "google_oauth2",
      :uid => "123456789",
      :info => {
          :name => "John Doe",
          :email => "john@company_name.com",
          :first_name => "John",
          :last_name => "Doe",
          :image => "https://lh3.googleusercontent.com/url/photo.jpg"
      },
      :credentials => {
          :token => "token",
          :refresh_token => "another_token",
          :expires_at => 1354920555,
          :expires => true
      },
      :extra => {
          :raw_info => {
              :sub => "123456789",
              :email => "user@domain.example.com",
              :email_verified => true,
              :name => "John Doe",
              :given_name => "John",
              :family_name => "Doe",
              :profile => "https://plus.google.com/123456789",
              :picture => "https://lh3.googleusercontent.com/url/photo.jpg",
              :gender => "male",
              :birthday => "0000-06-25",
              :locale => "en",
              :hd => "company_name.com"
          }
      }
  }
  User.find_or_create_from_google_callback google_response
  user_count = User.count

  user = User.find_or_create_from_google_callback google_response
  # Have we created a new record?
  refute_nil user
  assert_equal user_count, User.count
  assert_equal User, user.class
end  
{% endhighlight %}

Running these new tests should yield a failure and a few errors, since we haven't yet created the desired method. Let's do that now. Add the following to your User class:

{% highlight ruby %}
# app/models/user.rb

def self.find_or_create_from_google_callback google_response
end
{% endhighlight %}

We should still have those two failures since our method is not yet creating or finding any users. Let's address that now by updating the User method we created above:
{% highlight ruby %}
# app/models/user.rb

def self.find_or_create_from_google_callback google_response
  # Lets extract the parameters we're interested in
  user_params = google_response[:info].merge google_response[:credentials].except(:expires)
  user_params['uid'] = google_response[:uid]

  # Convert expires_at to a Datetime 
  user_params['expires_at'] = Time.at(user_params['expires_at'].to_i).to_datetime

  User.create! user_params
end
{% endhighlight %}

Only one more test needed to pass. We'll need to add some functionality to return a user if they have previously registered:

{% highlight ruby %}
# app/models/user.rb

def self.find_or_create_from_google_callback google_response
  # Check if a User exists
  user = User.find_by uid:google_response[:uid]
  return user unless user.nil?

  # Lets extract the parameters we're interested in
  user_params = google_response[:info].merge google_response[:credentials].except(:expires)
  user_params['uid'] = google_response[:uid]

  # Convert expires_at to a Datetime 
  user_params['expires_at'] = Time.at(user_params['expires_at'].to_i).to_datetime

  User.create! user_params
end
{% endhighlight %}

That should have all of our tests passing. Quickly taking stock, we now have:

  * A User model defined with appropriate validations
  * A nice class method for finding or initializing new Users
  * Reasonable test coverage

Things should start moving quicly now!

### Register your app with Google
