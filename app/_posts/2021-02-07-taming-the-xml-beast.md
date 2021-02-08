---
layout: post
title:  "Taming the XML beast!"
category: general
tags: XML PHP
summary: "Imagine a world ... in which you didn't have to deal with all PHP's strange XML quirks. The people following me on Twitter might have noticed that I don't like the XML tools that PHP currently provides. My main frustrations with the built-in tools can be divided in 2 big items: The lack of error handling and the bloated APIs."
image: 20210207/xml-beast-long.png
thumb: 20210207/xml-beast_thumb.png
---
  
<p>
Imagine a world ... in which you didn't have to deal with all PHP's strange XML quirks.
The people following me on <a href="https://twitter.com/toonverwerft" target="_blank">Twitter</a> might have noticed that I don't like the XML tools that PHP currently provides.
My main frustrations with the built-in tools can be divided in 2 big items: The lack of error handling and the bloated APIs.
</p>

## Lack of error handling

Most of the time, you'll be working with XML files of which you know they contain valid XML syntax.
In some situations however, you might receive invalid XML or XML that is not valid according to an XSD schema.
At this point, lib-xml gives you following:

{% highlight php %}
<?php

$doc = new DOMDocument();
$result = $doc->loadXML('<invalid>');

var_dump($result);
{% endhighlight %}

Resulting in:

```
Warning: DOMDocument::loadXML():
  Premature end of data in tag invalid line 1 in Entity,
  line: 1 in /in/0I8Or on line 4
  
bool(false)
```

Yes, that is right... It returns false and it triggers ... a PHP warning ...
This means that code execution is not being stopped and a shitload of other warnings will be triggered if you do not check the false case of the loadXML function.

If you want to cover up the warning and display the actual XML issue, you'll have to write code that looks like this.

{% highlight php %}
<?php

$doc = new DOMDocument();

$previousErrorReporting = libxml_use_internal_errors(true);
libxml_clear_errors();

$result = $doc->loadXML('<invalid>');

$errors = libxml_get_errors();
libxml_clear_errors();
libxml_use_internal_errors($previousErrorReporting);

if (!$result) {
    // Do something with the errors and break execution
}
{% endhighlight %}

Do note that most XML functions work in a similar way: they return false and trigger a warning.
So if you want to reliably work with XML, you need to wrap them all with your own error handling.

The code above can be found in so many packages out there.
It's like everybody is OK with this approach.
At least, I am not!

## Bloated API

The XML extensions provided by PHP contain functions for everything you want to do with XML.
The DOMDocument class contains about 50 functions and 20 properties (estimated values) which all have their own purpose.
This is only the main entry point of your XML, besides that you can find DOMElements, DOMNodes, DOMAttributes, ... which all have a similar amount of functions.

Don't get me wrong, I like this structure: It is created to do whatever you want to do with XML.
You want to load XML, fine! Or do you rather manipulate something, fine as well! Do whatever you please!
The downside of this all, is that you need to know a lot about the low level API!

To give you an example for day-to-day loading of XML:

{% highlight php %}
<?php

$document = new DOMDocument();
$document->encoding = 'UTF-8';
$document->preserveWhiteSpace = false;
$document->formatOutput = true;

// wrap error handling here ...
$result = $document->loadXML('<root />');

if (!$result) {
    // Custom error handling
}
{% endhighlight %}

With error handling, this is about 15 lines of code for just loading an XML file!
Not to mention that I needed to look up every property in there ...

The same goes for basically any manipulation you want to do.
Did you ever try to create an XML document with DOMDocument? It's aweful - because it is all this ultra flexible.

What if we were able to split all these functions and different XML types per use case?
That way we could create a smaller Document class that you can use to perform specific types of actions:

- Loading with configurable settings.
- Validating based on whatever schema you please.
- Building XML
- Manipulating XML
- Locating items inside an XML 
- ...


## Other components

I don't want to convert this blog post into a big rant, so I am not going to focus on all other XML components at this moment.
So instead of complaining, let's deal with these problems and let's create something that covers these problems instead.



# XML without worries

In order to tame the XML beast, I decided to create yet another package.
It's not that there isn't any XML library out there, but most focus on 1 specific problem or are bloated wrappers around DOMDocument.
With this package I want to improve ALL XML extensions that are available in PHP.

The goal of the package is to provide all tools for dealing with XML in PHP without worries.
You will find a type-safe, declarative API that deals with errors for you!

Currently the package provides DOM, XSD, XSLT, Error-Handling and Reader components.
In the future I will also focus on additional components like a memory-safe Writer and focus a bit on "simple"Xml.
There is a lot of stuff to discover, so let me give you a couple of highlights which I personally really like!


## Locating data inside an XML document

### Loading the XML document

You can load an XML document with configurable functions to determine how it will load.
These configurable functions can contain whatever you want: 

- Specify encoding
- Specify output
- Validate internal XSD schema's whilst loading
- You can create custom configurators that e.g. replace import statements with their eventual XML (flattening)
- ...

{% highlight php %}
<?php

use VeeWee\XML\DOM\Configurator;
use VeeWee\XML\DOM\Document;
use VeeWee\XML\DOM\Loader;
use VeeWee\XML\DOM\Validator;
use VeeWee\Xml\ErrorHandling\Issue\Level;

$doc = Document::configure(
    Configurator\utf8(),
    Configurator\trim_spaces(),
    Configurator\Loader(
      Loader\xml_file_loader('data.xml')
    ),
    Configurator\validator(
        Validator\internal_xsd_validator(),
        Level::warning()
    )
);
{% endhighlight %}


### XPath improvements

Next up is the actual querying. You start of by configuring how you want to create your XPath object.
In this case, you can configure it with lookup namespaces:

{% highlight php %}
<?php

use Psl\Type;
use VeeWee\XML\DOM\Xpath\Configurator;

$xpath = $doc->xpath(
    Configurator\namespaces([
        'acme' => 'http://acme.com',
    ])
);

$currentNode = $xpath->querySingle('//acme:products');
$count = $xpath->evaluate('count(.//item)', Type\int(), $currentNode);
{% endhighlight %}

After creating the XPath object, you can see a new function `querySingle`. 
I found myself frequently looking for one element and wanted to make that possible.
Another addition is typed evaluation.
By specifying a type, you can make sure that the $count variable actually contains an integer or throws an exception instead.

If you write an error in your XPath query, you will receive a meaningfull error message explaining you what is wrong with your query!

### Memory-safe Reading

If you have a very big XML file, you want to lookup nodes in a memory-safe way.
The improved version of the XMLReader looks like this:

{% highlight php %}
<?php

use VeeWee\Xml\Dom\Document;
use VeeWee\Xml\Reader\Configurator;
use VeeWee\Xml\Reader\Reader;
use VeeWee\Xml\Reader\Matcher;

$reader = Reader::fromXmlFile(
    'large-data.xml',
    Configurator\xsd_schema('schema.xsd')
);

/** @var \Generator<string> $provider */
$provider = $reader->provide(
    Matcher\all(
        Matcher\node_name('item'),
        Matcher\node_attribute('locale', 'nl-BE')
    )
);

foreach ($provider as $nlItem) {
    $dom = Document::fromXmlString($nlItem);
    // Do something with it
}
{% endhighlight %}

You can trust this reader to handle the boring node traversal for you.
By specifying matcher functions, you are in control of what XML strings are yielded.
From that point on, you can e.g. use a regular Document to deal with the smaller XML chunks.

The reader is able to detect errors or invalid schema implementations whilst iterating over the dataset lazily.
You will receive a meaningful exception at all time!

## Writing

Reading is a bug chunk of your day-to-day XML usage, but in some cases you mind want to create or manipulate an XML.
In the XML package, you will find some tools that make building XML quite intuitive.
You basically specify what the XML should look like:

{% highlight php %}
<?php

use VeeWee\Xml\Dom\Document;
use function VeeWee\Xml\Dom\Builder\attribute;
use function VeeWee\Xml\Dom\Builder\children;
use function VeeWee\Xml\Dom\Builder\element;
use function VeeWee\Xml\Dom\Builder\namespaced_element;
use function VeeWee\Xml\Dom\Builder\value;
use function VeeWee\Xml\Dom\Manipulator\append;

$doc = Document::empty();
$doc->manipulate(
    append(...$doc->build(
        element('root', children(
            element('foo',
                attribute('bar', 'baz'),
                value('hello')
            ),
            namespaced_element('http://namespace', 'foo',
                attribute('bar', 'baz'),
                children(
                    element('hello', value('world'))
                )
            )
        ))
    ))
);

$xml = $doc->toXmlString();

{% endhighlight %}


# In closing

The snippets above are only some highlights of the package.
There is a lot more in there for you to discover!

At this moment, the package already contains a lot of components that you can use on a day-to-day basis.
Since XML is very broad, there are still missing functions.
It might contain some rough corners and will surely grow in time.

It has to start somewhere, it has to start sometime
What better place than here, what better time than now?!

### Introducing <a href="https://github.com/veewee/xml" target="_blank">veewee/xml</a>!

Feel free to drop me any feedback.

Hope you love it!
