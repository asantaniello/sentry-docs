---
title: 'Unified API'
sidebar_order: 2
---

New Sentry SDKs should follow the unified API, use consistent terms to refer to concepts.  This
documentation explains what the unified API is and why it exits.

## Motivation

Sentry has a wide range of SDKs that have been developed over the years by different developers
and based on different ideas.  This has lead to the situation that the feature sets across the
SDKs are different, use different concepts and terms which has lead to the situation that it's
often not clear how to achive the same thing on different platforms.

Additionally those SDKs were purely centered around error reporting through explicit clients which
meant that certain integrations (such as breadcrumbs) were often not possible.

## Terminology

- **minimal**: A separate "facade" package that re-exports a subset of the SDK's functionality through interfaces or proxies. That package does not directly depend on the SDK, instead it should make every operation a noop if the SDK is not installed.

  The purpose of such a package is to allow random libraries to record breadcrumbs and set context data while not having a hard dependency on the SDK.

- **hub**: An object that manages the state. An implied global thread local or similar hub exists that can be used by default. Hubs can be created manually.

- **scope**: A scope holds data that should implicitly be sent with sentry events. It can hold context data, extra parameters, level overrides, fingerprints etc.

- **client**: A client is an object that is configured once and can be bound to the hub. The user can then auto discover the client and dispatch calls to it.  Users typically do not need to work with the client directly. They either do it via the hub or static convenience functions.

- **client options**: Are parameters that are language and runtime specific and used to configure the client. This can be release and environment but also things like which integrations to configure, how in-app works etc.

- **context**: Contexts give extra data to sentry. There are the special contexts (user and similar) and the generic ones (runtime, os, device) etc.  Check out [_Contexts_]({%- link _documentation/clientdev/interfaces/contexts.md -%}) for valid keys. *Note: In older SDKs you might encounter an unrelated concept of context, which is now deprecated by scopes*

- **tags**: Tags can be arbitrary string→string pairs by which events can be searched. Contexts are converted into tags.

- **extra**: Truly arbitrary data attached by client users.

- **transport**: The transport is an internal construct of the client that abstracts away the event sending. Typically the transport runs in a separate thread and gets events to send via a queue. The transport is responsible for sending, retrying and handling rate limits. The transport might also persist unsent events across restarts if needed.

- **integration**: Code that provides middlewares, bindings or hooks into certain frameworks or environments, along with code that inserts those bindings and activates them. Usage for integrations does not follow a common interface.

## "Static API"

The static API functions is the most common user facing API.  A user just imports these functions and can start
emitting events to sentry or configuring scopes.  These shortcut functions should be exported in the top-level
namespace of your package.  Behind the scenes they use scopes use hubs and scopes (see [Concurrency](#concurrency) for more information) if available on that platform:

- `capture_event(event, hint)`: Takes an already assembled event and dispatches it to the currently active hub.  The event object can be a plain dictionary or a typed object whatever makes more sense in the SDK.  It should follow the native protocol as close as possible ignoring platform specific renames (case styles etc.).

  For the hint parameter see [hints](#hints).

- `capture_exception(error)`: Report an error or exception object.  Depending on the platform different parameters are possible.  The most obvious version accepts just an error object but also variations are possible where no error is passed and the current exception is used.

- `capture_message(message, level)`: Reports a message. The level can be optional in language with default parameters in which case it should default to `info`.

- `add_breadcrumb(crumb, hint)`: Adds a new breadcrumb to the scope. If the total number of breadcrumbs exceeds the `max_breadcrumbs` setting, the oldest breadcrumb should be removed in turn. This works like the Hub api with regards to what `crumb` can be.

  For the hint parameter see [hints](#hints).

- `configure_scope(callback)`: Calls a callback with a scope object that can be reconfigured. This is used to attach contextual data for future events in the same scope.

- `last_event_id()`: Should return the last event ID emitted by the current scope.  This is for instance used to implement user feedback dialogs.

## Concurrency

All SDKs should have the concept of concurrency safe context storage. What this means depends on the language. The basic idea is that a user of the SDK can call a method to safely provide additional context information for all events that are about to be recorded.

This is implemented as a thread local stack in most languages, but in some (such as JavaScript) it might be global under the assumption that this is something that makes sense in the environment.

Here are some common concurrench patterns:

* **Thread bound hub**: in that pattern each thread gets its own "hub" which internally manages a stack of scopes.  If that pattern is followed one thread (the one that calls `init()`) becomes the "main" hub which is used as the based for newly spawned threads which will get a hub that is based on the main hub (but otherwise independent).

* **Internally scoped hub**: On some platforms such as .NET ambient data is available in which case the Hub can internally manage the scopes.

* **Dummy hub**: On some platforms concurrency just doesn't inherently exists.  In that case the hub might be entirely absent or just be a singleton without concurrenty management.

## Hub

Under normal circumstances the hub consists of a client and a stack of scopes.

The SDK maintains two variables: The *main hub* (a global variable) and the *current hub* (a variable local to the current thread or execution context, also sometimes known as async local or context local)

The `init()` you saw in the [_Overview_]({%- link _documentation/clientdev/overview.md -%}) typically creates / reinitializes the global hub which is propagated to all new threads/execution contexts, or a hub is created per thread/execution context. 

- `Hub::new(client, scope)`: Creates a new hub with the given client and scope.  The client can be reused between hubs. The scope should be owned by the hub (make a clone if necessary)

- `Hub::new_from_top(hub)` / alternatively native constructor overloads: Creates a new hub by cloning the top stack of another hub.

- `get_current_hub()` / `Hub::current()` / `Hub::get_current()`: Global function or static function to return the current (threads) hub

- `get_main_hub()` / `Hub::main()` / `Hub::get_main()`: In languages where the main thread is special ("Thread bound hub" model) this returns the main thread’s hub instead of the current thread’s hub. This might not exist in all languages.

- `Hub::capture_event` / `Hub::capture_message` / `Hub::capture_exception` Capture message / exception call into capture event. `capture_event` merges the event passed with the scope data and dispatches to the client.

- `Hub::push_scope()`: Pushes a new scope layer that inherits the previous data. This should return a disposable or stack guard for languages where it makes sense.  When the "internally scoped hub" concurrency model is used calls to this are often necessary as otherwise a scope might be accidentally incorrectly shared.

- `Hub::with_scope(func)` (optional): In Python this could be a context manager, in Ruby a block function. Pushes and pops a scope for integration work.

- `Hub::pop_scope()` (optional): Only exists in languages without better resource management. Better to have this function on a return value of `push_scope` or to use `with_scope`.  This is also sometimes called `pop_scope_unsafe` to indicate that this method should not be used directly.

- `Hub::configure_scope(callback)`: Invokes the callback with a mutable reference to the scope for modifiations  This can also be a `with` statement in languages that have it (Python).

- `Hub::add_breadcrumb(crumb, hint)`: Adds a breadcrumb to the current scope.

  - The argument supported should be:
    - function that creates a breadcrumb
    - an already created breadcrumb object
    - a list of breadcrumbs optionally
  - In languages where we do not have a basic form of overloading only a raw breadcrumb object should be accepted.

  For the hint parameter see [hints](#hints).

- `Hub::client()` / `Hub::get_client()` (optional): Accessor or getter that returns the current client or `None`.

- `Hub::bind_client(new_client)`: Binds a different client to the hub. If the hub is also the owner of the client that was created by `init` it needs to keep a reference to it still if the hub is the object responsible for disposing it.

- `Hub::unbind_client()` (optional): Optional way to unbind for languages where `bind_client` does not accept nullables.  

- `Hub::last_event_id()`: Should return the last event ID emitted by the current scope.  This is for instance used to implement user feedback dialogs.

- `Hub::run(hub, callback)`  `hub.run(callback)`, `run_in_hub(hub, callback)` (optional): Runs a callback with the hub bound as the current hub.

### Scope

A scope holds data that should implicitly be sent with sentry events. It can hold context data, extra parameters, level overrides, fingerprints etc.

The user should be able to modify the current scope easily (to set extra, tags, current user), through a global function `configureScope`.  `configure_scope` takes a callback function to which it passes the current scope. Here's an example from another place in the docs:

```javascript
Sentry.configure_scope((scope) => {
  scope.setExtra("character_name": "Mighty Fighter");
});
```

Why not just have a `get_current_scope()` function instead of this indirection?  If the SDK is disabled (e.g. by not providing a DSN), modifying a scope is pointless because there will never be any events to send. In that situation `configure_scope` may choose not to call the callback.  This is also used to more efficiently flush out scope changes in some languages.  When a `with` statement is used that always executes the scope needs to be a dummy scope object.

- `scope.set_user(data)`: Shallow merges user configuration (`email`, `username`, …).  Removing user data is SDK-defined, either with a `remove_user` function or by passing nothing as data.

- `scope.set_extra(key, value)`: Sets the extra key to an arbitrary value, overwriting a potential previous value. Removing a key is SDK-defined, either with a `remove_extra` function or by passing nothing as data.

- `scope.set_tag(key, value)`: Sets the tag to a string value, overwriting a potential previous value.  Removing a key is SDK-defined, either with a `remove_tag` function or by passing nothing as data.

- `scope.set_context(key, value)`: Sets the context key to a value, overwriting a potential previous value.  Removing a key is SDK-defined, either with a `remove_context` function or by passing nothing as data. The types are sdk specified.

- `scope.set_fingerprint(fingerprint[])`: Sets the fingerprint to group specific events together

- `scope.add_event_processor(processor)`: Registers an event processor function.  It takes an event and returns a new event or `None` to drop it.  This is the basis of many integrations.

- `scope.add_error_processor(processor)`: Registers an error processor function.  It takes an event and exception object and returns a new event or `None` to drop it.  This can be used to extract additional information out of an exception object that the SDK cannot extract itself.

- `scope.clear()`: resets a scope to default values (prevents inheriting).  This never clears registered event processors.

- `scope.apply_to_event(event[, max_breadcrumbs])`: Applies the scope data to the given event object. This also applies the event processors stored in the scope internally.  Some implementations might want to set a max breadcrumbs count here.

## Hints

Optionally a second parameter is supported to event capturing and breadcrumb adding: a hint.

A hint is SDK specific but provides high level information about the origin of
the event.  For instance if an exception was captured the hint might carry the
original exception object.  Not all SDKs are required to provide this.  The
parameter however is reserved for this purpose.

## Options

Many options are standardized across SDKs.  For a list of these refer to [the main options documentation]({% link _documentation/learn/configuration.md %}).
