# How I run Certbot (as non-root and automated)

I have previously noted that I get all my TLS certificates from [Let's Encrypt][le], but since my usage of the client
deviates quite a bit from the standard, I figured I should take a few minutes to describe my setup.

## Prelude: Configuration management

My system configuration is formatted as holograms that can be compiled with [holo-build][hb] and applied with
[Holo][ho]. This article is about my `hologram-letsencrypt`, so I'll be linking to [its source code][hl] a few times. If
you don't know about Holo, don't worry; the source code should be clear enough as to instruct you how to port this setup
to other configuration management schemes.

## Choosing the challenge method

In the ACME protocol, you need to prove your control over the domain name for which you are requesting a certificate.
The two most common methods are the *DNS challenge* (where you need to place a certain TXT record in the domain's DNS)
and the *webroot challenge* (where you need to configure your webserver to serve a certain file at a certain defined URL
on that domain). Since I want a method that works regardless of DNS providers (I have a lot of these, some without
automatable DNS zone API), I went with the webroot challenge.

This means that every server that wants to get TLS certificates needs to run an HTTP server on port 80 to respond to the
webroot challenge. I chose nginx, which is my go-to choice for HTTP servers, and [configured it][s1] to

* serve the `.well-known/acme-challenge/` path from `/srv/letsencrypt` (this is where the webroot challenge looks for
  the expected file), and
* upgrade all other requests to HTTPS via status code 302 and the `Location` header.

The latter implies that all my web domains exclusively use HTTPS, which is the case on all my servers since I don't care
about ad servers or other garbage.

The webserver is activated as a systemd unit called `nginx-for-letsencrypt.service`. When there are actual websites on
that server, another nginx will be listening on port 443 to serve these. This separation is useful to break a circular
dependency: The nginx that listens on port 443 needs the TLS certificates to start, but Certbot can only run if nginx is
running on port 80. If both nginx were the same, manual intervention would be required to resolve this dependency cycle
when setting up a new server or a new website. It's also nice because the nginx on port 443 can be left out entirely
when not needed, i.&nbsp;e.&nbsp;when the server needs TLS certificates for services that are not HTTPS
(e.&nbsp;g.&nbsp;XMPP or mail).

**Side-note:** This is also the reason why I haven't looked at webservers with integrated Let's Encrypt support, such as
[Caddy][ca]. If you only serve HTTPS, then Caddy is probably a good idea, but my method allows me to issue certificates
for all TLS-enabled protocols using the same procedure.

## Certbot

I started to use Certbot back in 2015 when it was still called `letsencrypt`. There were no other mature ACME clients at
this point, so I played it safe by going with the reference implementation. The big drawback of Certbot is that it
supposedly only runs with root privileges. I found this to be very untrue. I'm running it under user/group
`letsencrypt`. All that it takes is to `chown letsencrypt:letsencrypt` [a few things][s2]:

* `/etc/letsencrypt` (where Certbot puts the certificates, private keys, etc.)
* `/var/log/letsencrypt`
* `/srv/letsencrypt` (the document root for the webroot challenge, which is served by the nginx; see above)
* `/var/lib/letsencrypt` (the home directory of the `letsencrypt` user account; this may not even be necessary)

Now I can invoke Certbot like this:

```bash
sudo -u letsencrypt -g letsencrypt \
  certbot certonly --quiet --non-interactive --agree-tos \
  --keep-until-expiring --webroot -w /srv/letsencrypt/ -d "$domain"
```

That's a mouthful, so let's take this apart:

* `sudo -u letsencrypt -g letsencrypt certbot`: Run Certbot under that user and group instead of as root.
* `certonly`: Do not touch the webserver configuration, just provision the certificate.
* `--quiet --non-interactive`: Be less chatty and don't ask questions.
* `--agree-tos`: Without this switch, the first invocation will fail since one needs to agree to the ToS, but
  `--non-interactive` forbids the client to ask for agreement.
* `--keep-until-expiring`: This switch is generally useful when you're invoking Certbot from a cronjob. It means that
  Certbot will do nothing when there is already a valid certificate for the given domain, unless it will expire
  soon-ish.
* `--webroot -w /srv/letsencrypt/`: Use the webroot challenge method, where the document root is `/srv/letsencrypt`.
  We need to state the document root explicitly because we chose `certonly` earlier on and thus forbade Certbot from
  determining the document root from the webserver configuration.
* `-d "$domain"`: Get a certificate for that domain.

## Finding out which domains to handle

Now we basically just need to call Certbot once for every domain that we need a TLS certificate for. (You can also issue
certificates for multiple domain names at once, but I like to keep things neatly separated.) I use the fact that every
TLS certificate and private key must be mentioned in the configuration file of the server that provides the TLS-secured
endpoint. For example, an nginx configuration will have a statement like this:

```
ssl_certificate /etc/letsencrypt/live/www.example.org/fullchain.pem;
```

With some `grep` and `cut`, you can easily extract the domain names from the `nginx.conf`. This is what *collector
scripts* do in my setup. I have one for each service with TLS support, e.&nbsp;g.&nbsp; for [nginx][s3] or for
[Prosody][s4].

A small shell script, fittingly called [`letsencrypt-allofthem`][s5], runs all of these collectors and then invokes
Certbot once for each of the collected domains. That script also has hooks that allow to reload or restart services
afterwards, so that the new certificates and keys can be loaded before the old ones in memory expire.

## End-to-end example: Deploying a TLS-enabled static website in just 2 minutes

I have some further automation for static websites. I just dump the HTML files etc.&nbsp;into a directory
`/data/static-web/www.example.org`. Now only two steps are left: [`sudo holo apply`][ho] renders an nginx configuration
like this:

```
$ cat /etc/nginx/sites-enabled/static-web.conf
server {
    server_name www.example.org;
    include     /etc/nginx/server-baseline-https.inc;

    # CSP includes unsafe-inline to allow <style> tags in hand-written HTML
    add_header  Content-Security-Policy "default-src 'self' 'unsafe-inline'" always;

    ssl_certificate     /etc/letsencrypt/live/www.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.example.org/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/www.example.org/chain.pem;

    location / {
        root    /data/static-web/www.example.org;
        index   index.html index.htm;
        charset utf-8;
    }
}
```

The second step is `sudo letsencrypt-allofthem`, which provisions the TLS certificate and key for that domain, and
because that also reloads nginx at the end, the new configuration takes effect immediately.

[le]: https://letsencrypt.org
[hb]: https://github.com/holocm/holo-build
[ho]: https://github.com/holocm/holo
[hl]: https://github.com/majewsky/system-configuration/blob/master/hologram-letsencrypt.pkg.toml
[ca]: https://caddyserver.com/
[s1]: https://github.com/majewsky/system-configuration/blob/1ca1026a02a1a69324879af23ff379532c6eb1bf/hologram-letsencrypt.pkg.toml#L15-L48
[s2]: https://github.com/majewsky/system-configuration/blob/1ca1026a02a1a69324879af23ff379532c6eb1bf/hologram-letsencrypt.pkg.toml#L80-L120
[s3]: https://github.com/majewsky/system-configuration/blob/1ca1026a02a1a69324879af23ff379532c6eb1bf/hologram-nginx.pkg.toml#L106-L114
[s4]: https://github.com/majewsky/system-configuration/blob/1ca1026a02a1a69324879af23ff379532c6eb1bf/hologram-bethselamin-prosody.pkg.toml#L61-L67
[s5]: https://github.com/majewsky/system-configuration/blob/1ca1026a02a1a69324879af23ff379532c6eb1bf/hologram-letsencrypt.pkg.toml#L122-L148
