---
layout:     post
title:      Duplicating objects in Ruby
date:       2015-11-20 10:32:18
summary:    A brief look at the (sometimes) surprising effects of freezing simple and slightly more complex objects in Ruby, and the subsequent effects of manipulating the duplicate's properties.  
categories: technical
---

Let's start with a brief example:
{% highlight ruby %}
a = "foo"
b = a 
a.object_id == b.object_id 
{% endhighlight %}  

These two variables, `a` and `b`, are referencing the same object in memory (the String object with value "foo"). My first thought was that creating a new object and assigning it to `a` would replace the previous object in memory, causing variable `b` to reference this object as well: 

{% highlight ruby %}
# Continuing from above
a = "bar"  # New String object with value "bar"
puts b     # Variable 'b' references the original object 
=> "foo"    
a.object_id == b.object_id  
=> false
{% endhighlight %}  

Our two variables are now referencing different objects in memory. However, Ruby does have a replace method which would replace the original object in memory and maintain both variable's reference to the same object:

{% highlight ruby %}
a = "foo"  
b = a  

# We replace the object referenced by both 'a' and 'b'
a.replace "bar"  

puts a + " " + b
=> "bar bar"  

a.object_id == b.object_id  
=> true
{% endhighlight %}  

To summarize, when we assign a new object to a variable, Ruby, or more specifically [Matz's Ruby Interpretter](https://en.wikipedia.org/wiki/Ruby_MRI), creates a new object in memory and updates the variable's object_id (memory location). 

#### What happens when we modify an object? 

Will the following code yield `true` or `false`?

{% highlight ruby %}
a = "foo"
b = a

b[0] = "b"

a.object_id == b.object_id  # true
puts a  # "boo"
{% endhighlight %}  

Although using the assignment operator (=) creates a new object in memory, modifying an *existing* object does not. Above, the underlying object of variables `a` and `b` was affected. 

### Using `freeze` to prevent changes

Everything in Ruby is an Object, and every object has a `freeze` method we can use to prevent changes to an object (and will raise an exception if we try to make a change). Which of the following will raise an exception?

{% highlight ruby %}
# 1. New assignment to frozen object
a1 = "foo"  
a1.freeze
a1 = "bar"

# 2. Two variables reference frozen object
a2 = "foo"
b2 = a2
a2.freeze
b2 = "bar"

# 3. Replacing frozen object
a3 = "foo"
a3.freeze
a3.replace "bar"

# 4. Replacing frozen, referenced object
a4 = "foo"
b4 = a4
a4.freeze
b4.replace "bar"
{% endhighlight %}  

**Solutions**  

  1. Will *not* throw an exception. Remember, when we assign the String with value "bar" to the variable `a1` we are creating a new String object in memory and changing the object referenced by `a1`. Thus, the frozen String object with value "foo" is no longer referenced by `a1`. Hence, we are not modifying the frozen object. 
  2. Will *not* throw an exception. Similar to the explanation above, we are creating a new object when assign the String "bar" to variable `b2` - the frozen "foo" String referenced by `a2` is not being modified.
  3. *Will* throw an exception. We are attempting to modify the frozen string object since the Ruby method `replace` literally replaces an object in memory.
  4. *Will* throw an exception. Variables `a4` and `b4` are both referencing the same frozen object. When we try to replace the (frozen) object referenced by `b4`, Ruby raises an exception for us.

### More complicated objects

Suppose we have a simple class with one attribute:
{% highlight ruby %}
class Person
  attr_accessor :name
end
{% endhighlight %} 

How will `freeze` work in regards to a new Person object? Here's some counter-intuitive behavior:
{% highlight ruby %}
matt = Person.new
matt.name = "Matt"  
matt.freeze   

# Will a new string assignment work?
matt.name = "Matthew"  
=> RuntimeError: can't modify frozen Person

matt.name.replace "Matthew"
=> "Matthew"
{% endhighlight %} 














