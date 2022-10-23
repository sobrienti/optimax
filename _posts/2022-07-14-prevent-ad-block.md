---
title: Prevent Optimizely from being blocked by ad-blockers
date: 2022-07-13 01-00
categories: []
tags: [Optimizely Web, Optimizely Full Stack, Events]     # TAG names should always be lowercase
author: david
img_path: /assets/img/posts/prevent-ad-blockers
---

Some ad-blockers might block Optimizely from running. In this tutorial, we'll see how we can prevent this from happening. 

Ad-blockers will often look at the domain the request is originating from and if it happens to be from a list of known analytics tools (such as Optimizely.com), the ad blocker will block the network request from happening. 

The key solution is to proxy requests via an API gateway which will forward all requests back to Optimizely.com

Here's how. 

## If you are using AWS

### Step 1: Create an AWS API Gateway

Head the [AWS Management Console](https://aws.amazon.com) then head to the API gateway section

![AWS API Gateway](/image1.png)
_The AWS API Gateway Section_

Then, click on *Create New API*

![AWS API Gateway](/image2.png)
_Click to create a new API_

Select *HTTP API* as the API type:
![AWS API Gateway](/image3.png)
_Select API type_

### Step 2: Create the required routes 

Now we are going to create our API routes and where the API will forward the requests to. 

We'll need 3 routes, depending on the Optimizely product you use: 
1. One that allows us to retrieve the Optimizely snippet (if you are using Web). The snippet is fetched via a `GET` request done to https://cdn.optimizely.com. You can find the full snippet URL inside your Project settings. 
2. One that allows us to send decision & conversion events (logx.optimizely.com). This route will be fetched via a `POST` request to https://logx.optimizely.com/v1/event. 
3. One that allows us to retrieve the Optimizely datafile (if you are using Full Stack). This route will be fetched via a `GET` request to https://cdn.optimizely.com/datafiles. You can find your datafile URL inside your Project settings. 


![AWS API Gateway](/image4.png)
_Add to these values your Optimizely snippet and/or datafile_

Here's how it should look like once properly filled out: 

![AWS API Gateway](/image5.png)
_With snippet and/or datafile filled out_

AWS will ask you for a confirmation: 

![AWS API Gateway](/image6.png)
_Final review_

Change the ANY to be exactly the same as the method on the right-hand side. (POST, GET & GET)

Congrats you now have a working API which will proxy requests to Optimizely.com

### Step 3: Fetch Optimizely from your newly-created API gateway

Now that we've got a working API, it's time to update our website to start fetching from this API. 

#### Optimizely Web

Update your script tag that contains Optimizely to no longer fetch the file from cdn.optimizely.com but from your AWS API Gateway. 
You'll find the invoke URL on the main API page, as such: 

![AWS API Gateway](/image7.png)
_The invoke URL for your API_

Now to ensure the Optimizely snippet sends events to the API gateway instead of the default Optimizely endpoint, this is a custom snippet setting that can't be configured by a customer. You'll need to ask your account manager about it. They can amend your snippet to ensure the snippet dispatches events to your newly-created API. 

#### Optimizely Full Stack

For Full Stack, you'll need to customize the SDK's *createInstance* method to include a new datafile URL pointing to your API, as such

```javascript
const optimizely = require('@optimizely/optimizely-sdk');

const optimizelyClientInstance = optimizely.createInstance({
  sdkKey: '[YOUR_SDK_KEY]',
  datafileOptions: {
    autoUpdate: true,
    urlTemplate: 'https://[API_GATEWAY_INVOKE_URL]/datafiles/%s.json',
  },
});
```

You'll also need to provide a custom event dispatcher which will dispatch the events back to your newly AWS API Gateway. You can use [this built-in event dispatcher](https://github.com/optimizely/javascript-sdk/blob/master/packages/optimizely-sdk/lib/plugins/event_dispatcher/index.browser.ts#L39) and change line 39 to indicate the POST endpoint of your newly created API.
