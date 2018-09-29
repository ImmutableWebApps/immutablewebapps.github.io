# Immutable Web Apps

## Introduction

_Immutable Web Applications_ is a framework-agnostic methodology for building and deploying static [single-page applications](https://en.wikipedia.org/wiki/Single-page_application) that:

- Minimizes risk and complexity of live releases.
- Simplifies and maximizes caching.
- Minimizes the need for servers and administration of runtime environments.
- Enables continuous delivery through simple, flexible, atomic deployments.

## Principles

The methodology is based on the principles of ___strictly separating___:

- Dynamic content from static content.
- Configuration from code.
- Release tasks from build tasks.

These principles are inspired by [The Twelve-Factor App](https://12factor.net/).

## Decomposing a web application

The _Immutable Web App_ methodology was developed by through a process of decomposing web applications based on the principles declared above and then reconsidering the relationship between the different components. Before defining the methodology, this document will step through the decomposition. It starts with a monolithic web application.

### The Monolithic Web-Application

There generally are three types of network traffic commonly made in by web application from the browser:

#### static assets

- javascript, css, images, bundled assets
- long term cached
- artifacts live on the client

#### xhr/fetch

- dynamic
- no cache

#### the document

- typically in the form of `index.html`
- no caching
- returned for all routes that aren't physical files

> An image of typical network traffic for a simple single page application

A monolithic web application will handle the requests for all three types of requests. Express, a very popular node framework for building web applications, generally supports handling all three types of requests. Running the `express-generator` will create an application skeleton that supports routes for each:

```javascript
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');
var logger = require('morgan');

var indexRouter = require('./routes/index');
var usersRouter = require('./routes/users');

var app = express();

app.use(logger('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());

/* serves static assets */
app.use(express.static(path.join(__dirname, 'public')));

/* serves the document */
app.use('/', indexRouter);

/* serves dynamic requests */
app.use('/users', usersRouter);

module.exports = app;
```

A monolithic web application will also include a web application framework, like Angular, for building the single-page application assets. Using Express and Angular together in a single code base will results in a multi-stage build:

- First the static Angular apps are generated and
- Then the assets are embedded in the creation of the executable package (docker image, rpm).

The package is then typically stored in a versioned repository of packages before it is deployed to a runtime environment.

> An image of the monolith build process: code => build static assets => build docker image with assets baked in => publish image to docker repo => deployed to server

#### Characteristics of the monolithic web application

- separation release tasks from build tasks (good)
- high cohesion between the backend and frontend contract (good)
- ability to test the same build artifact in multiple environments (good)
- if the backend goes down, so does the frontend - serving static assets is much more reliable then dynamic backends, and gracefully handling backend outages is a better user experience than 404s and 500s (bad)
- zero downtime deploys can be tricky considering caching and existing browser sessions (bad)
- complicated project dependencies that represent two very different jobs (bad)
    - building a static web application assets
    - running a web server

### Separate frontend from backend

The first stage of decomposition is to separate the client from the server. This aligns with the principle of strict separation of dynamic content from static content.

> An image of the two separated build processes:
> code => build static assets => publish to static web server with routing rules
> code => build docker image => publish image to docker repo => deploy to server

#### The static web application

code base generates static assets,

The static assets can be deployed to a static web server that will have much higher reliability than the backend.

The static web apps should generally be cached for the long term, the backend should have caching turned off.

#### The backend api

code base generates an executable package.

generally you can read up on 12 factor app for help here, as it has been decoupled from many of the web application specific concerns

#### Characteristics of the frontend/backend web application

deployments require a little more careful sequencing to avoid zero-downtime. The backend may need to support multiple versions of an app as the frontend is deployed from one version to another. this is good, because it makes an preexisting problem more visible.

Code bases for generating the backend and the code base for generating the static application is separated.

Client side assets are often environment specific and deployed right after they are built as a part of a release.

### index.html as a special file

The second stage of decomposition continues to emphasize the strict separation of dynamic content from static content by separating `index.html`.

The document, `index.html`, is a bit of a special case. It may be a static file from the perspective of a specific deployment, but over the lifetime of multiple deployments, it is a dynamic file... evident by the fact that it cannot be cached at any location by the browser. Moreover, it has routing responsibilities that are unlike any of the other static assets or apis.

> An image of the three separated build processes:
> code => build static assets => publish to static web server
> code => build docker image => publish image to docker repo => deploy to server
> index.html => publish to static web server that always returns index.html

#### `index.html`

can be hosted by a static web server that for whatever path it is requested it always returns `index.html`

#### `static assets`

it allows us to simplify the hosting of all other static assets, all other static assets can be hosted in a completely different domain, and cached for long term if they are versioned or fingerprinted

#### Characteristics of the index/frontend/backend web application

assets can be build prior to deployment, a live release is effectively the act of updating `index.html`, but opportunities for testing them are limited.

### Separating code from config

Now that the app is decomposed into long term cached static assets, backend apis, and the special case of `index.html`, the relationships between the three parts look like this:

> An image of the three components and lines showing their dependencies (BEFORE)

- `index.html` has a strict dependency on static assets

- javascript in the static assets has strict dependencies on the backend apis (based on how modern web app frameworks handle configuration)

```typescript
// This file can be replaced during build by using the `fileReplacements` array.
// `ng build --prod` replaces `environment.ts` with `environment.prod.ts`.
// The list of file replacements can be found in `angular.json`.

export const environment = {
  production: false
};
```

- the backend apis generally have no dependencies on either the static assets or index.html

The environment configuration that creates a strict dependency between the javascript and the api violates the principle of strict separation of configuration from code. This practice forces the static assets to be rebuilt for each environment and for any configuration change. Rebuilding the static assets introduces risk and promotes a workflow that does not distinguish between build tasks and deploy tasks, adding even more risk and complexity to releases.

By expanding the perception of dynamic and static, index.html can be perceived as static from the perspective of one deployment, but dynamic over the lifetime of multiple deployments. Similarly, Javascript that contains configuration is static from the perspective of one environment, but it is _conceptually_ dynamic over multiple environments because of the differences in configuration. This dynamic quality is addressed by building the assets multiple times and publishing to multiple locations.

Moving the configuration out of the static javascript allows it to be _truly_ static. __It can be generated once, published once, cached forever and used in multiple different environments with multiple configurations.__

Where does the configuation live? It must live in a location that is dynamic (and therefore never cached), it must be unique to the environment and unique to the deployment. `index.html` has all of these qualities. If configuration is moved into `index.html`, the relationships now look like this:

> An image of the three components and lines showing their dependencies (AFTER)

Consider what it looks like to have all environment configuration defined on the global scope of the `index.html` generated by the `angular-cli`:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Immutable Web App Example</title>
    <base href="/">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
    <body>

        <!-- environment variables -->
        <script>
        env = {
            API: 'https://immutablewebapps.org/api',
            OAUTH: 'https://oauth.immutablewebapps.org',
            GA_TRACKING_ID: 'UA-98765432-1'
        }
        </script>

        <!-- application binding -->
        <app-root></app-root>

        <!-- fully-qualified static assets -->
        <script src="https://assets.immutablewebapp.org/c17080f1/runtime.js" type="text/javascript"></script>
        <script src="https://assets.immutablewebapp.org/c17080f1/polyfills.js" type="text/javascript"></script>
        <script src="https://assets.immutablewebapp.org/c17080f1/styles.js" type="text/javascript"></script>
        <script src="https://assets.immutablewebapp.org/c17080f1/vendor.js" type="text/javascript"></script>
        <script src="https://assets.immutablewebapp.org/c17080f1/main.js" type="text/javascript"></script>

    </body>
</html>
```

This file has interesting qualities.

- It is 100% configuration!
- It is strikingly similar to a kubectrl deployment YAML or a docker-compose.yaml

MORE STUFF HERE

### Conclusions

## Concepts

The following concepts define the core requirements for an _Immutable Web Application_. They are framework and infrastructure agnostic.

### _Permabundles_

Permabundles are the static assets that make up a single-page application.

- They must be available at a [permalink](https://en.wikipedia.org/wiki/Permalink) that is uniquely versioned or fingerprinted and _independent of the web application environment(s)_.

- They must be configured for long-term caching. (`cache-control: public, max-age=31536000, immutable`)

- __They must not contain any environment-specific configuration.__

### `index.html` <span style="font-size:larger;">___is___</span> configuration

 An `index.html` exists for each environment.

- They must contain fully-qualified references to the _permabundle_ assets.

- They must never be cached by the browser or a public cache that cannot be purged on-demand. (`cache-control: no-store`)

- __They must define all environment-specific configuration.__

## Workflow

The workflow is defined by its separation of build tasks and release tasks. Specifically, the publishing of permabundles as a _build_ and the publishing of index.html as a _release_.

### Publishing of permabundles as a build

Write code. Build assets. Publish permabundles.

It could be that simple. Of course, the specifics of this process will vary by the project and the team. The important part is that it is a process that results in published permabundles.

The publishing of a permabundle will typically be triggered from a manual action. Once the permabundle is published it must not be changed. Any required changes must result in publishing a different permabundle.

### Publishing of `index.html` as a release

When a permabundle is published and ready to be deployed, a new version of the `index.html` can be created that references the permabundle. Publishing the new `index.html` will instantly and atomically release the new version of the web application.

### Promoting permabundles

The _value_ of Immutable Web Apps occurs between the publishing of the permabundle and a live release of the web application. Between those events the permabundle can be promoted to production through a process of validation.

This validation might take the form of:

- Automation, regression, or exploratory testing
- Integration in a staging environment
- Canary testing in a production replica
- Blue/green deployment of backing services

If the permabundles fails any of this validation it is abandoned. If it succeeds it is immediately available to be released by publishing a new `index.html`. If an issue is found in production, rolling back is instant and atomic by simply publishing the previous revision of `index.html`.

## Setup

- Separate the management of `index.html` from the code repository that builds the static assets

- Manage your permabundles like they are a Javascript Library CDN, for example [http://cdnjs.org](https://cdnjs.com/libraries/jquery)

- Configure your web application host(s) to serve `index.html` for any path, no exceptions

- Use a CDN

- Favor a serverless cloud platform
