---
layout: post
title:  "Writing slack integrations with Lambda"
category: general
tags: nodejs lambda serverless
summary: ""
image: 20160731/lambda.jpg
thumb: 20160731/thumb-lambda.jpg
---

<p>
    As mentioned in my previous blogpost, I was creating some slack integrations with PHP.
    Because I did not want to configure a server to run these integrations, I decided to go serverless.
    After struggling with AWS, I've managed to get some Lambda functions up and running.
    In this blogpost I will describe how I created a <a href="https://www.meetup.com/" target="_blank">meetup.com</a> cronjob and slashcommand.
    Make sure to read about <a href="/blog/writing-your-first-nodjs-lambda-function/">creating lambda functions</a> 
    in my previous post if you are new to lambda functions!
</p>

<h2>Installing the right dependencies</h2>

<p>
    Because we do not want to write everyting ourselves, you can use following dependencies:
</p>

{% highlight json %}
{
  "dependencies": {
    "dateformat": "^1.0.12",
    "dotenv": "^2.0.0",
    "meetup-api": "^1.3.14",
    "named-regexp": "^0.1.1",
    "node-slack": "^0.0.7",
    "qs": "^6.2.0"
  },
}
{% endhighlight %}

<ul>
    <li>The `dateformat` package is used for parsing a Javascript date in a human readable format.</li>
    <li>The `dotenv` package is used for parsing environment variables from a `.env` file.</li>
    <li>The `meetup-api` package provides some methods to interact with meetup.com.</li>
    <li>The `named-regexp` package is used to create a slashcommand parser that will match a callback to a slashcommand.</li>
    <li>The `node-slack` package provides the tools to create an incomming webhook or respond to a slash command.</li>
    <li>The `qs` package is used to parse the RAW url encoded body from the API gateway into a usable javascript object.</li>
</ul>

<h2>Creating a cronjob</h2>





<h2>Conclusion</h2>

<p>
</p>
