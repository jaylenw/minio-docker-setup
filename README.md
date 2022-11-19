# My Nextcloud Docker Setup

**Goal**: Support a Docker Compose environment for running a single Minio node with data stored to an NFS. 

This project was inpired and references [Minio's Quickstart for Single-Node Single-Drive Container](https://min.io/docs/minio/container/index.html)
and [Minio's Deploy Docker Compose README.md](https://github.com/minio/minio/blob/master/docs/orchestration/docker-compose/README.md).

## System Requirements

**Docker Engine**: ^20.10.18

**Docker Compose**: ^2.10.2

You will also need an NFS server configured to support two environments, one for development testing and another for production use if you choose to use an NFS as
your  data storage for Minio.

## Getting Started

1.) Make sure you have [Docker Engine](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) installed.

2.) Clone this repository.

3.) Copy the `.minio-env.example` file to have `.minio-env` file in your directory. Proceed to set the appropriate values in the file.

4.) Copy both the `docker-compose.nfs.dev.yml.example` and `docker-compose.nfs.prod.yml.example` files to have `docker-compose.nfs.dev.yml` and `docker-compose.nfs.prod.yml` files in your directory.

### Local Testing

To get started with this, it is best that you are able to get Nextcloud running locally first before deploying it to your deployment environment (ex. your server).
We will first get Nextcloud working without NFS to make sure all is well.

1.) Run `docker compose pull`.

2.) Run `docker compose up` and after the initialization, open your browser to `http://localhost:9090/` to access the Minio console.

Congrats, we've successfully got Minio running with Docker Compose.

### Local Testing with a Dev NFS

**Caution:** When modifying the Docker volumes settings with Docker Compose, you may have to run `docker compose down -v` for the new
settings to take place. This will delete EVERYTHING in your volumes if you have something in there (ie. Minio data). Please be
careful running the command, make sure you know what you are doing, and take backups!

1.) Make the necessary changes to the `docker-compose.nfs.dev.yml` file for `device` and `o` under `driver_opts`.

2.) Run `docker compose -f docker-compose.yml -f docker-compose.nfs.dev.yml up`.

3.) Refer to the applicable information in the section above.

### Running in production with a Prod NFS

1.) Run `docker compose -f docker-compose.yml -f docker-compose.nfs.prod.yml up` after making applicable changes to the `docker-compose.nfs.prod.yml` file.

2.) Configure NGINX to forward traffic to your Nextcloud application. I strongly suggest having a proxy server in front of your Minio docker environment.

Below is an example of a reverse-proxy configuration to get you started using [NGNIX](https://www.nginx.com/).
*PLEASE CONFIGURE YOUR DEPLOYMENT WITH SSL. You can do so with [Certbot](https://certbot.eff.org/). It will take the configuration below
and modify it to support https.*

```
# configuration for nextcloud production environment
upstream minio-prod-backend-server {
    # the minio application in minio container
    server <localhost or whatever IP>:9000; # ip and use the port specified in the docker-compose.yml
    keepalive 64;
}

upstream minio-prod-backend-console {
    # the minio application in minio container
    server <localhost or whatever IP>:9090; # ip and use the port specified in the docker-compose.yml
    keepalive 64;
}

server {
    # the virtual host name of this
    listen 80;
    server_name <your-domain-or-subdomain>; #api.minio.example.com

    location / {
        proxy_pass http://minio-prod-backend-server;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        # HSTS policy, tell the browser that this domain must only be accessed using HTTPS, expire every 180 days
        add_header Strict-Transport-Security "max-age=15552000";
    }

}

server {
    # the virtual host name of this
    listen 80;
    server_name <your-domain-or-subdomain>; #minio.example.com

    location / {
        proxy_pass http://minio-prod-backend-console;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Ningx-Proxy true;
        proxy_connect_timeout 300;

        # To support websocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
            
        chunked_transfer_encoding off;
        
        # HSTS policy, tell the browser that this domain must only be accessed using HTTPS, expire every 180 days
        add_header Strict-Transport-Security "max-age=15552000";
    }

}
```
3.) Access your application in the browser and check that all is well.

4.) Once you have confirmed that all is well, you can Ctrl-C Docker Compose and restart it with `docker compose -f docker-compose.yml -f docker-compose.nfs.prod.yml up -d` to have Minio running the background on start on server boot up for example.

----------------------------------------------------------------------------------------------------------
Made with ♥ in Los Angeles CA.