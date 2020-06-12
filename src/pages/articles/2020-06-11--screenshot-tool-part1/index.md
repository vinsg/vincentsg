---
title: Creating and deploying a screenshot tool on AWS using CDK (PART 1)
date: "2020-06-11"
layout: post
draft: false
path: "/posts/screenshot-tool-part1/"
category: "CDK"
tags:
  - "CDK"
  - "Typescript"
  - "AWS"
description: "Building a screenshot tool utility that takes a picture of a website on a set interval and stores it into S3 using AWS CDK"
---

This is the introductory post for my blog which will focus on publishing my various adventures with cloud infrastructure and the CDK.  You are welcome to come around for the ride.

All the code is available on my [GitHub](https://github.com/vinsg/cdk-projects/tree/master/website-screenshot-cron).

## What is CDK?

Formally introduced at [re:Invent 2018](https://www.youtube.com/watch?v=Lh-kVC2r2AU), the CDK (Cloud Development Kit) is fast becoming an essential tool to model and deploy cloud infrastructures including serverless workloads on AWS. You can use various imperative programming languages like Typescript or Python to code your cloud infrastructure.  It accomplishes this by generating the CloudFormation code necessary to deploy the stack of resources defined in our code. This new paradigm greatly simplify the process of writing CloudFormation boilerplate and makes infrastructure-as-code much more accessible to programmers.

## Description

Our starting project will be a simple serverless screenshot tool entirely cloud native to AWS. The purpose of this project is to introduce the key concepts of the CDK and experience its components in action. We will be building a screenshot tool utility that takes a picture of a website on a set interval and stores it into S3. 

Here is a diagram of what we will be building.

![website-screenshot-diagram-v1.png](website-screenshot-diagram-v1.png)

## Environment Setup

This assumes you have the aws-cli installed and configured. See [AWS documentation.](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

Setup the CDK tools using npm.

```bash
npm install -g aws-cdk
```

If this is the first time running CDK, you need to run the bootstrap command.

```bash
cdk bootstrap
```

In an empty directory, setup a boilerplate typescript project.

```bash
cdk init --language typescript
```

Using a new terminal window, run this command to watch and compile all our typescript files during development.

```bash
npm run watch
```

## Building the Stack

CDK treats every AWS as a separate module, we need to install all our dependencies for all our services.

```bash
npm i @aws-cdk/aws-s3
npm i @aws-cdk/aws-lambda
npm i @aws-cdk/aws-events
npm i @aws-cdk/aws-events-targets
```

We start by importing our S3 dependency and defining our S3 bucket.

```tsx
import { Bucket } from '@aws-cdk/aws-s3';

// S3 bucket
const bucket = new Bucket(this, "url-bucket");
```

In order to screenshot a website, we will need to run a headless chrome instance. Lucky for us, instead of having to bundle all the dependencies with our lambda function, we can make use of lambda layers. [Shelf.io](http://shelf.io) makes available a chrome-aws-lambda through an Arn.

```tsx
import * as lambda from '@aws-cdk/aws-lambda';

// Chrome browser lambda layer
const layer = lambda.LayerVersion.fromLayerVersionAttributes(this, 'Layer', {
  layerVersionArn: "arn:aws:lambda:us-west-2:764866452798:layer:chrome-aws-lambda:8",
  compatibleRuntimes: [lambda.Runtime.NODEJS_12_X]
});
```

We then use this layer in our lambda definition.

```tsx
// lambda function
const fn = new lambda.Function(this, "screenshot", {
  runtime: lambda.Runtime.NODEJS_12_X,
  code: lambda.Code.fromAsset("lambda"),
  handler: "screenshot.handler",
  environment: {
    TARGET_URL: "https://www.google.com/",
    S3_BUCKET: bucket.bucketName
  },
  layers: [layer],
  memorySize: 1600,
  timeout: cdk.Duration.minutes(1)
});
```

CDK will load the lambda code found in the "lambda" directory of your project. We will create this function later.

```tsx
code: lambda.Code.fromAsset("lambda"),
```

Not only can we load environment variables, but we can also pass in our S3 bucket name which will be filled automatically during our deployment.

```tsx
environment: {
  TARGET_URL: "https://www.nytimes.com/",
  S3_BUCKET: bucket.bucketName
},
```

This line is where we pass our lambda layer we defined earlier.

```tsx
layers: [layer],
```

Crafting precise restrictive permissions IAM is very easy using the CDK. This line is all we need to setup the proper role to allow our lambda to write to our S3 bucket. 

```tsx
// grant permission for lambda function to write to s3 bucket
bucket.grantWrite(fn);
```

The only piece missing is our scheduling event. For this, we define a CloudWatch rule to run every day.

```tsx
import { Rule, Schedule } from '@aws-cdk/aws-events';

// CloudWatch rule
const eventRule = new Rule(this, "cron-screenshot", {
  schedule: Schedule.cron({minute: "0", hour: "10"}), // runs every day at 10:00 am
});
```

and we finally need to set our lambda function as a target for our rule.

```tsx
import * as Target from '@aws-cdk/aws-events-targets';

// Set function as rule target
eventRule.addTarget(new Target.LambdaFunction(fn));
```

## The lambda function

Now that our infrastructure is written, let's take care of our lambda function. In a new directory called 'lambda' create a new file 'screenshot.js'. This function will spin a headless Chrome instance, open the url passed by our TARGET_URL env variable and save a screenshot at our S3_BUCKET name with a date title.

```jsx
// dependencies
const AWS = require("aws-sdk");
const url = require("url");
const chromium = require("chrome-aws-lambda");

// S3 client
const s3 = new AWS.S3();

exports.handler = async function (event) {
  const browser = await chromium.puppeteer.launch({
    args: chromium.args,
    defaultViewport: chromium.defaultViewport,
    executablePath: await chromium.executablePath,
    headless: chromium.headless,
    ignoreHTTPSErrors: true,
  });
  const page = await browser.newPage();
  await page.goto(process.env.TARGET_URL, {
    waitUntil: "networkidle0",
    timeout: 3000000,
  });

  const screenshot = await page.screenshot({ fullPage: true });

  // Date
  const d = new Date();
  const day = d.getDate();
  const month = d.getMonth();
  const year = d.getFullYear();

  const key =
    url.parse(process.env.TARGET_URL).hostname +
    "/" +
    year +
    "-" +
    month +
    "-" +
    day +
    ".png";

  await s3
    .putObject({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: screenshot,
      ContentType: "image/png",
    })
    .promise();

  await browser.close();

  console.log("Screenshot ", key, "saved");
  return;
};
```

## Deploy

Running the command

```bash
cdk synth
```

will generate the CloudFormation file used to deploy the infrastructure.

To deploy the infrastructure and upload all our code, we simply run

```bash
cdk deploy
```

We then can go into the AWS console and see all the function code and the S3 bucket.

## Wrap-Up and What's Next?

Great now if everything went well, we should have our first cloud infrastructure deployed using the CDK. During this blog post, we setup an S3, a lambda function using layers, a CloudWatch Event Rule and tied it all up using properly scoped permissions.

Now that we have a first version of our project done, join me in PART 2 where we will refine our infrastructure by adding Testing, Monitoring and more improvements.