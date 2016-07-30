---
layout: post
title:  "Writing your first NodeJS Lambda function"
category: general
tags: nodejs lambda serverless
summary: "Last week I was playing around with writing some custom Slack slashcommands and cronjobs. Originally, I started writing these integrations with PHP. Since slack requires a publically available endpoint and I did not want to set up a server for these scripts, I decided to take a look at AWS Lambda. This Amazon service makes it possible to run scripts serverless. The scripts are triggered based on triggers inside of AWS. In this artile you can find out how to get started with writing lambda functions."
image: 20160730/lambda.jpg
thumb: 20160730/thumb-lambda.jpg
---

<p>
    Last week I was playing around with writing some custom <a href="https://slack.com/" target="_blank">Slack</a> slashcommands and cronjobs.
    Originally, I started writing these integrations with PHP. 
    Since slack requires a publically available endpoint and I did not want to set up a server for these scripts,
    I decided to take a look at <a href="https://aws.amazon.com/lambda/details/" target="_blank">AWS Lambda</a>.
    This Amazon service makes it possible to run scripts serverless. The scripts are triggered based on triggers inside of AWS.
    In this artile you can find out how to get started with writing lambda functions.
</p>

<h2>Selecting the right tools</h2>

<p>
    AWS Lambda can run Python, Java and NodeJS scripts.
    Since I've got some experience writing <a href="https://nodejs.org/en/" target="_blank">NodeJS</a>, I've choosen to go for this technology.
    There are many tools available for writing Lambda functions in NodeJS. 
    After investigating these tools, I've decided to go with <a href="https://github.com/Testlio/lambda-tools" target="_blank">lambda-tools</a>.
    This tool makes it easy to develop and deploy multiple lambda functions inside one codebase. 
    It comes with a Yeoman generator named <a href="https://github.com/Testlio/generator-lambda-tools">generator-lambda-tools</a>.
</p>

<h2>Getting started</h2>

<p>
    The first thing we'll want to do is scafold the application:
</p>

{% highlight sh %}
$ yo lambda-tools

? Service name: your-lambda-name
? Service description: Some Description
? License (API): MIT
? Author email your-email
? Author name your-name
? Default Lambda runtime Node.js 4.3
? Install dependencies lambda-tools, lambda-foundation
{% endhighlight %}

<p>
    When the command is finished, you'll find a project structure like this:
</p>

{% highlight sh %}
.
├── .lambda-tools-rc.json
├── .yo-rc.json
├── api.json
├── cf.json
├── lambdas/
└── node_modules/
{% endhighlight %}

<p>
    There are 2 hidden configuration files named `.lambda-tools-rc.json` and `.yo-rc.json`. 
    These are used to configure the yeoman generator and the lambda-tools command.
    The `api.json` is used for the <a href="https://aws.amazon.com/api-gateway/" target="_blank">AWS API Gateway</a> service.
    This service will make it possible to create a public endpoint that triggers the Lambda method we'll create next.
    <br />
    The `cf.json` is used for the <a href="https://aws.amazon.com/cloudformation/" target="_blank">AWS CloudFormation</a> service.
    This service will take care of orchestrating the related AWS services. 
    It will be used to deploy your lambda functions and let all the AWS services talk to each other.
    <br />
    The `lambdas` directory is currently empty, but will contain all the lambda functions you are going to write.
    <br />
    Finally there is the `node_modules` directory which contains all your project dependencies.
</p>

<h2>Writing your first lambda</h2>

{% highlight sh %}
$ yo lambda-tools:lambda

? Lambda function name: mylambda
? Create event.json: Yes
{% endhighlight %}

<p>
    The command above will create a folder named `mylambda` inside the `lambdas` directory.
    In this folder there are 2 files: `event.json` and `index.js`.
    The event.json contains the data that will be used as input for the Lambda function.
    Feel free to put it in `.gitignore` since you can only use this for testing purpose.
    The `index.js` file contains a `handler()` method and looks like this:
</p>

{% highlight js %}
'use strict';

exports.handler = function(event, context) {
    context.succeed('Hello!');
};
{% endhighlight %}

<p>
    The callback has 2 parameters:
    `event` which will be populated by AWS Lambda. During development this parameter can be linked to the `event.json` file.
    `context` which will make it possible to `succeed()` or `fail()` the lambda method.
    The parameters added to these 2 functions will be returned as result of the lambda:
</p>

{% highlight sh %}
$ node node_modules/lambda-tools/bin/lambda.js execute ./lambdas/mylambda/index.js -e lambdas/mylambda/event.json -v
{% endhighlight %}
{% highlight html %}
Executing: lambdas/mylambda/index.js
	--
	With event:
	{}
	--


	--
	Result 'Hello!'
{% endhighlight %}


<h2>Creating a cronjob</h2>

{% highlight sh %}

$ yo lambda-tools:periodic-lambda      

? Lambda function name cronjob
? Create event.json Yes
? Schedule/Rate expression cron(0 10 * * ? *)
? CloudWatch rule name (leave empty for automatic name) ten-o-clock
? CloudWatch rule logical ID (used in CloudFormation template) TenOClockEvent
? CloudWatch rule description (leave empty to skip)
? CloudWatch rule static input (JSON string, leave empty to skip)
? CloudWatch rule input path (JSONPath string, leave empty to skip)
? Is CloudWatch rule ENABLED? Yes
{% endhighlight %}

<p>
    In <a href="https://aws.amazon.com/cloudwatch/" target="_blank">AWS CloudWatch</a>, it is possible to create RATE or CRON rules.
    These rules can be the trigger for a Lambda function to execute. This means it is possible to create a lambda function that executes periodically.
    The generated code looks and behaves exactly the same as the `mylambda` code above.
    In the `cf.json` file, a event rule is added to cloudwatch and the event is linked to the lambda function.
</p>


<h2>Creating an API endpoint</h2>

{% highlight sh %}
$ yo lambda-tools:endpoint   

? Path for the endpoint /lambdas/apilambda
? HTTP Method POST
? Lambda function name lambdas-apilambda-post
? Map request body to event property (leave blank to skip) body
? Map HTTP headers? No
{% endhighlight %}

<p>
    By adding an API endpoint, you can create a publically available HTTPS endpoint that triggers a Lambda function.
    The generated code again looks the same as the `mylambda` code.
    The `api.json` file is altered and now describes the new endpoint:
</p>

{% highlight json %}
{
    "paths": {
        "/lambdas/apilambda": {
            "post": {
                "responses": {
                    "200": {
                        "description": "Default response",
                        "schema": {
                            "type": "object",
                            "properties": {},
                            "additionalProperties": true
                        }
                    }
                },
                "parameters": [],
                "x-amazon-apigateway-auth": {
                    "type": "none"
                },
                "x-amazon-apigateway-integration": {
                    "type": "aws",
                    "uri": "$lLambdasApilambdaPost",
                    "credentials": "$IamRoleArnApiGateway",
                    "httpMethod": "POST",
                    "requestParameters": {},
                    "requestTemplates": {
                        "application/json": "{\"path\":\"$context.resourcePath\",\"body\":$input.json('$')}"
                    },
                    "responses": {
                        "default": {
                            "statusCode": "200",
                            "responseTemplates": {
                                "application/json": "$input.json('$')"
                            }
                        }
                    }
                }
            }
        }
    }
}
{% endhighlight %}

<p>
    The added path describes the endpoint. 
    As you can see it only handles POST requests with a content-type of `application/json`.
    The JSON body of the request is added as the "body" parameter of the Lambda event.
    This is done internally through the `x-amazon-apigateway-integration` header.
    In the response you'll have to add the responseTemplate to make sure that the result of the lambda is returned as JSON.
    You can test the API endpoint with following commands:
</p>

{% highlight sh %}
# This command will start a server on port 3000:
$ node node_modules/lambda-tools/bin/lambda.js run -a api.json

# Send a curl request:
$ curl -X POST -H "Content-Type: application/json" -d '{request: "will be added to event as body param"}' "http://localhost:3000/lambdas/apilambda"
{% endhighlight %}

<p>
    This HTTP request will be handled as followed:
</p>

{% highlight html %}
  <-- POST /lambdas/apilambda
	<-- Configuring Lambda function
	--> Configuring Lambda function 1ms

	<-- Creating integration
	{
		"context": {
			"apiId": "local-lambda",
			"httpMethod": "POST",
			"identity": {},
			"requestId": "98e626ab-bbc7-4be2-a94c-b6f622b1d140",
			"resourceId": "POST /lambdas/apilambda",
			"resourcePath": "/lambdas/apilambda",
			"stage": "dev"
		},
		"event": {
			"path": "/lambdas/apilambda",
			"body": {
				"request": "will be added to event as body param"
			}
		}
	}
	--> Creating integration 14ms

	<-- Executing Lambda (lambdas-apilambda-post/index.handler)


	--> Executing Lambda function 151ms

	<-- Creating response
	{
		"status": 200,
		"headers": {},
		"type": "application/json",
		"body": "hello!"
	}
	--> Creating response 1ms

  --> POST /lambdas/apilambda 200 200ms 87b
{% endhighlight %}

<h2>Configuring AWS</h2>

<p>
    Now that we've created our lambda methods, the next step is to deploy them to AWS.
    This step is not documented very good, so I'll try to describe it as good as possible.
</p>

<p>
    First you'll need to create a user with the <a href="https://github.com/Testlio/lambda-tools#iam-permissions">IAM-permissions described on the lambda-tools page</a>.
    When you've created this user, you'll have to make sure that the user has an Access Key so that it can be used to automate stuff.
    Now that you've got both the Access Id and Secret Access Key, you'll have to generate an API token: 
</p>

{% highlight sh %}
$ aws configure
$ aws sts get-session-token
{% endhighlight %}


<p>
    Now that we have all the AWS parameters, we'll need to create a `.env` file so that the tool knows how to deploy the application:
</p>

{% highlight sh %}
AWS_ENVIRONMENT=dev
AWS_ACCESS_KEY_ID=your-key-id
AWS_SECRET_ACCESS_KEY=your-key-secret
AWS_SESSION_TOKEN=your-key-token
AWS_REGION=eu-west-1
AWS_RUNTIME=nodejs4.3
EXCLUDE_GLOBS="event.json *.env *.dist"
PACKAGE_DIRECTORY=build
{% endhighlight %}

<p>
    Finally we need to configure a <a href="https://aws.amazon.com/s3">AWS S3</a> bucket in which the lambda functions will be stored.
    This needs to be a unique index which can be configured in the `.lambda-tools-rc.json` file. 
    Add following `tools` to the root element:
</p>

{% highlight json %}
{
    "tools": {
        "resources": {
            "s3Bucket": "your-unique-s3-bucket-name"
        }
    }
}
{% endhighlight %}

<p>
    Now that we've configured the project source code to run with AWS, we'll have to set up everything in AWS.
    Luckilly this can be done with one simple command that only needs to run once:
</p>

{% highlight sh %}
$ node node_modules/lambda-tools/bin/lambda.js setup -r eu-west-1
{% endhighlight %}


<h2>Deploying your lambdas to AWS</h2>

{% highlight sh %}
$ node node_modules/lambda-tools/bin/lambda.js deploy -r eu-west-1
{% endhighlight %}

<p>
    By running the command above, the lambdas are build into some archive files.
    These archive files are stored in your AWS S3 Bucket.
    When this is done, the `cf.json` file is added to AWS CloudFormation which will start the orchestration of the installation:
</p>

<ul>
    <li>A CloudFormation stack will be created or updated based on the `cf.json` file.</li>
    <li>The source code of the lambdas are stored in your S3 Bucket</li>
    <li>IAM permissions will be added</li>
    <li>A new scheduling rule will be added to CloudWatch.</li>
    <li>New logging groups will be added to CloudWatch.</li>
    <li>The API gateway will be configured with the `api.json` file.</li>
    <li>The lambdas will be created and configured.</li>
</ul>

<h2>Adding configurable variables to the lambdas</h2>

<p>
    Adding custom configuration to a lambda function can be done based on environment variables.
    These variables can be added with the `-e` option during a deploy.
    When you've got a lot of configurable variables, another option is to add a .env file to every lambda directory and use the 
    <a href="https://github.com/ncthis/env-loader" target="_blank">env-loader</a>.
    You'll have to create an asset to make sure the .env file is added to your lambda source code.
    This can be done by adding a new `cf.json` file to every lambda directory that looks like:
</p>

{% highlight json %}
{
  "Assets": {
    ".env": ".env"
  },
}
{% endhighlight %}


<h2>Conclusion</h2>

<p>
    If you are writing your first AWS project, getting started is pretty complex.
    The documentation is rather big and you'll have to find your way in all the different AWS services.
    Once you found your way and got your project set-up and configured, writing Lambda functions is pretty fun.
    They are easy to understand and there are a lot of NPM packages available for Rapid Application Development.
    I am definitely looking forward in writing more Lambda integrations!
</p>
