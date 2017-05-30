# Docker + Nginx + Let's Encrypt

This docker-compose file provides a way for multiple websites to run behind a dockerized Nginx reverse proxy and served via HTTPS using free [Let's Encrypt](https://letsencrypt.org) certificates. New sites can be added automatically by running new Docker containers with a couple of environment variables set. When the new containers start, the proxy's Nginx config is automatically updated and certificates (if needed) are automatically acquired; no manual steps necessary.

This is derived from <https://gilyes.com/docker-nginx-letsencrypt>.


## Running the example
### Prerequisites
* [docker](https://docs.docker.com/engine/installation/) (>= 1.10)
* [docker-compose](https://github.com/docker/compose/releases) (>= 1.8.1)
* access to (sub)domain(s) pointing to a publicly accessible server (required for TLS)

### Installation
* Clone the [repository](https://github.com/brentkearney/docker-nginx-letsencrypt-sample.git) on your server.
* Create the directory ./volumes/certs
* Update or remove the DEFAULT_HOST from docker-compose.yml
* Add these environment variables to the containers that you want proxied:
  * VIRTUAL_HOST=fqdn.yourdomain.com
  * VIRTUAL_NETWORK=nginx-proxy
  * VIRTUAL_PORT=80
* Add these environment variables to the containers that you want automatic, free SSL certificates for:
  * LETSENCRYPT_HOST=fqdn.yourdomain.com (use a comma-separated list for multiple domains per certificate)
  * LETSENCRYPT_EMAIL=your@email.com (the email address you want to be associated with the certificates)


### Running
In the main directory run:
```
docker-compose up -d
docker-compose logs
```

This will perform the following steps:

* Download the required images from Docker Hub ([nginx](https://hub.docker.com/_/nginx/), [docker-gen](https://hub.docker.com/r/jwilder/docker-gen/), [docker-letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/)).
* Create containers from them.
* Start up the containers.
  * *docker-letsencrypt-nginx-proxy-companion* inspects containers' metadata and tries to acquire certificates as needed (if successful then saving them in a volume shared with the host and the Nginx container).
  * *docker-gen* also inspects containers' metadata and generates the configuration file for the main Nginx reverse proxy

Next, start your other containers with the environment variables specified above (VIRTUAL_HOST, VIRTUAL_NETWORK, VIRTUAL_PORT, LETSENCRYPT_HOST, LETSENCRYPT_EMAIL).

If everything went well then you should now be able to access your website(s).

### Troubleshooting
* To view logs run `docker-compose logs`.
* To view the generated Nginx configuration run `docker exec -ti nginx cat /etc/nginx/conf.d/default.conf`

## How It Works

The system consists of 4 main parts:

* Main Nginx reverse proxy container.
* Container that generates the main Nginx config based on container metadata.
* Container that automatically handles the acquisition and renewal of Let's Encrypt TLS certificates.
* The actual websites living in their own containers.

### The main Nginx reverse proxy container
This is the only publicly exposed container, routes traffic to the backend servers and provides TLS termination.

Uses the official [nginx](https://hub.docker.com/_/nginx/) Docker image.

It is defined in `docker-compose.yml` under the **nginx-proxy** service block:

```
services:
  nginx-proxy:
    restart: always
    image: nginx
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/etc/nginx/conf.d"
      - "/etc/nginx/vhost.d"
      - "/usr/share/nginx/html"
      - "./volumes/proxy/certs:/etc/nginx/certs:ro"
      - "./volumes/vhost.d:/etc/nginx/vhost.d"
```

As you can see it shares a few volumes:
* Configuration folder: used by the container that generates the configuration file.
* Default Nginx root folder: used by the Let's Encrypt container for challenges from the CA.
* Certificates folder: written to by the Let's Encrypt container, this is where the TLS certificates are maintained.
* Virtual hosts folder: if you have site-specific nginx configurations, e.g. to enable CORS, add a file with the same name as the FQDN of the service, followed by "_location". For example, "service.example.com_location". You can add nginx configuration directives in that file. For example, to add CORS support, the file would contain:
```
  set $cors '';
  if ($http_origin ~* 'https?://(localhost|someothersite.com|.+\.yetanothersite.com|.+\.andanother\.org)') {
     set $cors 'true';
  }

  if ($cors = 'true') {
     add_header 'Access-Control-Allow-Origin' "$http_origin";
     add_header 'Access-Control-Allow-Credentials' 'true';
     add_header 'Access-Control-Allow-Methods' 'GET';
     add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With';
  }
```

### The configuration generator container
This container inspects the other running containers and based on their metadata (like **VIRTUAL_HOST** environment variable) and a template file it generates the Nginx configuration file for the main Nginx container. When a new container is spinning up this container detects that, generates the appropriate configuration entries and restarts Nginx.

Uses the [jwilder/docker-gen](https://hub.docker.com/r/jwilder/docker-gen/) Docker image.

It is defined in `docker-compose.yml` under the **nginx-gen** service block:

```
services:
  ...

  nginx-gen:
    restart: always
    image: jwilder/docker-gen
    container_name: nginx-gen
    volumes:
      - "/var/run/docker.sock:/tmp/docker.sock:ro"
      - "./volumes/proxy/templates/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro"
    volumes_from:
      - nginx-proxy
    entrypoint: /usr/local/bin/docker-gen -notify-sighup nginx-proxy -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

The container reads the `nginx.tmpl` template file (source: [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy)) via a volume shared with the host.

It also mounts the Docker socket into the container in order to be able to inspect the other containers (the `"/var/run/docker.sock:/tmp/docker.sock:ro"` line).
**Security warning**: mounting the Docker socket is usually discouraged because the container getting (even read-only) access to it can get root access to the host. In our case, this container is not exposed to the world so if you trust the code running inside it the risks are probably fairly low. But definitely something to take into account. See e.g. [The Dangers of Docker.sock](https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/) for further details.

NOTE: it would be preferrable to have docker-gen only handle containers with exposed ports (via `-only-exposed` flag in the `entrypoint` script above) but currently that does not work, see e.g. <https://github.com/jwilder/nginx-proxy/issues/438>.

### The Let's Encrypt container
This container also inspects the other containers and acquires Let's Encrypt TLS certificates based on the **LETSENCRYPT_HOST** and **LETSENCRYPT_EMAIL** environment variables. At 1-minute intervals it checks and renews certificates as needed.

Uses the [jrcs/letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/) Docker image.

It is defined in `docker-compose.yml` under the **letsencrypt-nginx-proxy-companion** service block:

```
services:
  ...

  letsencrypt-nginx-proxy-companion:
    restart: always
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: letsencrypt-nginx-proxy-companion
    volumes_from:
      - nginx-proxy
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./volumes/proxy/certs:/etc/nginx/certs:rw"
    environment:
      - NGINX_DOCKER_GEN_CONTAINER=nginx-gen
```

The container uses a volume shared with the host and the Nginx container to maintain the certificates.

It also mounts the Docker socket in order to inspect the other containers. See the security warning above in the docker-gen section about the risks of that.

### Example websites
These two very simple examples are running in their own respective containers. They are defined in their own `docker-compose.yml` file:

```
services:
  ...

  sample-api:
    restart: always
    image: sample-api
    build: ./samples/api
    container_name: sample-api
    environment:
      - VIRTUAL_HOST=sampleapi.example.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=3000
      - LETSENCRYPT_HOST=sampleapi.example.com
      - LETSENCRYPT_EMAIL=email@example.com
    networks:
      - nginx-proxy

  sample-website:
    restart: always
    image: sample-website
    build: ./samples/website
    container_name: sample-website
    volumes:
      - "./volumes/nginx-sample-website/conf.d/:/etc/nginx/conf.d"
      - "./volumes/config/sample-website/config.js:/usr/share/nginx/html/config.js"
    environment:
      - VIRTUAL_HOST=samplewebsite.example.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=80
      - LETSENCRYPT_HOST=sample.example.com
      - LETSENCRYPT_EMAIL=email@example.com
    networks:
      - nginx-proxy

    networks:
      nginx-proxy:
        external: true
```
The important part here are the environment variables. These are used by the config generator and certificate maintainer containers to set up the system.


## Conclusion
This can be a fairly simple way to have easy, reproducible deploys for websites with free, auto-renewing TLS certificates.

