title: How-To's
description: Examples and Tutorials
template: doc
uri: docs/howto.html
---

<h2 id="pg-unix-socket">Use a Unix socket for Postgresql <a class="anchor" href="#pg-unix-socket" title="Permalink">¶</a></h2>

If you would like to connect via a Unix socket to Postgresql, set the parameter `host=/path/to/socket/folder`.

Example:

```bash
export DATABASE_URL="user=postgres password=postgres dbname=miniflux2 sslmode=disable host=/path/to/socket/folder"
./miniflux
```

<h2 id="privileged-ports">How to run Miniflux on port 443 or 80 <a class="anchor" href="#privileged-ports" title="Permalink">¶</a></h2>

Ports less than 1024 are reserved for privileged users.
If you have installed Miniflux with the RPM or Debian package, systemd run the process as the `miniflux` user.

To give Miniflux the ability to bind to privileged ports as a non-root user, add the capability `CAP_NET_BIND_SERVICE` to the binary:

```bash
setcap cap_net_bind_service=+ep /usr/bin/miniflux
```

Check that the capability is added:

```bash
getcap /usr/bin/miniflux
/usr/bin/miniflux = cap_net_bind_service+ep
```

<div class="info">
Another way of doing this is to use the Systemd Socket Activation or a reverse-proxy like Nginx.
</div>

<h2 id="reverse-proxy">Reverse-Proxy Configuration <a class="anchor" href="#reverse-proxy" title="Permalink">¶</a></h2>

You can use the reverse-proxy software of your choice, here an example with Nginx:

```bash
server {
    server_name     my.domain.tld;
    listen          80;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

This example assumes that you are running the Miniflux daemon on `127.0.0.1:8080`.

<h2 id="reverse-proxy-subfolder">Reverse-Proxy with a subfolder <a class="anchor" href="#reverse-proxy-subfolder" title="Permalink">¶</a></h2>

Since version 2.0.2, you can host your Miniflux instance under a subfolder.

You must define the environment variable `BASE_URL` for Miniflux, for example:

```bash
export BASE_URL=http://example.org/rss/
```

### Nginx Example

```bash
server {
    server_name     my.domain.tld;
    listen          80;

    location /rss/ {
        proxy_pass http://127.0.0.1:8080/rss/;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Alternative Nginx Configuration

```bash
server {
    server_name     my.domain.tld;
    listen          80;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

This example assumes that you are running the Miniflux daemon on `127.0.0.1:8080`.

Now you can access your Miniflux instance at `http://example.org/rss/`.
In this configuration, cookies are using the path `/rss`.

### Apache 2.4 Example

This configuration assumes the same base-url as the nginx-example.

```
ProxyRequests Off
<Proxy *>
    Order allow,deny
    Allow from all
</Proxy>

<Location "/rss/">
    ProxyPreserveHost On
    ProxyPass http://127.0.0.1:8080/rss/
    ProxyPassReverse http://127.0.0.1:8080/rss/
</Location>
```

Place this snippet inside your vhosts config, needed modules: `mod_proxy` and `mod_proxy_http`.

<h2 id="reverse-proxy-unix-socket">Reverse-Proxy with a Unix socket <a class="anchor" href="#reverse-proxy-unix-socket" title="Permalink">¶</a></h2>

If you prefer to use a Unix socket, change the environment variable `LISTEN_ADDR` to the path of your socket.

Configure Miniflux to use a Unix socket:

```bash
LISTEN_ADDR=/run/miniflux/miniflux.sock
```

The socket folder must be writeable by the miniflux user:

```bash
sudo mkdir /run/miniflux
sudo chown miniflux: /run/miniflux
```

Example with Nginx as reverse-proxy:

```bash
server {
    server_name     my.domain.tld;
    listen          80;

    location / {
        proxy_pass  http://unix:/run/miniflux/miniflux.sock;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

- By default, the socket has the permissions `0666` to make it accessible from other processes like Nginx or others.
- If you don't set the header `X-Forwarded-For`, Miniflux won't be able to determine the remote IP address.
- Listening on Unix socket is available only since Miniflux v2.0.13.

<h2 id="systemd-socket-activation">Systemd Socket Activation <a class="anchor" href="#systemd-socket-activation" title="Permalink">¶</a></h2>

<div class="warning">
This feature is experimental.
</div>

In this example, we are going to expose Miniflux on port 80 via Systemd.

Miniflux will be started by systemd at boot or on-demand.
The process is running under an unprivileged user `miniflux` and we load the environment variables from `/etc/miniflux.conf`.

Create a file `/etc/systemd/system/miniflux.socket`:

    [Unit]
    Description=Miniflux Socket

    [Socket]
    NoDelay=true

    # Listen on port 80.
    # To use a unix socket, define the absolute path here.
    ListenStream=80

    [Install]
    WantedBy=sockets.target

Create a file `/etc/systemd/system/miniflux.service`:

    [Unit]
    Description=Miniflux Service
    Requires=mniflux.socket

    [Service]
    ExecStart=/usr/bin/miniflux
    EnvironmentFile=/etc/miniflux.conf
    User=miniflux
    NonBlocking=true

    [Install]
    WantedBy=multi-user.target

Enable this:

    sudo systemctl enable miniflux.socket
    sudo systemctl enable miniflux.service

Tell systemd to listen on port 80 for us:

    sudo systemctl start miniflux.socket

If you go to `http://127.0.0.1/`, systemd will start the Miniflux service automatically.

If you watch the logs with `journalctl -u miniflux.service`, you will see `[INFO] Listening on systemd socket`.

<div class="info">
<ul>
    <li>Systemd socket activation is available only since Miniflux v2.0.13.</li>
    <li>When using this feature, Miniflux ignores <code>LISTEN_ADDR</code>.</li>
</ul>
</div>

<h2 id="lets-encrypt">Let's Encrypt Configuration <a class="anchor" href="#lets-encrypt" title="Permalink">¶</a></h2>

You could use Let's Encrypt to manage the SSL certificate automatically and activate HTTP/2.

```bash
export CERT_DOMAIN=my.domain.tld
miniflux
```

- Your server must be reachable publicly on port 443 and port 80 (http-01 challenge)
- In this mode, `LISTEN_ADDR` is automatically set to `:https`
- A cache directory is required, by default `/tmp/cert_cache` is used, it could be overrided by using the variable `CERT_CACHE`

<div class="note">
Miniflux supports http-01 challenge since the version 2.0.2.
</div>

<h2 id="https">Manual HTTPS Configuration <a class="anchor" href="#https" title="Permalink">¶</a></h2>

Here an example to generate a self-signed certificate:

```bash
# Generate the private key:
openssl genrsa -out server.key 2048
openssl ecparam -genkey -name secp384r1 -out server.key

# Generate the certificate:
openssl req -new -x509 -sha256 -key server.key -out server.crt -days 3650
```

Start the server like this:

```bash
# Configure the environment variables:
export CERT_FILE=/path/to/server.crt
export KEY_FILE=/path/to/server.key
export LISTEN_ADDR=":https"

# Start the server:
miniflux
```

Then you can access to your server by using an encrypted connection with the HTTP/2 protocol.

<h2 id="oauth2">OAuth2 Authentication <a class="anchor" href="#oauth2" title="Permalink">¶</a></h2>

OAuth2 allows you to sign in with an external provider.
As of now, only Google is supported.

1. Create a new project in Google Console
2. Create a new OAuth2 client
3. Set an authorized redirect URL, for example `https://my.domain.tld/oauth2/google/callback`
4. Define the OAuth2 environment variables and start the process

```bash
export OAUTH2_PROVIDER=google
export OAUTH2_CLIENT_ID=replace_me
export OAUTH2_CLIENT_SECRET=replace_me
export OAUTH2_REDIRECT_URL=https://my.domain.tld/oauth2/google/callback

miniflux
```

Now from the settings page, you can link your existing user to your Google account.

If you would like to authorize anyone to create a user account, you must set `OAUTH2_USER_CREATION=1`.
Since Google do not have the concept of username, the email address is used as username.