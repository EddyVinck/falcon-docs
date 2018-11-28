---
title: Basics
---

Falcon Server is the entrypoint for backend features of Falcon stack. It acts as API server for Falcon Client - provides data and features required by Falcon Client. It can also act as standalone API server for other services.

Falcon Server is implemented with [Koa](https://koajs.com/) and [Apollo Server](https://www.apollographql.com/docs/apollo-server/).

Falcon Server is just a "glue" that realises all the functionalities via [extensions](#extensions-system)

## Installation

With npm:

```bash
npm install @deity/falcon-server
```

or with yarn:

```bash
yarn add @deity/falcon-server
```

## Usage

```js
const FalconServer = require('@deity/falcon-server');
const config = {
  port: 4000
};
const server = new FalconServer(config);
server.start();
```

## Configuration

`config: object`
 * `port: number` - port number that server should be running on (default is set to 4000)
 * `apis: []` - array of APIs configuration. See [APIs configuration](#apis-configuration).
 * `extensions: []` - array of extensions configuration. See [Extensions configuration](#extensions-configuration)
 * `session: object` - session configuration, [see the details](#session-configuration)
 * `maxListeners: number` (`20` by default) - number of max listeners per event
 * `verboseEvents: boolean` (`false` by default) - toggling "Logger.trace" call for each event handler
 * `logLevel: string` - Logger level
 * `debug: boolean` (`false` by default) - whether Falcon Server should start in "debug" mode (enabling "tracing" flags)

### APIs configuration

`apis` array provides list of APIs that should be used along with options that should be passed to those APIs. Additionally, if API should be available for other extensions its configuration should have `"name"` property that later can be used to get instance of particular extension.

```js
const FalconServer = require('@deity/falcon-server');
const config = {
  "apis": [
    {
      "package": "@deity/falcon-wordpress-api",
      "name": "api-wordpress",
      "options": {
        "host": "mywordpress.com",
        "protocol": "https",
      }
    }
  ]
};
const server = new FalconServer(config);
server.start()
```

### Extensions configuration

`config` object can contain `extensions` array that provides list of extensions that should be used along with options that should be passed to those extensions.
Extensions should be added by specifying package name of the extension, and `options` object that is passed to extension constructor:

```js
const FalconServer = require('@deity/falcon-server');
const config = {
  "extensions": [
    {
      "package": "@deity/falcon-blog-extension",
      "config": {}
    }
  ]
};
const server = new FalconServer(config);
server.start()
```

If extension requires an API to work correctly the API can be either implemented inside the extension, but it can also be implemented as separate package. Then, such API can be added via [`apis`](#apis-configuration) and used by extension.

This is especially handy when extension realised some piece of functionality that can use data from various 3rd party services - e.g. blog extension can use wodpress for content fetching, but also any other service that can deliver data in the format accepted by blog extension.

```js
const FalconServer = require('@deity/falcon-server');
const config = {
  "apis": [
    {
      "package": "@deity/falcon-wordpress-api",
      "name": "api-wordpress", // set name for that extension
      "options": {
        // options for this api instance
      }
    }
  ],
  "extensions": [
    {
      "package": "@deity/falcon-blog-extension",
      "options": {
        "api": "api-wordpress" // use API named "api-wordpress"
      }
    }
  ]
};
const server = new FalconServer(config);
server.start()
```

### Session configuration
Falcon Server uses [koa-session](https://www.npmjs.com/package/koa-session) for session implementation, so all the options passed in `session.options` will be passed directly to `koa-session`. Additionally you can provide `session.keys` array with kesy used by `koa` instance:

```js
const FalconServer = require('@deity/falcon-server');
const config = {
  "session": {
    "keys": ["some secret key"], // this value will be set as "keys" property of koa instance
    "options": { // all options allowed by koa-session
      "maxAge": 86400000
    }
  }
};
const server = new FalconServer(config);
server.start()
```

## Extensions system

The essence of Falcon Server is realisation of the access to the external services via GraphQL. That objective is achieved via simple extensions system. Extensions are JavaScript classes that implement particuar methods to deliver the data or modify existing data.

Each extension can extend GraphQL configuration by providing its own config. When all extensions are instantiated FalconServer asks each and every extension for the configuration, combines all the received configurations and passes the combined configuration to Apollo Server configuration.

During startup Falcon Server calls method `getGraphQLConfig()` method that should return configuration object that provides configuration for the Apollo Server. The following options are implemented:
 * `typeDefs: object` - gql expression with type definitions - helpful especially when extension needs to extend a type defined by other extension
 * `schema: object` - GrahQL schema generated by [makeExecutableSchema()](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html) or [makeRemoteExecutableSchema()](https://www.apollographql.com/docs/graphql-tools/remote-schemas.html). Passing that option will override `typeDefs` property
 * `resolvers: object` - map with resolvers required by defined data types
 * `context: function` - function that modifies GraphQL context passed to resolvers during execution
 * `dataSources: object` - map with data sources that can be used by resolvers

All the values passed via `typeDefs` and `schema` properties will be merged using [Apollo's schema stitching mechamism](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html)

## Dynamic Route Resolver

`DynamicRouteResolver` handles dynamic route resolution for the client application.
When user goes to a page that is not defined in React routing - then client asks back-end for the type of content for that particular url:

- It asks every extension to get "fetch URL priority" from the assigned API DataSource instance
- It sorts API DataSource instances that are able to determine dynamic content type by their "priority"
- It calls `fetchUrl` method on every available API DataSource until it gets a proper response

### Example

The simplest example is handling content from shop and blog. Let's say we want to resolve content for `/sport.html`

```js
class Magento2Api {
    getFetchUrlPriority(url) {
        // if url ends with .html then it's highly possible that it should be handed by shop
        // because it might be a product page under url generated by Magento
        return url.endsWith('.html') ? ApiUrlPriority.HIGH : ApiUrlPriority.NORMAL;
    }

    fetchUrl(obj, args, context, info) {
        // implementation of the resolver
    }
}
```

```js
class WordpressApi {
    getFetchUrlPriority(url) {
        // return just static value so others can be higher or lower
        return ApiUrlPriority.NORMAL;
    }

    fetchUrl(obj, args, context, info) {
        // implementation of the resolver
    }
}
```

`DynamicRouteResolver` will try to call `getFetchUrlPriority(url)` method for both extensions:

- ShopExtension/Magento2Api will return `ApiUrlPriority.HIGH`
- BlogExtension/WordpressApi will return `ApiUrlPriority.NORMAL`

So in that case `DynamicRouteResolver` will call `ShopExtension.fetchUrl()` first, if that call returns `null`,
it will call `BlogExtension.fetchUrl()` second.

### DynamicRouteResolver result structure

The correct response must match the following structure:

```json
{
    "id": 1,
    "path": "/foo.html",
    "type": "foo-type",
    "redirect": false
}
```

- `id` - must represent a unique ID for your back-end
- `path` - must represent a canonical path for your entity (this will be used for a possible redirection situation)
- `type` - must represent a unique entity type that will be later used by `DynamicRoute` component on `falcon-client`
- `redirect` - must set a flag whether the given URL must be redirected to the `path` address (to ensure canonical URL)

### Requirements

If any extension should handle dynamic routing it should implement both methods:

- `getFetchUrlPriority(url)` - accepts url (`string`) that should be resolved and returns the priority (`number`) for the extension
- `fetchUrl(obj, args, context, info)` - fetches the data from the remote source or performs any other async logic to determine

the type of the given url and returns a correct result or `null` (if the given URL is not determined by your back-end)