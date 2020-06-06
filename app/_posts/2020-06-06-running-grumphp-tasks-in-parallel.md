---
layout: post
title:  "Running GrumPHP tasks in parallel"
category: general
tags: grumphp
summary: "Last week, we launched a new version of GrumPHP. One of the main new features inside this relase is the ability to run your tasks in parallel. This feature will safe you a lot of waiting, both during pre-commit and on the CI environments!"
image: 20200606/parallel-grumpy-long.png
thumb: 20200606/parallel-grumpy_thumb.png
---

<p>
    Last week, we launched <a href="https://github.com/phpro/grumphp/releases/tag/v0.19.0" target="_blank">a new version of GrumPHP</a> (0.19.0).
    One of the main new features inside this relase is the ability to run your tasks in parallel.
    This feature will safe you a lot of waiting, both during pre-commit and on the CI environments!
</p>

<p>
<img src="/images/blog/20200606/parallel.gif" width="100%" />
</p>

<h2>How do I get this madness to work?</h2>

<p>
    Actually, you don't have to do anything! Parallel task execution is enabled by default.
    All tasks will run in parallel, unless you specify a different priority on a task.
    This way, you can specify parallel groups inside your configuration.
    Imagine following setup:
</p>

{% highlight yaml %}
# grumphp.yaml

grumphp:
  tasks:
    phpcs:
      standard: PSR2
    phpunit: ~
    clover_coverage:
      clover_file: 'test/clover.xml'
      metadata:
        priority: -10

{% endhighlight %}

<p>
    This configuration will run both `phpunit` and `phpcs` in parallel since they are both in priority group 0.
    When both tasks are ready, the `clover_coverage` task is started
</p>

<p>
Enjoy and let me know what you think of it!
</p>
