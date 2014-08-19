---
layout: post
title:  "Generating ZF2 template maps on the fly"
category: general
tags: blog grunt zf2
summary: One of the performance boosters in Zend Framework 2 is using template_map instead of template_path_stack in the view manager. While developing, it is easier to use the stack. This is because you don't have to add a mapping every time you create a new view. At the end of the development you might forget to transform the stack into a template map. This is why it is useful to <strong>never</strong> use the stack and automate the generation of the template_map array instead.
image: 20140819/auto-template-mapping.jpg
thumb: 20140819/thumb-auto-template-mapping.jpg
---

<p>
    One of the performance boosters in
     <a href="http://framework.zend.com/" target="_blank">Zend Framework 2</a>
     is using `template_map` instead of the `template_path_stack` in the view manager.
     While developing, it is easier to use the stack.
     This is because you don't have to add a mapping every time you create a new view.
     At the end of the development you might forget to transform the stack into a template map.
     This is why it is useful to <strong>never</strong> use the stack and automate the generation of the template_map array instead.
</p>
<p>
    There is already a great tool called
     `<a href="https://github.com/zendframework/zf2/blob/master/bin/templatemap_generator.php" target="_blank">templatemap_generator.php</a>`
     included in the bin folder of zf2.
     It can be used to generate a template map for every module in your application.
     In the next step, we want to make sure that this tool is called every time a view file is created.
     This kind of automation can be done with a task manager like
     `<a href="http://gruntjs.com/" target="_blank">Grunt</a>`.
</p>

## Sample configuration

{% highlight js %}
grunt.initConfig({

    phplint: {
      applicationViews: ['module/Application/**/*.phtml'],
    }

    watch: {
      applicationViews: {
        files: ['module/Application/view/**/*.phtml'],
        tasks: ['phplint:applicationViews', 'exec:generate_application_template_map']
      }
    },

    exec: {
      generate_application_template_map: {
        cmd: '../../vendor/bin/templatemap_generator.php',
        cwd: 'module/Application'
      }
    }

});

grunt.registerTask('generate-template-maps', ['exec:template_map_application']);
{% endhighlight %}

<p>
    By using this configuration, Grunt will watch for file changes in the `view` folder of the `Application` module.
     When a file has changed, PHPLint will check for syntax errors and finally a new template map file will be generated.
     There is also a command named `generate-template-maps` which you can run manually.
</p>

<p>
    <em>
        <strong>Note:</strong>
        Make sure to include the npm packages
        <a href="https://www.npmjs.org/package/grunt-exec" target="_blank">grunt-exec</a>
        and
        <a href="https://npmjs.org/package/grunt-phplint" target="_blank">grunt-phplint</a>.
    </em>
</p>

## Conclusion

<p>
    By automating all of the boring tasks, you can gain a lot of time during development.
     If you want to automate some more tasks during PHP development, you can check an earlier blog called
     `<a href="http://veewee.github.io/blog/validating-your-php-code-on-the-fly/">Validating your PHP code on the fly</a>`.
</p>