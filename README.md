# lua-resty-session

**lua-resty-session** is a secure, and flexible session library for OpenResty.

## Hello World with lua-resty-session

```nginx
http {
    server {
        listen       8080;
        server_name  localhost;
        default_type text/html;
        location / {
            content_by_lua '
                ngx.say("<html><body><a href=/start>Start the test</a>!</body></html>")
            ';
        }
        location /start {
            content_by_lua '
                local session = require "resty.session".start()
                session.data.name = "OpenResty Fan"
                session:save()
                ngx.say("<html><body>Session started. ",
                        "<a href=/test>Check if it is working</a>!</body></html>")
            ';
        }
        location /test {
            content_by_lua '
                local session = require "resty.session".open()
                ngx.say("<html><body>Session was started by <strong>",
                        session.data.name or "Anonymous",
                        "</strong>! <a href=/destroy>Destroy the session</a>.</body></html>")
            ';
        }
        location /destroy {
            content_by_lua '
                local session = require "resty.session".start()
                session:destroy()
                ngx.say("<html><body>Session was destroyed. ",
                        "<a href=/check>Is it really so</a>?</body></html>")
            ';
        }
        location /check {
            content_by_lua '
                local session = require "resty.session".open()
                ngx.say("<html><body>Session was really destroyed, you are known as ",
                        "<strong>",
                        session.data.name or "Anonymous",
                        "</strong>! <a href=/>Start again</a>.</body></html>")
            ';
        }
    }
}
```

## Roadmap

1. Implement cookieless server-side session support using `ssl_session_id` as a `session.id` (using a server-side storage).
2. Add support for cipher plugins.
3. Add support for `lua-resty-nettle` for more wide variety of encryption algorithms as a plugin.

## Installation

Just place [`session.lua`](https://github.com/bungle/lua-resty-session/blob/master/lib/resty/session.lua) somewhere in your `package.path`, preferably under `resty` directory. If you are using OpenResty, the default location would be `/usr/local/openresty/lualib/resty`.

### Using LuaRocks or MoonRocks

If you are using LuaRocks >= 2.2:

```Shell
$ luarocks install lua-resty-session
```

If you are using LuaRocks < 2.2:

```Shell
$ luarocks install --server=http://rocks.moonscript.org moonrocks
$ moonrocks install lua-resty-session
```

MoonRocks repository for `lua-resty-session` is located here: https://rocks.moonscript.org/modules/bungle/lua-resty-session.

## About The Defaults

`lua-resty-session` does by default set session only cookies (non-persistent, and `HttpOnly`) so that
the cookies are not readable from Javascript (not subjectible to XSS in that matter). It will also set
`Secure` flag by default when the request was made via SSL/TLS connection. Cookies send via SSL/TLS
don't work when sent via HTTP and vice-versa (unless the checks are disabled). By default the HMAC key
is generated from session id (random bytes generated with OpenSSL), expiration time, unencrypted
data, and Nginx variables `ssl_session_id` (if requested with TLS/SSL), `http_user_agent` and `scheme`.
You may also configure it to use `remote_addr` as well by setting `set $session_check_addr on;`
(but this may be problematic with clients behind proxies or NATs that change the remote address
between requests).

The data part is encrypted with AES-algorithm (by default it uses OpenSSL `EVP_aes_256_cbc` and
`EVP_sha512` functions that are provided with `lua-resty-string`. They come pre-installed with
the default OpenResty bundle. The `lua-resty-session` library is not tested with all the
`resty.aes` functions (but the defaults are tested to be working). Please let me know or contact
`lua-resty-string` project if you hit any problems with different algorithms. In a future we
might implement a pluggable cipher support.

Session identifier length is by default 16 bytes (randomly generated data with OpenSSL
`RAND_pseudo_bytes` function). The server secret is also generated by default with this same
function, and its length is determined by calculating the used `$session_cipher_size` divided
by 8 (so by default it uses 32 bytes). This will work until Nginx is restarted, but you might want
to consider setting your own secret using `set $session_secret 623q4hR325t36VsCD3g567922IC0073T;`,
for example (this will work in farms installations as well, but you are then responsible for
rotating the secret). On farm installations you should also configure other session configuration
variables the same on all the servers in the farm.

Cookie parts are encoded with cookie safe Base64 encoding (we also support pluggable encoders).
Before encrypting and encoding the data part, the data is serialized with JSON encoding (so you can
use basic Lua types in data, and expect to receive them back as the same Lua types). JSON encoding
is done by the bundled OpenResty cJSON library (Lua cJSON). We do support pluggable serializers as
well, though only serializer currently supplied is JSON. Cookie's path scope is by default `/`
(meaning that it will be send to all paths in the server. The domain scope is not set by default,
and it means that the cookie will only be sent back to same domain where it originated.

For session data we do support pluggable storage adapters. The default adapter is `cookie` that
stores data to client-side cookie. Currently we do also support a few server side storages: `shm`
(aka a shared dictionary), `memcache`, and `redis`.

## Notes About Turning Lua Code Cache Off

In issue ([#15](https://github.com/bungle/lua-resty-session/issues/15)) it was raised that there may
be problems of using `lua-resty-session` when the `lua_code_cache` setting has been turned off.

Nginx:

```nginx
lua_code_cache off;
```

The problem is caused by the fact that by default we do generate session secret automatically with
a random generator (on first use of the library). If the code cache is turned off, we regenerate
the secret on each request. That will invalidate the cookies aka making sessions non-functioning.
The cure for this problem is to define the secret in Nginx or in Lua code (it is a good idea to
always have session secret defined).

Nginx:

```nginx
set $session_secret 623q4hR325t36VsCD3g567922IC0073T;
```

Lua:

```lua
local session = require "resty.session".start{ secret = "623q4hR325t36VsCD3g567922IC0073T" }
-- or
local session = require "resty.session".new()
session.secret = "623q4hR325t36VsCD3g567922IC0073T"
```

## Pluggable Storage Adapters

With version 2.0 we started to support pluggable session data storage adapters. We do currently have
support for these backends:

* `cookie` aka Client Side Cookie (this is the default adapter)
* `shm` aka Lua Shared Dictionary
* `memcache` aka Memcached Storage Backend (thanks @zandbelt)
* `redis` aka Redis Backend

Here are some comparisons about the backends:

|                               | cookie | shm  | memcache | redis |
| :---------------------------- | :----: | :--: | :------: | :---: |
| Stateless                     | ✓      |      |          |       |
| Lockless                      | ✓      | ¹    | ¹        | ¹     |
| Works with Web Farms          | ✓      |      | ✓        | ✓     |
| Session Data Stored on Client | ✓      |      |          |       |
| Zero Configuration            | ✓      |      |          |       |
| Extra Dependencies            |        |      | ✓        | ✓     |

¹ Can be configured lockless.

The storage adapter can be selected from Nginx config like this:

```nginx
set $session_storage shm;
```

Or with Lua code like this:

```lua
local session = require "resty.session".new()
session.storage = "shm"
```

#### Cookie Storage Adapter

Cookie storage adapter is the default adapter that is used if storage adapter has not been configured. Cookie
adapter does not have any settings.

Cookie adapter can be selected with configuration (if no configuration is present, the cookie adapter is picked up):

```nginx
set $session_storage cookie;
```

#### Shared Dictionary Storage Adapter

Shared dictionary uses OpenResty shared dictionary and works with multiple worker processes, but it isn't a good
choice if you want to run multiple separate frontends. It is relatively easy to configure and has some added
benefits on security side compared to `cookie`, although the normal cookie adapters is really secure as well.

Shared dictionary adapter can be selected with configuration:

```nginx
set $session_storage shm;
```

But for this to work, you will also need a storage configured for that:

```nginx
http {
   lua_shared_dict sessions 10m;
}
```

Additionally you can configure the locking and some other things as well:

```nginx
set $session_shm_store         sessions;
set $session_shm_uselocking    on;
set $session_shm_lock_exptime  30;
set $session_shm_lock_timeout  5;
set $session_shm_lock_step     0.001;
set $session_shm_lock_ratio    2;
set $session_shm_lock_max_step 0.5;
```

#### Memcache Storage Adapter

Memcache storage adapter stores the session data inside Memcached server.
It is scalable and works with web farms. 

Memcache adapter can be selected with configuration:

```nginx
set $session_storage memcache;
```

Additionally you can configure Memcache adapter with these settings:

```nginx
set $session_memcache_prefix        sessions;
set $session_memcache_socket        unix:///var/run/memcached/memcached.sock;
set $session_memcache_host          127.0.0.1;
set $session_memcache_port          11211;
set $session_memcache_uselocking    on;
set $session_memcache_spinlockwait  10000,
set $session_memcache_maxlockwait   30,
set $session_memcache_pool_timeout  45
set $session_memcache_pool_size     10
}
```

## Lua API

### Functions and Methods

#### table session.new(opts or nil)

With this function you can create a new session table (i.e. the actual session instance). This allows
you to generate session table first, and set invidual configuration before calling `session:open()` or
`session:start()`. You can also pass in `opts` Lua `table` with the configurations.

```lua
local session = require "resty.session".new()
-- set the configuration parameters before calling start
session.cookie.domain = ".mydomain.com"
-- call start before setting session.data parameters
session:start()
session.data.uid = 1
-- save session and update the cookie to be sent to the client
session:save()
```

This is equivalent to this:

```lua
local session = require "resty.session".new{ cookie = { domain = ".mydomain.com" } }
session:start()
session.data.uid = 1
session:save()
```

As well as with this:

```lua
local session = require "resty.session".start{ cookie = { domain = ".mydomain.com" } }
session.data.uid = 1
session:save()
```

#### table, boolean session.open(opts or nil)

With this function you can open a new session. It will create a new session Lua `table` on each call (unless called with
colon `:` as in examples above with `session.new`). Calling this function repeatedly will be a no-op when using colon `:`.
This function will return a (new) session `table` as a result. If the session cookie is supplied with user's HTTP(S)
client then this function validates the supplied session cookie. If validation is successful, the user supplied session
data will be used (if not, a new session is generated with empty data). You may supply optional session configuration
variables with `opts` argument, but be aware that many of these will only have effect if the session is a fresh session
(i.e. not loaded from user supplied cookie). The second `boolean` return argument will be `true` if the user
client send a valid cookie (meaning that session was already started on some earlier request), and `false` if the
new session was created (either because user client didn't send a cookie or that the cookie was not a valid one). This
function will not set a client cookie. You need to call `session:start()` to really start the session. This open is mainly
used if you only want to read data and avoid automatically sending a cookie (see also issue
[#12](https://github.com/bungle/lua-resty-session/issues/12)).

```lua
local session = require "resty.session".open()
-- Set some options (overwriting the defaults or nginx configuration variables)
local session = require "resty.session".open{ identifier = { length = 32 }}
-- Read some data
if session.present then
    ngx.print(session.data.uid)
end
-- Now let's really start the session
-- (session.started will be always false in this example):
if not session.started then 
    session:start()
end
session.data.greeting = "Hello, World!"
session:save()
```

#### table, boolean session.start(opts or nil)

With this function you can start a new session. It will create a new session Lua `table` on each call (unless called with
colon `:` as in examples above with `session.new`). Right now you should only start session once per request as calling
this function repeatedly will overwrite the previously started session cookie and session data. This function will return
a (new) session `table` as a result. If the session cookie is supplied with user's HTTP(S) client then this function
validates the supplied session cookie. If validation is successful, the user supplied session data will be used
(if not, a new session is generated with empty data). You may supply optional session configuration variables
with `opts` argument, but be aware that many of these will only have effect if the session is a fresh session
(i.e. not loaded from user supplied cookie). This function does also manage session cookie renewing configured
with `$session_cookie_renew`. E.g. it will send a new cookie with a new expiration time if the following is
met `session.expires - now < session.cookie.renew`. The second `boolean` return argument will be `true` if the user
client send a valid cookie (meaning that session was already started on some earlier request), and `false` if the
new session was created (either because user client didn't send a cookie or that the cookie was not a valid one).

```lua
local session = require "resty.session".start()
-- Set some options (overwriting the defaults or nginx configuration variables)
local session = require "resty.session".start{ identifier = { length = 32 }}
```

#### boolean, string session:regenerate(flush or nil)

This function regenerates a session. It will generate a new session identifier (`session.id`) and optionally
flush the session data if `flush` argument evaluates `true`. It will automatically call `session:save` which
means that a new expires flag is set on the cookie, and the data is encrypted with the new parameters. With
client side sessions (`cookie` storage adapter) this overwrites the current cookie with a new one (but it
doesn't invalidate the old one as there is no state held on server side - invalidation actually happens when
the cookie's expiration time is not valid anymore). This function returns a boolean value if everything went
as planned. If not it will return error string as a second return value.

```lua
local session = require "resty.session".start()
session:regenerate()
-- Flush the current data
session:regenerate(true)
```

#### boolean, string session:save()

This function saves the session and sends (not immediate though, as actual sending is handled by Nginx/OpenResty)
a new cookie to client (with a new expiration time and encrypted data). You need to call this function whenever
you want to save the changes made to `session.data` table. It is advised that you call this function only once
per request (no need to encrypt and set cookie many times). This function returns a boolean value if everything
went as planned. If not it will return error string as a second return value.

```lua
local session = require "resty.session".start()
session.data.uid = 1
session:save()
```

#### boolean session:destroy()

This function will immediately set session data to empty table `{}`. It will also send a new cookie to
client with empty data and Expires flag `Expires=Thu, 01 Jan 1970 00:00:01 GMT` (meaning that the client
should remove the cookie, and not send it back again). This function returns a boolean value if everything went
as planned.

```lua
local session = require "resty.session".start()
session:destroy()
```

### Fields

#### string session.id

`session.id` holds the current session id. By default it is 16 bytes long (raw binary bytes).
It is automatically generated.

#### boolean session.present

`session.present` can be used to check if the session that was opened with `session.open` or `session.start`
was really a one the was received from a client. If the session is a new one, this will be false.

#### boolean session.opened

`session.opened` can be used to check if the `session:open()` was called for the current session
object.


#### boolean session.started

`session.started` can be used to check if the `session:start()` was called for the current session
object.

#### boolean session.destroyed

`session.destroyed` can be used to check if the `session:destroy()` was called for the current session
object. It will also set `session.opened`, `session.started`,  and `session.present` to false.


#### number session.identifier.length

`session.identifier.length` holds the length of the `session.id`. By default it is 16 bytes.
This can be configured with Nginx `set $session_identifier_length 16;`.

#### string session.key

`session.key` holds the HMAC key. It is automatically generated. Nginx configuration like
`set $session_check_ssi on;`, `set $session_check_ua on;`, `set $session_check_scheme on;` and `set $session_check_addr on;`
 will have effect on the generated key.

#### table session.data

`session.data` holds the data part of the session cookie. This is a Lua `table`. `session.data`
is the place where you store or retrieve session variables. When you want to save the data table,
you need to call `session:save` method.

**Setting session variable:**

```lua
local session = require "resty.session".start()
session.data.uid = 1
session:save()
```

**Retrieving session variable (in other request):**

```lua
local session = require "resty.session".open()
local uid = session.data.uid
```

#### number session.expires

`session.expires` holds the expiration time of the session (expiration time will be generated when
`session:save` method is called).

#### string session.secret

`session.secret` holds the secret that is used in keyed HMAC generation.

#### boolean session.cookie.persistent

`session.cookie.persistent` is by default `false`. This means that cookies are not persisted between browser sessions (i.e. they are deleted when the browser is deleted). You can enable persistent sessions if you want to by setting this to `true`. This can be configured with Nginx `set $session_cookie_persistent on;`.

#### number session.cookie.renew

`session.cookie.renew` holds the minimun seconds until the cookie expires, and renews cookie automatically
(i.e. sends a new cookie with a new expiration time according to `session.cookie.lifetime`). This can be configured
with Nginx `set $session_cookie_renew 600;` (600 seconds is the default value).

#### number session.cookie.lifetime

`session.cookie.lifetime` holds the cookie lifetime in seconds in the future. By default this is set
to 3,600 seconds. This can be configured with Nginx `set $session_cookie_lifetime 3600;`. This does not
set cookie's expiration time on session only (by default) cookies, but it is used if the cookies are
configured persistent with `session.cookie.persistent == true`. See also notes about [ssl_session_timeout](#nginx-configuration-variables).

#### string session.cookie.path

`session.cookie.path` holds the value of the cookie path scope. This is by default permissive `/`. You
may want to have a more specific scope if your application resides in different path (e.g. `/forums/`).
This can be configured with Nginx `set $session_cookie_path /forums/;`.

#### string session.cookie.domain

`session.cookie.domain` holds the value of the cookie domain. By default this is automatically set using
Nginx variable `host`. This can be configured with Nginx `set $session_cookie_domain openresty.org;`.
For `localhost` this is omitted.

#### boolean session.cookie.secure

`session.cookie.secure` holds the value of the cookie `Secure` flag. meaning that when set the client will
only send the cookie with encrypted TLS/SSL connection. By default the `Secure` flag is set on all the
cookies where the request was made through TLS/SSL connection. This can be configured and forced with
Nginx `set $session_cookie_secure on;`.

#### boolean session.cookie.httponly

`session.cookie.httponly` holds the value of the cookie `HttpOnly` flag. By default this is enabled,
and I cannot think of an situation where one would want to turn this off. By keeping this on you can
prevent your session cookies access from Javascript and give some safety of XSS attacks. If you really
want to turn this off, this can be configured with Nginx `set $session_cookie_httponly off;`.

#### string session.cookie.delimiter

`session.cookie.delimiter` is used to configure how the different parts of the data stored in a cookie is
delimiter. By default it is a pipe character, `|`. It is up to storage adapter to decide if this configuration
parameter is used.

#### number session.cipher.size

`session.cipher.size` holds the size of the cipher (`lua-resty-string` supports AES in `128`, `192`,
and `256` bits key sizes). See `aes.cipher` function in `lua-resty-string` for more information.
By default this will use `256` bits key size. This can be configured with Nginx
`set $session_cipher_size 256;`.

#### string session.cipher.mode

`session.cipher.mode` holds the mode of the cipher. `lua-resty-string` supports AES in `ecb`, `cbc`,
`cfb1`, `cfb8`, `cfb128`, `ofb`, and `ctr` modes (ctr mode is not available with 256 bit keys).
See `aes.cipher` function in `lua-resty-string` for more information. By default `cbc` mode is
used. This can be configured with Nginx `set $session_cipher_mode cbc;`.

#### function session.cipher.hash

`session.cipher.hash` is used in ecryption key, and iv derivation (see: OpenSSL
[EVP_BytesToKey](https://www.openssl.org/docs/crypto/EVP_BytesToKey.html)). By default `sha512` is
used but `md5`, `sha1`, `sha224`, `sha256`, and `sha384` are supported as well in `lua-resty-string`.
This can be configured with Nginx `set $session_cipher_hash sha512;`.

#### number session.cipher.rounds

`session.cipher.rounds` can be used to slow-down the encryption key, and iv derivation. By default
this is set to `1` (the fastest). This can be configured with Nginx `set $session_cipher_rounds 1;`.

#### boolean session.check.ssi

`session.check.ssi` is additional check to validate that the request was made with the same SSL
session as when the original cookie was delivered. This check is enabled by default on non-persistent
sessions and disabled by default on persistent sessions. Please note that on TLS with TLS Tickets enabled,
this will be empty) and not used. This is discussed on issue #5 (https://github.com/bungle/lua-resty-session/issues/5).
You can disable TLS tickets with Nginx configuration:

```nginx
ssl_session_tickets off;
```

#### boolean session.check.ua

`session.check.ua` is additional check to validate that the request was made with the same user-agent browser string
as where the original cookie was delivered. This check is enabled by default.

#### boolean session.check.addr

`session.check.addr` is additional check to validate that the request was made from the same remote ip-address
as where the original cookie was delivered. This check is disabled by default.

#### boolean session.check.scheme

`session.check.scheme` is additional check to validate that the request was made using the same protocol 
as the one used when the original cookie was delivered. This check is enabled by default.

## Nginx Configuration Variables

You can set default configuration parameters directly from Nginx configuration. It's **IMPORTANT** to understand
that these are read only once (not on every request), for performance reasons. This is especially important if
you run multiple sites (with different configurations) on the same Nginx server. You can of course set the common
parameters on Nginx configuration even on that case. But if you are really supporting multiple site with different
configurations (e.g. different `session.secret` on each site), you should set these in code (see: `session.new`
and `session.start`).

Please note that Nginx has also its own SSL/TLS caches and timeouts. Especially note `ssl_session_timeout` if you
are running services over SSL/TLS as this will end sessions regardless of `session.cookie.lifetime`. Please adjust
that accordingly or disable `ssl_session_id` check `session.check.ssi = false` (in code) or
`set $session_check_ssi off;` (in Nginx configuration).

You may want to add something like this to your Nginx SSL/TLS config (quite a huge cache in this example, 1 MB is
about 4.000 SSL sessions):

```nginx
ssl_session_cache shared:SSL:100m;
ssl_session_timeout 60m;
```

Also note that the `ssl_session_id` may be `null` if the TLS tickets are enabled. You can disable tickets in Nginx
server with the configuration below:

```nginx
ssl_session_tickets off;
```

Right now this is a workaround and may change in a future if we find alternative ways to have the added security
that we have with `ssl_session_id` with TLS tickets too. While TLS tickets are great, they also have effect on
(Perfect) Forward Secrecy, and it is adviced to disable tickets until the problems mentioned in
[The Sad State of Server-Side TLS Session Resumption Implementations](https://timtaubert.de/blog/2014/11/the-sad-state-of-server-side-tls-session-resumption-implementations/)
article are resolved.

Here is a list of `lua-resty-session` related Nginx configuration variables that you can use to control
`lua-resty-session`:

```nginx
set $session_name              session;
set $session_secret            623q4hR325t36VsCD3g567922IC0073T;
set $session_storage           cookie;
set $session_encoder           base64;
set $session_serializer        json;
set $session_cookie_persistent off;
set $session_cookie_renew      600;
set $session_cookie_lifetime   3600;
set $session_cookie_path       /;
set $session_cookie_domain     openresty.org;
set $session_cookie_secure     on;
set $session_cookie_httponly   on;
set $session_cookie_delimiter  |;
set $session_cipher_mode       cbc;
set $session_cipher_size       256;
set $session_cipher_hash       sha512;
set $session_cipher_rounds     1;
set $session_check_ssi         on;
set $session_check_ua          on;
set $session_check_scheme      on;
set $session_check_addr        off;
set $session_identifier_length 16;
```

## License

`lua-resty-session` uses two clause BSD license.

```
Copyright (c) 2015, Aapo Talvensaari
All rights reserved.

Redistribution and use in source and binary forms, with or without modification,
are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this
  list of conditions and the following disclaimer in the documentation and/or
  other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
```
