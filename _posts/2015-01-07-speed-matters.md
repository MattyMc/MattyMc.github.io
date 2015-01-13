---
layout:     post
title:      Speed Matters
date:       2015-01-13 16:32:18
summary:    I believe it's impossible to completely dissociate the speed at which we perform a task with how others perceive our competence in performing that task. This post is about how I got blown out of the water, speed-wise, in my first paired programming session and the drastic steps I'm taking to ensure it never happens again. 
categories: technical
---

In early December I had the opportunity to participate in a paired programming session with a developer at Shopify. Shopify has some great developers on staff, and having never formally worked in the field I knew I would almost certainly be overmatched in skill level. Boy was I right.

"Do you prefer to use MiniTest or RSpec for testing?" he asked. 

*Great*, I thought. Testing is (was?) one of my weaker areas and this guy's comfortable with both MiniTest and RSpec? 

"MiniTest", I replied.

"What text editor do you prefer?" 

"Sublime would be great," finally confident in one of my answers. 

Curious as to his text editor of choice, I inquired "what do you use?" 

"Vim" he replied without hesitation. 

For those reading without a technical background, Vim is a less popular text editor that has a somewhat cult following. Although you can become a very quick programmer using Vim, most programmers stay away because of it's steep learning curve and heavy reliance on keyboard shortcuts. Aesthetically, it looks like it's from 1990; because it is. 

"How long have you been using Vim for?" 

"Since I was 7 years old."

Well if I wasn't intimidated before, I certainly was now. 

###Time to code

As an exercise we were to build a URL shortener, something akin to [bit.ly](https://bitly.com/). 

Once we launched our Rails project and started diving into code I felt *much* more comfortable. This was the same ol' Ruby on Rails that I'd used to build many applications, and although I was technically overmatched, the difference in our skill levels now felt much more narrow. 

We whipped together our URL shortener reasonably quickly and had a nice time doing it. He was a super-nice guy and very accomodating to my more junior skill level. It was great fun.  

Although I had focused so much of my preparation on my knowledge of Ruby and Rails, I had never anticipated how much *faster* of a programmer I would be paired with. In terms of speed, I simply couldn't keep up. For me at least this was a big problem. 

###Two problems I didn't anticipate

  1. I was incredibly clumsy on the keyboard. At first I blamed this on having cold hands, however as the session progressed it became much more clear that I was a *much* slower typer than my partner. Moreover, in trying to keep up with his blazing typing speed I was making even *more* mistakes. 
  2. I was much slower at debugging. Specifically, when our program would throw errors or raise exceptions he was able to locate the line numbers much more quickly than I was. Having never programmed with any sort of time pressure I didn't see this problem coming.
  
I can't imagine the effect these experiences had on my partner's perception of my programming ability. I imagine the times I've sat with my mother at the keyboard, watching her stike each key with one of her two index fingers. Or rather, standing over the shoulder of a friend instructing him to click a particular link on a website and waiting as they strugle to locate it. 

Our instincts are to take over in these instances. As my partner sat beside me, patiently waiting for me to located the line number of the error or to type out the next line of code I wonder what he was thinking in those moments. I'll likely never know, yet I'm certain it wasn't what I wanted him thinking: *Matt is a great programmer*.

###Speeding up

I never want to be the slower typist or debugger in a paired programming session ever again. Rather, I want to be the fastest programmer in the room. I want my speed to be impressive. Here's how I'm going about getting to that level:

####Typing Speed

I've committed to improving my typing speed through daily practice. I found a great web application with a beautiful interface, smart pedagogy, and excellent data tracking; [keybr.com](http://keybr.com). Here's my progress so far:

![Typing Data](/images/typing_improvement.png)

My typing speed originally was a measly 45-48 WPM. Now, I'm averaging somewhere between 80-85 WPM. 

This improvement has largely been the result of daily practice. At the time of this post, I have practice every day without exception for more than five weeks.

![Typing Data](/images/typing_calendar.png)

I couldn't be more happy with my progress. However, to truly be a programming wizard on the keyboard I'll need to improve the speed at which I can:

  * Type CamelCase text
  * Type underscore_text
  * Navigate my text editor, both within a file and between files

Luckily, [keybr](http://keybr.com) allows for custom text. I'll be taking advantage of this feature in the near future by populating the application with common Ruby syntax.

####Debugging and otherwise

I'm attempting to become more conscious of repetive tasks I perform in building an application in the hopes of improving the time required to perform said tasks. To use debugging as an example, consider the following task:

You run your tests. One of your tests either fails or throws an error message. Where is the line number of the error or failure located?

*Error message:*
{% highlight bash %}
Error:
ItemsControllerTest#test_should_not_create_a_new_category_even_if_there_are_leading_spaces:
SyntaxError: /Users/Matt/Dropbox/programming/webdev/sharey/app/controllers/items_controller.rb:31: syntax error, unexpected tIDENTIFIER, expecting ')'
        original_request: item_params["description"]
                        ^
/Users/Matt/Dropbox/programming/webdev/sharey/app/controllers/items_controller.rb:32: syntax error, unexpected ')', expecting keyword_end
{% endhighlight %} 

In an error, the line number is the first result in the `SyntaxError` hash. 

*Test failure*:
{% highlight bash %}
Failure:
ItemsControllerTest#test_should_update_an_item_if_a_user_already_has_saved_that_item [/Users/Matt/Dropbox/programming/webdev/sharey/test/controllers/items_controller_test.rb:122]:
should create a new category.
Expected: 3
  Actual: 2
{% endhighlight %}

Here, the line number is located at the end of the square brackets - the value returned after the method name.

Although these insights may seem trivial, I believe spending a few moments analyzing small, repetitive tasks such as these will result in large improvements in programming speed over the long term. I've certainly noticed a difference.

###Final thoughts

I learned a great deal from my first paired programming experience. In addition to all the learning I'm thankful I had the chance to spend some time programming with such a nice, talented person. 

It was a super positive experience. Hopefully one I'll have the opportunity to repeat soon.