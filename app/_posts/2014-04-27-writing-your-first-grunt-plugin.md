---
layout: post
title:  "Writing your first Grunt plugin"
category: general
tags: grunt plugin
summary: For one of my projects, I needed a tool to list all used translations keys from a PHP project. Because I want to keep the translation database clean, I decided to create a Grunt plugin. There was one big problem... How do you create a Grunt plugin? So I started to read the documentation, blog posts, etc. and noticed that the documentation was pretty confusing for beginners like me. Let me take you on a tour through my first plugin.
image: 20140427/grunt.jpg
thumb: 20140427/thumb-grunt.jpg
---

For one of my projects, I needed a tool to list all used translations keys from a PHP project.
  Because I want to keep the translation database clean, I decided to create a
  <a href="http://gruntjs.com/" target="_blank">Grunt</a> plugin.
  There was one big problem: How do you create a Grunt plugin?
  So I started to read the documentation, some blog posts, etc. and noticed
  that the documentation was pretty confusing for beginners like me.
  Let me take you on a tour through my first plugin.

## Where to start?

### Create the plugin skeleton
The Grunt team has created a tool called <a href="http://gruntjs.com/project-scaffolding" target="_blank">grunt-init</a>.
  It's purpose is to quickly scaffold your project with some basic templates.
  The tool does not include any templates, so you have to download them manually.
  There is a tempalte for creating a
  <a href="https://github.com/gruntjs/grunt-init-gruntplugin" target="_blank">grunt-plugin</a>
  and another one to add a
  <a href="https://github.com/gruntjs/grunt-init-gruntfile" target="_blank">Gruntfile</a>
  to your project.
  When you run the tool, a wizard is launched to personalize the template.

{% highlight sh %}
$ grunt-init gruntplugin
{% endhighlight %}

After running this command, all the template files are being copied to your folder.
  The structure is pretty straight forward.
  There is a `tasks` folder which contains the Grunt tasks.
  Another folder is the `test` folder. This is the place where you write some node tests for your plugin.
  Finally, there is also a Gruntfile which will help you with some basic tasks like testing the plugin.

### Configure NPM
Once you got through the installation wizard, you should personalize the NPM file.
  My plugin is heavy project specific, so I decided to make it a private repository.
  I also like using
  <a href="http://lodash.com/" target="_blank">Lo-Dash</a>
  for handling arrays, so I added this one as a dependency of the plugin.
  All these parameters can be configured in the `package.json` file.

{% highlight js %}
  "private": true,
  "dependencies": {
    "lodash": "~2.4.1"
  },
{% endhighlight %}

When the configuration is saved, the next step is to install the dependencies. This can be done by running the command:

{% highlight sh %}
$ npm install
{% endhighlight %}


## Writing the task

### Task structure
You can add one or more tasks to your plugin. By default there is one
  <a href="http://gruntjs.com/creating-tasks#multi-tasks" target="_blank">multi-task</a>
  in the tasks folder.
  It looks like this:

{% highlight js %}
'use strict';

module.exports = function(grunt) {
  grunt.registerMultiTask('findTranslations', 'Find all translations in your project.', function() {

    // The task code comes here

  };
};
{% endhighlight %}

### Initialize dependencies
Grunt has some standard tools to handle
  <a href="http://gruntjs.com/api/grunt.file" target="_blank">file</a> actions and
  <a href="http://gruntjs.com/api/grunt.log" target="_blank">logging</a>.
  When you want to use custom libraries, you can include them through
  <a href="http://requirejs.org/" target="_blank">require</a>.

{% highlight js %}
    var _ = require('lodash');
{% endhighlight %}

### Options
If you ever worked with Grunt before, you know that all commands are configurable.
  Some of the options will have a default value.
  In the plugin, you can add custom <a href="http://gruntjs.com/api/grunt.option">options</a> as followed:

{% highlight js %}
    var options = this.options({
      format: 'json',
    });
{% endhighlight %}

The options in the object are the default options and can be overwritten in the Gruntfile.

### Files
Next to the options, there is a basic configuration parameter to specify source and destinations files.
The configuration looks like:

{% highlight js %}
  files: [
    {
      src: [
        'test/fixtures/php-controller-plugin',
      ],
      dest: 'tmp/translations-json'
  }
],
{% endhighlight %}

As you can see, it is possible to add multiple src / dest collections.
  The source can consist out of multiple files and the names can contain regular expressions like wildcards.
  In the plugin, it is very easy to loop through the files:

{% highlight js %}
    this.files.forEach(function (f) {

      var translations = [];
      f.src.forEach(function(filePath) {
        var content = grunt.file.read(filePath);
        // Todo: Find translation keys
      });

      grunt.file.write(f.dest, translations.join("\n");
    });
{% endhighlight %}

### Locating translation keys
To find the translation keys in a file, I needed to write some regular expressions to find the key.
  Luckily for me, I use a very basic translation method.
  In my project I have 2 different ways of retrieving translations: one for the controller and one for the view.

{% highlight js %}
    var translationKeyRegex = "[\s]?.*[\\\'\\\"]{1}([a-z0-9\\-\\_]*)[\\\'\\\"]{1}[\s]?.*";
    var regexs = {
      'phpControllerPlugin': "translator->get\\(" + translationKeyRegex + "\\)",
      'phpViewHelper': "_t->get\\(" + translationKeyRegex + "\\)"
    };
{% endhighlight %}

This may look a bit hard at first, but it is actually very easy. There are only 2 patterns:

- In the controller: `$translator->get('KEY_123');`
- In the view: `_t->get('KEY_123');`

After the regular expressions were written, I created a function that returns the translation keys:
{% highlight js %}
    var findTranslationKeys = function(content, regex) {
      var found = [];
      var match;

      while ((match = regex.exec(content)) !== null) {
        found.push(match[1]);
      }

      return found;
    };
{% endhighlight %}

Okay, the hard part of the plugin is written.
  Now the only thing left is adding this functionality to the part where we read the file.
  I want to execute all regular expressions patterns on every file.
  The resulting array should only contain unique entries.
  This can easily be done with the Lo-Dash forEach and union method:

{% highlight js %}
  var found, regex;
  _.forEach(regexs, function(expression) {
    regex = new RegExp(expression, 'gi');
    found = findTranslationKeys(content, regex);

    // Only use unique translation keys:
    translations = _.union(translations, found);
  });
{% endhighlight %}

### Formatting the output
As you can see in the files part, at the moment we only output the result as newline-separated text.
  Because I want to clean up my database through PHP, I decided to make JSON the default format.
  To specify which format will be used I added one other function:

{% highlight js %}
    var formatTranslations = function(translations, format) {
      var formatted = '';
      switch(format) {
        case 'newlines':
          formatted = translations.join("\n");
          break;
        case 'json':
          formatted = JSON.stringify(translations, null, 4);
          break;
        default:
          formatted = 'Invalid export format';
          break;
      }
      return formatted;
    };
{% endhighlight %}

The translations parameter is the array of found translation keys and the format is configured in the `options.format` variable.

### Logging
At the end of the command, I find it useful to log the results of the task.
 In this case, I only print an indication of the amount of found items.

{% highlight js %}
    grunt.log.writeln('Saved ' + translations.length + ' translation keys to file ' + f.dest);
{% endhighlight %}

## Testing
For now, I only talked about writing the plugin.
  Of course it is also recommended to write some test for your plugin.
  As far as I can see in other plugins, there are only functional tests.
  In the test directory there are 2 folders.
  The `fixtures` folder is where you add the files where you want to find translation keys in.
  In the `expected` folder you place the output file that you expect the plugin to generate.

Now the first thing that you need to do, is to configure your task in the Gruntfile.
  When you run the test command, your configured tasks will run first.
  So in the tasks you can configure to run the tasks with the fixtures file and save it in a `tmp` folder.
  When you know which files are being read and written, it is easy to create your test:

{% highlight js %}
  json_format: function(test) {
    test.expect(1);

    var actual = grunt.file.read('tmp/translations-json');
    var expected = grunt.file.read('test/expected/json_format');
    test.equal(actual, expected, 'should return translations in json.');

    test.done();
  },
{% endhighlight %}

Now the only command you need to run to test your code is:
{% highlight sh %}
$ grunt test
{% endhighlight %}

## Deployment
Once your plugin is ready, you want to place it in GIT.
  Because this is a private repository, I placed it on a private repository on BitBucket.
  Now the next problem arose: How can I load my custom plugin through NPM in another project?
  In the latest versions of NPM, it is possible to add packages that are not managed on NPM.
  The real problem is that it is a private repository.
  After some searching, I found out that the only good way the require the package is to use the SSH url of the package.
  This is what I placed in the `package.json` file of the main project.

{% highlight js %}
  "devDependencies": {
    "grunt-find-bo-translations": "git+ssh://git@bitbucket.org:organization/package.git"
  }
{% endhighlight %}

**Note:** Don't forget to `npm install`

In the Gruntfile of the project, I configured the task to search for translations in all .php and .phtml files.
  The result of the tasks will be saved in the logs folder.

{% highlight js %}
    findTranslations: {
      json_format: {
        options: {
            format: 'json',
        },
        files: [
          {
            src: [
              '{,*/}*.php',
              '{,*/}*.phtml',
            ],
            dest: 'logs/translation-keys.json'
          }
        ],
      },
    },
{% endhighlight %}

Finally my task was ready to use in my project:
{% highlight sh %}
$ grunt findTranslations

>> Running "findTranslations:json_format" (findTranslations) task
>> Saved 250 translation keys to file logs/translation-keys.json
{% endhighlight %}
