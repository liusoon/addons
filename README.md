# Building Add-on integrations with Netlify

A Netlify add-on is a way for Netlify users to extend their site functionality.

**Some examples of add-ons:**

- Automatically inject Sentry error tracking into your app
- Provision a new Fauna database
- Setup `env` variables in Netlify's build context for users Automatically
- ...

## Table of Contents
<!-- AUTO-GENERATED-CONTENT:START (TOC) -->
- [Getting Started](#getting-started)
- [The Add-on Provisioning Flow](#the-add-on-provisioning-flow)
- [Creating Your Add-on API](#creating-your-add-on-api)
  * [Create an Add-on Instance](#create-an-add-on-instance)
  * [Updating an Add-on Instance](#updating-an-add-on-instance)
  * [Deleting an Add-on Instance](#deleting-an-add-on-instance)
  * [Getting an Add-on Instance](#getting-an-add-on-instance)
- [Proxied URLs](#proxied-urls)
  * [Request Headers](#request-headers)
  * [Verification with JWS](#verification-with-jws)
  * [User-Level Authentication with JWTs](#user-level-authentication-with-jwts)
- [Registering your add-on](#registering-your-add-on)
- [Example Implementations](#example-implementations)
<!-- AUTO-GENERATED-CONTENT:END -->

## Getting Started

Each netlify add-on service must offer a [management API](#creating-your-add-on-api) that Netlify will use to provision the service for new projects, manage configuration settings and update the plans for your service.

## The Add-on Provisioning Flow

This is a diagram of how Netlify provisions your add-on service.

<img width="100%" alt="provisioning flow for repo" src="https://user-images.githubusercontent.com/532272/45775428-93c74000-bc04-11e8-9a27-084170353563.png">


## Creating Your Add-on API

In order for Netlify customers to create and manage their own instances of your service, you'll need to create a management API that exposes the following endpoints:

```
GET     /manifest       # returns the manifest for the API
POST    /instances      # create a new instance of your service
GET     /instances/:id  # get the current configuration of an instance
PUT     /instances/:id  # update the configuration of an instance
DELETE  /instances/:id  # delete an instance
```

### Manifest Endpoint

The manifest endpoint of a Netlify add-on is used to return information about your service to the Netlify user.

**The manifest includes:**

- `name` - The name of your add-on
- `description` - A brief description of your add-on
- `config` - The inputs required from the user for the add-on to provision itself.

**Example response**

```js
{
  statusCode: 200,
  body: JSON.stringify({
    name: "My Awesome Integration",
    description: "This addon does XYZ.",
    admin_url: 'https://your-admin-url.com',
    config: {
      "optionOne": {
        // An alternate, human-friendly name.
        "displayName": "Twilio Account SID",
        // Type of field
        "type": "string",
        // If is required or not
        "required": true,
      },
      "optionTwo": {
        "displayName": "Twilio Account Authentication Token",
        "type": "string"
      },
      "fooBarZaz": {
        "displayName": "Twilio Account Phone number(s)",
        "description": "Number(s) required for service to function",
        "type": "string",
      },
    },
  })
}
```

### Create an Add-on Instance

When a customer adds an instance of your add-on to a site, Netlify will `POST` to `your-management-api.com/instances/`

A Netlify user can add an instance of your service by running:

```bash
netlify addons:create your-addon-namespace --valueOne xyz --otherConfigValue abc
```

That kicks off the following flow:

1. **The `POST` request from Netlify to your service occurs**

    Here is an example request body **to your management endpoint**:

    ```js
    {
      // Unique ID generated by Netlify
      uuid: '2e65dd70-523d-48d8-8826-a93229d7ec01',
      account: '5902622bcf321c7359e97e52',
      config: {
        site_url: 'https://calling-site-from-netlify.netlify.com',
        jwt: {
          secret: 'xyz-netlify-secret'
        },
        // User defined configuration values
        config: {
          name: 'woooooo'
        },
        // Netlify Site id
        site_id: '2e65dd70-523d-48d8-8826-a93229d7ec01',
        // Your service ID slug
        service_id: 'express-example',
        service_instance: {
          config: { name: 'woooooo' }
        },
        // If your add-on needs to trigger site rebuilds we will send a build hook
        incoming_hook_url: 'https://api.netlify.com/build_hooks/123xyz' } }
      }
    }
    ```

    - `uuid`: Unique ID generated by Netlify.
    - `config`: Fields and values you need for configuring your service for a customer.

    You will want to take this data, provision your application resources and return a response.

2. **Return a `201` response from your service back to Netlify**

    ```js
    {
      // `id` (required) - A unique ID generated by you, for reference within your own API
      id: uuid(),
      // `endpoint` (optional) - Proxied endpoint.
      // This will be callable at https://user-netlify-site.com/.netlify/your-addon-namespace
      endpoint: "https://my-endpoint.example.com",
      /* `config` (optional) - This can return back exactly what was received in the POST request, or include additional fields or altered values. This should also be what is returned in response to a GET request to /instances/:id */
      config: {},
      // `env` (optional) - Environment Keys accessible by Netlify user in build context & in functions
      env: {
        'YOUR_SERVICE_API_SECRET': 'value'
      },
      // `snippets` (optional) - JS Snippet content to inject into the calling Netlify site
      snippets: [
        {
          title: 'Snippet From Demo App',
          position: 'head',
          html: `<script>console.log("Hello from ${logValue}")</script>`
        }
      ]
    }
    ```

    - `id` A unique ID generated by you, for reference within your own API. Any string is valid for our purposes. This will be included in the headers and JWS for all API calls from Netlify. This `id` is also what is used in all subsequent `instances/${id}` update/get/delete calls to your remote API.
    - `env`: Set Environment variable for the Netlify user to access during site build or inside of their Netlify functions context.
    - `snippets`: Inject javascript snippets into the header or footer of the calling Netlify Site.

    Though not implemented yet, we plan to include a `state` field, which will allow your service to handle async provisioning, in case it takes some amount of time to activate the new service.

### Updating an Add-on Instance

You can allow Netlify users to update your service instance.

This is achieved by the Netlify user running:

```bash
# Option 1. Run through configuration prompts from the /manifest endpoint
netlify addons:config your-addon-namespace

# Option 2. Run command with values for no prompts
netlify addons:config your-addon-namespace --valueOne xyz --otherConfigValue abc
```

1. **The `PUT` request from Netlify to your service `/instances/${id}` occurs**

    The ID of the service instance is included in the path parameters. This `id` was generated initially by your `POST` `/instances` implementation. (see "Create an Instance")

    The body of the update request looks like this:

    ```js
    {
      config: {
        name: 'noooooooo'
      }
    }
    ```

    Run your services update logic here and then return any updated values back to the Netlify site.

2. **Return a `200` response from your service back to Netlify**

    Return any updated values from the users request

    Here is an example response with updated `env` values and an updated `snippet`

    ```js
    {
      env: {
        'YOUR_SERVICE_API_SECRET': 'updated-env-value'
      },
      snippets: [
        {
          title: 'Snippet From Demo App',
          position: 'head',
          html: `<script>console.log("Updated snippet content")</script>`
        }
      ]
    }
    ```

### Deleting an Add-on Instance

When a Netlify user removes your add-on, Netlify sends a `DELETE` request to your service to handle deprovisioning.

Users can remove add-ons like so:

```bash
netlify addons:delete your-addon-namespace
```

When this happens, the following occurs:

1. **Netlify sends a `DELETE` request from to your service `/instances/${id}`**

    This request has no body but includes the `id` of the instance. This `id` was generated initially by your `POST` `/instances` implementation. (see "Create an Instance")

    Run your deletion logic and optionally return data back to the client (CLI)

2. **Return a `204` response from your service back to Netlify to verify the deletion was successful**

### Getting an Add-on Instance

Netlify users get information about your service instance like so:

```bash
netlify addons:list
```

To implement this endpoint

1. **The `GET` request from Netlify to your service `/instances/${id}` occurs**

    This request has no body but includes the `id` of the instance.

    Run your logic to fetch details about your instance and return them back to the CLI

2. **Return a `200` response from your service back to Netlify**

    ```
    {
      env: {
        'YOUR_SERVICE_API_SECRET': 'value'
      },
      snippets: [
        {
          title: 'Snippet From Demo App',
          position: 'head',
          html: '<script>console.log("Hello from App")</script>'
        }
      ]
    }
    ```

## Proxied URLs

When creating an add-on, if you return an `endpoint` with a URL to your service, a Netlify customer can send requests to

```
https://users-netlify-site.netlify.com/.netlify/<your-addon-namespace>/*
```

**Example:**

If a user adds the `status-page` add-on and the creation of the `status-page` add-on returns an `endpoint` url of `http://third-party-service.com/xzy123/statue-page`, that endpoint will be available at:

```
https://users-netlify-site.netlify.com/.netlify/status-page/*
```

A call to `https://users-netlify-site.netlify.com/.netlify/status-page/` will be proxied through to the original service URL of `http://third-party-service.com/xzy123/statue-page`.

### Request Headers

All requests will include the following headers:

```
X-NF-UUID: 1234-1234-1234
X-NF-ID: 5a76f-b3902
X-NF-SITE-URL: https://my-project.netlify.com
```

- `X-NF-UUID`: The unique UUID generated by Netlify and sent to your management API when provisioning the instance.
- `X-NF-ID`: The unique ID returned from your management API when provisioning the instance.
- `X-NF-SITE-URL`: The URL of the Netlify-hosted site associated with the instance.

### Verification with JWS

Though the headers above can be useful for quick troubleshooting and testing during development, they are vulnerable to impersonation attacks, and should not be used in production. Instead, use the JSON Web Signature (JWS) we'll include the header `X-NF-SIGN`. The JWS secret will match the bearer token generated when you first register your microservice on the platform, as described in [Getting started](getting-started.md). By verifying this secret, you'll know that the request is coming from Netlify. Also, because it's unique to your microservice, you can be sure that a leaked secret on another microservice will not impact the security of yours.

The JWS payload will be JSON with this format

```json
{
  "exp": epoch-seconds,
  "site_url": "https://my-project.netlify.com",
  "id": "5a76f-b3902",
  "netlify_id": "1234-1234-1234"
}
```

- `exp`: Standard JWS expiration field.
- `site_url`: The URL of the Netlify-hosted site associated with the instance, like `X-NF-SITE-URL` above.
- `id`: The unique ID returned from your management API when provisioning the instance, like `X-NF-ID` above.
- `netlify_id`: The unique UUID generated by Netlify and sent to your management API when provisioning the instance, like `X-NF-SITE-URL` above.


### User-Level Authentication with JWTs

While the JWS described above may be used to verify the origin of a request, you may also want to verify certain information about the actual site user who triggered that request. Netlify handles this with JSON Web Tokens (JWTs) as a stateless identity layer.

Netlify provides a built-in Identity service (which is actually a microservice addon itself, based on our open source [GoTrue](https://www.gotrueapi.org) project). However, any authentication service that can issue JWTs will work with Netlify's microservice gateway. Site builders can include one of these JWTs as a bearer token in the Authorization header for a site route, and Netlify will verify its signature against the site's identity secret stored in Netlify's backend. If the route points to your microservice (using the path at the beginning of this doc), Netlify will re-sign the JWT with your unique secret before passing the request on to your microservice.

## Registering your add-on

Our current plan is to create a developer panel where new partners can register a new add-on in our UI. It will require an application title, slug, description, icon, and an endpoint for the app's management API.

For now, you can register your add-on namespace/endpoint by [filling out this form](https://cli.netlify.com/register-addon) and we will get you set up in Netlify's add-on marketplace.

When we register your add-on, we’ll generate an add-on secret that is unique to your service. All requests from Netlify to your add-on’s management API will contain an `X-Nf-Sign` authorization header. You can verify request are coming from Netlify by verifying the `X-Nf-Sign` header against your add-on secret.

Your management API should verify this secret before accepting any requests.

Example:

```js
const jwt = require('jsonwebtoken')
const Your_Addon_Secret = 'generated-when-addon-registered'

// inside your request handler:
const netlifySignedToken = req.headers['X-Nf-Sign']
const fromNetlify = jwt.verify(netlifySignedToken, Your_Addon_Secret)

if (fromNetlify) {
  // do stuff
}
```

After your namespace is setup in Netlify, you will be able to run CLI command to test out your provisioning logic

```bash
netlify addons:create your-name-space
```

It’s common for add-on partners to create a `-staging` version of their add-on to test new features, like `your-addon-namespace-staging`. This gives a safe place for us to test and update changes to an addon before shipping updates to the master `your-addon-namespace` that is live for all Netlify users.

## Example Implementations

We have created a couple example implementations of how an add-on REST API should work.

- [An express app running on Heroku](./examples/express) |  [code](https://github.com/netlify/netlify-addons/blob/master/examples/express/index.js)
- [A REST API running on AWS Lambda + APIGateway](./examples/serverless) |  [code](https://github.com/netlify/netlify-addons/blob/master/examples/serverless/handler.js)

Please let us know if you have any questions on how these work! Open an issue in this repo or ping us directly on slack.
