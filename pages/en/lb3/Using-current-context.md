---
title: "Using current context"
redirect_from: /doc/en/lb2/Using%20current%20context.html
lang: en
layout: page
keywords: LoopBack
tags:
sidebar: lb3_sidebar
permalink: /doc/en/lb3/Using-current-context.html
summary:
---

LoopBack applications sometimes need to access context information to implement the business logic, for example to:

* Access the currently logged-in user.
* Access the HTTP request (such as URL and headers).

A typical request to invoke a LoopBack model method travels through multiple layers with chains of asynchronous callbacks. It's not always possible to pass all the information through method parameters. 

In LoopBack 2.x, we have introduced current-context APIs which used the module
[continuation-local-storage](https://www.npmjs.com/package/continuation-local-storage)
to provide a context object preserved across asynchronous operations.
Unfortunately, as we learned later, this module is not reliable and have many
problems (for example, see [issue #59](https://github.com/othiym23/node-continuation-local-storage/issues/59)).
As a result, our current-context feature does not work in many situations,
as can be seen from issues reported in LoopBack's
[issue tracker](https://github.com/strongloop/loopback/issues?utf8=%E2%9C%93&q=is%3Aissue%20getCurrentContext).

To address this problem, we have moved all current-context-related code to a
standalone module [loopback-context](https://github.com/strongloop/loopback-context)
and removed all current-context APIs in the version 3.0 (see
[Release Notes](3.0-Release-Notes.html#current-context-api-and-middleware-removed) for
more details).

Having done that, we do recognize the need of accessing information like the
currently logged-in user deep inside application logic, for example in
[Operation Hooks](Operation-hooks.html). Until there is a reliable
implementation of continuation-local-storage available for Node.js, we are
recommending to explicitly pass any additional context via `options` parameter
of (remote) methods.

All built-in methods like
[PersistedModel.find](http://apidocs.strongloop.com/loopback/#persistedmodel-find)
or
[PersistedModel.create](http://apidocs.strongloop.com/loopback/#persistedmodel-create)
were already modified to accept an optional `options` argument.

[Operation Hooks](Operation-hooks.html) expose the `options` argument
as `context.options`.

The missing piece is how to initialize the `options` parameter when a method is
invoked via REST API, and do it in a safe manner, so that clients cannot
override sensitive information like the currently logged-in user.

The solution we have adopted has two parts.

## Annotate "options" parameter in remoting metadata

Methods accepting an `options` argument should declare this argument in their
remoting metadata and use a special value `optionsFromRequest` as the `http`
mapping.

```json
{
  "arg": "options",
  "type": "object",
  "http": "optionsFromRequest"
}
```

Under the hood, this "magic string" value is converted by `Model.remoteMethod`
to a function that will be called by strong-remoting for each incoming request
in order to build the value for this parameter.

{% include tip.html content='
Computed "accepts" parameters have been around for a while and they are well supported by LoopBack tooling. For example, the Swagger generator excludes computed properties from the API endpoint description. As a result, the "options" parameter will not be described in the Swagger documentation.
' %}

All built-in method have been already modified to include this new "options"
parameter.

{% include note.html content='
In LoopBack 2.x, this feature is disabled by default for compatibility reasons.  You can enable it by adding `"injectOptionsFromRemoteContext": true` to your model JSON file.
' %}

## Customize the value provided to "options"

When strong-remoting is resolving the "options" argument, it will call model's
method `createOptionsFromRemotingContext`. The default implementation of this
method returns an object with a single property `accessToken` containing
the `AccessToken` instance used to authenticate the request.

There are several ways how to customize this value.

### Override `createOptionsFromRemotingContext` in your model

```js
MyModel.createOptionsFromRemotingContext = function(ctx) {
  var base = this.base.createOptionsFromRemotingContext(ctx);
  return extend(base, {
    currentUserId: base.accessToken && base.accessToken.userId,
  });
};
```

A better approach is to write a mixin that overrides this method and that can
be shared between multiple models.

### Use a "beforeRemote" hook

Because the "options" parameter is a regular method parameter, it can be
accessed from remote hooks via `ctx.args.options`.

```js
MyModel.beforeRemote('saveOptions', function(ctx, unused, next) {
  if (!ctx.args.options.accessToken) return next();
  User.findById(ctx.args.options.accessToken.userId, function(err, user) {
    if (err) return next(err);
    ctx.args.options.currentUser = user;
    next();
  });
})
```

Again, a hook like this can be reused by placing the code in a mixin.

Note that remote hooks are executed in order controller by the framework, which
may be different from the order in which you need to modify the options
parameter and then read the modified values. Please use the next suggestion if
this order matters in your application.

### Use a custom strong-remoting phase

Internally, strong-remoting uses phases similar to [Middleware
Phases](https://loopback.io/doc/en/lb3/Defining-middleware.html). The framework
defines two built-in phases: `auth` and `invoke`. All remote hooks are run in
the second phase `invoke`.

Applications can define a custom phase to run code before any remote hooks are
invoked, such code can be placed in a boot script for example.

```js
module.exports = function(app) {
  app.remotes().phases
    .addBefore('invoke', 'options-from-request')
    .use(function(ctx, next) {
      if (!ctx.args.options.accessToken) return next();
      User.findById(ctx.args.options.accessToken.userId, function(err, user) {
        if (err) return next(err);
        ctx.args.options.currentUser = user;
        next();
      });
    });
};
```

