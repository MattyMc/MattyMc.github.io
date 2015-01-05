---
layout:     post
title:      A minimalist's tutorial on Rails and OmniAuth, powered by TDD
date:       2015-01-05 11:21:29
summary:    A step-by-step tutorial on building an authentication system using Rails, OmniAuth and Google's OAuth 2.0 (easily swappable for Facebook, Twitter or Github's APIs). We're going to setup a nice development environment, follow test-driven development best practices, and end up with a barebones app you can build on.
categories: technical
---

### Introduction
This is a step-by-step, test-driven tutorial for web developers in creating a Rails application that will allow users to sign-in with their Google Accounts. Here’s what you’ll end up with:

  * A Rails application that allows users to login/out using their Google credentials
  * Test coverage somewhere between mediocre and good (would love suggestions in the comments section)
  * Version control using GIT
  * Your application up and running on Heroku
  * All your secret keys/passwords in your app kept secret (so you can push to GitHub, if desired)

### Motivation

I spent the better part of two days implementing a **Sign-in with Google** button. I'm not proud it took me this long, but I really wanted to do it *well*. I'd implemented authentication systems previously with help from gems like [Devise](https://github.com/plataformatec/devise), but admittingly there were a few times where I'd be left scratching my head from Devise's magic and I definitely let the magician do his thing, if you know what I mean. Not this time. I wanted to have a full understanding of the entire process, which for me meant:

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

Before we build out our User model, what information exactly will we have access to? Looking at the `omniauth-google-oauth2` [documentation](https://github.com/zquestz/omniauth-google-oauth2) shows exactly the type of JSON response we can expect from Google:
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

Fixtures can come in very handy and we'll be using them a little later. However, you may notice that both fixtures are using the value `MyString` for both their `uid` and `email` attributes. Since we just enforced a unique constraint on our database, these fixtures will throw an error on every test. For now, lets comment out these fixtures by changing our `test/fixtures/user.yml` to the following:

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

After generating our User model, Rails has kindly generated a test file that we'll now make use of. Open up `test/models/user_test.rb` and making use of the [Shoulda gem](https://github.com/thoughtbot/shoulda) we'll create some failing tests. Add the following at the top of your `UserTest` class:
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

Running `rake test` (or simply `rake`) should render many failures. Let's add some validations to our User model to help these pass.

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

Running these new tests should yield a failure and a few errors since we haven't yet created the desired method. Let's do that now. Add the following to your User class:

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

### Google and ngrok

Before we get going with Google and ngrok you're going to need your Heroku URL. You can find ths by entering `heroku open` in the terminal and copying the URL from your browser's address bar. It should look something like: https://**guarded-coast-30**.herokuapp.com. I'll refer to this as your Heroku URL, and to the part before *.herokuapp.com*, **guarded-coast-30** in this case, as your heroku sub-domain.

#### Register with ngrok

Ngrok is an excellent free service that will allow us to simulate a User sign-up while in development mode on our local machine. Effectively, it opens a port on our computer and tunnels traffic through the ngrok URL to our machine. Cool eh?! 

Head on over to [ngrok](https://ngrok.com/) and sign-up. Although registering is optional, we're going to need it when we define a custom sub-domain.

Next, download ngrok and unzip into your Rails application directory. Later, you may wish to create a new directory for ngrok but for now this will do fine.

#### Register your app with Google

Time to head on over to the [Google Developer’s Console](https://console.developers.google.com/project). Once you arrive, you'll want to:

  1. Click *Create Project*. Give your app a name of *Google Login*. You do not need to set the project id.
  2. Click on your project, click *APIs and auth*, *APIs* and enable *Contacts API* and *Google+ API* (you can do what you'd like with the others).
  3. Click *Credentials* and *Create new Client ID*. Check *Web Application* on the dialog box.
  4. Under *Redirect URIs* enter the following being sure to replace *guarded-coast-30* with your Heroku sub-domain:

    https://guarded-coast-30.herokuapp.com/auth/google_oauth2/callback
    https://guarded-coast-30.ngrok.com/auth/google_oauth2/callback

  5. Under Javascript origins, enter the following again using your Heroku sub-domain:

    https://guarded-coast-30.herokuapp.com/
    https://guarded-coast-30.ngrok.com

  6. Copy the CLIENT_ID and CLIENT_SECRET attributes that Google will provide. Head back to your Rails project and go ahead and paste these in your `config/application.yml` file:

{% highlight ruby %}
# config/application.yml
CLIENT_ID: "YOUR_CLIENT_ID_FROM_GOOGLE.apps.googleusercontent.com"
CLIENT_SECRET: "YOUR_SECRET_TOKEN"
{% endhighlight %} 

*Note: This is where our Figaro gem works its magic. The Figaro gem will help keep information in this file safe. You're definitely going to want to ensure that your CLIENT_ID and CLIENT_SECRET remain secret through the lifetime of your application.*

Finally, note that Google is going to need upwards of 10 minutes to roll out the new redirect URLS your provided. In the mean time, we'll get up and running with OmniAuth.

### Configuring OmniAuth

We're going to add a few initializers to our `config/initializers` directory to get up and running with OmniAuth (and Figaro). First, let's create the following file:

{% highlight ruby %}
# config/initializers/figaro.rb

# Will throw an error if environment variables are not set
Figaro.require_keys("CLIENT_ID", "CLIENT_SECRET")
{% endhighlight %}

This will ensure that we get a build error if we try to deploy our application without our `CLIENT_ID` or `CLIENT_SECRET`. Super handy since our application won't work without these. Next, add the following file:

{% highlight ruby %}
# config/initializers/omniauth.rb

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2, ENV['CLIENT_ID'], ENV['CLIENT_SECRET']
end

# This will route failed login attempts to a SessionsController action
OmniAuth.config.on_failure = SessionsController.action(:oauth_failure)
{% endhighlight %}
*Note that we will need to create the above action (`oauth_failure`) to catch failed login attempts*

### Setup SessionController tests

Again, we'll take on a few steps at a time. I'm going to assume you're now somewhat familiar with the OAuth 2.0 process. Here are a few important things to remember:

  * We provided Google with the callback URL `/auth/google_oauth2/callback`. After our users sign-in with Google they will be redirected to this path and OmniAuth will takeover and do it's [magic (see the magic in action in this post)]({% post_url 2015-01-2-omniauth-under-the-hood %}). 
  * Upon completion, OmniAuth will populate our `request.env['omniauth.auth']` object with the user's information. In our functional controller tests we can simulate these parameters by populating the `@request.env['omniauth.auth']` instance variable
  * We're not going to test the entire OmniAuth process. OmniAuth is a well tested project, we're going to forego the OAuth2.0 handshake and parachute in at the end by populating the `request.env['omniauth.auth']` hash as OmniAuth would upon completion.

Let's go ahead and test that our application:

  1. Passes control of the root url ("/") to the "sessions#index" action
  2. Throws an exception if the "sessions#create" action is called without a `request.env['omniauth.auth']` object defined
  3. Creates a new user if `request.env['omniauth.auth']` is defined AND that user does not yet exist
  4. Returns an existing user if `request.env['omniauth.auth']` is defined AND that user does exist
  5. Sets a session variable, `session['current_user_id']` if a user signs-in or is created (we'll later use this to quickly retrieve users)
  6. Destroys the user's session (by setting `session['current_user_id']`) to `nil` and redirects the user to our `root_path` upon clicking `sign-out`
  7. Redirects to root_path after completion of `create` and `destroy` actions

Those are quite a few tests. To help us out we'll add a few fixtures in our `test/fixtures/users.yml` file and then our tests in `test/controllers/sessions_controller_test.rb`. Feel free to replace the contents of this file with code below:

{% highlight yaml %}
# test/fixtures/users.yml

pam:
  name: Pam Fake
  first_name: Pamela
  last_name: McInnis
  email: pam@email.com
  image: http://www.something.com/pam.jpg
  uid: 930428394820394
  token: ab3b234lbj3fbjkb234
  refresh_token: dnflaf3nkjb23bb3kj2b
  expires_at: 2014-12-22 14:47:49

matt:
  name: Matt McInnis
  first_name: Matt
  last_name: McInnis
  email: matt@email.com
  image: http://www.something.com/matt.jpg
  uid: 930428394820397
  token: ahfjk4hjfkh4898fhjksdf
  refresh_token: asdfhkj33382372398749
  expires_at: 2014-12-22 14:47:49
{% endhighlight %}

Time for some tests:

{% highlight ruby %}
# test/controllers/sessions_controller_test.rb

require 'test_helper'


class SessionsControllerTest < ActionController::TestCase

  # A helper to format a Google response object from a user's attributes
  # Note: could not seem to extend the User class as I had hoped
  def to_request user
    request = {
      :provider => "google_oauth2",
      :uid => user.uid,
      :info => {
        :name => user.name,
        :email => user.email,
        :first_name => user.first_name,
        :last_name => user.last_name,
        :image => user.image
      },
      :credentials => {
        :token => user.token,
        :refresh_token => user.refresh_token,
        :expires_at => Time.at(1419290669).to_datetime,
        :expires => true
      },
      :extra => {
        :raw_info => {
          :sub => "123456789",
          :email => user.email,
          :email_verified => true,
          :name => user.name,
          :given_name => user.first_name,
          :family_name => user.last_name,
          :profile => "https://plus.google.com/123456789",
          :picture => user.image,
          :gender => "male",
          :birthday => "0000-06-25",
          :locale => "en",
          :hd => "company_name.com"
        }
      }
    }.with_indifferent_access
  end

  # A simple test to retrieve the index action
  test "should get index" do
    get :index
    assert_response :success
  end

  test "should route root_path to sessions#index" do
    assert_routing({ path: '/', method: :get }, { controller: 'sessions', action: 'index' })
  end  

  test "should raise an exception if request.env['omniauth.auth'] is not defined" do
    assert_raises(RuntimeError) { get :create, provider: "google_oauth2" }
  end

  test "should create a new user on first login" do
    @request.env['omniauth.auth'] = {
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
    }.with_indifferent_access
    user_count = User.count

    get :create, provider: "google_oauth2"

    assert_equal user_count+1, User.count
    assert_equal User.find_by_uid("123456789"), assigns["current_user"]
  end

  test "should set session['current_user_id']" do
    # cookies[:sharey_session_cookie] = users(:pam).sharey_session_cookie
    pam = users(:pam)
    @request.env['omniauth.auth'] = to_request pam
    user_count = User.count

    get :create, provider: "google_oauth2"
    assert_equal users(:pam), assigns["current_user"]

    assert_equal user_count, User.count
  end

  test "should have access to helper current_user generated from session[current_user_id]" do
    # cookies[:sharey_session_cookie] = users(:pam).sharey_session_cookie
    session['current_user_id'] = users(:pam).id

    get :index
    assert_equal users(:pam), assigns["current_user"]
  end

  test "should destroy session and redirect to root_path" do
    # cookies[:sharey_session_cookie] = users(:pam).sharey_session_cookie
    session['current_user_id'] = users(:pam).id
    pam = users(:pam)
    @request.env['omniauth.auth'] = to_request pam

    get :create, provider: "google_oauth2"

    get :destroy
    assert_redirected_to root_path
    assert_nil session['current_user_id']
  end
end
{% endhighlight %}

Running these tests in the terminal should invoke quite a few errors and failures. Lets get these tests passing.

### Setup routes

Jump into your `config/routes.rb`. Change your routes file to look like this:
{% highlight ruby %}
# config/routes.rb

Rails.application.routes.draw do
  root to:'sessions#index'

  get '/sign_out' => 'sessions#destroy', :as => :sign_out
  get "/auth/:provider/callback" => 'sessions#create'
end
{% endhighlight %}

Worthy of some explanation, our last `get` command above is what will be *catching* the Google callback; this is the redirect URL we provided Google. As a reminder, here's our process when a user sign-in occurs:

  1. User is directed to `/auth/google_oauth2` where OmniAuth will again redirect them to Google (`https://accounts.google.com/o/oauth2/auth?...`) for sign-in.
  2. Upon granting access and completing the sign-in process, the user will be internally directed to our `create` action inside our `SessionsController` along with the large Hash of information we tested against eariler. 

### A few helper methods in ApplicationController
These will be available to all of our controllers in our application. Feel free to replace your ApplicationController with the following:

{% highlight ruby %}
# app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  # Prevent CSRF attacks by raising an exception.
  # For APIs, you may want to use :null_session instead.
  protect_from_forgery with: :exception

  helper_method :current_user
  helper_method :user_signed_in?

  private

  def current_user
    begin
      @current_user ||= User.find(session[:current_user_id]) if session[:current_user_id]
    rescue Exception => e
      nil
    end
  end

  def user_signed_in?
    return true if current_user
  end
end
{% endhighlight %}



### Setting up our Sessions Controller
Time to add some functionality to our Sessions Controller. Feel free to replace the contents of your SessionController with the following:

{% highlight ruby %}
class SessionsController < ApplicationController

  before_filter :get_current_user, :only => [:index]

  def index
  end

  def create
    # raise "foo"
    @current_user = User.find_or_create_from_google_callback google_response
    # reset_session
    # cookies.permanent[:sharey_session_cookie] = @current_user.sharey_session_cookie
    session['current_user_id'] = @current_user.id

    get_current_user
    redirect_to root_url, :notice => 'Signed in!'
  end

  def destroy
    reset_session
    redirect_to root_url, :notice => 'Signed out!'
  end

  def oauth_failure
    # TODO: Render something appropriate here
    render text:"failed..."
  end

  private

  def get_current_user
    @current_user ||= current_user
  end

  def google_response
    raise "Missing parameters" if request.nil?
    raise "Missing parameters" if request.env.nil?
    raise "OmniAuth error. Parameters not defined." if request.env['omniauth.auth'].nil?

    request.env['omniauth.auth']
  end
end
{% endhighlight %}

Let's explain a little of what's happening here:

  * The **create** action is locating or creating a new user, adding a session variable (`current_user_id`) and finally redirecting the user to the root_url.
  * The **destroy** action simplly removes the session `current_user_id` session variable
  * The **google_response** private method is just a helper to check whether our OmniAuth object is present in the `response` object

### Setup a view 

Lets add some simple code in your `index.html.erb` view:

{% highlight erb %}
<!-- app/views/sessions/index.html.erb -->

<h1>GoogleLogin Homepage - Sessions#index</h1>
<ul>
<% if user_signed_in? %>
  <li><%= "Hello " + @current_user.first_name + "!" %></li>
  <li><%= link_to 'Sign out', sign_out_path %></li>
<% else %>
  <li><%= link_to "Sign in with Google", '/auth/google_oauth2' %></li>
<% end %>
</ul>
{% endhighlight %}

Here we will display a sign-in or sign-out button based on whether we recognize the session token of the user. Note that the path `/auth/google_oauth2` is automatically configured and handled for us by OmniAuth; this is where the sign-in process will begin. 

### Up and running!

Hopefully your tests are all passing. You should now be able to fire up your local server and take a run through your application. We'll fire-up ngrok using your Heroku sub-domain:

{% highlight bash %}
$ ./ngrok --subdomain=yourHerokuSubDomain
{% endhighlight %}

In another terminal window:
{% highlight bash %}
$ rails s
{% endhighlight %}

Open your web browser and point it to the URL supplied by ngrok. You should now have a fully-functional Google authentication system!

### Touch up the edges

Secure user authentication is largely predicated on having encrypted requests using SSL. Let's enforce this behaviour in Rails by changing the following settings:

*Just above the `end` tag:*
{% highlight ruby %}
# config/environments/development.rb

# Force all access to the app over SSL, use Strict-Transport-Security, and use secure cookies.
config.force_ssl = true
{% endhighlight %}

*Uncomment the following line:*
{% highlight ruby %}
# config/environments/production.rb

config.force_ssl = true
{% endhighlight %}

### Push to Heroku

{% highlight bash %}
$ git add .
$ git commit -m "added authentication system"
$ git push heroku master
{% endhighlight %}

Now point your browser to your live application and test it out:
{% highlight bash %}
$ heroku open
{% endhighlight %}

### Conclusion
I hope you found this tutorial helpful. Please email me with any comments.

### Additional Resources

If you're looking to gain a more rich understanding of this tutorial, take a read through of the following:

  * [OmniAuth Documentation](https://github.com/intridea/omniauth/wiki) - Will gives you a better understanding of the process and provides a list of 'Strategies' (ie sign-in with Twitter, instead of Google).
  * [OmniAuth Google gem](https://github.com/zquestz/omniauth-google-oauth2) - Provides details for extending functionality beyond what we've done here.
  * [Google OAuth 2.0 API Documentation](https://developers.google.com/accounts/docs/OAuth2) - Very useful for adding features involving Gmail, Contacts, Maps, etc. 

### Refreshing tokens

Although I won't officially support the following, you may wish to implement some functionality in your SessionsContoller to refresh access tokens. This should do the trick:

{% highlight ruby %}
# app/controllers/sessions_controller.rb

def to_params
  {
    'refresh_token' => refresh_token,
    'client_id' => ENV['CLIENT_ID'],
    'client_secret' => ENV['CLIENT_SECRET'],
    'grant_type' => 'refresh_token'
  }
end


def request_token_from_google
  url = URI('https://accounts.google.com/o/oauth2/token')
  NET::HTTP.post_form(url, self.to_params)
end

def refresh!
  response = request_token_from_google
  data = JSON.parse(response.body)
  update_attributes(
    access_token: data['access_token'],
    expires_at: Time.now + (data['expires_in'].to_i).seconds
  )
end

def expired?
  expires_at < Time.now
end

def fresh_token
  refresh! if expired?
  access_token
end
{% endhighlight %}

