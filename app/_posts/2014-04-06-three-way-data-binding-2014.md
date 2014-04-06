---
layout: post
title:  "Three-way data binding"
category: general
tags: angular firebase angularfire
summary: One of the cool features in AngularJS is the two-way data binding. When you change a property of an object, the value is instantly changed in every view or service that is using this property. A while ago, I saw a video about the combination of AngularJS and Firebase. This combination makes it possible to create three-way data binding in AngularJS.
image: 20140406/angularfire.png
thumb: 20140406/thumb-angularfire.jpg
---

One of the cool features in <a href="http://angularjs.org/" target="_blank">AngularJS</a> is the two-way data binding.
When you change a property of an object, the value is instantly changed in every view or service that is using this property.
A while ago, I saw a video about the combination of AngularJS and
<a href="https://www.firebase.com/" target="_blank">Firebase</a>.
This combination makes it possible to create three-way data binding in AngularJS.

## What is Firebase?
Firebase is an online service that provides an API to store and sync data in realtime.
This means that when the data is saved, all connected clients will retrieve the latest data without reloading the page.
You can see Firebase as a database service with some extra features like authentication.
The cool thing about this service is that you don't need to create a database schema.
You can create collections just by specifying a path to the database in the URL.
An additional package '<a href="https://www.firebase.com/quickstart/angularjs.html" target="_blank">AngularFire</a>' is available for integration with AngularJS.

## The world's easiest guestbook

### Getting started
A good way to start a test project is by using <a href="http://yeoman.io/" target="_blank">Yeoman</a>.
Both AngularJs and AngularFire have a generator that will create your basic application in no time.
Just run following commands and follow the wizard:

{% highlight sh %}
$ yo angular
$ yo angularfire
$ grunt serve
{% endhighlight %}

### The view
{% highlight html %}
{% raw  %}
<div class="jumbotron">
  <h1>'Allo, 'Allo!</h1>
  <p class="lead">
    Please leave a message if your like my page!
  </p>

  <div class="form-group">
      <input ng-model="input.name" placeholder="Name" class="form-control" />
  </div>
  <div class="form-group">
      <textarea ng-model="input.message" class="form-control" placeholder="Message"></textarea>
  </div>
  <p><a class="btn btn-lg btn-success" data-ng-click="addMessage()">Splendid!</a></p>
</div>

<div class="messages" ng-cloack="">
    <div class="message" id="message-{{ message.$id }}" ng-repeat="message in messages | orderByPriority | reverse">
        <div class="name">{{ message.name }}</div>
        <div class="content">
            {{ message.message }}
        </div>
    </div>
</div>
{% endraw %}
{% endhighlight %}

### Initializing your firebase collection
The AngularFire package provides some ways to interact with Firebase.
The easiest way to connecto to a collection is by using the syncData service.
We can create the connection with one line of code:

{% highlight js %}
angular.module('app')
  .controller('MainCtrl', function ($scope, syncData) {

    $scope.messages = syncData('messages');

  });
{% endhighlight %}

### Add a message
Now that we set up the connection, we want to add messages to the collection.
This part is very easy: you just pass a JSON object to the collection:

{% highlight js %}
$scope.input = {
  message: '',
  name: ''
};

$scope.addMessage = function()
{
    $scope.messages.$add({
      name: $scope.input.name,
      message: $scope.input.message,
      timestamp: Date.now()
    });

    $scope.input.message = '';
}
{% endhighlight %}

### Sort messages in reverse order
When you want to display the messages in reverse order you should create one extra filer called `reverse`.
This filter is allready written in one of the example AngularFire projects.
<a href="https://github.com/firebase/angularFire-seed/blob/master/app/js/filters.js" target="_blank">You can find the code over here.</a>

A quick way to create a custom filter is by using the Yeoman angular:filter generator:
{% highlight sh %}
$ yo angular:filter reverse
{% endhighlight %}


### Hosting
It is also possible to use Firebase hosting.
A custom CLI tool is provided an can be installed through NPM:
{% highlight sh %}
$ npm install -g firebase-tools
{% endhighlight %}

The next step is to create a firebase configuration file. This can be done with the command:
{% highlight sh %}
$ firebase init
{% endhighlight %}

*Note: The destination folder should be `dist`*

The next step is to deploy the application. This is also very easy:
{% highlight sh %}
$ grunt build
$ firebase deploy
{% endhighlight %}

The sample application is now available at: <a href="https://veewee.firebaseapp.com/" target="_blank">Firebase App</a>.

