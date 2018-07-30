# RabbitMQ Access Control Cache Plugin

This plugin provides a caching layer for [access control operations](http://rabbitmq.com/access-control.html)
performed by RabbitMQ nodes.

## Project Maturity

This plugin is relatively young but has known production users.
It's a candidate for inclusion into the RabbitMQ distribution.

## Overview

This plugin provides a way to cache [authentication and authorization backend](http://rabbitmq.com/access-control.html)
results for a configurable amount of time.
It's not an independent auth backend but a caching layer for existing backends
such as the built-in, [LDAP](github.com/rabbitmq/rabbitmq-auth-backend-ldap), or [HTTP](github.com/rabbitmq/rabbitmq-auth-backend-http)
ones.

Cache expiration is currently time-based. It is not very useful with the built-in
(internal) [authn/authz backends](http://rabbitmq.com/access-control.html) but can be very useful for LDAP, HTTP or other backends that
use network requests.

## RabbitMQ Version Requirements

As with all authentication plugins, this plugin requires requires 2.3.1 or later.

`master` branch targest RabbitMQ master (future `3.7.0`). To use this plugin with RabbitMQ 3.6.x, see
the `stable` branch.

## Erlang Version Requirements

This plugin requires Erlang `18.3` or a later version.

## Binary Builds

Binary builds can be obtained [from project releases](https://github.com/rabbitmq/rabbitmq-auth-backend-cache/releases/) on GitHub.

## Building

You can build and install it like any other plugin (see
[the plugin development guide](http://www.rabbitmq.com/plugin-development.html)).

## Authentication and Authorization Backend Configuration

To enable the plugin, set the value of the `auth_backends` configuration item
for the `rabbit` application to include `rabbit_auth_backend_cache`.
`auth_backends` is a list of authentication providers to try in order.


So a configuration fragment that enables this plugin *only* (this example is intentionally incomplete)
would look like this:

``` erlang
[
  {rabbit, [
            {auth_backends, [rabbit_auth_backend_cache]}
            ]
  }
].
```

This plugin wraps another auth backend (an "upstream" one) to reduce load on it.
To configure an upstream backend, use the `rabbitmq_auth_backend_cache.cached_backend` configuration key.

The following configuration uses the LDAP backend for both authentication and authorization
and wraps it with caching:

``` erlang
[
  {rabbit, [
    %% ...
  ]},
  {rabbitmq_auth_backend_cache, [
                                  {cached_backend, rabbit_auth_backend_ldap}
                                ]},
  {rabbit_auth_backend_ldap, [
    %% ...
  ]},
].
```

The following example combines this backend with the [rabbitmq-auth-backend-http](https://github.com/rabbitmq/rabbitmq-auth-backend-http/tree/v3.6.x) and its example
Spring Boot application, and falls back to the internal backend for all requests that the HTTP backend and cache
do not report success for:

``` erlang
[
 {rabbit, [
           {auth_backends, [rabbit_auth_backend_cache, rabbit_auth_backend_internal]}
          ]
 },
 {rabbitmq_auth_backend_cache, [
                                {cached_backend, rabbit_auth_backend_http}
                               ]
  },
  {rabbitmq_auth_backend_http, [{http_method,   post},
                                {user_path,     "http://127.0.0.1:8080/auth/user"},
                                {vhost_path,    "http://127.0.0.1:8080/auth/vhost"},
                                {resource_path, "http://127.0.0.1:8080/auth/resource"}
                               ]
  }
].
```

It is still possible to [use different backends for authorization and authentication](https://www.rabbitmq.com/access-control.html).

The following example configures plugin to use LDAP backend for authentication
but internal backend for authorisation:

``` erlang
[
  {rabbit, [
    %% ...
  ]},
  {rabbitmq_auth_backend_cache, [{cached_backend, {rabbit_auth_backend_ldap,
                                                  rabbit_auth_backend_internal}}]}].
```

## Cache Configuration

You can configure TTL for cache items, by using `cache_ttl` configuration item, specified in **milliseconds**

``` erlang
[
  {rabbit, [
    %% ...
  ]},
  {rabbitmq_auth_backend_cache, [
                                 {cached_backend, rabbit_auth_backend_ldap},
                                 {cache_ttl, 5000}
  ]}
].
```

You can also use a custom cache module to store cached requests. This module
should be an erlang module implementing `rabbit_auth_cache` behaviour and (optionally)
define `start_link` function to start cache process.

This repository provides several implementations:

 * `rabbit_auth_cache_dict` stores cache entries in the internal process dictionary. **This module is for demonstration only and should not be used in production**.
 * `rabbit_auth_cache_ets` stores cache entries in an [ETS](http://learnyousomeerlang.com/ets) table and uses timers for cache invalidation. **This is the default implementation**.
 * `rabbit_auth_cache_ets_segmented` stores cache entries in multiple ETS tables and does not delete individual cache items but rather
   uses a separate process for garbage collection.
 * `rabbit_auth_cache_ets_segmented_stateless` same as previous, but with minimal use of `gen_server` state, using ets tables to store information about segments.

To specify module for caching you should use `cache_module` configuration item and
specify start args with `cache_module_args`.
Start args should be list of arguments passed to module `start_link` function

    [{rabbitmq_auth_backend_cache, [{cache_module, rabbit_auth_backend_ets_segmented},
                                    {cache_module_args, [10000]}]}].

Default values are `rabbit_auth_cache_ets` and `[]`, respectively.

## License and Copyright

(c) 2016 Pivotal Software Inc.

Released under the Mozilla Public License 1.1, same as RabbitMQ.
