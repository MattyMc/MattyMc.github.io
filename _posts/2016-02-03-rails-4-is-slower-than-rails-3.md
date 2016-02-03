---
layout:     post
title:      Is Rails 4 slower than Rails 3? 
date:       2016-01-26 12:32:18
summary:    In describing their process upgrading from Rails 3.2 to Rails 4.2 at February's Ruby Lightning Talks T.O., Canadian startup 500px experienced significant drops in performance in some areas. They shared one example of code that was outright baffling. This article is an explanation of my investigation into this problem.
categories: technical
---

In describing their process upgrading from Rails 3.2 to Rails 4.2 at [February's Ruby Lightning Talks T.O.](http://www.meetup.com/ruby-lightning-to/events/227145915/?), Canadian startup [500px](http://www.500px.com/) experienced significant drops in performance. To underscore the issue, they shared the following example:  

{% highlight bash %}
# Rails 3.2
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}
=> 0.370000 0.000000 0.370000 ( 0.371422)
{% endhighlight %}

{% highlight bash %}
# Rails 4.2.5
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}
=> 1.620000 0.030000 1.650000 ( 1.649495) 
{% endhighlight %}

####Wait a minute. Rails 4 is slower???
Sure seems that way. This simple test shared by [Kevin](https://github.com/trustyknave/) at [500px](http://www.500px.com/‎) sure seems clear: `(0..5).each { |i| }` is 4 times faster on Rails 3.2 compared to Rails 4.2. This is a significant drop in performance. Lets find out why.

#### Isolating the problem
This was my approach to try to find out what's wrong:  

  1. Replicate the problem. We'll setup skinny applications of both Rails 3.2 and Rails 4.5.2 to investigate and run the code above to make sure we get similarly shitty results.
  2. Eliminate Ruby as a culprit. Lots of ways to do this, most simply I'll compare my benchmarks with that obtained in a pure ruby environment (ie `irb`, no Rails).
  3. Dig into the `Range` and `Enumerator` classes. There are lots of ways to do looping in Ruby, are they all benchmarking so poorly between versions? This should help us figure out which docs we need to dig into.
  4. Dig into documentation between versions to figure out what's happening. 

#### Setting up an environment
{% highlight bash %}
# Some setup
mkdir rails-speed-test
cd rails-speed-test

# Install the gems we need
gem install rails -v 3.2.0
gem install rails -v 4.5.1 

# Generate our rails environments 
rails _3.2.0_ new rails-3.2.0
rails _4.5.1_ new rails-4.5.1

# Run bundle install in each directory
{% endhighlight %}

From here, I would suggest opening three terminal tabs:  

  - One tab in each of our new rails applications. Run `rails c` to enter the console. 
  - One tab running `irb` for pure ruby comparisons.

#### Setting up our environments

Making sure we can replicate the results:
{% highlight bash %}
# irb
require 'benchmark'
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}.total
=> 0.38

# Rails 3.2.0
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}.total
=> 0.39

# Rails 4.5.1
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}.total
=> 1.43
{% endhighlight %}

Summary: Rails 3.2 and No Rails (`irb`) are approximately equal, Rails 4.5 is ~4 times slower. All three environments are running the same version of Ruby, so this is definitely a Rails issue.

#### Is it just Enumerator?

Running `(0..5).each.class` returns an `Enumerator` object. I decided to test whether other loops/objects would produce similar slowdowns. Here is a sample of some of the tests I ran and their results:  

{% highlight bash %}
# What about just a boolean? 
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { 
      true 
    } 
  } 
}.total
# result: Rails 4 is much slower

# What about a different type of loop? 
Benchmark.measure { 
  1_000_000.times { 
    i = 0
    while i < 6
      i += 1
    end
  } 
}.total
# result: No difference between Rails 3 and Rails 4
{% endhighlight %}

So it seems the damage might be limited to the `Enumerator` class. Since this class can be created in a number of ways, lets see if all objects of class `Enumerator` are experiencing the slow down, regardless of how they were created: 

{% highlight bash %}
# A few different ways to define an Enumerator:
[].each.class
4.times.class
Range.new(0,5).each  #  <- The "bad one"
# etc.

# Testing some out:
# How about defining an Enumerator with an array? 
Benchmark.measure { 
  1_000_000.times { 
    [1,2,3,4,5,6].each do 
      true 
    end
  } 
}.total
# No difference between Rails 3 (0.71s) and Rails 4 (0.75s)

# How about defining an Enumerator with Integer.times? 
Benchmark.measure { 
  1_000_000.times { 
    6.times do 
      true 
    end
  } 
}.total
# No difference between Rails 3 (0.42s) and Rails 4 (0.42s)
{% endhighlight %}

Very surprising. Both `[].each` and `6.times` create an Enumerator object, yet neither experience the major slow down that we're experiencing when instantiating an `Enumerator` object from the `Range` class (ie `(0..5).each`). 

This is a big hint. Since both `irb` and Rails 3.x are not experiencing the same slow down that's happening in Rails 4, and the `Array` and `Fixnum` classes seem to be working well it's reasonable to assume that Rail's `Range#each` method has been overwritten. [ActiveSupport](https://github.com/rails/rails/tree/master/activesupport) is the Rails library that extends these methods. Lets take a look. 

#### ActiveSupport

In [rails/activesupport/lib/active_support/core_ext](https://github.com/rails/rails/tree/4-2-5/activesupport/lib/active_support/core_ext), switching to version 4.2.5, we find [range/each.rb](https://github.com/rails/rails/blob/4-2-5/activesupport/lib/active_support/core_ext/range/each.rb). First, note that if we switch to version 3.2 on GitHub there's now `each.rb` file - good sign. Here's the code: 

{% highlight ruby %}
require 'active_support/core_ext/module/aliasing'

class Range #:nodoc:

  def each_with_time_with_zone(&block)
    ensure_iteration_allowed
    each_without_time_with_zone(&block)
  end
  alias_method_chain :each, :time_with_zone

  def step_with_time_with_zone(n = 1, &block)
    ensure_iteration_allowed
    step_without_time_with_zone(n, &block)
  end
  alias_method_chain :step, :time_with_zone

  private
  def ensure_iteration_allowed
    if first.is_a?(Time)
      raise TypeError, "can't iterate from #{first.class}"
    end
  end
end
{% endhighlight %}

Notice the line `alias_method_chain :each, :time_with_zone`? From the [documentation](http://apidock.com/rails/ActiveSupport/CoreExtensions/Module/alias_method_chain), the following are equivalent: 

{% highlight ruby %}
alias_method_chain :foo, :feature
# and:
alias_method :foo_without_feature, :foo
alias_method :foo, :foo_with_feature
{% endhighlight %}

This means that the `Range#each` instance method is being overwritten presumably with a much, much slower version, since:
{% highlight ruby %}
alias_method_chain :each, :time_with_zone
# equivalent to: 
alias_method :each_without_time_with_zone, :each
alias_method :each, :each_with_time_with_zone
{% endhighlight %}

#### Final test
Lets jump into our Rails-free console (`irb`) and see if we can duplicate the problem. First lets get a baseline benchmark: 

{% highlight bash %}
require 'benchmark'
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}.total
=> 0.39
{% endhighlight %}

Next, lets overwrite the the `Range#each` method in the same way that ActiveSupport is doing. Note that I've commented the call to the private function `ensure_iteration_allowed` and replaced `alias_method_chain` with its functionally equivalent two-line version described above:  

{% highlight bash %}
class Range
  def each_with_time_with_zone(&block)
    # ensure_iteration_allowed
    each_without_time_with_zone(&block)
  end
  alias_method :each_without_time_with_zone, :each
  alias_method :each, :each_with_time_with_zone
end
{% endhighlight %}

Finally, lets run our benchmark:

{% highlight bash %}
Benchmark.measure { 
  1_000_000.times { 
    (0..5).each { |i| 
    } 
  } 
}.total
=> 1.23
{% endhighlight %}

Again, our benchmark has gone from 0.39 seconds to 1.23 seconds. We've found our culprit. 

#### Conclusion

Simply overwriting the `Range#each` method as ActiveSupport has been doing in since verison 4.0 causes a major slowdown. It might be reasonable to assume that other changes in ActiveSupport have caused much of the slowdown that [500px](http://www.500px.com), and likely many other organizations are experiencing. 