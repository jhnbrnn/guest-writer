---
layout: post
title: "Securing AWS HTTP APIs with JWT Authorizers"
description: "A tutorial on integrating Auth0 with HTTP APIs built on AWS API Gateway"
date: "2019-12-23 08:30"
author:
  name: "John Brennan"
  url: "jhnbrnn"
  mail: "brennan.john@mac.com"
  avatar: "https://twitter.com/jhnbrnn/profile_image?size=original"
related:
- 2017-11-15-an-example-of-all-possible-elements
---

**TL;DR:** JSON Web Tokens (JWTs) can be used to authorize requests to HTTP APIs built on Amazon Web Services’ API Gateway. This tutorial will walk you through integrating Auth0 with such an HTTP API.

# Securing AWS HTTP APIs with JWT Authorizers
​
**TL;DR** JSON Web Tokens (JWTs) can be used to authorize requests to HTTP APIs built on Amazon Web Services’ API Gateway. This tutorial will walk you through integrating Auth0 with such an HTTP API.
​
## An introduction to HTTP APIs in API Gateway
​
AWS API Gateway offers a number of solutions for building scalable APIs. HTTP APIs ([link](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html)) are a simple form of API that proxies requests to either another HTTP endpoint or an AWS Lambda. Unlike building REST APIs in API Gateway, HTTP APIs can’t leverage other AWS services, such as AWS Identity and Access Management (IAM), as _Authorizers_ - services that allow or restrict API access to clients based on criteria such as user, roles, IP addresses, and so on. 

HTTP APIs can, however, use JSON Web Tokens (JWTs) to provide access control using a **JWT Authorizer**.

## What is a JWT Authorizer?
​
A JWT Authorizer configures the endpoint of your HTTP API to expect a token, created by an Identity Provider such as Auth0, to be provided in HTTP requests to the API. The authorizer will validate the token to ensure the requesting client, as well as optionally restricting access based on scopes in the token.

A benefit of JWT Authorization is that you can pair existing identity provider solutions with the low overhead of HTTP APIs for security and authorization without needing to rely on heavier processes such as AWS IAM or resource policy files.

In this tutorial, you’ll learn how to pair these two pieces together!

## What will you build?

​You’ll be building an HTTP API on AWS using API Gateway, a Lambda function, and DynamoDB. The API will support both GET and POST requests and read/write data from a DynamoDB table. POST requests will be restricted to authorized users and GET requests will be unauthorized, meaning anyone can access them.

## Prerequisites

* Auth0 Account
* AWS Account
* knowledge of Node.js (for creating the Lambda)
	​
## Configure Auth0
### Create new Auth0 API

After signing into Auth0, head to your dashboard and click the APIs link in the lefthand menu. Click the “Create new API” button and fill out the form with the following values:
**Name:** AWS JWT Authorizer
**Identifier:** “https://auth0-jwt-authorizer.com”
**Signing Algorithm:** RS256 (default value)
Click “Create”, and you’ll be taken to the definition page for your new API.

![an example of the Auth0 API settings](images/jwt1.png)

You’ll want to copy or write down your API’s identifier; we’ll need it later on.
​
## Create HTTP API (Wish List API) on AWS

For this step, we’ll be using three services in AWS:
* DynamoDB (a NoSQL database to store our wish list items)
* Lambda (the business logic that will power our API, implemented as a serverless function)
* API Gateway (to create the public API endpoint itself and add the JWT Authorizer)

### Create DynamoDB

First, head to the [AWS DynamoDB dashboard](https://console.aws.amazon.com/dynamodb/home), signing into AWS if needed. Click the “Create Table” button to get started.

We’ll name our table “WishList” and give the Primary Key a name of “id”, leaving its type as the default (String). Leave the rest of the form as-is; just scroll down and click Create.

![Settings for the DynamoDB table](images/jwt2.png)

That’s all you need to do for setup, so it’s time to head over to Lambdas!

### Create Lambda

Go to the [Lambda dashboard](https://console.aws.amazon.com/lambda/home) and click the “Create function” button in the table’s top right corner.

The Lambda service provides a lot of helpful blueprints and samples, but for this tutorial you’ll be creating our functionality from the ground up, so leave “Author from scratch” selected at the top of the form.

Give your function the name “wish-list-service” and keep the runtime as the default (Node 12.x). In the Permissions section, open up the section for creating an execution role; for our purposes, we’ll use a template for setting permissions to allow the Lambda to use the DynamoDB table we just created.

For Execution Role, Select “Create a new role from AWS policy templates”. Name the role “wish-list-service-role” and, in the Policy templates dropdown, select “Simple microservice permissions”.

![Settings for the Lambda](images/jwt3.png)

Click “create function” in the bottom right corner of the page, and you’ll shortly be taken to the details page for your new Lambda. And now, time to code!

You’ll want to start out by clearing the “Hello World” example. Replace any existing code with the following empty handler function:

```js
exports.handler = async (event, context) => {

};
```

You’re using the `async` keyword because the code will asynchronously access DynamoDB, which you’ll see in a moment. `event` and `context` will both be used in a moment to retrieve information about the API request.

Next, you’ll want to set up the connection between your Lambda and DynamoDB to perform your reads and writes:

```js
// NEW: import the AWS JavaScript SDK and create a new DocumentClient
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();

const TABLE_NAME = "WishList";

// Existing code
exports.handler = async (event, context) => {

}
```

`DocumentClient` will allow you to perform actions against your DynamoDB tables. The AWS documentation for DocumentClient can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html).

Since you’re handling both GET and POST requests in one handler, a simple `switch` statement against the HTTP method should do the trick:

```js
// ...

exports.handler = async (event, context) => {
	switch(event.httpMethod) {
		case "GET":
			break;
		case "POST:
			break;
		default:
			break;
	}
}

```

With the scaffolding in place, it’s time to set up our GET handler. It’s simple enough: you’ll retrieve records from the `WishList` table and return them:

```js
// ...
switch(event.httpMethod) {
	case "GET":
		// scan() will retrieve all records found in the table matching the TableName parameter
		const data = await ddb.scan({ TableName: TABLE_NAME }).promise();

		// output from the Lambda must match a specific format expected by API Gateway. Information about the format can be found here: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format
		return {
			statusCode: 200,
            body: JSON.stringify(data),
            headers: {
				'Content-Type': 'application/json',
			}
		};
		
```

Next up: POST requests. You’ll store three properties in the database that you can expect in the request body:
* `name` (the name of the item on your wish list)
* `description` (a helpful description of the item)
* `url` (a link to where the item can be purchased). 

As for the ID of the new item that will be added to DynamoDB, You can use the guaranteed-to-be-unique `awsRequestId` property from the `context` parameter provided to the handler function. After the item’s been added to the database, you can return the ID in the body of the successful response.

Here’s what the code’s going to look like:

```js
// ...
switch(event.httpMethod) {
	// ...
	case "POST":
        const {name, url, description } = event.body;
		await ddb.put({
			TableName: TABLE_NAME,
			Item: {
				id: context.awsRequestId,
				name,
				url,
				description
			}
		}).promise();
            
		return {
			statusCode: 201,
			body: JSON.stringify({
				id: context.awsRequestId,
			}),
			headers: {
				"Content-Type": "application/json",
			}
		};
```

And that’s it! Here’s a complete look at  your Lambda’s code:
 
```js
const AWS = require('aws-sdk');

const ddb = new AWS.DynamoDB.DocumentClient();

const TABLE_NAME = "WishList";

exports.handler = async (event, context) => {
  switch(event.httpMethod) {
    case "GET":
      const data = await ddb.scan({ TableName: TABLE_NAME }).promise();
      return {
        statusCode: 200,
        body: JSON.stringify(data),
        headers: {
          'Content-Type': 'application/json',
        }
      };
    case "POST":
      const {name, url, description } = event.body;
            
      await ddb.put({
        TableName: TABLE_NAME,
        Item: {
          id: context.awsRequestId,
          name,
          url,
          description
        }
      }).promise();
            
      return {
        statusCode: 201,
        body: JSON.stringify({
          id: context.awsRequestId,
        }),
        headers: {
          "Content-Type": "application/json",
        }
      };
    default:
      break;
  }
};
```

After you’ve added your code, click the orange “Save” button in the top right corner. After the save is completed, your Lambda’s functionality is now live, and you’re ready to create the HTTP API that will make this Lambda’s functionality publicly available.

### Create HTTP API in front of Lambda

There are a few ways to create an HTTP API; for the purposes of this tutorial, you’ll be creating the API from the Lambda details page in which you’ve been working for the above section. The benefit of creating it here is that AWS will handle the API’s permission settings by default, allowing it to execute the Lambda without any additional configuration.

Scroll to the top of the page and open the “Designer” section at the top. Click the “Add trigger” button in the section’s left side. Select “Api Gateway” in the form’s dropdown, then “Create a new API” from the next dropdown that appears. In the form elements that follow, HTTP API should be selected by default, and you don’t need to change any of the settings - just head to the bottom of the form and click “Add”.

![Settings for the HTTP API Trigger](images/jwt4.png)

By creating the API via the triggers section, everything should automatically wire up with the Lambda, so let’s give it a test using cURL to make sure. From the Designer section of the Lambda details page, click on the new API Gateway trigger object. From the API Gateway section that appears, grab the URL if your API and open up a new terminal window.

First, make a POST request to your endpoint to create a new wish list item:

```bash
$ curl -H "Content-Type: application/json" \
  -X POST \
  -d '{"name": "Test Item", "description": "Test Description", "url": "https://www.amazon.com"}' \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

You should get back a JSON payload containing an `id` key.

Next, make a GET request to ensure your new item was created:

```bash
$ curl https://[DOMAIN].amazonaws.com/default/wish-list-service
```

The return payload should contain a key called `Items`, the value of which is an array of wish list items. Your new item will appear in that array.

### Add JWT Authorizer using Auth0 App

Now for the final step - setting up your Auth0 API as a JWT Authorizer. Head over to the [API Gateway dashboard](https://console.aws.amazon.com/apigateway/main/apis) and click “wish-list-service-API” to open up the API’s details page.

First, click “Routes” in the left side menu. Currently, the API is configured to allow any type of request to the `wish-list-service` endpoint, so you’ll need to change that first. Click “ANY” under the “/wish-list-service” Route in the left column, then click “Edit” in the details card’s header on the right. In the dropdown, change “ANY” to “GET” and click save.

![Changing the route to GET only](images/jwt5.png)

You’ve now restricted the existing behaviour - allowing unauthorized requests to the Lambda - to only GET requests.

Once you’re back on the Routes page, click “Create” in the left column header to create a new route. Select “POST” from the methods dropdown and set the URL to “/wish-list-service”, then click Save. 

![Adding a POST method to the route](images/jwt6.png)

You’ll be redirected back to the Routes page; click “POST” under the “/wish-list-service” route in the left column to open up the details card.

You’ll first need to configure the route to actually call your Lambda. To do so, click “Attach Integration” near the bottom of the details card. This will route you to the Integrations section of the API; once there, select “wish-list-service” from the dropdown of existing integrations and click “Attach Integration”.

![Attaching Lambda integration to your API](images/jwt7.png)

Your API now calls the Lambda for POST requests!

Next, head to the “Authorization” section via the left hand menu. In the left column, click “Post” under the “/wish-list-service” route, then click “Create and attach an authorizer”. Fill out the form with the following details:

**Name**: auth0
**Identity source**: $request.header.Authorization (this will use the Authorization header to check for the JWT)
**Issuer URL:** https://[YOUR-DOMAIN].auth0.com (this can be found from your JWT metadata endpoint: https://[YOUR-DOMAIN].auth0.com/.well-known/openid-configuration, in the `issuer` field.
**Audience**: your Auth0 API’s unique identifier from the “Create new Auth0 API” step. If you didn’t write it down, you can find it by going to your Auth0 dashboard and navigating to the APIs page in the lefthand menu; the identifier for your API is listed on the landing page.

![Auth0 API Settings](images/jwt8.png)

![JWT Authorizer settings](images/jwt9.png)

Click “Create and attach”, and you’re done! You should now see a green “JWT” pill next to the POST method on your endpoint in the left column of the “Authorization” page.

From this page, you could add scopes if you were using Role-Based Access Control in your Auth0 API, but for the purposes of this post, you’re all set up. The only thing left is to test it out.

## Try it out!

First, let’s ensure the Authorizer is working properly. In your terminal window, make another POST request to the endpoint, ensuring that the `-i` flag is included:

```bash
curl -H "Content-Type: application/json" \
  -X POST \
  -d '{"name": "Test Item 2", "description": "Test Description 2", "url": "https://www.ebay.com"}' \
  -i \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

You should receive a 401 response back, the body of which includes a `message` key with a value of `Unauthorized`. We can no longer create items while unauthorized!

Your GET request should also still work correctly, allowing unauthorized requests. Make a GET request with `-i`:

```bash
$ curl -i https://[DOMAIN].amazonaws.com/default/wish-list-service
```

And you should see a 200 response.

### Get Access Token from Auth0 Dashboard

The moment of truth is here! Head to your Auth0 API dashboard and navigate to the details page of the custom API you created back at the beginning of this tutorial. Go to the “Test” section and copy the `access_token` property from the example response. 

![The access_token property from Auth0 test section](images/jwt10.png)

We’ll be adding this to our cURL POST request as follows:

```
curl -H "Content-Type: application/json" \
  -H "Authorization: [ACCESS_TOKEN]" \
  -X POST \
  -d '{"name": "Test Item 2", "description": "Test Description 2", "url": "https://www.ebay.com"}' \
  -i \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service

```

(One important thing to note: JWT Authorizers don’t require the `bearer` keyword in the Authorization header. The value of the header is simply the access token itself.)

Executing that request should return a 201 as before - and if it does, you’ve successfully integrated Auth0 as a JWT Authorizer on an AWS HTTP API!

## Summary

In this tutorial, you’ve built a functional API using AWS Lambda, DynamoDB, and API Gateway. You’ve then added a JWT Authorizer, restricting access for POST requests to your endpoint to users that have authenticated with an Auth0 application. 

From here, there are a lot of directions you could go. For example, you could expand the functional capabilities of your API in the Lambda itself, or you could refine access by adding authorization scopes within Auth0 and restricting your API endpoints or methods using role-based access control.

The sample code used to run the lambda can be found here: https://github.com/brennanjohn/a0-aws-jwt

Happy building!