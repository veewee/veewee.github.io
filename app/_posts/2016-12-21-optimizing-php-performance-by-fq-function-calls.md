---
layout: post
title:  "Optimizing PHP performance by using fully-qualified function calls"
category: general
tags: php performance
summary: "Today, a little conversation on Twitter escalated rather quickly. Apparently PHP runs function calls differently depending on namespaced or non namespaced context. When calling functions in a namespaced context, additional actions are triggered in PHP which result in slower execution. In this article, I'll explain what happens and how you can speed up your application."
image: 20161221/speed.jpg
thumb: 20161221/thumb-speed.jpg
---


<p>
    Today, a little conversation on Twitter escalated rather quickly.
    Apparently PHP runs function calls differently depending on namespaced or non namespaced context.
    When calling functions in a namespaced context, additional actions are triggered in PHP which result in slower execution.
    In this article, I'll explain what happens and how you can speed up your application.
</p>

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Wondering how much <a href="https://twitter.com/hashtag/PHP?src=hash">#PHP</a> I could break by removing the local namespace lookup semantics for namespaced vs. core functions...</p>&mdash; Reviewed, BLYATIFUL! (@Ocramius) <a href="https://twitter.com/Ocramius/status/811504929357660160">December 21, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<p>
    The conversation started with the tweet above. 
    To understand the difference between global and namespaced function calls better, I'll explain what is going on under the hood.
</p>


<h2>Calling functions in the global namespace</h2>

Function calls in the global namespace look like this:

{% highlight php %}
<?php
// global.php

function foo() {
    echo 'bar';
}

call_user_func('foo');
{% endhighlight %}

After parsing this script, the opcodes look like this:

{% highlight sh %}
$ php -d vld.active=1 -d vld.execute=0 global.php

...
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   4     0  E >   EXT_STMT
         1        NOP
   8     2        EXT_STMT
         3        EXT_FCALL_BEGIN
         4        SEND_VAL                                                 'foo'
         5        DO_FCALL                                      1          'call_user_func'
         6        EXT_FCALL_END
         7      > RETURN                                                   1

...
{% endhighlight %}

As you can see, this is a simple `EXT_FCALL` sequence of opcodes.


<h2>Calling functions in a namespace</h2>

Function calls in a namespace look like this:

{% highlight php %}
<?php
// namespaced.php

namespace baz;

function foo() {
    echo 'bar';
}

call_user_func('foo');
{% endhighlight %}

After parsing this script, the opcodes look like this:

{% highlight sh %}
$ php -d vld.active=1 -d vld.execute=0 global.php

...
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   4     0  E >   NOP
   6     1        EXT_STMT
         2        NOP
  10     3        EXT_STMT
         4        INIT_NS_FCALL_BY_NAME
         5        EXT_FCALL_BEGIN
         6        SEND_VAL                                                 'foo'
         7        DO_FCALL_BY_NAME                              1
         8        EXT_FCALL_END
         9      > RETURN                                                   1
...
{% endhighlight %}

The opcodes look pretty much the same as in the global namespace. 
However, there is one additional opcode added: `INIT_NS_FCALL_BY_NAME`.
When PHP runs over this opcode, it will check if the function `call_user_func()` is found in the namespace.
If the function exists in the namespace, PHP will run this one.
When the function does not exist in current namespace, PHP will check if it exists in the global namespace and execute that one.

This handy "feature" is frequently (ab)used during testing.
A good example for this (ab)use is overwriting the functions to read from or write to the filesystem.
In the source files, you can for example use the `fopen()` function.
During the tests, you can mock this function by placing it in the namespace of the class that you are testing.
An example of this can be found in the [local adapter of flysystem](https://github.com/thephpleague/flysystem/blob/master/tests/LocalAdapterTests.php).


<h2>Benchmarks</h2>

One of the next tweets stated that a performance gain of 4.5% was made by using fully qualified function calls.
Of course, this is just a non-proven number and depends on the project you are working on.
To make sure that I am not writing nonsense, I made a little benchmark in PHP 7.1:

{% highlight php %}
$ php -v

PHP 7.1.0 (cli) (built: Dec  2 2016 11:29:13) ( NTS )
Copyright (c) 1997-2016 The PHP Group
Zend Engine v3.1.0-dev, Copyright (c) 1998-2016 Zend Technologies
{% endhighlight %}

Next, I wrote code that can be run with the [phpbench tool](https://github.com/phpbench/phpbench).
There are 4 cases I covered:

- Run a global function in a fully qualified way.
- Run a global function in a non-fully qualified way.
- Run an overridden function that exists globally and in the namespace.
- Run a namespaced function.

I've chosen a rather big amount of revs and iterations to make sure the results are accurate.
The code in the benchmark looks like this:

{% highlight php %}
<?php

namespace {
    function a(){}
    function b() {}
}

namespace foo {

    function b() {}
    function c() {}

    /**
     * @Revs(10000)
     * @Iterations(100)
     */
    class MixedBench
    {
        public function benchFqGlobalFunction()
        {
            \a();
        }

        public function benchGlobalFunction()
        {
            a();
        }

        public function benchOverriddenFunction()
        {
            b();
        }

        public function benchNamespacedFunction()
        {
            c();
        }
    }
}
{% endhighlight %}

This is an overview of the results:

{% highlight sh %}
$ phpbench run bench

\foo\MixedBench

    benchFqGlobalFunction         I99 P0 	[μ Mo]/r: 0.145 0.141 (μs) 	[μSD μRSD]/r: 0.018μs 12.59%
    benchGlobalFunction           I99 P0 	[μ Mo]/r: 0.148 0.145 (μs) 	[μSD μRSD]/r: 0.021μs 14.38%
    benchOverriddenFunction       I99 P0 	[μ Mo]/r: 0.157 0.157 (μs) 	[μSD μRSD]/r: 0.022μs 13.82%
    benchNamespacedFunction       I99 P0 	[μ Mo]/r: 0.157 0.159 (μs) 	[μSD μRSD]/r: 0.019μs 12.38%

4 subjects, 400 iterations, 40,000 revs, 0 rejects
(best [mean mode] worst) = 0.124 [0.152 0.151] 0.225 (μs)
⅀T: 60.773μs μSD/r 0.020μs μRSD/r: 13.293%
suite: 133a2c5566a4e9fb57b0251cebfd189bc150f104, date: 2016-12-21, stime: 22:06:03

+-------------------------+-------+-----+---------+--------+
| subject                 | revs  | its | mean    | diff   |
+-------------------------+-------+-----+---------+--------+
| benchFqGlobalFunction   | 10000 | 100 | 0.145μs | 0.00%  |
| benchGlobalFunction     | 10000 | 100 | 0.148μs | +2.26% |
| benchNamespacedFunction | 10000 | 100 | 0.157μs | +8.55% |
| benchOverriddenFunction | 10000 | 100 | 0.157μs | +8.38% |
+-------------------------+-------+-----+---------+--------+
{% endhighlight %}

As expected, the fully qualified global function call is the fastest one. 
This is because PHP does not need to go through the `INIT_NS_FCALL_BY_NAME` opcode.
When calling the global function in a non-fully qualified way, it is slower.
Running functions inside a namespace are always slower then running global functions.

Of course, this is not a big overhead in this simple benchmark.
It could be a big overhead if you think about the amount of function calls per run.
PHP is not able to optimize this since it is possible that functions get defined during runtime.


<h2>Speeding up your application</h2>

Currently the only way to speed up the function calls, is by making sure that the global functions are called in a fully qualified way.
This is a tedious manual action. You could do this in one of these 2 ways:

{% highlight php %}
<?php
// solution 1:
namespace baz;
\foo();

// solution 2:
namespace baz;
use function foo;
foo();
{% endhighlight %}

Luckily for us, the community is very creative and alert when it comes to performance.
In the future, maybe one of the following solutions is less boring to implement:

- Add a `declare(no_dynamic_functions=1)` on top of the PHP file or maybe in a future [`namespace_scoped_declares`](https://wiki.php.net/rfc/namespace_scoped_declares) method.
- [Autocompletion to FQ function names in PHPStorm.](https://youtrack.jetbrains.com/issue/WI-34446)


<h2>Conclusion</h2>

<p>
    As you can see, performance killers can hide in small corners.
    I'm glad to see how this little tweet can get this much feedback from the community in no time.
    Let's hope that a good solution for this performance problem will be added to PHP soon so that we can easily speed up our applications even more.
    Now that the secret behind this optimization is revealed, I am looking forward to discover more of these little performance killers.
</p>
