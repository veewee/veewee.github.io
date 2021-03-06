---
layout: post
title:  "Grand Opening!"
category: general
tags: blog jekyll yeoman grunt
summary: With a new year, come some new resolutions. I was already thinking of creating a blog for quite some time, but the new year forced me to finally get this site up and running. So why did I start this blog? Not to tell you who I am, but to inform you about some cool stuff I encounter during the development of web applications.
image: 20140114/grand-opening.jpg
thumb: 20140114/thumb-grand-opening.jpg
---

<p>
    With a new year, come some new resolutions. I was already thinking of creating a blog for quite some time,
    but the new year forced me to finally get this site up and running. So why did I start this blog?
    Not to tell you who I am, but to inform you about some cool stuff I encounter during the development of web applications.
</p>
<p>
    Just another blog you think? Ok that's true ... Except ...<br />
    Last year I found myself digging in the latest subjects of Web Development.
    Some of these subjects are poor documented and are not handled by many blog articles.
    My goal is to help you fix the issues that I encountered while using open-source modules.
    That, and off course to tell you more about some cool tools / projects I found.
</p>

<p>
    Let's not let my first blog article go to waste and add some technical information about this blog.
</p>

## Choose the right platform

<p>
    The first problem while creating this blog, was deciding which platform to use.
    At first there were the obvious platforms like Wordpress or some well-known hosted services like Blogger.
    Because I rather didn't want to use these platforms, I went on a blog-platform-quest.
    After a while I bumped into <a href="http://jekyllrb.com/" target="_blank">Jekyll</a>.
    This Ruby based static site generator is easy to use and is perfect for building something as simple as this blog.
    Extra bonus: you can deploy to github pages instead of wasting your own web space!
</p>
<p>
    Off course, there were dozens of people who already worked with Jekyll before me.
    That is Why I searched for some good articles about building a blog for GitHub pages.
    One of the results I found was the
    <a href="http://erjjones.github.io/blog/How-I-built-my-blog-in-one-day/" target="_blank">&quot;How to build a blog in one day&quot;</a>
    article by
    <a href="http://erjjones.github.io/" target="_blank">Eric Jones</a>.
    This was a good place to start, but after some boring repeated tasks like the configuration of tools,
    I found a great tool to build this site:
</p>

## Installation

<p>
    By installing the
    <a href="https://github.com/robwierzbowski/generator-jekyllrb" target="_blank">jekyllrb</a>
    generator for
    <a href="http://yeoman.io/" target="_blank">yeoman</a>, all the work was done for me. Following items were configured:
    <ul>
        <li>Files structure</li>
        <li>Grunt configuration</li>
        <li>Compass</li>
        <li>File watcher - which is much better than the default Jekyll watcher!</li>
        <li>LiveReload</li>
        <li>Jekyll configuration options</li>
        <li>Bower: Twitter Bootstrap / jQuery / ...</li>
    </ul>

    All that it took to let me focus on the lay-out and content, were 2 commands:
</p>

{% highlight sh %}
~$ npm install -g generator-jekyllrb
~$ yo jekyllrb
{% endhighlight %}

## Development
<p>
    The real power of this generator, is the integration with <a href="gruntjs.com">grunt</a>.
    Several handy tasks are pre-configured for you to use. For example:
</p>

{% highlight sh %}
~$ grunt serve
{% endhighlight %}

<p>
    This command will launch the Jekyll server and will rebuild the static website on file changes.
    When the static site is build, the livereload task will refresh your browser, so you can validate your changes right away.
    Also your compass files will be automatically converted to usable CSS files.
</p>

## Publishing
<p>
    So, you finished your blog and want to place it on github.io?
    I forgot to tell you there is one little problem left.
    Github.io uses the root of the master branch to find your index.html file.
    Because yeoman creates your site in a subfolder called "app", github.io will not find your files.
    That is why we need to separate our development code from the final site.
    The distribution version will be placed on the 'master' branch and the main code will be placed on a separate branch named 'develop'.
    In GIT, this can be done by creating an <a href="http://www.git-tower.com/files/applicationHelp/pgs/Refs_Branches_DetachedOrphaned.html">'orphan'</a> branch, in this case named 'develop'.
    When you finished your blog, the final site should be committed to 'master' and the source code should be committed to 'develop'.
    I hear you say: So much work to deploy my blog?
    Fortunately there is a grunt tool named <a href="https://npmjs.org/package/grunt-gh-pages">gh-pages</a> which does the job.
    I used following configuration which will commit my 'dist' folder of the 'develop' branch to the 'master' branch in git.
</p>

{% highlight js %}
'gh-pages': {
  options: {
    base: 'dist',
    branch: 'master',
    message: 'Auto-generated build',
  },
  src: '**/*'
}

grunt.registerTask('github-deploy', [
    'default',
    'gh-pages'
]);

{% endhighlight %}

<p>
    Now, when you want to publish your site, it's just one simple command:
</p>
{% highlight sh %}
~$ grunt github-deploy
{% endhighlight %}
<p>
    This will first clean up all temporary files and validate your Jekyll site, javascript and compass files.
    When everything is good to go, it will build the static version of your website for distribution.
    The files in the distribution folder will be minimized and optimized for quick access.
    After all actions are done, the optimized version will be pushed to your master branch, using the gh-pages tool in grunt.
</p>
## Conclusion
<p>
    With these easy to use, but incredibly powerful tools,
    I hope to focus myself on writing articles about some interesting topics.
    If you want to take a look at my configuration or want to re-use it for your own project,
    feel free to fork <a href="http://github.com/veewee/veewee.github.io">veewee.github.io</a>
    from my <a href="http://github.com/veewee/" target="_blank">GitHub</a> account.
</p>
