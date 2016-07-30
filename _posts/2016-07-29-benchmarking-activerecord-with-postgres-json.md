---
layout:     post
title:      The strange story of ActiveRecord, Postgres, and large JSON files.
date:       2016-07-29 17:15:32
summary:    When storing large JSON files, how efficiently does ActiveRecord's jsonb field store and retrieve large JSON files when compared to a json field, or a regular String field?
categories: technical
---

### When storing large JSON files, how efficiently does ActiveRecord's jsonb field store and retrieve data when compared to using a regular String field??

This is an experiment I performed to help me solve a problem with long response times. Full disclosure, I experienced some strange results here that I can't explain. I'd love to be able to understand these better (help, svp!), but for now I'm simply posing the problem. All code is included.

If you decide to duplicate, ensure you're using at least **Postgres v9.4**.

There've been a few great posts about how speedy ActiveRecord is in storing JSON and JSONB files. In particular, [Nando Vieira](http://nandovieira.com/) shows that [inserting 30,000 records in json versus jsonb "didn't have any real difference"](http://nandovieira.com/using-postgresql-and-jsonb-with-ruby-on-rails), with yielding response times of 12.76s vs 12.57s respectively (only a 1.5% efficiency decrease). He does however note that there querying JSON is much faster with JSONb.  

I wish to extend Nano's study to see how large the performance difference is when JSON files are larger (such as an array of 2000 objects). Is it larger than the 1.5% drop that Nando calculated? If yes, how much larger?  

### Setting up Rails & Postgres

I launched a new rails application `rails new storing_large_json_test_app --database=postgresql` with `bundle install` and generated three models using:

{% highlight bash %}
rails g model Test1string data:string
rails g model Test2Json data:json
rails g model Test3Jsonb data:jsonb

rake db:create db:migrate
{% endhighlight %}

In your migration files you should see, respectively:  
{% highlight ruby %}
# Test1string
t.string :data

# Test2Json
t.json :data

# Test3Jsonb
t.jsonb :data
{% endhighlight %}

### Getting data

I will be comparing an array of objects, where each object is a set of 4 randomly generated, 2 character strings. Note that all our objects will be the same.

{% highlight ruby %}
data = [
  {
    first_key: rs1,
    second_key: rs2,
    third_key: rs3,
    fourth_key: rs4
  },
  ...
]  
{% endhighlight %}


### Benchmarking & Results


Lets generate four random strings using this handy [code snippet from StackOverflow](http://stackoverflow.com/questions/88311/how-best-to-generate-a-random-string-in-ruby), store them within an array of objects. Next, we'll convert this to a string to mimic our server and then run some benchmarks:  

{% highlight ruby %}
# Helper:
def r_string
  (0...2).map { (65 + rand(26)).chr }.join
end

# Generate an array of objects
entry = []
2000.times { entry.push({first_key:r_string, second_key:r_string, third_key:r_string, fourth_key:r_string}) }
entry = entry.to_s

# Benchmark string field
string_test = Benchmark.measure do
  10.times { Test1string.create!(data:entry) }
end

# Benchmark json field
json_test = Benchmark.measure do
  10.times { Test2Json.create!(data:entry) }
end

# Benchmark jsonb field
jsonb_test = Benchmark.measure do
  10.times { Test3Jsonb.create!(data:entry) }
end

# handy print
p "String test: " + string_test.real.to_s + " | " + "Json test: " + json_test.real.to_s + " | " + "Jsonb test: " + jsonb_test.real.to_s
{% endhighlight %}

Results:  

  - String test: 0.748323 seconds
  - json test:   0.020513 seconds
  - jsonb test:  0.021677 seconds (only marginally slower than json)

As you can see above, the object associated with a string field is *significantly* slower when storing a string of JSON. I wasn't able to determine exactly why, to me this seems a bit counterintuitive. Note-to-self: *Use JSON fields when storing a String of JSON*.  

However, what about the opposite situation? Namely, what about when we're storing a Ruby Array or Ruby String object that has not been coerced to a string? I ran the tests two more times, first by leaving the object as a Ruby Array, and next by coercing the `entry` object to JSON before storing it (user: `entry = entry.to_json`).

Results with String (`to_s`, from above):  

  - String test: 0.748323 seconds
  - json test:   0.020513 seconds
  - jsonb test:  0.021677 seconds  

Results with Ruby Array:  

  - String test: 0.851061 seconds
  - json test:   4.765974 seconds
  - jsonb test:  6.193291 seconds  

Results with JSON (`to_json`):  

  - String test: 0.846517 seconds
  - json test:   4.310821 seconds
  - jsonb test:  6.614038 seconds


### Summary

It's important to know what type of object you're storing if you have a large amount of JSON to store.   

  - If you have a string of JSON, store in a JSON field
  - If you have a Ruby Object, coerce to string, store in a json or string field
  - If you have a coerced string of JSON, consider storing in a string field

Interestingly, if we have a Ruby object and coerce it to a string (`to_s`) it will store nice and quickly in a JSON file. Similarly if we have an escaped JSON string and coerce it non-escaped JSON string we can bring performance times down. For example, if we have an escaped JSON string (example #3):

{% highlight ruby %}
# Helper:
def r_string
  (0...2).map { (65 + rand(26)).chr }.join
end

# Generate an array of objects
entry = []
2000.times { entry.push({first_key:r_string, second_key:r_string, third_key:r_string, fourth_key:r_string}) }
entry = entry.to_json

# Benchmark string field
string_test = Benchmark.measure do
  10.times { Test1string.create!(data:entry) }
end

# Benchmark json field
json_test = Benchmark.measure do
  json_as_string = JSON.parse(entry).to_s
  10.times { Test2Json.create!(data:json_as_string) }
end

# Benchmark jsonb field
jsonb_test = Benchmark.measure do
  json_as_string = JSON.parse(entry).to_s
  10.times { Test3Jsonb.create!(data:json_as_string) }
end

# handy print
p "String test: " + string_test.real.to_s + " | " + "Json test: " + json_test.real.to_s + " | " + "Jsonb test: " + jsonb_test.real.to_s
{% endhighlight %}  

We get some baffling results here:  

Results coercing to string:  

  - String test: 0.751831 seconds
  - json test:   0.136969 seconds
  - jsonb test:  0.05749 seconds

Now, jsonb is faster than storing a string. Moral of the story: **Run Benchmarks when you're storing JSON!**.  **:/**
