---
layout: layouts/home.njk
title: Quickstart
templateClass: home
eleventyNavigation:
  key: Quickstart
  order: 1
---


# Quickstart Guide

You can install the MicroBin executable on its own from [Cargo](#from-cargo), from [AUR](#from-aur), from the [GitHub releases page](https://github.com/szabodanika/microbin/releases) and by [building it yourself](#building-microbin). You can also deploy MicroBin on [Docker](#docker), [Unraid OS](https://forums.unraid.net/topic/112086-support-joshndroids-docker-repo-support-thread/) and services like [Render](https://render.com/deploy?repo=https://github.com/szabodanika/microbin). 

If you obtain the excecutable on its own, you may need to [configure the service](#microbin-as-a-service) yourself.

You can run the service behind a reverse proxy like [NGINX](#nginx-configuration) or [Caddy](#caddy-configuration).

Before installing MicroBin, look at the configuration options in the [Documentation](documentation/#command-line-arguments), or just use the [recommended configuration](documentation/#recommended-configuration).

### From Cargo

Install from Cargo:

`cargo install microbin`

Remember, MicroBin will create your database and file storage wherever you execute it. I recommend that you create a folder for it first and execute it there:

`mkdir ~/microbin/`

`cd ~/microbin/`

`microbin --highlightsyntax --editable`

### From AUR

Install `microbin` package from AUR on any Arch-based Linux distribution and start/enable microbin systemd service. Systemd will start server on  127.0.0.1:8080 with almost all features enabled, but this can be changed in `/etc/microbin.conf`.

### Building MicroBin

Simply clone the repository, build it with `cargo build --release` and run the `microbin` executable in the created `target/release/` directory. It will start listening on 0.0.0.0:8080. You can change the port or bind address with CL arguments `-p (--port)` or `-b (--bind)` respectively . For other arguments see [the Documentation](/documentation).

<pre>
git clone https://github.com/szabodanika/microbin.git
cd microbin
cargo build --release
./target/release/microbin -p 80
</pre>

### Docker

The official automated docker images are available on [Docker Hub at danielszabo99/microbin](https://hub.docker.com/r/danielszabo99/microbin). Alternatively you can also build an image yourself, following the steps below:

<pre>
git clone https://github.com/szabodanika/microbin.git
cd microbin
docker build -t microbin-docker .
</pre>

Then, for `docker compose` you can repurpose the following example in your compose file:

<pre>
services:
  paste:
    image: microbin-docker
    restart: always
    ports:
     - "80:8080"
    volumes:
     - ./microbin-data:/app/pasta_data
</pre>
To pass command line arguments you must edit the Dockerfile and change the CMD line. In this example we add the syntax highlighting option and enable private pastas:

<pre>
CMD ["microbin", "--highlightsyntax", "--private"]
</pre>
You then need to rebuild the image and recreate your container.


**Note:** If you are getting the following error about domain name resolution:

<pre>
warning: spurious network error (2 tries remaining): failed to resolve address for github.com: Temporary failure in name resolution; class=Net (12)
warning: spurious network error (1 tries remaining): failed to resolve address for github.com: Temporary failure in name resolution; class=Net (12)
</pre>

You might need to run `docker build` with the `--network` option:

<pre>
docker build --network host -t microbin-docker .
</pre>

### MicroBin as a service

To install it as a service on your Linux machine, create a file called `/etc/systemd/system/microbin.service`, paste this into it with the `[username]` and `[path to installation directory]` replaced with the actual values. If you installed MicroBin from cargo, your executable will be in your cargo directory, e.g. `/Users/daniel/.cargo/bin/microbin`.

<pre>
[Unit]
Description=MicroBin
After=network.target

[Service]
Type=simple
Restart=always
User=[username]
RootDirectory=/
WorkingDirectory=[path to installation directory]
ExecStart=[path to installation directory]/target/release/microbin

[Install]
WantedBy=multi-user.target
</pre>

Here is my `microbin.service` for example, with some optional arguments:


<pre>
[Unit]
Description=MicroBin
After=network.target

[Service]
Type=simple
Restart=always
User=ubuntu
RootDirectory=/

# This is the directory where I want to run microbin. It will store all the pastas here.
WorkingDirectory=/home/ubuntu/server/microbin

# This is the location of my executable - I also have 2 optional features enabled
ExecStart=/home/ubuntu/server/microbin/target/release/microbin --editable --highlightsyntax

# I keep my installation in the home directory, so I need to add this
ProtectHome=off

[Install]
WantedBy=multi-user.target
</pre>

Then start the service with `systemctl start microbin` and enable it on boot with `systemctl enable microbin`. To update your MicroBin, simply update or clone the repository again, build it again, and then restart the service with `systemctl restart microbin`. An update will never affect your existing pastas, unless there is a breaking change in the data model (in which case MicroBin just won't be able to import your DB), which will always be mentioned explicitly.


### NGINX configuration

<pre>
server {
	# I have HTTPS enabled using certbot - you can use HTTP of course if you want!
  listen 443 ssl; # managed by Certbot

	server_name	microbin.myserver.com;

	location / {
			# Make sure to change the port if you are not running MicroBin at 8080!
    	proxy_pass	        http://127.0.0.1:8080$request_uri;
	    proxy_set_header	Host $host;
	    proxy_set_header	X-Forwarded-Proto $scheme;
	    proxy_set_header    X-Real-IP $remote_addr;
	    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
	}

	# Limit content size - I have 1GB because my MicroBin server is private, no one else will use it.
	client_max_body_size 1024M;

  ssl_certificate /etc/letsencrypt/live/microbin.myserver.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/microbin.myserver.com/privkey.pem; # managed by Certbot
  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
</pre>

#### Caddy configuration

An example of running MicroBin behind the reverse proxy `caddy` on the path `/paste` (using systemd)

**microbin.service**

<pre>
[Unit]
Description=Micobin Paste

[Service]
Type=simple
User=microbin
# Path to your binary
ExecStart=/home/microbin/microbin/target/release/microbin
# Set your desired working directory, eg where microbin places its data
WorkingDirectory=/home/pi/bin/micro

# bind to localhost, as its exposed via Caddy
Environment=MICROBIN_BIND=127.0.0.1
# bind to a unused port
Environment=MICROBIN_PORT=31333
# All your other config (change and add as needed)
Environment=MICROBIN_EDITABLE=true
Environment=MICROBIN_HIGHLIGHTSYNTAX=true
Environment=MICROBIN_THREADS=2
Environment=MICROBIN_FOOTER_TEXT="Bin the bytes"
# Set your **public** url. Eg the one you will later use to access microbin
# Ensure it either starts with http(s):// or with a /
Environment=MICROBIN_PUBLIC_PATH="http://100.127.233.32/paste"

[Install]
WantedBy=multi-user.target
</pre>

**Caddyfile**

<pre>
example.com {
    # Your normal http root
    root * /var/www/html
    file_server
    
    # Route all requests to past
    handle_path /paste/* {
        reverse_proxy http://127.0.0.1:31333
    }
}
</pre>

</pre>