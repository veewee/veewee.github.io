---
layout: post
title:  "Hijacking your code on runtime with streams"
category: general
tags: blog streams
summary: Lately, you might have noticed that streams are getting more important in the PHP scene. They are for example widely used in PSR-7 and implemented in Diactoros. Yesterday I was scrolling down the issues list of phpspec and bumped in on a pull request. This pull request shows that it is possible to convert code in-memory and later require the modified content to be executed. Wow, that is awesome! Let's take a look at it.
image: 20151026/hijacking.jpg
thumb: 20151026/thumb-hijacking.jpg
---

<p>
    Lately, you might have noticed that streams are getting more important in the PHP scene.
    They are for example widely used in 
    <a href="http://www.php-fig.org/psr/psr-7/" target="_blank">PSR-7</a> and implemented in
    <a href="https://github.com/zendframework/zend-diactoros" target="_blank">Diactoros</a>.
    Yesterday I was scrolling down the issues list of <a href="http://phpspec.readthedocs.org/en/latest/" target="_blank">Phpspec</a> 
    and bumped in on <a href="https://github.com/phpspec/phpspec/pull/788" target="_blank">this pull request</a>.
    This pull request shows that it is possible to convert code in-memory and later require the modified content to be executed.
    Wow, that is awesome! Let's take a look at it.
</p>

## The concept
<p>
    You probably already used some build-in PHP streams like regular files, `php://memory` or `data://...` 
    in combination with the `fopen()` method or the `StdFileObject` class.
    Beside those build-in streams, it is possible to configure your own stream wrappers.
    For this post I will be creating a blacklist stream that will remove blacklisted methods from your code.
    The blacklist stream wrapper will read a PHP file and replaces blacklisted methods from the code.
    When the file is required through the blacklist wrapper, all blacklisted methods will be removed from the code.
</p>


## Show me some code

{% highlight php %}
<?php

class StreamWrapper
{

    private $realPath;
    private $fileResource;
    private static $blacklistedMethods = [];

    public static function blacklistMethod($method)
    {
        self::$blacklistedMethods[] = $method;
    }

    public function stream_open($path, $mode, $options, &$opened_path)
    {
        $this->realPath = preg_replace('|^blacklist://|', '', $path);
        if (!file_exists($this->realPath)) {
            return false;
        }

        $content = file_get_contents($this->realPath);
        foreach (self::$blacklistedMethods as $blacklistedMethod) {
            $content = preg_replace('/^' . $blacklistedMethod . '\([^\)]*\);$/im', '', $content);
        }

        $this->fileResource = fopen('php://memory', 'w+');
        fwrite($this->fileResource, $content);
        rewind($this->fileResource);

        $opened_path = $this->realPath;
        return true;
    }

    public function stream_stat()
    {
        return stat($this->realPath);
    }

    public function stream_read($count)
    {
        return fread($this->fileResource, $count);
    }

    public function stream_eof()
    {
        return feof($this->fileResource);
    }
}

{% endhighlight %}

## Usage

To use the stream wrapper, you have to register it with the name blacklist.
Next you can require the file with the new stream URL. 
Internally PHP will open the file with your custom stream reader.
This means that you can alter the content of the PHP file in an in-memory stream which will be used by PHP.
Here is what the file looks like.

{% highlight php %}
<?php
// run.php

require('StreamWrapper.php');
StreamWrapper::blacklistMethod('die');
stream_wrapper_register('blacklist', 'StreamWrapper');

require 'blacklist://test.php';
{% endhighlight %}

The above code needs a file test.php. This file looks like this:

{% highlight php %}
<?php
// test.php

die('now you see me ...');
echo('now you dont!');
{% endhighlight %}

When executing the `run.php` you will see the result:

{% highlight sh %}
$ php run.php
now you dont!
{% endhighlight %}


## What the hell did I just read...
<p>
    I know, most projects won't need in-memory code adjustments. 
    The blacklist wrapper in this post will probably never be needed, but it is used to show you what is possible.
    Streams are pretty cool and will probably become more important in the future.
</p>
