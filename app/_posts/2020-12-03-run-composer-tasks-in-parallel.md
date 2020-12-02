---
layout: post
title:  "Run composer tasks in parallel"
category: general
tags: composer
summary: "Composer has a built-in way of running multiple composer scripts by declaring a script alias. This alias runs all specified scripts one by one and fails if one of the scripts fails.  To save you some time, I've written the veewee/composer-run-parallel plugin that can run multiple composer scripts in parallel and won't stop before all tasks have been executed."
image: 20201203/composer-parallel-long.png
thumb: 20201203/composer-parallel_thumb.png
---
  
<p>
Composer has a built-in way of running multiple composer scripts by declaring a script alias.
This alias runs all specified scripts one by one and fails if one of the scripts fails.
To save you some time, I've written the <a href="https://github.com/veewee/composer-run-parallel" target="_blank">veewee/composer-run-parallel</a> plugin that can run multiple composer scripts in parallel and won't stop before all tasks have been executed: 
</p>
<br />
<p align="center">
<img src="/images/blog/20201203/composer-parallel.gif" alt="example" width="100%" />
</p>

# Why I created this package?

<p>
  As the creator of <a href="https://github.com/phpro/grumphp" target="_blank">GrumPHP</a>,
  I am used that <a href="http://veewee.github.io/blog/running-grumphp-tasks-in-parallel/">all tasks run in parallel</a>.
  Some of the open-source project that I work on don't have a GrumPHP configuration (yet).
  Most of those projects run Quality Assurance tools crafted out of composer scripts.
  Even on those projects, I want to make sure to commit good code before pushing it to a Pull Request.
  Since I got bored of waiting for the composer scripts to run one by one, I decided to dive into the composer internals in order to make composer scripts run in parallel as well.
</p>


# Shut up and tell me how to use it!

<p>
  Configuring parallel scripts is like configuring any regular composer script.
  This package provides a `@parallel` script that takes other composer script aliases as arguments:
</p>

{% highlight json %}
{
  "scripts": {
  
      "ci": "@parallel cs tests",
  
      "cs": "php-cs-fixer fix --dry-run",
      "tests": "phpunit"
  }
}
{% endhighlight %}

<p>
  The code above adds a `ci` composer script to your project, that runs the coding styles and unit tests in parallel.
  You can run the script like this:
</p>

{% highlight bash %}
composer run ci
{% endhighlight %}

<p>
  Most of the times, you don't want to introduce dependencies to projects you don't own.
  Therefore, this plugin also provides a `composer parallel` command.
  You can install this package globally, meaning you can unleash the power of parallel task execution even on packages that don't have this plugin installed:
</p>

{% highlight bash %}
composer parallel cs tests
{% endhighlight %}

# In closing

<p>
  No more waiting!
  This <a href="https://github.com/veewee/composer-run-parallel" target="_blank">veewee/composer-run-parallel</a> plugin can be quickly added to any project and is fun to use.
  It contains a few more options. Head over to the readme on GitHub to learn more.
</p>

<p>
  Whilst writing the plugin, I was pleasantly surprised by what is available in the composer codebase by default.
  Writing the package was a matter of linking the various existing components together.
  Big-ups to the composer team!
</p>

<p>
  This plugin comes with one big limitation:
  Since it is only setting composer as a dependency, it is limited to the `Loop` and `ProcessExecutor` that is provided by composer.
  To display the result of a script, both the stderr and stdout are printed out to the end user.
  This could result in non-sequential output if the script you are executing mixes stdout and stderr messages.
</p>
