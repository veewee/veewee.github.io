---
layout: post
title:  "Authenticating with X.509 client certificates"
category: general
tags: ssl authentication symfony php
summary: Last week, I was diving in different authentication systems for API's. One of the better authentication types is authentication through X.509 client certificates. This type of authentication is harder to set-up, but sure is secure, managaeble and powerfull. While searching for documentation on the subject, I was surprised there weren't a lot of good articles. In this article, I will try to explain every step as easy as possible.
image: 20151207/security.jpg
thumb: 20151207/thumb-security.jpg
---

<p>
    Last week, I was diving in different authentication systems for API's.
    One of the better authentication types is authentication through <a href="https://en.wikipedia.org/wiki/X.509" target="_blank">X.509 client certificates</a>.
    This type of authentication is harder to set-up, but sure is secure, managaeble and powerfull.
    While searching for documentation on the subject, I was surprised there weren't a lot of good articles.
    In this article, I will try to explain every step as easy as possible.
</p>

<h2>Why should I use X.509 authentication?</h2>
<p>
    The main advantage is that the client is not sending a username or password to the server. 
    This means that a man-in-the-middle attack is nearly impossible. 
    It is much easier to steal a username/password login, for example by bruteforcing, then stealing a certificate.
    Because the certificate is signed, it is only possible to connect to the real server.
    It is possible to revoke and manage these certificates in an easy way.
</p>

<h2>Configuring the server</h2>

<p>
    Client Certificate authentication can only be done while running HTTPS.
    So first of all, make sure the server is running HTTPS. 
    This can be done with a self-signed or a signed certificate.
    The creation of this certificate is not part of this article.
    Your apache vhost configuration should look more or less like this:
</p>

{% highlight apache %}
SSLEngine on
SSLCertificateFile      "/etc/ssl/certs/server.pem"
SSLCertificateKeyFile   "/etc/ssl/private/server.key"
SSLProtocol             TLSv1 TLSv1.1 TLSv1.2
SSLCipherSuite          "......."
 
# When using signed certificates:
SSLCertificateChainFile "/etc/ssl/yourDomainName.ca-bundle"
{% endhighlight %}

<p>
    Now that you got HTTPS up and running, you should be able to browse to your application through HTTPS. 
</p>

<h3>Creating a certification authority</h3>

<p>
    A certification authority (CA) hands out a digital certificate in which the CA says that a public key in the certificate, 
    belongs to the person, organisation, server or entity that is mentioned in the certificate.
    In our example, this will be done based on the e-mail address that is provided in the certificate.
    The task of the CA is to control the identiy of the issuer, so that the client that is using the certificates from the CA can be trust.
</p>

<p>
    First we will need to create the CA private key and certificate:
</p>

{% highlight bash %}
cd /etc/ssl/certs/
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
{% endhighlight %}

<div class="alert alert-warning">
    <strong>Note:</strong> Make sure to set the CN to a valid server domain. Otherwise this might result in exceptions!
</div>

<p>
    Now that we have the CA key and certificate, we can register it to Apache to validate the client certificates:
</p>

{% highlight apache %}
SSLCACertificateFile "/etc/ssl/certs/ca.crt"
SSLVerifyClient optional
SSLVerifyDepth 1
SSLOptions +StdEnvVars
{% endhighlight %}

<p>
    As you can see the client certificate verification is optional. 
    This will make it possible to add another type of authentication like basic authentication when there is nog client certificate.
    The verify depth is set to 1 so that it only accepts certificates signed by the configured CA.
    Finally the StdEnvVars are registered so that the additional SSL server variables are available in PHP.
</p>


<h2>Configuring the client</h2>

<p>
    Now that we got or server fully configured, it is time to create a client certificate.
    Following commands should be ran on the client machine to create a Certificate Signing Request (CSR):
</p>

{% highlight bash %}
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr
{% endhighlight %}

<div class="alert alert-warning">
    <strong>Note:</strong> Make sure to set the CN to a valid server domain. Otherwise this might result in exceptions!<br />
    <strong>Note:</strong> The email field will be used to authenticate the user.
</div>

<p>
    The CSR file is useless when it is not signed by the CA.
    So the logical next step is to transfer the certificate to the server and sign it with the CA.
</p>

{% highlight bash %}
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt
{% endhighlight %}


<p>
    The command above results in a usefull client certificate. 
    To make it easier to use the certificate, we will pack the client private key and the certificate in one file.
    This action should run on the client machine:
</p>

{% highlight bash %}
cat client.crt client.key > client.pem
{% endhighlight %}

<div class="alert alert-warning">
    <strong>Note:</strong> it's important to place the crt before the key in the pem file!
</div>

<p>
    In above example the CSR was created on the client, to make it clear that the certificate + key should only be known by the client.
    However, it is perfectly possible to run all these commands on the server and send the .pem file to the client who will be using the certificate.
    This means that it is perfectly possible to automate the creation of the client certificates.
    It is even possible to create your own user interface to make the keys manageable per user of your application.
    This can for example be done with the built-in <a href="http://php.net/manual/en/book.openssl.php" target="_blank">openssl</a> extension of PHP.
</p>


<h2>Requesting resources with a client certificate</h2>

<p>
    Ok, we configured our server and requested a client certificate. 
    Now how do I use this certificate to get my resources?
    The easiest way is the add the "<em>-cert</em>" attribute to the curl command.
    Since we are using Guzzle for HTTP requests, the client configuration will look like this:
</p>

{% highlight php %}
<?php
$client = new Guzzle\Http\Client($baseUrl, [
    'ssl.certificate_authority' => 'system',
    'request.options' => [
        'cert' => 'client.pem',
    ]
]);
{% endhighlight %}

<p>
    As you can see it is possible to specify the certificate in the request.options part of the configuration.
    Another option is the ssl.certificate_authority. This one can be used to specify which CA that should be used. 
    By default the built-in CA file is being used.
    You can choose to disable ssl verification or add your own ca file.
    For example, when using self-signed certificates, you can run following command:
</p>

{% highlight bash %}
openssl s_client -showcerts -connect my.host.com:443 > ca-file.pem
{% endhighlight %}

<p>
    This ca-file.pem file will contain the certificate of the certification authority and mark it as trusted. 
    Now it can be used in the PHP configuration as followed:
</p>

{% highlight php %}
<?php
$client = new Guzzle\Http\Client($baseUrl, [
    'ssl.certificate_authority' => 'ca-file.pem',
    'request.options' => [
        'cert' => 'client.pem',
    ]
]);
{% endhighlight %}


<h2>X.509 authentication in Symfony</h2>

<p>
    One of the less known features of Symfony is X.509 authentication.
    It can easily be configured with the default x509 authentication security adapter.
    The configuration is pretty easy:
</p>

{% highlight yaml %}
security:
    providers:
        client_certificate:
            memory:
                users:
                    email@certification.file:
                      roles: ROLE_SUPER_ADMIN
 
    firewalls:
        main:
            x509:
                provider: client_certificate
 
    access_control:
            - { path: ^/, roles: ROLE_SUPER_ADMIN, requires_channel: https }
{% endhighlight %}

<div class="alert alert-warning">
    <strong>Note:</strong> This type of authentication only works with HTTPS. You might want to enforce HTTPS!
</div>

<p>
    As you can see HTTPS is enforced and authentication will try X.509 authentication by default.
    When a valid client certificate is found, 
    Symfony will try to match the email that is configured inside the certificate with a user in the client_certificate user provider.
    In this case we are using an in-memory provider that links an e-mail to a security role.
</p>

<h2>X.509 authentication in PHP</h2>

<p>
    That was easy! But how does it work?
    By adding the "SSLOptions +StdEnvVars" configuration in Apache, there are some additional "<em>SSL_</em>" environment variables available.
    These variables contain the email in the client certificate. For example:
</p>

{% highlight php %}
<?php
$email = $_SERVER['SSL_CLIENT_S_DN_Email'];
if (!$email && preg_match('#/emailAddress=(.+\@.+\..+)(/|$)#', $_SERVER['SSL_CLIENT_S_DN'], $matches)) {
    $email = $matches[1]
}
{% endhighlight %}

<p>
    Now that we have the e-mail, it is possible to search this email in the list of configured client_certificates.
    Symfony will throw an exception when the email can't be found.
</p>

<h2>Conclusion</h2>

<p>
    Even though it is a lot of work to get this type of authentication running,
    it sure is a powerful type of authentication. 
    The users of your application will be able to connect to your application in an improved and secure way. 
    It also comes in very handy for server to server communication in which you don't always want to hardcode user credentials.
    Your customers will surely thank you for the extra effort you made!
</p>
