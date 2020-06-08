---
layout: post
title:  "Let GrumPHP fix your code!"
category: general
tags: grumphp
summary: "One of the most requested features is the ability to automatically fix broken code. Since previously we could not do this in a controlled an safe way, we decided that this won't be a GrumPHP feature. In the new GrumPHP version (0.19.0), we rewrote the task runner system. Now it is possible to do these kind of code manipulations. Let's see how it works... "
image: 20200607/fixer-grumpy-long.png
thumb: 20200607/fixer-grumpy_thumb.png
---

<p>
    One of the most requested features is the ability to automatically fix broken code.
    Since previously we could not do this in a controlled an safe way, we decided that this won't be a GrumPHP feature.
    <a href="https://github.com/phpro/grumphp/releases/tag/v0.19.0" target="_blank">In the new GrumPHP version</a> (0.19.0), we rewrote the task runner system.
    Now it is possible to do these kind of code manipulations. 
    Let's see how it works...
</p>

<p align="center">
<img src="/images/blog/20200607/fixer.gif" width="100%" />
</p>


<h2>How do I get this madness to work?</h2>

<p>
    No worries: Fixing is enabled by default!
    Currently we added fixers for `phpcs` and `PHP-CS-Fixer`.
    This list might be extended in the future and you can of course also provide fixers for your own custom tasks!
</p>

<h2>How does GrumPHP knows what to fix?</h2>
<p>
    GrumPHP will first run the configured tasks as it always did.
    When a task fails, we already provided the commandline command to fix the code.
    This resulted in manual copy/pasting the code, which is a stupid repeatable task.
    With this new feature, the task can now determine if it can autofix the code by e.g. running the console command it displayed.
    In a last phase before finishing the task runner, the autofixer process is initialized.
    It will look for fixable tasks and after asking for your consent, it will run the fix command that is provided from the task.
    GrumPHP will not add any new files to GIT or won't automatically commit these changes.
</p>

<h2>You are in full control!</h2>
<p>
    In previous versions, we only displayed the command line tool that you can use to fix your code.
    We never wanted GrumPHP to change code during a pre-commit or inside a CI cycle.
    Therefore, we decided to fix the code but don't stage the changes to git.
    This way, you can always review the changes that were made before recommitting the code.
</p>

<h2>Don't touch my code!</h2>

<p>
    I can imagine that you don't want GrumPHP to touch your code.
    Luckily, the fixer is configurable:
    You can choose to completely disable it or you can choose to change the default consent answer.
</p>

<p>
{% highlight yaml %}
# grumphp.yaml

grumphp:
    fixer:
        enabled: true
        fix_by_default: false
{% endhighlight %}
</p>

<p>
    <em>Note:</em>
    The `fix_by_defaut` parameter will also be used in situations where CLI input is not supported.
    Depending on your CLI, this could be e.g. during pre-commit. 
</p>

<p>
Enjoy and let me know what you think of it!
</p>
