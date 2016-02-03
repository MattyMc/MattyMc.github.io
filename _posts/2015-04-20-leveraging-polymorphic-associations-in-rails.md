---
layout:     post
title:      Leveraging Polymorphic Associations in Rails
date:       2015-04-20 9:54:18
summary:    In adding the ability in my application for registered users to invite new, unregistered users I created a few polymorphic relationships. This tutorial walks through adjusting the Item model to be polymorphic, and highlights a few interesting consequences of doing so.
categories: technical
---

### The Problem
Sharey is an application whose value lies in allowing users to share web content easily between friends. When Matt decides to invite a new friend who has not yet registered, I would like Matt to be able to share things with this new, unregistered user so that when this user does sign up he/she may see the content that people have previously shared with him/her.  

Since I'm authenticating users using Google's OAuth2 protocol, maintaining fidelity to Google's response hash and its user table was a priority. I created a new model for an Unregistered User. 

### The Models - Relationships in Question

  * User    - `has_many :items`
  * UnregisteredUser  - `has_many :items`
  * Item    - Will be polymorphic, and belong_to either a User or UnregisteredUser

### How polymorphic associations work
There are two steps involved in setting up a polyorphic relationship.  

  1. Adjusting your database table to include a new field. Since I wanted the user_id field in the Item model to point to either the User model or UnregisteredUser model, I needed to add a user_type field to Item. 
  2. Defining the polymorphic relationships in the ActiveRecord models: Item, User, UnregisteredUser.  

*Note: There's a third step often required, where you'll need to adjust any code that relies on the relationships between these models. I'll discuss more below.* 

### Setting up the database
Let's get the database setup first:  
{% highlight ruby %}
class CreateItems < Activerecord::Migration
  def change
    create_table :items do |t|
      ...
      # Added user_type 
      t.belongs_to :user, index: true, null: false
      t.string :user_type, null: false
      ...

      t.timestamps null: false
    end
    # Needed to update the line below to include user_type
    add_index :items, [:user_id, :user_type, :document_id], unique: true
  end
end
{% endhighlight %}

### Relationships
Next, time to define our relationships. In the Item model, setting up a polymorphic relationship is as easy as adding the polymorphic option to your belongs_to method:  
{% highlight ruby %}
# app/models/item.rb
belongs_to :user, polymorphic: true
{% endhighlight %}  

In the User and Unregistered models:

{% highlight ruby %}
# app/models/user.rb
has_many :items, as: :user

# app/models/unregistered_user.rb
has_many :items, as: :user
{% endhighlight %}

Easy!

### All my tests are failing!
Luckily we have nice test coverage in this application. Here're some of the things that needed changing and a few suprises along the way:  

##### Fixtures
You need to tell your fixtures about the new polymorphic relationship by adding the associated model in brackets.  

{% highlight yaml %}
matts_item:
  document: some_video
  user: matt (User)  # <-- Add (User) or (UnregisteredUser)
{% endhighlight %}  

##### Uniqueness
I needed to update my uniqueness constraints. Since Items can now belong to either a User or an UnregisteredUser, it is no longer sufficient to check for uniqueness on just the user_id field:  
{% highlight ruby %}
# app/models/item.rb
validates :user_id, uniqueness: { scope: [:user_type, :document_id] }
{% endhighlight %}

##### Eager Loading
To avoiding hitting the database (ie the N+1 problem), I was eager loading Users referenced by a collection of Items. Since our user_id field no longer points to a single database table, this is no longer possible in this way. I need to find a different way to use eager loading, other than the `includes(:user)` method. 

### Conclusion
Overall, Rails and ActiveRecord combine to make setting up, testing and using polymorphic relationships very easy.  

Hope this helped!

