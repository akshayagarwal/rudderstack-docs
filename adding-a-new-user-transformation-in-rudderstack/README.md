---
description: Detailed technical documentation on RudderStack's Transformations feature.
---

# User Transformations in RudderStack

Transformations is one of RudderStack's key features. It allows you to leverage custom JavaScript functions to transform the events in a destination-specific format. These transformations can be used across your Event Stream, Warehouse Actions, and Cloud Extract pipelines, and help you implement a variety of use-cases, like:

* Filtering and/or sampling events.
* Enriching events by implementing static logic or by leveraging an external API.
* Cleaning data by removing unnecessary bits.
* Data masking or removing sensitive PII information in the events to ensure data privacy.
* Data aggregation / rolling-up at a micro-batch level.
* Implementing external actions on the events in the stream by means of an API.

{% hint style="success" %}
Our [Sample User Transformations](https://github.com/rudderlabs/sample-user-transformers) GitHub repository contains some useful templates that you can use to create your own user transformations.
{% endhint %}

{% hint style="warning" %}
**Please note that the user transformations only work for the cloud mode destinations.** 

**To know more about the cloud mode in RudderStack, check out the** [**RudderStack Connection Modes**](https://docs.rudderstack.com/get-started/rudderstack-connection-modes) **guide.**
{% endhint %}

## Creating a New Transformation in RudderStack

* Log into your [RudderStack dashboard](https://app.rudderstack.com/).
* Click on the [Transformations](https://app.rudderstack.com/transformations) Link from the left panel in the dashboard, as shown:

![](../.gitbook/assets/1%20%2816%29.png)

* Click on the **Create New** option as shown:

![](../.gitbook/assets/2%20%2822%29.png)

* Next, assign a name for this new transformation. Enter the transformation function's code in the **Transformation** window, as shown:

![](../.gitbook/assets/3%20%2819%29.png)

* Add your transformation code within the `transformEvent` function in the **Transformation** window. You can also add other functions and call them from within `transformEvent`.

![](../.gitbook/assets/screenshot-2021-05-05-at-4.51.46-pm.png)

{% hint style="success" %}
Our [Sample User Transformations](https://github.com/rudderlabs/sample-user-transformers) GitHub repository contains some useful templates that you can use to create your own user transformations. 

You can copy-paste the entire code of any of the functions present in this repository into the **Transformation** window. 

**Note**: Do remember to delete the pre-populated `transformEvent` function in such cases, before pasting your code.
{% endhint %}

* RudderStack also gives you the ability to test your transformation function with the **Run Test** option as shown below:

![](../.gitbook/assets/image-6-.png)

* In case you want to perform any aggregation/roll-up operation on a micro batch of events, use the `transformBatch` feature, as shown:

```javascript
/***  
* This function gets executed on a batch of events before it gets pushed to a destination
* events    => JSON list of events sent to rudder
* metadata (optional) => Javascript function which can be used to access metadata of
                        the given event by calling metadata(event)  
* After all the transformations are done, the final event that 
* needs to be pushed to the destination should be returned by this function
***/

export function transformBatch(events, metadata) {      
  return events
}
```

{% hint style="warning" %}
**Disclaimer for using `transformBatch`** 

Make sure you pass on the `messageID` from the input event onto the output event. If the `messageID` is passed on, the order of delivery of events is preserved. **Without it, this order may not be maintained**. It is highly recommended to use `transformEvent` in every possible case as it ensures event ordering in all cases. 

In case a `transformBatch` is required and event ordering is important, make sure to pass on the `messageID`.
{% endhint %}

## Libraries

One of the key features of RudderStack transformations is the ability to reuse code written for a transformation in other transformations as well. For this, RudderStack lets you create libraries or functions that can be reused in different transformations. 

To create a library, follow these steps:

* Click on the [Transformations](https://app.rudderstack.com/transformations) Link to see the **Libraries** section. Use **Create New** option to add new libraries.

![](../.gitbook/assets/4%20%2819%29.png)

* Add the library's **Name** and include the custom functions that you need to reuse across all other transformations as shown:

![](../.gitbook/assets/screenshot-2021-05-05-at-6.29.27-pm.png)

* More functions can be added under a single library, as shown:

![](../.gitbook/assets/screenshot-2021-05-05-at-6.33.40-pm.png)

{% hint style="warning" %}
RudderStack does not support the deletion of libraries as of now. This is so that you don't break any existing transformations that use the library that you might be trying to delete.
{% endhint %}

## Using Libraries in Transformations

To use the libraries in your existing transformation, simply take the name of your library and convert it into **camel case without spaces** - this becomes your library handle. 

As an example, if your library name is **`is rudder email`**, then the library handle would be **`isRudderEmail`**.

![](../.gitbook/assets/screenshot-2021-05-05-at-6.39.13-pm.png)

Let's say you want to import a function called **`rudderEmail`**, which returns `true` for the emails from a specific domain \(e.g. RudderStack\) and `false` otherwise, from the **`is rudder email`** library. Also, you want to use this function to filter the events that don't have the email address of the specified domain. 

We can implement this in the following way:

```javascript
import { rudderEmail } from 'isRudderEmail'

export function transformEvent(event) {
    const email = event.context && event.context.traits && event.context.traits.email;
    if (email) {
      if (!rudderEmail(email)) return;
    }
    return event;
}
```

On running a test, an example event not having the specified email domain is filtered out, as shown below:

![](../.gitbook/assets/image-7-.png)

## Accessing Metadata

RudderStack injects a function `metadata(event)` into your transformations as an argument. This allows you to access the event metadata variables that help you customize your transformations.

{% hint style="info" %}
`metadata()` takes the event as input and returns the metadata of the event.
{% endhint %}

The following properties, if available, are present in the metadata response:

| Property name | Description |
| :--- | :--- |
| `sourceId` | This refers to the ID of the source configured on your RudderStack dashboard. Note that it different from the source **Write Key**. Refer to the image below for more details. |
| `destinationId` | ID of the destination configured on your RudderStack dashboard. |
| `messageId` | Corresponds to the unique ID for each event. |
| `sessionId` | If sessions are enabled, this corresponds to the value of the session ID. |

Since you may not need the metadata in every transformation, you can optionally access it by including it in your function signature as an argument wherever required. 

An example of this is as shown below:

```javascript
export function transformEvent(event, metadata) {
  const meta = metadata(event);
  event.sourceId = meta.sourceId
  
  return event;
}
```

![Source ID corresponding to a source configured in RudderStack.](../.gitbook/assets/1%20%2817%29.png)

![](../.gitbook/assets/image-8-.png)

## External API Requests

You can make any number of external API requests in your transformer functions and use the response to enrich your events data. RudderStack injects an asynchronous `fetch(url)` function into your transformations. It makes an API call to the given URL and returns the response in a JSON format. 

Examples of how to use the `fetch` function in transformations are shown below:

{% tabs %}
{% tab title="Basic" %}
```javascript
export async function transformEvent(event) {
    const res = await fetch("any_api_endpoint");
    event.response = JSON.stringify(res);
    return event;
}
```
{% endtab %}

{% tab title="POST" %}
```javascript
export async function transformEvent(event) {
    const res = await fetch("post_url", {
        method: "POST", // POST, PUT, DELETE, GET, etc.
        body: JSON.stringify(event),
        headers: {
         "Content-Type": "application/json;charset=UTF-8",
        },
    });
    event.response = JSON.stringify(res);
    return event;
}
```
{% endtab %}

{% tab title="Headers" %}
```javascript
export async function transformEvent(event) {
    const res = await fetch("post_url", {
        method: "POST", // POST, PUT, DELETE, GET, etc.
        headers: {
         "Content-Type": "application/json;charset=UTF-8",
         "Authorization": "Bearer <your_authorization_token>"
        },
        body: JSON.stringify(event),
    });
    event.response = JSON.stringify(res);
    return event;
}
```
{% endtab %}
{% endtabs %}

You can check out the [Clearbit enrichment example](https://github.com/rudderlabs/sample-user-transformers/blob/sampleTransformationV1/EnrichWithClearbit.js) which uses the `fetch` function.

{% hint style="success" %}
Use `batch` API requests instead of a separate API request for each event whenever possible for improved performance.
{% endhint %}

## Limits

You should take into account the memory and time limits when adding a new user transformation. Each invocation of the user transformation should not exceed these limits. 

The limits are:

| Parameter | Limit |
| :--- | :--- |
| Memory Limit | 8 MB |
| Time Limit | 4 seconds |

{% hint style="warning" %}
The user transformation fails if these limits are exceeded.
{% endhint %}

## Contact Us

If you want to know more about transformations and libraries in RudderStack, feel free to [contact us](mailto:%20docs@rudderstack.com). You can also talk to us on our [Slack](https://resources.rudderstack.com/join-rudderstack-slack) channel.

