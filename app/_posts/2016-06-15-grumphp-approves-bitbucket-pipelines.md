---
layout: post
title:  "GrumPHP approves Bitbucket Pipelines!"
category: general
tags: php grumphp docker
summary: "Today, my invitation for Bitbucket Pipelines Beta got accepted! For those who haven't heard about it: Pipelines is a Continuous Delivery tool integrated in Bitbucket. When you push your changes to your Bitbucket repository, it will spin up a Docker container and run the actions you specify. I was quite exited about the concept and immediately started to set up an automated test scenario with GrumPHP."
image: 20160615/grumphpipelines.jpg
thumb: 20160615/thumb-grumphpipelines.jpg
---

<p>
    Today, my invitation for 
    <a href="https://bitbucket.org/" target="_blank">Bitbucket</a>
    Pipelines Beta got accepted!
    For those who haven't heard about it: 
    <a href="https://bitbucket.org/product/features/pipelines" target="_blank">Pipelines</a> 
    is a Continuous Delivery tool integrated in Bitbucket.
    When you push your changes to your Bitbucket repository, it will spin up a 
    <a href="http://docker.io/" target="_blank">Docker</a> 
    container and run the actions you specify.
    I was quite exited about the concept and immediately started to set up an automated test scenario with 
    <a href="https://github.com/phpro/grumphp" target="_blank">GrumPHP</a>.
</p>

<h2>Configuration</h2>

<p>
    The configuration of Pipelines is similar to existing services like e.g. 
    <a href="https://travis-ci.org/" target="_blank">Travis</a>.
    It all starts by adding a 
    <a href="https://confluence.atlassian.com/bitbucket/configure-bitbucket-pipelines-yml-792298910.html" target="_blank">bitbucket-pipelines.yml</a> 
    file to the root of your project. 
</p>

<p><strong>bitbucket-pipelines.yml:</strong></p>

{% highlight yaml %}
image: php:7.0-cli

pipelines:
  default:
    - step:
        script:
          - apt-get update && apt-get install -y git
          - php -r "readfile('https://getcomposer.org/installer');" | php
          - php composer.phar global require hirak/prestissimo
          - php composer.phar install --prefer-dist
          - ./vendor/bin/grumphp run
{% endhighlight %}

<p>
    After every commit, the pipelines file will be loaded.
    Since we've added a default pipeline, it will spin up a basic <a href="https://hub.docker.com/_/php/" target="_blank">PHP 7.0 container</a>.
    Because it's a basic container, we'll have to install git and composer to make sure the test scenario works.
    The installation of these packages will happen on every commit,
    so it is better to create your own container which contains all the tools you need to test your code.
    Finally, the composer dependencies are installed and the GrumPHP command is being triggered.
</p>

<h2>Analysing the results</h2>

<p class="text-center">
    <a href="/images/blog/20160615/pipelines-overview.png" target="_blank"><img src="/images/blog/20160615/pipelines-overview.png" alt="GrumPHP Pipelines" class="img-responsive" /></a>
<p>

<p>
    As you can see, the Pipelines overview page is very clean and focuses on the output of your tasks.
    Every commit will trigger a new run and a new results overview page.
    This makes it very easy to spot problems before they got merged in!
</p>


<h3>Pros</h3>
<ul>
    <li>It is a unique service that integrates completely on your git server!</li>
    <li>The interface is very simple and focuses on the errors.</li>
    <li>Pipelines is integrated in every screen in Bitbucket. You get feedback about the committed code in a wink of an eye.</li>
    <li>It is possible to let Pipelines handle your Continuous Delivery flow!</li>
    <li>You can specify different steps per git branch.</li>
    <li>It works perfectly fine with GrumPHP!</li>
    <li>It is FREE!</li>
</ul>


<h3>Cons</h3>

<ul>
    <li>The docker image is loaded from <a href="https://hub.docker.com" target="_blank">DockerHub</a> which takes a while.</li>
    <li>You can only specify 1 image, which means you cannot test every commit against multiple PHP versions.</li>
    <li>It is not possible to cache the composer packages. All dependencies are downloaded on every commit.</li>
</ul>

<h2>Conclusion</h2>

<p>
    Bitbucket has done a really great job creating ther unique Pipelines service! 
    Currently it is still in Beta which means they are still working on making it even better. 
    I hope to see some additional features like running tests on multiple images, caching volumes, ... in a near future.
    In a next phase, I will be playing around with the Continuous Delivery options and maybe link Pipelines 
    with an 
    <a href="https://www.ansible.com/" target="_blank">Ansible</a>
    script to automatically deploy the changes to a development server.
</p>
