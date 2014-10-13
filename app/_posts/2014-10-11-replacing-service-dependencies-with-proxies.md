---
layout: post
title:  "Replacing service dependencies with proxies"
category: general
tags: blog zf2 proxy doctrine
summary: When using the Zend Framework 2 service manager, it is possible to create shared services that will be loaded only once. In some situations however, it is very hard to switch the already injected dependencies in this service. You could mark the service as unshared even if this is often unnecessary. Another solution is to wrap the service with a proxy object and use the proxy instead of the service. 
image: 20141011/three-shell-game.jpg
thumb: 20141011/thumb-three-shell-game.jpg
---

<p>
    When using the Zend Framework 2 service manager, it is possible to create shared services that will be loaded only once.
     In some situations however, it is very hard to switch the already injected dependencies in this service.
     You could mark the service as unshared even if this is often unnecessary.
     Another solution is to wrap the service with a proxy object and use the proxy instead of the service. 
</p>

## Real-life Scenario
<p>
    In an application with multiple tenants, there is one database per tenant.
     When browsing the application the tenant's database connection is always known.
     This connection doesn't ever change when browsing the application.
     Therefor the Doctrine DocumentManager is injected and the services are marked as shared.
</p>
<p>
    On the server, there are some long-running processes to handle async commands.
     These processes run globally and are not dependant on a specific tenant.
     This means that the processes need to switch the DocumentManager based on the tenant they are working for at the moment.
</p>
<p>
    Off course, the registered services have to work with the current DocumentManager, even if they are marked as shared.
</p>

## Delegators
<p>
    One of the cool features of ZF2 are 
     <a href="http://framework.zend.com/manual/2.3/en/modules/zend.service-manager.delegator-factories.html" target="_blank">service delegators</a>.
     These delegators make it possible to attach or wrap custom functionality to an instance.
     In this case we will use the delegator to wrap the actual service in a proxy.
     The instance of the service will still be created in the service manager like before, 
     but instead of returning this instance we will return a proxy object.
</p>

## Proxies
<p>
    In this post, I already talked a lot about using proxies. But what are those proxies?
</p>
<p>
    The short answer: Proxies are objects that serve as a gateway to the actual object. 
</p>
<p>
    The long answer: There are many different types of proxies.
     Here you can find a good overview of all 
     <a href="http://ocramius.github.io/presentations/proxy-pattern-in-php" target="_blank">proxy patterns</a>
     in PHP.
</p>
<p>
    In this specific case, a Virtual Proxy is the right proxy to use.
     It extends the actual base class and has exactly the same API.
     An over-simplified proxy of a Doctrine DocumentManager could look like this:
</p>


{% highlight php %}
<?php

class DocumentManagerProxy
    extends DocumentManager
{

    protected $instance;

    public function __construct(DocumentManager $instance)
    {
        $this->instance = $instance;
    }
    
    public function find($className, $id)
    {
        return $this->instance->find($className, $id);
    }
    
    // ... All other methods of the DocumentManager ...

}
{% endhighlight %}

<p>
    Because this kind of objects will result in a lot of effort in maintaining,
     it is better to use a library that automatically generates the proxy objects.
     One of those libraries is 
     <a href="https://github.com/Ocramius/ProxyManager" target="_blank">ProxyManager</a> by Ocramius.
</p>


# Putting it all together

## ServiceManager configuration
{% highlight php %}
<?php

return [
    'factories' => [
        'documentmanager' => new \DoctrineMongoODMModule\Service\DocumentManagerFactory('manager_key'),
        'Application\DocumentManagerProxyDelegator' => 'Application\Factory\DocumentManagerProxyDelegatorFactory',
        'Application\DynamicDocumentManager' => 'Application\Factory\DynamicDocumentManagerFactory',
    ],
    'delegators' => [
        'documentmanager' => ['Application\DocumentManagerProxyDelegator'],
    ]
];
{% endhighlight %}

## Delegator

<p>
    The delegator is responsible for initializing the actual DocumentManager and creating the proxy object.
     Another service is added that is responsible for maintaining and loading the different DocumentManagers.
     Make sure that the proxy initializer returns false, 
     so that the proxy is initialized with the current DocumentManager on every method call.
</p>

{% highlight php %}
<?php

namespace Application;

use ProxyManager\Factory\LazyLoadingValueHolderFactory;
use Zend\ServiceManager\DelegatorFactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class DocumentManagerProxyDelegator implements DelegatorFactoryInterface
{

    /**
     * @var LazyLoadingValueHolderFactory
     */
    protected $proxyFactory;

    /**
     * @var DynamicDocumentManager
     */
    protected $dynamicDocumentManager;

    /**
     * @param $dynamicDocumentManager
     * @param $proxyFactory
     */
    public function __construct($dynamicDocumentManager, $proxyFactory)
    {
        $this->dynamicDocumentManager = $dynamicDocumentManager;
        $this->proxyFactory = $proxyFactory;
    }

    /**
     * {@inheritdoc}
     */
    public function createDelegatorWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName, $callback)
    {
        $dynamicDocumentManager = $this->dynamicDocumentManager;
        $currentDocumentManager = $callback();
        $dynamicDocumentManager->setCurrentManager($currentDocumentManager);

        $proxy = $this->proxyFactory->createProxy(
            'Doctrine\ODM\MongoDB\DocumentManager',
            function (& $wrappedObject) use ($dynamicDocumentManager) {
                $wrappedObject = $dynamicDocumentManager->getCurrentManager();
                return false;
            }
        );

        return $proxy;
    }

}
{% endhighlight %}

## Delegator Factory
<p>
    The delegator factory creates an instance of the factory.
     For this example It is very straight forward.
     If you are planning to use it in production, 
     you might add some extra caching configuration to the proxy factory.
     This configuration can be based on
     `<a href="https://github.com/zendframework/zf2/blob/master/library/Zend/ServiceManager/Proxy/LazyServiceFactoryFactory.php" target="_blank">Zend\ServiceManager\Proxy\LazyServiceFactoryFactory</a>`. 
</p>

{% highlight php %}
<?php

namespace Application;

use Application\DocumentManagerProxyDelegator;
use ProxyManager\Factory\LazyLoadingValueHolderFactory;
use Zend\ServiceManager\FactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class DocumentManagerProxyDelegatorFactory implements FactoryInterface
{

    public function createService(ServiceLocatorInterface $serviceLocator)
    {
        $dynamicDocumentManager = $serviceLocator->get('Application\DynamicDocumentManager');
        $proxyFactory = new LazyLoadingValueHolderFactory();
        return new DocumentManagerProxyDelegator($dynamicDocumentManager, $proxyFactory);
    }
    
}
{% endhighlight %}


## Dynamic DocumentManager
<p>
    The dynamic DocumentManager is responsible for loading and managing the DocumentManagers.
     While creating an instance for the DocumentManager, the real instance is injected in the DynamicDocumentManager.
     An implementation might look like this:
</p>

{% highlight php %}
<?php

interface DynamicDocumentManagerInterface
{

    public function getCurrentManager();
    public function setCurrentManager(DocumentManager $documentManager);
    public function loadForTenant(Tenant $tenant);

}

{% endhighlight %}

<p>
    The `setCurrentManager()` is called by the delegator to set the current manager.
</p>
<p>
    The `getCurrentManager()` is called while initializing the proxy. 
     This way, the current manager will always be used when using the `documentmanager` in a shared service.
</p>
<p>
    Before executing a command in the long-running process, the `loadForTenant()` method is being called.
     This method will close the old manager and reset the service manager keys for all Doctrine services.
     Finally it will recreate the `documentmanager`, which will trigger the `setCurrentManager()` method again.
</p>


# Final thoughts
<p>
    This technique might look a little bit `hacky`, 
     but is actually a very neat trick to make it easier to change instances on the fly.
</p> 
<p> 
    You don't have to mark all services that rely on the DocumentManager as unshared.
     In the codebase, you can still use the actual instance type in the annotations.
</p>
<p>
     You might think of some other good use-cases to implement this powerful technique yourself.
</p>