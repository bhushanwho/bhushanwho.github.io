---
layout: design-post
title: "an nginx primer"
date: 2026-09-07 08:21:21 +0530
tags: [dev]
---

## example.conf

here's a complete, working nginx config for a reverse proxy setup with load balancing. everything below is a breakdown of this.

<pre><code class="language-go">
upstream backend_pool {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080 backup;
}

server {
    listen 80;
    server_name myapp.com www.myapp.com;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /static/ {
        root /var/www;
    }

    location /images/ {
        alias /var/www/media/images/;
    }
}
</code></pre>

request comes in on port 80, nginx checks it against each location block, and either forwards it into backend_pool or serves a file straight off disk.

## the upstream block

<pre><code class="language-go">
upstream backend_pool {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080 backup;
}
</code></pre>

this defines a named pool of backend servers. it does nothing by itself — a server block has to reference it via `proxy_pass` for it to matter.

- `least_conn` — the load balancing method. this one picks the backend with the fewest active connections. covered in more detail further down.
- `server 10.0.0.1:8080` — one backend in the pool, address and port.
- `weight=3` — this server gets roughly three times the traffic of a server with the default weight of 1.
- `backup` — this server only gets used if all the non-backup servers are down.

other per-server options you'll run into: `max_fails` and `fail_timeout` (how many failures in what window before nginx marks a server dead), and `down` (manually take a server out of rotation without deleting the line).

## the server block

<pre><code class="language-go">
server {
    listen 80;
    server_name myapp.com www.myapp.com;
    ...
}
</code></pre>

this is what actually receives connections.

- `listen 80` — the port (and optionally ip) nginx listens on. `listen 443 ssl` for https.
- `server_name` — which hostnames this block answers for. lets one nginx instance host multiple domains, each with its own server block.

## the location block

<pre><code class="language-go">
location / {
    proxy_pass http://backend_pool;
    ...
}
</code></pre>

`location` matches request paths against a pattern and decides what happens to matching requests. a server block usually has several of these, one per route or asset type. matching isn't strictly first-match — nginx has its own precedence rules (exact match, then longest prefix, then regex), so more specific locations don't have to be listed in any particular order to win.

## proxy_pass vs root vs alias

these three are the actual workhorses inside a location block.

`proxy_pass` forwards the request to another server and relays the response back. use it for anything dynamic — an app server, an api, another nginx instance.

<pre><code class="language-go">
location /api/ {
    proxy_pass http://backend_pool;
}
</code></pre>

the trailing slash on the target url changes the forwarded path:

<pre><code class="language-go">
location /api/ {
    proxy_pass http://10.0.0.1:8080/;
}
# /api/users -> forwarded as /users (prefix stripped)

location /api/ {
    proxy_pass http://10.0.0.1:8080;
}
# /api/users -> forwarded as /api/users (full path kept)
</code></pre>

`root` serves a file straight from disk, appending the full request path to the root path.

<pre><code class="language-go">
location /static/ {
    root /var/www;
}
# /static/logo.png -> /var/www/static/logo.png
</code></pre>

`alias` also serves from disk, but replaces the location prefix instead of appending to it.

<pre><code class="language-go">
location /images/ {
    alias /var/www/media/images/;
}
# /images/logo.png -> /var/www/media/images/logo.png
</code></pre>

mixing up `root` and `alias` is a common source of 404s and wrong-file bugs. `root` keeps the matched prefix in the final path; `alias` throws it away.

## proxy_set_header

<pre><code class="language-go">
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
</code></pre>

when nginx proxies a request, the backend sees nginx as the client by default, not the original visitor. these headers pass the real information through:

- `Host` — preserves the original hostname the client requested.
- `X-Real-IP` — the client's actual ip address.
- `X-Forwarded-For` — the chain of ips the request passed through, useful if there are multiple proxies.

## load balancing methods

### round robin

the default. no directive needed. requests rotate through the server list in order — one, two, three, back to one. stateless, no memory of past requests.

### least_conn

<pre><code class="language-go">
upstream backend_pool {
    least_conn;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
</code></pre>

sends each new request to whichever backend currently has the fewest open connections. better than round robin when requests vary a lot in how long they take to handle.

### ip_hash

<pre><code class="language-go">
upstream sticky {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
</code></pre>

hashes the client's ip address to consistently pick the same backend for the same client. this is the standard open-source way to get sticky behavior.

### hash (generic)

<pre><code class="language-go">
upstream backend_pool {
    hash $request_uri consistent;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
</code></pre>

like `ip_hash` but you choose the key — a url, a header, a cookie value, whatever nginx variable makes sense. `consistent` enables consistent hashing, which minimizes remapping when servers are added or removed (regular hashing reshuffles almost everything when the pool size changes; consistent hashing only reshuffles a fraction).

## when to actually use sticky/hash-based routing

only when the backend is holding state locally — session data in memory, a websocket connection, an in-progress upload. if session state lives in a shared store like redis, none of this matters and plain round robin or `least_conn` works fine, which is usually the better place to put that state anyway since it also gives you failover.
