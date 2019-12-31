---
layout: post
title: "Securing AWS HTTP APIs with JWT Authorizers"
description: "A tutorial on integrating Auth0 with HTTP APIs built on AWS API Gateway"
date: "2019-12-23 08:30"
author:
  name: "John Brennan"
  url: "https://github.com/jhnbrnn"
  mail: "brennan.john@mac.com"
  avatar: "https://twitter.com/jhnbrnn/profile_image?size=original"
tags:
- AWS
- node
- javascript

---

**TL;DR:** [JSON Web Tokens (JWTs)](https://auth0.com/resources/ebooks/jwt-handbook) can be used to authorize requests to HTTP APIs built on AWS API Gateway. This tutorial will walk you through integrating Auth0 with such an HTTP API.

---

## An introduction to HTTP APIs in API Gateway

AWS API Gateway offers several solutions for building scalable APIs. [AWS HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html) are simple APIs that receive HTTP requests and send them to [Integrations](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-concepts.html#http-api-concepts.integrations) such as another public HTTP endpoint or an AWS Lambda. Unlike API Gateway's REST APIs, HTTP APIs can’t leverage other AWS services, such as AWS Identity and Access Management (IAM), as _Authorizers_: services that allow or restrict API access to clients based on criteria such as user, roles, IP addresses, and so on. 

HTTP APIs can, however, use JSON Web Tokens (JWTs) to provide access control using a **JWT Authorizer**.

## What is a JWT Authorizer?

A JWT Authorizer configures the endpoint of your HTTP API to expect a token, created by an Identity Provider such as Auth0, to be provided in HTTP requests to the API. The Authorizer will validate the token and ensure the requesting client is allowed to access the endpoint. JWT Authorizers can also optionally restrict access based on authorization scopes in the JWT.

A benefit of JWT Authorization is the ability to pair existing identity provider solutions with the low overhead of HTTP APIs. This provides security and authorization without needing to rely on heavier processes such as AWS IAM or resource policy files.

## What will you build?

In this tutorial, you’ll learn how to add a JWT Authorizer to an HTTP API, using Auth0 as the identity provider. You’ll be building an HTTP API on AWS using API Gateway, a Lambda function, and DynamoDB. The API will support both `GET` and `POST` requests and read/write data from a DynamoDB table. `POST` requests will be restricted to authorized users and `GET` requests will be unauthorized, meaning anyone can access them.

Your API will manage the creation and retrieval of a simple public wish list of items, similar to Amazon's Wish List functionality. Anyone will be able to read the wish list, but only authorized users will be able to add items to the list.

## Prerequisites

In order to create the various parts of the API, you'll need an AWS account. You can [sign up for a free tier account here](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc).

You'll also need an Auth0 API that will work as the JWT Authorizer for your API. If you don't have one already, you can [sign up for an Auth0 account here](https://auth0.com/signup).

Lastly, you'll need some knowledge of modern JavaScript (ES2017 and, specifically, [async functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)) and Node.js in order to create the Lambda that will power your API.

## Create a new Auth0 API

Your first step will be to create a new Auth0 API. After you've signed in (or [signed up](https://auth0.com/signup)), head to your [**Auth0 Dashboard**](https://manage.auth0.com/#/) and click “APIs” in the left-hand menu. Click the “Create new API” button and fill out the form with the following values:
**Name:** AWS JWT Authorizer

**Identifier:** https://auth0-jwt-authorizer.com

**Signing Algorithm:** RS256 (default value)

Click the “Create” button, and you’ll be taken to the definition page for your new API.

![an example of the Auth0 API settings](images/jwt1.png)

You’ll want to copy or write down your API’s identifier; we’ll need it later on.

## Create HTTP API (Wish List API) on AWS

For this step, we’ll be using three services in AWS:

* DynamoDB (a NoSQL database to store our wish list items)

* Lambda (the business logic that will power our API, implemented as a serverless function)

* API Gateway (to create the public API endpoint itself and add the JWT Authorizer)

### Create DynamoDB

Navigate to the [AWS DynamoDB dashboard](https://console.aws.amazon.com/dynamodb/home), signing into AWS if needed. Click the “Create Table” button to get started.

We’ll name our table “WishList” and give the Primary Key a name of “id”, leaving its type as the default (String). Leave the rest of the form as-is; just scroll down and click Create.

![Settings for the DynamoDB table](images/jwt2.png)

That’s all you need to do for setup, so it’s time to head over to Lambdas!

### Create Lambda

Navigate to the [Lambda dashboard](https://console.aws.amazon.com/lambda/home) and click the “Create function” button in the table’s top right corner.

The Lambda service provides a lot of helpful blueprints and samples, but for this tutorial you’ll be creating your Lambda's functionality from the ground up, so leave “Author from scratch” selected at the top of the form.

Give your function the name “wish-list-service” and set the runtime to `Node 12.x`. In the Permissions section, open up the accordion for creating an execution role; for this tutorial, you’ll use a template for setting permissions to allow the Lambda to use the DynamoDB table you just created.

In the "Execution Role" radio group, select “Create a new role from AWS policy templates.” Name the role “wish-list-service-role” and, in the Policy templates dropdown, select “Simple microservice permissions.” This policy template will automatically create the AWS IAM role that allows your Lambda to access DynamoDB. If you're curious, you can go to the [AWS IAM Roles Dashboard](https://console.aws.amazon.com/iam/home#/roles) and click "wish-list-service-role" to understand exactly what permissions are granted.

![Settings for the Lambda](images/jwt3.png)

Click “Create Function” in the bottom right corner of the page, and you’ll shortly be taken to the details page for your new Lambda. And now, it's time to code!

Start by replacing any existing code with the following empty handler function:

```js
exports.handler = async (event, context) => {

};
```

You’re using the `async` keyword because the code will asynchronously access DynamoDB, which you’ll see in a moment. The `event` argument is a [core Lambda concept](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-concepts.html#gettingstarted-concepts-event) - it's an object representing the data for the Lambda to process, and in your case, it represents the HTTP request received from your API. An [example event from API Gateway](https://docs.aws.amazon.com/lambda/latest/dg/with-on-demand-https.html) can be found in the Lambda documentation, if you're looking for a more detailed rundown of what's available in the event. The [`context` argument](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-prog-model-context.html) contains useful information about the execution environment; we'll be using it for its `awsRequestId` property in a moment.

`event` and `context` will both be used in a moment to retrieve information about the API request.

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

[`AWS.DynamoDB.DocumentClient()`](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html) creates an object that allows you to perform actions against your DynamoDB tables.

Now you're ready to write your handler method. Since you’re handling both `GET` and `POST` requests in one handler, a simple `switch` statement based on the HTTP method should do the trick:

```js
// ...

exports.handler = async (event, context) => {
	switch(event.httpMethod) {
		case "GET":
			break;
		case "POST":
			break;
		default:
			break;
	}
}

```

With the scaffolding in place, it’s time to implement the `GET` case. It’s simple enough: when a `GET` request is received, you’ll retrieve records from the `WishList` table and return them:

```js
// ...
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
		
```

`ddb.scan()` Will retrieve all records found in a DynamoDB table; the table is specified by the `TableName` key in the function parameter.

The object that the Lambda is returning adheres to a [specific format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format) for the return value, expected by API Gateway.

Next up: `POST` requests. This implements a request to create a new item on your Wish List. The request should send three properties that you'll store in the database:

* `name` (the name of the item on your wish list)

* `description` (a helpful description of the item)

* `url` (a link to where the item can be purchased). 

You'll also need an ID for the item to match the `id` field you created in DynamoDB. For simplicity, you can use the guaranteed-to-be-unique `awsRequestId` property from the `context` parameter provided to the handler function.

Finally, after the item’s been added to the database, you can return the ID in the body of the successful response.

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

And that’s it! Here’s a complete look at your Lambda’s code:
 
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

After you’ve added your code, click the orange “Save” button in the top right corner. Your Lambda is now live, and you’re ready to create the HTTP API that will make this Lambda’s functionality publicly available.

### Create an HTTP API in front of your Lambda

There are a few different ways to create an HTTP API; for this tutorial, you’ll be creating the API from the Lambda details page in which you’ve been working. By creating it from the Lambda, AWS will configure the API’s permission settings by default, allowing it to execute the Lambda when a request is received.

Scroll to the top of the page and open the Designer section - this is a visual representation of your Lambda, plus other AWS services that are connected to it. These take the form of Triggers &mdash; various AWS services that can call your Lambda &mdash; and Destinations &mdash; services to which the Lambda's execution results can be routed. More information about this can be found in the [Function Configuration](https://docs.aws.amazon.com/lambda/latest/dg/resource-model.html) docs on AWS.

In the Function Designer, click the “Add trigger” button in the diagram's left side. Select “API Gateway” in the form’s dropdown, then “Create a new API” from the next dropdown that appears. In the form elements that follow, HTTP API should be selected by default, and you don’t need to change any of the settings - just head to the bottom of the form and click the “Add” button.

![Settings for the HTTP API Trigger](images/jwt4.png)

You should be returned to the configuration page for your Lambda. In the Designer diagram, the left branch should how contain a linked child on its left branch called API Gateway, which should be selected. Below the Function Designer, a section called API Gateway should be open and include details about your newly-created HTTP API. 

![The newly created API in the Function Designer](images/jwt11.png)

By creating the API through the Function Designer, the API should be automatically connected to the Lambda, so let’s give it a test to make sure. Grab the URL of your API from the API Gateway section and open up a new terminal window.

First, make a `POST` request to your endpoint using cURL to create a new wish list item:

```bash
$ curl -H "Content-Type: application/json" \
  -X POST \
  -i \
  -d '{"name": "Test Item", "description": "Test Description", "url": "https://www.amazon.com"}' \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

The `-i` flag is included to provide more information about the response from the API.

You should get back a `201` response with a JSON payload containing an `id` key.

Next, make a `GET` request to ensure your new item was created:

```bash
$ curl -i https://[DOMAIN].amazonaws.com/default/wish-list-service
```

The returning JSON payload should contain a key called `Items`, the value of which is an array of wish list items. Your new item will appear in that array.

### Add a JWT Authorizer to your API

Now for the final step - setting up your Auth0 API as a JWT Authorizer. Head over to the [API Gateway dashboard](https://console.aws.amazon.com/apigateway/main/apis) and click “wish-list-service-API” to open up the API’s details page.

First, navigate to the “Routes” section from the left-hand menu. Currently, the API is configured to allow any type of request to the `wish-list-service` endpoint, so you’ll need to change that first. Click “ANY” under the “/wish-list-service” route in the left column, then click “Edit” in the details card’s header on the right. In the dropdown, change “ANY” to “GET” and click save.

![Changing the route to GET only](images/jwt5.png)

You’ve now restricted the existing behavior &mdash; allowing requests to execute the Lambda &mdash; to only work on `GET` requests.

To test that your configuration works, you can make another `POST` request, as before:

```bash
$ curl -H "Content-Type: application/json" \
  -X POST \
  -i \
  -d '{"name": "Test Item", "description": "Test Description", "url": "https://www.amazon.com"}' \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

You should receive the following payload in response:

```
{"message":"Internal Server Error"}
```

Once you’re back on the Routes page, click “Create” in the left column header to create a new route. Select “POST” from the methods dropdown and set the URL to `/wish-list-service`, then click the "Create" button. 

![Adding a POST method to the route](images/jwt6.png)

You’ll be redirected to the Routes page; click “POST” under the “/wish-list-service” route in the left column to open up the details card.

First, you’ll need to configure the route to call your Lambda. To do so, click “Attach Integration” near the bottom of the details card. This will route you to the Integrations section of the API; once there, select “wish-list-service” from the dropdown of existing integrations and click the “Attach Integration” button.

![Attaching Lambda integration to your API](images/jwt7.png)

Your API now allows `POST` as well as `GET` requests to run the Lambda.

Run the followig cURL command to verify that this works:

```bash
$ curl -H "Content-Type: application/json" \
  -X POST \
  -i \
  -d '{"name": "Another Test Item", "description": "Another Description", "url": "https://www.amazon.com"}' \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

You should receive a `201` in response - `POST`s are working again!

Next, head to the “Authorization” section via the left-hand menu. In the left column, click “Post” under the “/wish-list-service” route, then click the “Create and Attach an Authorizer” button. Fill out the form with the following details:

#### Name:

auth0

#### Identity source:

`$request.header.Authorization` (this will use the Authorization header to check for the JWT)

#### Issuer URL:

This can be found from your OpenID Configuration endpoint: https://[YOUR-DOMAIN].auth0.com/.well-known/openid-configuration, in the `issuer` field.

#### Audience:

Your Auth0 API’s unique identifier from the “Create new Auth0 API” step. If you didn’t write it down, you can find it by going to your [Auth0 dashboard](https://manage.auth0.com/#/) and navigating to the APIs page in the lefthand menu; the identifier for your API is listed on the landing page.

![Auth0 API Settings](images/jwt8.png)

![JWT Authorizer settings](images/jwt9.png)

Click “Create and attach”, and you’re done! You should now see a green “JWT” pill next to the `POST` method on your endpoint in the left column of the “Authorization” page.

From this page, you could add scopes if you were using Role-Based Access Control in your Auth0 API, but for this tutorial, you’re all set up. The only thing left is to test it out.

## Test it out!

First, you should ensure the Authorizer is working properly. In your terminal window, make another `POST` request to the endpoint:

```bash
curl -H "Content-Type: application/json" \
  -X POST \
  -d '{"name": "Test Item 3", "description": "Test Description 3", "url": "https://www.amazon.com"}' \
  -i \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

You should receive a `401` response, the body of which includes a `message` key with a value of `Unauthorized`. Your API no longer allows unauthorized creation of wish list items!

Your `GET` request should still work correctly, allowing unauthorized requests:

```bash
$ curl -i https://[DOMAIN].amazonaws.com/default/wish-list-service
```

And you should see a `200` response.

### Get Access Token from Auth0 Dashboard

The moment of truth is here! Head to your [Auth0 API dashboard](https://manage.auth0.com/#/apis) and navigate to the details page of the custom API you created back at the beginning of this tutorial. Go to the “Test” section and click the "copy token" button in the top right of the code snippet (see screenshot below) to copy the `access_token` property from the Response section. 

![The access_token property from Auth0 test section](images/jwt10.png)

We’ll be adding this to our cURL `POST` request as follows:

```bash
curl -H "Content-Type: application/json" \
  -H "Authorization: [ACCESS_TOKEN]" \
  -X POST \
  -d '{"name": "Test Item 2", "description": "Test Description 2", "url": "https://www.ebay.com"}' \
  -i \
  https://[SUBDOMAIN].amazonaws.com/default/wish-list-service
```

(One important thing to note: JWT Authorizers don’t require the `bearer` keyword in the Authorization header. The value of the header is simply the access token itself.)

That request should return a `201` status code - which means you’ve successfully integrated Auth0 as a JWT Authorizer on an AWS HTTP API!

## Summary

In this tutorial, you’ve built a functional API using AWS Lambda, DynamoDB, and API Gateway. You’ve then added a JWT Authorizer, restricting access for `POST` requests to your endpoint to users that have authenticated with an Auth0 application. 

From here, there are several next steps you could take. For example, you could expand the functional capabilities of your API in the Lambda itself, or you could refine access by adding authorization scopes within Auth0 and restricting your API endpoints or methods using role-based access control.

The sample code used to run the Lambda can be found [in this GitHub repository](https://github.com/brennanjohn/a0-aws-jwt).

Happy building!
