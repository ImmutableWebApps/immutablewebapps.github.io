# Immutable Web Apps

## Introduction

_Immutable Web Applications_ is a framework-agnostic methodology for building and deploying static [single-page applications](https://en.wikipedia.org/wiki/Single-page_application) that:

- Minimizes risk and complexity of live releases.
- Simplifies and maximizes caching.
- Minimizes the need for servers and systems administration.
- Enables continuous delivery through simple, flexible, atomic deployments.

## Principles

The methodology is based on the principles of ___strictly separating___:

- Configuration from code.
- Release tasks from build tasks.
- Dynamic content from static content.

These principles are inspired by [The Twelve-Factor App](https://12factor.net/).

## Concepts

The following concepts define the core requirements for an _Immutable Web Application_. They are framework and infrastructure agnostic.

### _Permabundles_

Permabundles are the static assets that make up a single-page application.

- They must be available at a [permalink](https://en.wikipedia.org/wiki/Permalink) that is uniquely versioned or fingerprinted and _independent of the web application environment(s)_.

- They must be configured for long-term caching. (`cache-control: public, max-age=31536000, immutable`)

- __They must not contain any environment-specific configuration.__

### `index.html` <span style="font-size:larger;">___is___</span> configuration

 An `index.html` exists for each deployment environment.

- They must contain fully-qualified references to the _permabundle_ assets.

- They must never be cached by the browser or a public cache that cannot be purged on-demand. (`cache-control: no-store`)

- __They must define all environment-specific configuration.__

_The `index.html` of your Angular app might look like this... it's all configuration!_:

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
            API: 'https://api.immutablewebapps.org',
            OAUTH: 'https://oauth.immutablewebapps.org',
            GA_TRACKING_ID: 'UA-98765432-1'
        }
        </script>

        <!-- application binding -->
        <app-root></app-root>

        <!-- fully-qualified permabundles -->
        <script src="https://immutablewebapp.org/assets/c17080f1/runtime.js" type="text/javascript"></script>
        <script src="https://immutablewebapp.org/assets/c17080f1/polyfills.js" type="text/javascript"></script>
        <script src="https://immutablewebapp.org/assets/c17080f1/styles.js" type="text/javascript"></script>
        <script src="https://immutablewebapp.org/assets/c17080f1/vendor.js" type="text/javascript"></script>
        <script src="https://immutablewebapp.org/assets/c17080f1/main.js" type="text/javascript"></script>

    </body>
</html>
```

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
