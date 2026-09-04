# Hands-On Labs - API Gateway, Lambda and Deployment Strategies

## Lab outcome

Build a small HTTP API using **Amazon API Gateway HTTP API** and **AWS Lambda**, inspect application logs in CloudWatch, intentionally generate an error, and use Lambda versions and aliases to demonstrate controlled deployment, canary routing, promotion and rollback.

By the end of the lab, you will understand this flow:

```text
Client
  |
  v
API Gateway HTTP API
  |
  v
Lambda
  |
  v
CloudWatch Logs
```

You will then use Lambda versions and an alias to demonstrate:

```text
live alias
   |
   +---- 90% ----> Version 1
   |
   +---- 10% ----> Version 2
```

> **Important:** The API Gateway portion of this introductory lab demonstrates how an HTTP API invokes Lambda. The deployment-strategy portion demonstrates Lambda versions and weighted aliases directly in Lambda. This keeps the first lab simple and avoids mixing API Gateway integration reconfiguration into the deployment exercise.

---

## Prerequisites

- Access to the AWS Management Console.
- Permission to manage:
  - AWS Lambda
  - Amazon API Gateway
  - IAM roles
  - Amazon CloudWatch Logs
- Select one AWS Region and use it throughout the lab.

---

# Lab 1 - Create the Lambda function

## Step 1 - Create the function

1. Open **AWS Lambda**.
2. In the left navigation, choose **Functions**.
3. Choose **Create function**.
4. Select **Author from scratch**.
5. Enter:

   - **Function name:** `architecture-orders-api`
   - **Runtime:** Select the latest available **Python 3.x** runtime.
   - **Architecture:** `x86_64`

6. Under permissions, keep the option that creates a new execution role with basic Lambda permissions.
7. Choose **Create function**.

---

## Step 2 - Add the function code

1. Open the **Code** tab if it is not already selected.
2. In the code editor, replace the existing function code with:

```python
import json

def lambda_handler(event, context):

    path = event.get("rawPath", "/")

    method = (
        event.get("requestContext", {})
             .get("http", {})
             .get("method", "UNKNOWN")
    )

    print({
        "method": method,
        "path": path
    })

    return {
        "statusCode": 200,
        "headers": {
            "content-type": "application/json"
        },
        "body": json.dumps({
            "version": "v1",
            "method": method,
            "path": path,
            "message": "Orders API is running"
        })
    }
```

3. Choose **Deploy**.

---

## Step 3 - Test the Lambda function

Instead of using an empty default test event, create an event that resembles the event sent by an API Gateway HTTP API.

1. Choose **Test**.
2. Create a new test event.
3. Enter:

   - **Event name:** `get-orders-test`

4. Replace the event JSON with:

```json
{
  "rawPath": "/orders",
  "requestContext": {
    "http": {
      "method": "GET"
    }
  }
}
```

5. Save the test event.
6. Choose **Test** again.

You should receive a successful response similar to:

```json
{
  "statusCode": 200,
  "headers": {
    "content-type": "application/json"
  },
  "body": "{\"version\": \"v1\", \"method\": \"GET\", \"path\": \"/orders\", \"message\": \"Orders API is running\"}"
}
```

The Lambda console shows the complete Lambda response. Later, API Gateway will return the JSON contained in the `body` field to the client.

---

# Lab 2 - Create the HTTP API

## Step 1 - Start API creation

1. Open **Amazon API Gateway**.
2. Choose **Create API**.
3. Locate **HTTP API**.
4. Choose **Build**.

> API Gateway offers three API types: HTTP API for lightweight, low-cost RESTful APIs, REST API for advanced API-management features such as API keys, request validation, caching, WAF and per-client throttling, and WebSocket API for persistent two-way real-time communication. For this lab, use HTTP API because we only need a simple GET /orders → Lambda integration; use REST API when you need the richer enterprise/API-management capabilities.

---

## Step 2 - Add the Lambda integration

1. Under **Integrations**, choose **Add integration**.
2. Select **Lambda**.
3. Select the same AWS Region used for the Lambda function.
4. Select:

```text
architecture-orders-api
```

5. Enter the API name:

```text
orders-http-api
```

6. Continue to the next step.

---

## Step 3 - Configure the route

Configure the route as:

| Setting | Value |
|---|---|
| Method | `GET` |
| Resource path | `/orders` |
| Integration target | `architecture-orders-api` |

The route is therefore:

```text
GET /orders
```

Continue to the next step.

---

## Step 4 - Configure the stage

1. Keep the **`$default`** stage.
2. Keep **Auto-deploy** enabled.
3. Review the configuration.
4. Choose **Create**.

Your request path is now:

```text
Client
   |
   | GET /orders
   v
API Gateway
   |
   v
Lambda
architecture-orders-api
```

---

# Lab 3 - Validate the API

## Step 1 - Find the invoke URL

1. Open:

   **API Gateway → APIs → orders-http-api**

2. Find the **Invoke URL** for the `$default` stage.

It will look similar to:

```text
https://API-ID.execute-api.REGION.amazonaws.com
```

---

## Step 2 - Invoke the API

Append `/orders`:

```text
https://API-ID.execute-api.REGION.amazonaws.com/orders
```

Open the URL in a browser.

You should receive a response similar to:

```json
{
  "version": "v1",
  "method": "GET",
  "path": "/orders",
  "message": "Orders API is running"
}
```

---

## Step 3 - Understand the request flow

The request now follows this path:

```text
Browser
   |
   | HTTPS GET /orders
   v
API Gateway HTTP API
   |
   | Lambda proxy integration
   v
architecture-orders-api
   |
   v
Lambda response
   |
   v
API Gateway
   |
   v
Browser
```

API Gateway converts the incoming HTTP request into a Lambda event.

That is why the Lambda function can read:

```python
event["rawPath"]
```

and:

```python
event["requestContext"]["http"]["method"]
```

---

# Lab 4 - Inspect Lambda logs in CloudWatch

Every Lambda invocation produces logs in Amazon CloudWatch Logs.

1. Open **CloudWatch**.
2. In the left navigation, choose:

   **Logs → Log groups**

3. Find:

```text
/aws/lambda/architecture-orders-api
```

4. Open the log group.
5. Open the latest log stream.

Find the application log generated by:

```python
print({
    "method": method,
    "path": path
})
```

You should see a value similar to:

```text
{'method': 'GET', 'path': '/orders'}
```

You will also see Lambda-generated entries such as:

```text
START RequestId: ...
END RequestId: ...
REPORT RequestId: ...
```

This establishes the basic troubleshooting path:

```text
API request
    |
    v
Lambda invocation
    |
    v
CloudWatch Logs
```

---

# Lab 5 - Observe a controlled application error

Now intentionally break the Lambda function so that you can observe how an application failure appears.

## Step 1 - Introduce the error

Return to:

**Lambda → Functions → architecture-orders-api → Code**

Add the following line as the first statement inside the handler:

```python
raise Exception("Controlled lab error")
```

The handler should temporarily look similar to:

```python
def lambda_handler(event, context):

    raise Exception("Controlled lab error")

    path = event.get("rawPath", "/")
```

Choose **Deploy**.

---

## Step 2 - Invoke through API Gateway

Open:

```text
https://API-ID.execute-api.REGION.amazonaws.com/orders
```

The request should now fail.

---

## Step 3 - Inspect the error

Open:

**CloudWatch → Logs → Log groups → /aws/lambda/architecture-orders-api**

Open the latest log stream.

Find:

```text
Controlled lab error
```

and the associated Python exception/stack trace.

The troubleshooting path is:

```text
Client
   |
   v
API Gateway
   |
   v
Lambda
   |
   X
Exception
   |
   v
CloudWatch Logs
```

---

## Step 4 - Restore the application

Remove:

```python
raise Exception("Controlled lab error")
```

Choose **Deploy**.

Invoke the API again and confirm that the working response has returned.

---

# Lab 6 - Publish Lambda version 1

Until now you have been modifying:

```text
$LATEST
```

`$LATEST` is the editable, unpublished version of the Lambda function.

A published version is a fixed snapshot of the function code and relevant configuration.

Conceptually:

```text
$LATEST
   |
   | Publish
   v
Version 1
```

---

## Step 1 - Verify version 1 code

Before publishing, confirm that the function contains:

```python
"version": "v1"
```

and that the API works correctly.

---

## Step 2 - Publish the version

1. Open:

   **Lambda → Functions → architecture-orders-api**

2. Open the **Versions** tab.
3. Choose **Publish new version**.
4. Enter the description:

```text
Stable API v1
```

5. Choose **Publish**.

AWS assigns the version number automatically.

For a new function, this will normally be:

```text
1
```

Record the version number shown by the console.

---

# Lab 7 - Create the `live` alias

An alias is a named pointer to a published Lambda version.

Instead of clients needing to know:

```text
architecture-orders-api:1
```

they can conceptually use:

```text
architecture-orders-api:live
```

The alias can later be moved to another version.

---

## Step 1 - Create the alias

1. Open:

   **Lambda → Functions → architecture-orders-api**

2. Choose the **Aliases** tab.
3. Choose **Create alias**.
4. Enter:

   - **Name:** `live`
   - **Description:** `Live Orders API`
   - **Version:** Select the version published in Lab 6.

5. Choose **Save**.

The mapping is now:

```text
live
 |
 v
Version 1
```

---

# Lab 8 - Publish Lambda version 2

Now create the next release.

## Step 1 - Return to `$LATEST`

Open:

**Lambda → Functions → architecture-orders-api → Code**

Make sure you are editing:

```text
$LATEST
```

Published versions cannot be edited.

---

## Step 2 - Modify the code

Change:

```python
"version": "v1"
```

to:

```python
"version": "v2"
```

Also change:

```python
"message": "Orders API is running"
```

to:

```python
"message": "Orders API v2 is running"
```

Choose **Deploy**.

---

## Step 3 - Test `$LATEST`

Use the existing:

```text
get-orders-test
```

test event.

Confirm the response contains:

```json
{
  "version": "v2",
  "message": "Orders API v2 is running"
}
```

---

## Step 4 - Publish version 2

1. Open the **Versions** tab.
2. Choose **Publish new version**.
3. Enter:

```text
Candidate API v2
```

4. Choose **Publish**.

Record the new version number.

If version 1 was the first published version, this will normally be:

```text
2
```

You now have:

```text
$LATEST  -> editable development copy

Version 1 -> stable release
Version 2 -> candidate release

live -> Version 1
```

---

# Lab 9 - Configure a canary deployment using the `live` alias

Lambda aliases can distribute invocations between two published versions.

For this lab:

```text
             +---- 90% ----> Version 1
             |
live alias --+
             |
             +---- 10% ----> Version 2
```

---

## Step 1 - Configure weighted routing

1. Open:

   **Lambda → Functions → architecture-orders-api → Aliases**

2. Select:

```text
live
```

3. Choose **Edit**.
4. Keep the primary **Version** set to version 1.
5. Expand **Weighted alias**.
6. For **Additional version**, select version 2.
7. For **Weight (%)**, enter:

```text
10
```

8. Choose **Save**.

The alias now sends approximately:

```text
90% -> Version 1
10% -> Version 2
```

> Weighted routing is probabilistic. With a small number of requests, the observed percentage may differ significantly from exactly 90/10.

---

# Lab 10 - Test the canary alias

The purpose of this lab is to invoke the **alias**, not `$LATEST`.

## Option A - Lambda console

1. Open the Lambda function.
2. Select the `live` alias.
3. Use the existing test event:

```text
get-orders-test
```

4. Invoke the alias repeatedly.

Observe the response field:

```json
"version": "v1"
```

or:

```json
"version": "v2"
```

Most invocations should reach v1, while some should reach v2.

Because only 10% of the traffic is configured for v2, a very small number of requests might not hit v2 at all.

Run enough test invocations to demonstrate that both versions can be reached.

---

## Optional - Use AWS CLI for repeated testing

If AWS CLI is configured, retrieve the alias repeatedly with:

```bash
aws lambda invoke \
  --function-name architecture-orders-api \
  --qualifier live \
  --payload '{"rawPath":"/orders","requestContext":{"http":{"method":"GET"}}}' \
  response.json
```

Then inspect:

```bash
cat response.json
```

Repeat the invocation several times.

> The CLI is optional. The lab can be completed using only the AWS Management Console.

---

# Lab 11 - Promote version 2

After validating the candidate version, simulate a full production promotion.

## Step 1 - Update the alias

1. Open:

   **Lambda → Functions → architecture-orders-api → Aliases → live**

2. Choose **Edit**.
3. Change the primary version to version 2.
4. Remove the weighted secondary version.
5. Choose **Save**.

The alias now points entirely to:

```text
live
 |
 v
Version 2
```

---

## Step 2 - Validate

Invoke the `live` alias.

The response should contain:

```json
"version": "v2"
```

The deployment occurred without modifying version 1 or version 2.

Only the alias pointer changed.

---

# Lab 12 - Roll back to version 1

Now simulate a rollback because version 2 has an issue.

## Step 1 - Change the alias

1. Open:

   **Lambda → Functions → architecture-orders-api → Aliases → live**

2. Choose **Edit**.
3. Change the primary version back to version 1.
4. Ensure no weighted secondary version is configured.
5. Choose **Save**.

The alias is now:

```text
live
 |
 v
Version 1
```

---

## Step 2 - Validate the rollback

Invoke the `live` alias again.

The response should contain:

```json
"version": "v1"
```

No code was edited.

No old version had to be rebuilt.

The rollback was performed simply by changing the alias.

---