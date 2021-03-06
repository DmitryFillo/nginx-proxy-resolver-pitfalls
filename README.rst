===============================================
NGINX proxy pitfalls related with DNS resolving
===============================================

If you're using ``proxy_pass`` and your endpoint's IPs can vary in time, please read this article to avoid misunderstandings about how nginx works.

.. contents::

TL;DR
=====

If you want to force nginx resolve your endpoints, you should:

* Use variables within ``proxy_pass`` directive, e.g. ``proxy_pass https://$endpoint/;``, where ``$endpoint`` can be manually setted or extracted from location regexp. `Read more <https://github.com/DmitryFillo/nginx-proxy-pitfalls#add-variables>`_.
* Make sure that your endpoint isn't used in the another locations w/o variables, because in this case resolving won't work. To fix this move endpoint domain to the ``upstream`` or use variables in the ``proxy_pass`` in all locations to make resolving works. `Read more <https://github.com/DmitryFillo/nginx-proxy-pitfalls#caveats>`_.
* `You can have both resolve and non-resolve locations for same domain <https://github.com/DmitryFillo/nginx-proxy-pitfalls/blob/master/README.rst#you-can-have-both-resolve-and-non-resolve-locations-for-same-domain>`_.
* When a variable is used in proxy_pass directive, the location header is not longer adjusted. To get around this, simply set ``proxy_redirect``. `Read more <https://github.com/DmitryFillo/nginx-proxy-pitfalls#caveats>`_.

But I recommend to read full article, because it's interesting.

Explanatory example
===================

.. code:: nginx

  location /api/ {
      proxy_pass http://api.com/;
  }

In this case nginx will resolve api.com only once at startup (or reload). But there are some cases when your endpoint can be resolved to any IP, e.g. if you're using load balancer which doing magic failover via DNS mapping. If api.com will point to another IP your proxying will fail.

Finding the solution
====================

Add a resolver directive
------------------------

You can check `official nginx documentation <http://nginx.org/en/docs/>`_ and find `resolver <http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver>`_ directive:

.. code:: nginx

  location /api/ {
      resolver 8.8.8.8;
      proxy_pass https://api.com/;
  }

No, it will not work. Even this will not work:

.. code:: nginx

  location /api/ {
      resolver 8.8.8.8 valid=1s;
      proxy_pass https://api.com/;
  }

It's because of nginx doesn't respect ``resolver`` directive in this case. It will resolve api.com only at startup (or reload) by system resolver (/etc/resolv.conf), even if real TTL of A/AAAA record api.com is 1s.

Add variables
-------------

You can google a bit and find that `nginx try to resolve proxy endpoint with variables <https://trac.nginx.org/nginx/ticket/723>`_. Also `official documentation for proxy_pass directive notices this too <http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass>`_. Hmmm, I think this should be noticed in the ``resolver`` description, but let's try anyway:

.. code:: nginx

  location = /proxy/ {
      set $endpoint proxy.com;
      resolver 8.8.8.8 valid=10s;
      proxy_pass https://$endpoint/;
  }

Works as expected, nginx will query proxy.com every 10s on particular requests. These configurations works too:

.. code:: nginx

  set $endpoint api.com;
  location ~ ^/api/(.*)$ {
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/$1$is_args$args;
  }

.. code:: nginx

  location ~ ^/(?<dest_proxy>[\w-]+)(?:/(?<path_proxy>.*))? {
      resolver 8.8.8.8 ipv6=off valid=60s;
      proxy_pass https://${dest_proxy}.example.com/${path_proxy}$is_args$args;
  }

Notice that nginx will start even without ``resolver`` directive, but will fail with 502 at runtime, because "no resolver defined to resolve".

Caveats
-------

.. code:: nginx

  location = /api_version/ {
      proxy_pass https://api.com/version/;
  }

  location ~ ^/api/(.*)$ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/$1$is_args$args;
  }

In this case nginx will resolve api.com once at startup with system resolver and then will never do re-resolve even for /api/ requests. *Example with /api_version/ is just synthetic example, you can use more complex scenarios with headers set, etc.*

Use variables everywhere to make it work as expected:

.. code:: nginx

  location = /api_version/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/version/;
  }

  location ~ ^/api/(.*)$ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/$1$is_args$args;
  }

You can move ``set`` and ``resolver`` to the ``server`` or ``http`` (or use ``include``) directives to avoid copy-paste (also I assume that it will increase perfomance a bit, but I haven't tested it).

If response from proxy contains ``Location`` header, as in the case of a redirect, nginx will automatically replace these values as needed. However, if variables are used in ``proxy_pass``, this must be done explicitly via ``proxy_redirect``:

.. code:: nginx

  location = /api_version/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/version/;
      proxy_redirect https://$endpoint/ /;
  }

  location ~ ^/api/(.*)$ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/$1$is_args$args;
      proxy_redirect https://$endpoint/ /;
  }

Single line in `nginx docs <http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_redirect>`_ that mention it:
    
    The default parameter is not permitted if proxy_pass is specified using variables.

Upstreams
=========

If you're using nginx plus, you can use ``resolve`` parameter, `check out documentation <http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server>`_. I assume that it will be efficient, because documentation says "monitors changes of the IP addresses that correspond to a domain name of the server", while solutions listed above will query DNS on the particular requests. But if you're using open source nginx, no honey is available for you. No money — no honey.

You can have both resolve and non-resolve locations for same domain
-------------------------------------------------------------------

.. code:: nginx

  upstream proxy {
      server proxy.com:443;
  }

  server {
      listen      80;
      server_name fillo.me;

      location = /proxy-with-resolve/ {
         set $endpoint proxy.com;
         resolver 8.8.8.8 valid=1s;
         proxy_pass https://$endpoint/;
      }

      location = /proxy-without-resolve/ {
         proxy_pass https://proxy/;
         proxy_set_header Host proxy.com;
      }
  }

Yes, http://fillo.me/proxy-with-resolve/ will resolve proxy.com every 1s on particular requests, while http://fillo.me/proxy-without-resolve/ will not resolve proxy.com (nginx will resolve proxy.com at startup/reload once). This magic works because ``upstream`` directive is used.

Another example:

.. code:: nginx

  upstream api_version {
      server version.api.com:443;
  }

  server {
      listen      80;
      server_name fillo.me;

      location = /api_version/ {
         proxy_pass https://api_version/version/;
         proxy_set_header Host version.api.com;
      }

      location ~ ^/api/(?<dest_proxy>[\w-]+)(?:/(?<path_proxy>.*))? {
          resolver 8.8.8.8 valid=60s;
          proxy_pass https://${dest_proxy}.api.com/${path_proxy}$is_args$args;
      }
  }

* If you will open http://fillo.me/api_version/ then no resolve will be done, because of nginx resolved version.api.com at startup.
* If you will open http://fillo.me/api/version/version/ then it will work as expected, nginx will resolve version.api.com every 60s on particular request.
* If you will open http://fillo.me/api/checkout/items/ then it will work as expected, nginx will resolve checkout.api.com every 60s on particular request.

Tested on
=========

* 1.9.6
* 1.10.1

Although I think it works for many other versions.

Further research
================

* `This issue <https://trac.nginx.org/nginx/ticket/723>`_ says that changing HTTPS to the HTTP helps. Check how protocol changes affects examples above.
* Compare perfomance with and without resolving.
* Compare perfomance with different variables scopes.
* How to force upstream resolving.
