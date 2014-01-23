---
layout: post
title:  "Validating your PHP code on the fly"
category: general
tags: grunt phpspec zf2 php
summary: With a new year, come some new resolutions. I was already thinking of creating a blog for quite some time, but the new year forced me to finally get this site up and running. So why did I start this blog? Not to tell you who I am, but to inform you about some cool stuff I encounter during the development of web applications.
image: 20140121/grunt-php.jpg
thumb: thumb-grunt.jpg
---

<p>
    It's amazing how much time you can loose while running PHP tests.
    First you alter your code or test. Next you start your test and wait for it to finish.
    Finally you find out you messed up and start the process all over again.
    In this article I am going to show you how to automate PHP testing,
    so you can focus on adding business value to your application.
</p>
<p>
    For this article I created a simple
    <a href="http://framework.zend.com/" target="_blank">Zend Framework 2</a>
    application with
    <a href="http://www.phpspec.net/" target="_blank">Phpspec</a> tests.
    Even though this article focuses on these 2 packages,
    it is possible to create this kind of automation for every possible package.
</p>


## Application structure
<p>
    First, let's take a look at the directory structure.
    In the representation below, you can see a default ZF2 skeleton.
    The root folder contains the directories config, module, public and vendor.
    The `config` folder provides the application configuration.
    In the `module` folder you can see 2 modules that should be tested: Export and Import.
    Next you can find the public directory which will bootstrap the application.
    And finally there is the vendor folder that will be filled with dependencies by
    <a href="https://getcomposer.org/" target="_blank">Composer</a>.
</p>
<pre>
.
|-- config
|   |-- application.config.php
|-- module
|   |-- Export
|   |   |-- config
|   |   |-- spec
|   |   |-- src
|   |   |  `-- Export
|   |   |       |-- Service
|   |   |       `-- Module.php
|   |   |-- view
|   |   `-- Module.php
|   `-- Import
|       |-- config
|       |-- spec
|       |-- src
|       |  `-- Import
|       |       |-- Service
|       |       `-- Module.php
|       |-- view
|       `-- Module.php
|-- public
|-- vendor
</pre>

## Configuring composer
<p>
    Because nobody wants to download all the dependencies manually, we will be using composer to get the required packages.
    We also want to configure composer to automatically load our custom modules: Export and Import.
    This way, no additional autoloading is required and Phpspec will be able to find all classes by itself.
</p>

<p><strong>composer.json</strong></p>

<pre class="prettyprint lang-js">
{
    "name": "veewee/validating-on-the-fly",
    "require": {
        "zendframework/zendframework": "2.*",
    },
    "require-dev": {
        "phpspec/phpspec": "dev-master",
        "fabpot/PHP-CS-Fixer": "*",
    },
    "minimum-stability": "dev",
    "prefer-stable": true,
    "autoload": {
        "psr-0": {
          "Export": "module/Export/src/",
          "Import": "module/Import/src/"
        }
    }
}
</pre>
<p>
    As you can see, composer will download Zend Framework 2 to the vendor directory.
    When you add the option `--dev` during the installation, Phpspec and PHP-CS-Fixer will also be downloaded.
    The last part of this configuration file will provide autoloading for the custom Export and Import module.
    To download and install all the dependencies, one command is being used:
</p>
<pre class="prettyprint lang-sh">
~$ composer install --dev --prefer-dist
</pre>

## Configuring phpspec
<p>
    As you can see in the directory overview, there is already a spec folder available.
    The Phpspec tests will be placed in this directory.
    This way, every module has it's own scope and can be tested separately.
</p>
<p>
    Now the tricky part: let's tell Phpspec where to find the custom source and spec files.
    Because we need to test multiple modules on multiple locations, we need to add multiple test suites to Phpspec.
    This is done by creating a phpspec.yml file in the root folder of the application.
    A test suite describes where the source and spec files of a custom namespace are located.
    Let's take a look at the configuration file for our application:
</p>
<p><strong>phpspec.yml</strong></p>
<pre class="prettyprint lang-yml">
formatter.name: dot
suites:
  Export:
    namespace: Export
    src_path: 'module/Export/src/'
    spec_path: 'module/Export/'

  Import:
    namespace: Import
    src_path: 'module/Import/src/'
    spec_path: 'module/Import/'
</pre>

<p>
    One of the downsides of Phpspec is that, at the moment, you can not run suites based on suite name.
    To test this configuration, you can run following commands:
</p>
<pre class="prettyprint lang-sh">
~$ ./vendor/bin/phpspec run module/Export/src
~$ ./vendor/bin/phpspec run module/Import/src
</pre>

## Configuring php-cs-fixer
<p>
    Everybody wants clean, readable and maintainable code.
    That is why the code of the modules should be validated against the desired PHP standards.
    In this case, we want to check our classes against the
    <a href="https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md" target="_blank">PSR2</a>
    standard.
    Per module, you should add a file called .php_cs in the root directory of the module.
    In this file you can tell the
    <a href="https://github.com/fabpot/PHP-CS-Fixer" target="_blank">Php-CS-Fixer</a>
    which paths to check and add some default configuration settings.
</p>
<p>
    <strong>.php_cs</strong>
</p>
<pre class="prettyprint lang-php">
&lt;?php
$finder = Symfony\CS\Finder\DefaultFinder::create()
    ->exclude('language')
    ->exclude('view')
    ->exclude('config')
    ->in(__DIR__ . '/src')
    ->in(__DIR__ . '/spec');

$config = Symfony\CS\Config\Config::create();
$config->fixers(Symfony\CS\FixerInterface::PSR2_LEVEL);
$config->finder($finder);
return $config;
</pre>

<p>
    To test this configuration, you can run following commands:
</p>
<pre class="prettyprint lang-sh">
~$ ./vendor/bin/php-cs-fixer fix module/Export --dry-run
~$ ./vendor/bin/php-cs-fixer fix module/Import --dry-run
</pre>

## Automate the testing process
<p>
    Before the automation, every time you change a file in your module,
    you need to run Phpspec and occasionally the Php-CS-Fixer to validate your code.
    Because you have to define which module to check every time you run the command,
    you will spend some time searching for the right command in your console.
    A better way is to automate this process, so that a file change will trigger the command to test your code.
    This will save you some time and will give you an indication of when and where you messed up.
    What was that tool again to automate tasks? Exactly:
    <a href="http://gruntjs.com/" target="_blank">Grunt</a>!
</p>
<p>
    There are some grunt plugins that can be used to automate PHP validation.
    In this article we are using
    <a href="https://npmjs.org/package/grunt-phpspec" target="_blank">Phpspec</a>,
    but there are also plugins for other testing libraries like
    <a href="https://npmjs.org/package/grunt-phpunit" target="_blank">Phpunit</a>
    and
    <a href="https://npmjs.org/package/grunt-parallel-behat" target="_blank">Behat</a>.
    In the next part, I will tell you how to test your code on syntax errors,
    coding standard errors and bad functionality.
</p>

### Grunt-Phpspec
<p>
    For validating bad functionality we use
    <a href="https://npmjs.org/package/grunt-phpspec">grunt-phpspec</a>.
    This tool will run Phpspec on our custom modules.
    For every PHP module we need to check, an entry in the configuration is required.
    Here is the sample configuration for the current project:
</p>

<pre class="prettyprint lang-js">
phpspec: {
  options: {
    prefix: './vendor/bin/'
  },
  export: {
    specs: 'module/Export/src'
  },
  import: {
    specs: 'module/Import/src'
  }
}
</pre>

<p>
    To run spec tests, you can use following commands:
</p>
<pre class="prettyprint lang-sh">
~$ grunt phpspec
~$ grunt phpspec:export
~$ grunt phpspec:import
</pre>


### Grunt-php-cs-fixer
<p>
    The tool for validating PHP coding standards is
    <a href="https://npmjs.org/package/grunt-php-cs-fixer" target="_blank">grunt-php-cs-fixer</a>.
    As the name suggests: it does not only check for issues, it also fixes them.
    Therefore it must be possible to validate and fix errors with Grunt.
    This can be done by adding the parameter `--fixcs`.
    Here is the sample configuration for the current project:
</p>

<pre class="prettyprint lang-js">
// Place this parameter before grunt.initConfig() method:
var fixCs = grunt.option('fixcs') || false;

// Place this configuration in grunt.initConfig() method:
phpcsfixer: {
  options: {
    bin: 'vendor/bin/php-cs-fixer',
    verbose: true,
    dryRun: !fixCs,
    ignoreExitCode: fixCs,
    level: 'psr2',
    standard: 'Zend'
  },
  export: {
    dir: 'module/Export'
  },
  import: {
    dir: 'module/Import'
  }
}
</pre>

<p>
    To run the cs fixer, you can use following commands:
</p>
<pre class="prettyprint lang-sh">
// Validating:
~$ grunt phpcsfixer
~$ grunt phpcsfixer:export
~$ grunt phpcsfixer:import

// Fixing
~$ grunt phpcsfixer --fixcs
~$ grunt phpcsfixer:export --fixcs
~$ grunt phpcsfixer:import --fixcs
</pre>

### Grunt-Phplint
<p>
    To validate PHP files for syntax errors you can use the commnd `php -l`.
    For automation, there is also a Grunt plugin available named
    <a href="https://npmjs.org/package/grunt-phplint" target="_blank">grunt-phplint</a>.
    This tool is added to the test suite
    to make sure the CS-fixer and spec tests only run when there is no syntax error.
    Here is the sample configuration for the current project:
</p>

<pre class="prettyprint lang-js">
phplint: {
  export: ['module/Export/**/*.php'],
  import: ['module/Import/**/*.php']
}
</pre>

<p>
    To run PHP lint, you can use following commands:
</p>
<pre class="prettyprint lang-sh">
~$ grunt phplint
~$ grunt phplint:export
~$ grunt phplint:import
</pre>


### Grunt-watch
<p>
    Now that we configured all tasks to validate the PHP code,
    we want the tests to run every time we change a class or spec test in our module.
    This is where
    <a href="https://github.com/gruntjs/grunt-contrib-watch" target="_blank">grunt-contrib-watch</a>
    comes in to play.
    This task will listen for file changes and will configured tasks for these files.
    When one of the tasks fail, the other tasks will not run.
    This is why I decided to run the tests in the order: lint, csfixer, phpspec.
    Here is the sample configuration for the current project:
</p>
<pre class="prettyprint lang-js">
watch: {
  export: {
    files: ['module/Export/**/*.php'],
    tasks: ['phplint:export', 'phpcsfixer:export', 'phpspec:export']
  },
  import: {
    files: ['module/Import/**/*.php'],
    tasks: ['phplint:import', 'phpcsfixer:import', 'phpspec:import']
  }
},
</pre>

<p>
    To start the watcher and automate the tests, you only have to use one command:
</p>
<pre class="prettyprint lang-sh">
~$ grunt watch
</pre>


### Run all tests in one
<p>
    For Continious Integration it would be nice if all tests could be run in one command.
    Well off course this is possible! It is just one line of configuration:
</p>
<pre class="prettyprint lang-js">
grunt.registerTask('test', ['phplint', 'phpcsfixer', 'phpspec']);
</pre>
<p>
    To start the complete test, you can use following command:
</p>
<pre class="prettyprint lang-sh">
~$ grunt test
</pre>


## Conclusion
<p>
    As you can see there are some tools to configure.
    Once the tools are configured, this set-up will save you some valuable time while testing your code.
    I highly recommend you to use a similar set-up because it is very easy to use yet very powerful.
    You won't regret!
</p>


