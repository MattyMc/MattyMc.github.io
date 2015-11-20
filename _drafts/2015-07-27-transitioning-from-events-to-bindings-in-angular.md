---
layout:     post
title:      AngularJS - A transition from an event driven approach to monitoring keyboard events to utilizing Angular's two-way bindings
date:       2015-01-13 16:32:18
summary:    In designing an Rails application front-ended by AngularJS, I needed to pass keyboard events to multiple controllers. This is how I transitioned from suing multiple $broadcast and $on methods in my controllers, to monitoring the events in one model instance and using $watch. 
categories: technical
---

### The Problem
I'm building an application that will rely heavily on "keydown" and "keyup" events. Multiple Angular controllers will respond to these events, and I want to design my application efficiently from the start. 

### Using $broadcast and $on
Angular makes broadcasting events and detecting these events reasonably simple with `$broadcast` and `$on`. Here's how I was using these in my app:  

{% highlight javascript %}
// My application 

// Setup my module
var app = angular.module("ptt", []);

// KeyboardService - Used to broadcast keydown listeners in app.run below
app.service("keyboardService", function() {
    this.keyCode = null;
    this.watch = function (event_type, callback) {
        document.addEventListener(event_type, callback, false);
    }
});

// Allows broadcasts the keydown and keyup events on $rootScope
app.run(function ($rootScope, keyboardService) {
    // event_type = "keydown"
    keyboardService.watch("keydown", function (e) {
        keyboardService.keyPressed = e.keyCode; 
        $rootScope.$apply(function() {
            $rootScope.$broadcast("keydown", e);
        });
    });

    keyboardService.watch("keyup", function (e) {
        $rootScope.$apply(function() {
            $rootScope.$broadcast("keyup", e);
        });
    });
});


{% endhighlight %}

Next, I could easily detect these events in my controllers:

{% highlight javascript %}
// Define my controller
app.controller('KeyboardController', function($scope, keyboardUtilities) {
  $scope.current_key = "";
  $scope.keysdown = [];

  $scope.$on("keydown", function(a, e) {
    keyboardUtilities.blockDefaultBehaviourOfTheseKeys(e);

    var key = keyboardUtilities.getKeyPressed(e, true);
    $scope.current_key = key.toLowerCase();

    // Make sure a key cannot be added twice in an array
    if (!$scope.isKeyDown($scope.current_key)) {
      $scope.keysdown.push($scope.current_key);
    }
  });
});
{% endhighlight%}

This approach works just fine. However, I intend on having multiple controllers that will respond to the keydown/keyup events, and I don't want to be running the same methods (ie `keyboardUtilities.getKeyPressed(e)`) and storing the same variables (`current_key`, `keysdown`, etc.) in each controller. 

### New approach - Use Angular's two-way data binding

Instead, I will modify my service `keyboardUtilities` to process all the keydown events (determine key pressed, add to an array, etc.) and set the results to some internal variables. Instead of broadcasting these changes to all of my controllers, I will use the $watch feature to detect any changes. This will allow me to keep all my keyboard related application logic in one place, and result in some big models and skinny controllers:



