# Misago image

This is a repository to create a misago container image for use in unsupported applications not covered by the misago officially supported installation scheme

Refer to [misago-docker](https://github.com/rafalp/misago_docker) for the official repository and installation

# customizing the installation

The use of .env variables to customze the container is possible using the following;

## misago settings

## if not using behind a reverse proxy to provide certificates
LETSENCRYPT_EMAIL=<certificate@email>
LETSENCRYPT_HOST=<certificate_hostname>

## misago configuration settings
MISAGO_ADDRESS=https://myinstance.domain.com/
MISAGO_DAILY_BACKUP=no
MISAGO_DEBUG=yes
MISAGO_DEFAULT_FROM_EMAIL=Forum email <forum@domain.com>
MISAGO_EMAIL_HOST=<smtp host>
MISAGO_EMAIL_PASSWORD=<smtp password>
MISAGO_EMAIL_PORT=587
MISAGO_EMAIL_PROVIDER=smtp
MISAGO_EMAIL_USER=<smtp user>
MISAGO_EMAIL_USE_SSL=yes
MISAGO_EMAIL_USE_TLS=no
MISAGO_LANGUAGE_CODE=en-gb
MISAGO_SEARCH_CONFIG=english
MISAGO_SECRET_KEY=<misago secret>
MISAGO_TIME_ZONE=Europe/London
VIRTUAL_HOST=myinstance.domain.com

## database settings
POSTGRES_USER="postgresql_user" \
POSTGRES_PASSWORD="postgresql_user_password" \
POSTGRES_DB="postgresql_database" \
POSTGRES_HOST="postgresql_host" \

# Required containers for deployment

There are three containers required for a working deployment, you should note that a proxy to route traffic, and supply certificates is also required for a public facing service.

## service containers

### Create the redis container

redis is required to provide cache

```bash
docker run -d
  --name=redis --label io.containers.autoupdate=local \
  -v [app]/myinstance/redis:/data \
  localhost/redis:latest \
  redis-server  --port 6379 --save 600 1 --loglevel warning
```

### Create the misago container

```bash
docker run -d --name=myinstance-misago \
    -v [app]/misago/media:/misago/media \
    -v [app]/misago/static:/misago/static \
    -v [app]/misago/avatargallery:/misago/avatargallery \
    -v [app]/misago/theme:/misago/theme \
    -v [app]/logs/misago:/misago/logs \
    -e MISAGO_ADDRESS=https://[your.domain.com]/ \
    -e MISAGO_DAILY_BACKUP=no \
    -e MISAGO_DEBUG=no \
    -e MISAGO_DEFAULT_FROM_EMAIL='Your Forum <youradmin@yourmail.domain>' \
    -e MISAGO_EMAIL_PROVIDER=smtp \
    -e MISAGO_EMAIL_USE_TLS=yes \
    -e MISAGO_EMAIL_USE_SSL=no \
    -e MISAGO_EMAIL_HOST=[yourmail_server] \
    -e MISAGO_EMAIL_PASSWORD=[your_email_server.password] \
    -e MISAGO_EMAIL_USER=[your_email_server.login] \
    -e MISAGO_EMAIL_PORT=587 \
    -e MISAGO_LANGUAGE_CODE=en-gb \
    -e MISAGO_TIME_ZONE=Europe/London \
    -e MISAGO_SEARCH_CONFIG=english \
    -e MISAGO_SECRET_KEY=[your_misago_secret] \
    -e CACHE_REDIS_URL="redis://myinstance-redis/1" \
    -e CELERY_BROKER_URL="redis://myinstance-redis/0" \
    -e POSTGRES_USER=[postgres_user] \
    -e POSTGRES_PASSWORD=[postgres_password] \
    -e POSTGRES_DB=[postgres_database] \
    -e POSTGRES_HOST=[postgres_host] \
    -e VIRTUAL_HOST=[your.domain.com] \
    -e VIRTUAL_PROTO=uwsgi \
    localhost/misago:latest
```

### Create the celery container

```bash
docker run -d --name=myinstance-celery \
    -v [app]/misago/media:/misago/media \
    -v [app]/misago/static:/misago/static \
    -v [app]/misago/theme:/misago/theme \
    -v [app]/logs/misago:/misago/logs \
    -e MISAGO_ADDRESS=https://[your.domain.com]/ \
    -e MISAGO_DAILY_BACKUP=no \
    -e MISAGO_DEBUG=no \
    -e MISAGO_DEFAULT_FROM_EMAIL='Your Forum <youradmin@yourmail.domain>' \
    -e MISAGO_EMAIL_PROVIDER=smtp \
    -e MISAGO_EMAIL_USE_TLS=yes \
    -e MISAGO_EMAIL_USE_SSL=no \
    -e MISAGO_EMAIL_HOST=[yourmail_server] \
    -e MISAGO_EMAIL_PASSWORD=[your_email_server.password] \
    -e MISAGO_EMAIL_USER=[your_email_server.login] \
    -e MISAGO_EMAIL_PORT=587 \
    -e MISAGO_LANGUAGE_CODE=en-gb \
    -e MISAGO_TIME_ZONE=Europe/London \
    -e MISAGO_SEARCH_CONFIG=english \
    -e MISAGO_SECRET_KEY=[your_misago_secret] \
    -e CACHE_REDIS_URL="redis://myinstance-redis/1" \
    -e CELERY_BROKER_URL="redis://myinstance-redis/0" \
    -e POSTGRES_USER=[postgres_user] \
    -e POSTGRES_PASSWORD=[postgres_password] \
    -e POSTGRES_DB=[postgres_database] \
    -e POSTGRES_HOST=[postgres_host] \
    -e VIRTUAL_HOST=[your.domain.com] \
    -e VIRTUAL_PROTO=uwsgi \
    celery -A misagodocker worker --loglevel=info
```

## Finalise the Installation

### Essential migration/setup

use docker exec to access the misago container, and setup the application.

```bash
docker exec -it myinstance-misago bash
```

initialise the database

```bash
./.run initialize_default_database
```

add the superuser credentials when prompted

SUPERUSER_USERNAME=[admin] 
SUPERUSER_EMAIL=[admin_email] 
SUPERUSER_PASSWORD=[admin_password] 

### Complete

You now should have a working misago installation, which you can view successfully via your reverse proxy. Configuration of that is out of scope here, but you should configure it to produce certificates for the site, forward to https, and pass on the X-Forwarded headers.


## Optional Setup

these commands can be accessed by exec'ing into bash in the container

```bash
docker exec -it myinstance-misago bash
```

### email test

You can check the site email with a test

```
python manage.py sendtestemail test@email.address
```

### maintennance tasks

Perform regular maintennance tasks manually

```bash
./cron
```

You should see an output similar to this:

```
Deleted users: 0


Categories were pruned

Building active posters ranking...
Finished after 0.05s


No unused attachments were cleared


No expired entries were found
Bans invalidated: 0
Ban caches emptied: 0
IP addresses older than 90 days have been removed.
Automatic deletion of inactive user accounts is currently disabled.
Data downloads expired: 0
Data downloads prepared: 0
```

you can also automate this directly without exec'ing bash in the container. This allows you to create a system level `cron` job, or a `systemctl timer` to automate maintennance.

```bash
docker exec -it myinstance-misago ./cron
```

### additional maintenance options

these scripts are the standard scripts that the wizard uses in the dockler-compose install, and are mentioned here as they are accessible through bash in the container.

#### to migrate database

```bash
python manage.py migrate
```

#### load avatar gallery

```bash
python manage.py loadavatargallery
```

#### to setup admin account

```bash
python manage.py createsuperuser
```

#### to collect static assets after that you will have to run

```bash
python manage.py collectstatic
```

#### something something versioned caches

```bash
python manage.py invalidateversionedcaches 
```

## Note on backups

To backup an instance there are two aspects that we need to consider. One is the database, and one the static files. 

The database should be achieved using a dump of the database. [borgmatic](https://torsion.org/borgmatic/) has database hooks that can automate this as part of the backup process. Depending on the nature of your postgresql provision, there are various options for that, which are not in scope here.

The static files should be backed up when there is no chance of a conflicting write during the process. It may be possible to snapshot the storage, and thus perform an uninterrupted backup. This will depend on the storage provision on your individual setup. If this is not possible, then it is best to temporarily stop the service, backup, and then restart the service. 


# Example: Install misago in a podman pod under linux, with external database, behind reverse proxy

---

This is provided as an example, on the off-chance it might be useful to someone, and it does not claim to be authoratative. No warranty is implied or given. Use at your own risk. This may not be the best way to do this, and it may have unforseen failings.

requirements: podman, git

The use case here is to install an opinionated production ready misago in a podman pod, with an external postgresql database (ie in another pod, or externally provisioned - one can also add a postgresql container to this pod), and behind a reverse-proxy that handles certificates and access authentication. In my case this is caddy2, but it should work with the usual suspects. Backup, in my case, is handled by the excellent [borgmatic](https://torsion.org/borgmatic/) which dumps the database and de-duplicate and backups that and the static files. So you should note that this guide does not address backup.  

### Define the image to use

Tag this image as the latest image. This will allow us the facility to build future versions and tag as latest the image that we wish our deployment to use, as well as the option for automatic updates controlled by podman.

```bash
podman image tag localhost/misago:0.29 localhost/misago:latest
```

## Preparing the installation

Choose the preferred location for your containers persistent files. I tend to use `/var/lib/misago` for systems level installations, but you don't have to. You may prefer something in the user home directory if you are running the containers rootless (you are on your own there, I'm keeping this simple by way of example). Let's call this location `[app]`

### create directory structure

This is the file structure your containers will need to persist values.

```bash
mkdir -p [app]/{logs,misago,nginx,redis}
mkdir -p [app]/nginx/vhost.d
mkdir -p [app]/logs/{nginx,misago,celery}
mkdir -p [app]/misago/{avatargallery,media,static,theme,userdata}
mkdir -p [app]/misago/theme/{static,template}
```

### Add the nginx configuration

Into the [app]/nginx directory put the custom `nginx.conf` to serve misago to the reverse proxy of your choice. In the server block, this needs to reflect the pod name that you create for the install. In this case `myinstance` (see `server_name myinstance; # substitute your machine's IP address or FQDN`). Note that the location alias of /media and /static refers to within the container, and does not require changing.

```
# nginx.conf

events {}

http {

    include /etc/nginx/mime.types;

    upstream misago {
    # server unix:///path/to/your/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:3031; # for a web port socket (we'll use this first)
    }

    server {
        # the port your site will be served on
        listen      80;
        # the domain name it will serve for
        server_name myinstance; # substitute your machine's IP address or FQDN
        charset     utf-8;

        # max upload size
        client_max_body_size 75M;   # adjust to taste

        # Django media
        location /media  {
            alias /misago/media;  # your Django project's media files - amend as required

            uwsgi_param Host $host;
            uwsgi_param X-Real-IP $remote_addr;
            uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
            uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;

        }

        location /static {
            alias /misago/static; # your Django project's static files - amend as required

            uwsgi_param Host $host;
            uwsgi_param X-Real-IP $remote_addr;
            uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
            uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;

        }

        # Finally, send all non-media requests to the Django server.
        location / {
            uwsgi_pass  misago;
            include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed

            uwsgi_param Host $host;
            uwsgi_param X-Real-IP $remote_addr;
            uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
            uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;

        }
    }
}
```

create [your.domain.com]_location in the nginx/vhost.d directrory

```
# your.domain.com_location

# Additional Nginx configuration for Misago vhost

# Set max upload size at 16 megabytes
client_max_body_size 16M;

# Enable GZIP
gzip  on;
gzip_http_version 1.0;
gzip_comp_level 2;
gzip_min_length 1100;
gzip_buffers     4 8k;
gzip_proxied any;
gzip_types
    # text/html is always compressed by HttpGzipModule
    text/css
    text/javascript
    text/xml
    text/plain
    text/x-component
    application/javascript
    application/json
    application/xml
    application/rss+xml
    font/truetype
    font/opentype
    application/vnd.ms-fontobject
    image/svg+xml;

location /static/ {
    root /misago;
    expires max;

    # Enable GZIP
    gzip  on;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_min_length 1100;
    gzip_buffers     4 8k;
    gzip_types
        # text/html is always compressed by HttpGzipModule
        text/css
        text/javascript
        text/xml
        text/plain
        text/x-component
        application/javascript
        application/json
        application/xml
        application/rss+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject
        image/svg+xml;
}

location /media/ {
    root /misago;
    expires max;

    # Enable GZIP
    gzip  on;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_min_length 1100;
    gzip_buffers     4 8k;
    gzip_types
        # text/html is always compressed by HttpGzipModule
        text/css
        text/javascript
        text/xml
        text/plain
        text/x-component
        application/javascript
        application/json
        application/xml
        application/rss+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject
        image/svg+xml;
```

## Deploying the service

Modify the volume mounts, and the env variables, (along with any pod/container name changes) below to suit your own installation. The label `--label io.containers.autoupdate=local` is included to allow for podman autoupdate of images. Omit if not required. All images not built are pulled from docker hub and labelled locally (not covered, but eg `podman pull nginx:1.23.3-alpine-slim` - then use `podman image tag nginx:1.23.3-alpine-slim localhost/nginx:latest` to label for locally upgradeable images).  

### Create the pod

```bash
podman pod create --name myinstance
```

In the following containers, the label `--pod=myinstance` ensures that they are created in this pod.

### Create the redis container

```bash
podman run \
  -d --pod=myinstance \
  --name=myinstance-redis --label io.containers.autoupdate=local \
  -v [app]/myinstance/redis:/data \
  localhost/redis:latest \
  redis-server  --port 6379 --save 600 1 --loglevel warning
```

### Create the misago container

```bash
podman run -d --pod=myinstance --name=myinstance-misago \
    -v [app]/misago/media:/misago/media \
    -v [app]/misago/static:/misago/static \
    -v [app]/misago/avatargallery:/misago/avatargallery \
    -v [app]/misago/theme:/misago/theme \
    -v [app]/logs/misago:/misago/logs \
    -e MISAGO_ADDRESS=https://[your.domain.com]/ \
    -e MISAGO_DAILY_BACKUP=no \
    -e MISAGO_DEBUG=no \
    -e MISAGO_DEFAULT_FROM_EMAIL='Your Forum <youradmin@yourmail.domain>' \
    -e MISAGO_EMAIL_PROVIDER=smtp \
    -e MISAGO_EMAIL_USE_TLS=yes \
    -e MISAGO_EMAIL_USE_SSL=no \
    -e MISAGO_EMAIL_HOST=[yourmail_server] \
    -e MISAGO_EMAIL_PASSWORD=[your_email_server.password] \
    -e MISAGO_EMAIL_USER=[your_email_server.login] \
    -e MISAGO_EMAIL_PORT=587 \
    -e MISAGO_LANGUAGE_CODE=en-gb \
    -e MISAGO_TIME_ZONE=Europe/London \
    -e MISAGO_SEARCH_CONFIG=english \
    -e MISAGO_SECRET_KEY=[your_misago_secret] \
    -e CACHE_REDIS_URL="redis://myinstance-redis/1" \
    -e CELERY_BROKER_URL="redis://myinstance-redis/0" \
    -e POSTGRES_USER=[postgres_user] \
    -e POSTGRES_PASSWORD=[postgres_password] \
    -e POSTGRES_DB=[postgres_database] \
    -e POSTGRES_HOST=[postgres_host] \
    -e VIRTUAL_HOST=[your.domain.com] \
    -e VIRTUAL_PROTO=uwsgi \
    --label io.containers.autoupdate=local localhost/misago:latest
```

### Create the celery container

```bash
podman run -d --pod=myinstance --name=myinstance-celery \
    -v [app]/misago/media:/misago/media \
    -v [app]/misago/static:/misago/static \
    -v [app]/misago/theme:/misago/theme \
    -v [app]/logs/misago:/misago/logs \
    -e MISAGO_ADDRESS=https://[your.domain.com]/ \
    -e MISAGO_DAILY_BACKUP=no \
    -e MISAGO_DEBUG=no \
    -e MISAGO_DEFAULT_FROM_EMAIL='Your Forum <youradmin@yourmail.domain>' \
    -e MISAGO_EMAIL_PROVIDER=smtp \
    -e MISAGO_EMAIL_USE_TLS=yes \
    -e MISAGO_EMAIL_USE_SSL=no \
    -e MISAGO_EMAIL_HOST=[yourmail_server] \
    -e MISAGO_EMAIL_PASSWORD=[your_email_server.password] \
    -e MISAGO_EMAIL_USER=[your_email_server.login] \
    -e MISAGO_EMAIL_PORT=587 \
    -e MISAGO_LANGUAGE_CODE=en-gb \
    -e MISAGO_TIME_ZONE=Europe/London \
    -e MISAGO_SEARCH_CONFIG=english \
    -e MISAGO_SECRET_KEY=[your_misago_secret] \
    -e CACHE_REDIS_URL="redis://myinstance-redis/1" \
    -e CELERY_BROKER_URL="redis://myinstance-redis/0" \
    -e POSTGRES_USER=[postgres_user] \
    -e POSTGRES_PASSWORD=[postgres_password] \
    -e POSTGRES_DB=[postgres_database] \
    -e POSTGRES_HOST=[postgres_host] \
    -e VIRTUAL_HOST=[your.domain.com] \
    -e VIRTUAL_PROTO=uwsgi \
    --label io.containers.autoupdate=local localhost/misago:latest  celery -A misagodocker worker --loglevel=info
```

### Create the nginx container

```bash
podman run \
  -d --pod=myinstance \
  --name=myinstance-nginx --label io.containers.autoupdate=local \
  -e ENABLE_IPV6=true \
  -v [app]/misago/media:/misago/media \
  -v [app]/misago/static:/misago/static \
  -v [app]/logs/nginx:/var/log/nginx \
  -v [app]/nginx/nginx.conf:/etc/nginx/nginx.conf \
  -v [app]/nginx/vhost.d:/etc/nginx/vhost.d \
  localhost/nginx:latest
```

## Finalise the Installation

### Essential migration/setup

use podman exec to access the misago container, and setup the application.

```bash
podman exec -it myinstance-misago bash
```

initialise the database

```bash
./.run initialize_default_database
```

add the superuser credentials when prompted

SUPERUSER_USERNAME=[admin] 
SUPERUSER_EMAIL=[admin_email] 
SUPERUSER_PASSWORD=[admin_password] 

### Complete

You now should have a working misago installation, which you can view successfully via your reverse proxy. Configuration of that is out of scope here, but you should configure it to produce certificates for the site, forward to https, and pass on the X-Forwarded headers.

In caddy that is as simple as adding to the caddyfile:

```
your.domain.com {
        reverse_proxy myinstance:80
}
```

## Optional Setup

these commands can be accessed by exec'ing into bash in the container

```bash
podman exec -it myinstance-misago bash
```

### email test

You can check the site email with a test

```
python manage.py sendtestemail test@email.address
```

### maintennance tasks

Perform regular maintennance tasks manually

```bash
./cron
```

You should see an output similar to this:

```
Deleted users: 0


Categories were pruned

Building active posters ranking...
Finished after 0.05s


No unused attachments were cleared


No expired entries were found
Bans invalidated: 0
Ban caches emptied: 0
IP addresses older than 90 days have been removed.
Automatic deletion of inactive user accounts is currently disabled.
Data downloads expired: 0
Data downloads prepared: 0
```

you can also automate this directly without exec'ing bash in the container. This allows you to create a system level `cron` job, or a `systemctl timer` to automate maintennance.

```bash
podman exec -it myinstance-misago ./cron
```

### additional maintenance options

these scripts are the standard scripts that the wizard uses in the dockler-compose install, and are mentioned here as they are accessible through bash in the container.

#### to migrate database

```bash
python manage.py migrate
```

#### load avatar gallery

```bash
python manage.py loadavatargallery
```

#### to setup admin account

```bash
python manage.py createsuperuser
```

#### to collect static assets after that you will have to run

```bash
python manage.py collectstatic
```

#### something something versioned caches

```bash
python manage.py invalidateversionedcaches 
```

## Using systemctl to control the pod/containers

The containers created will not restart if stopped, nor on reboot. They will have to be manually started and stopped. Which is fine while testing. Once you are happy with your installation you may wish to choose automating the handling of the service using `systemctl` (you may not, but you are on your own with that).

### create systemctl files

...in a scratch directory (or in your backup path), create the necessary systemd files.

```bash
podman generate systemd --files --new --name myinstance
```

this should output something like:

```
[working_directory]/container-myinstance-nginx.service
[working_directory]/pod-myinstance.service
[working_directory]/container-myinstance-misago.service
[working_directory]/container-myinstance-celery.service
[working_directory]/container-redis.service
```

In order to work with misago the redis service is just called 'redis' within the pod, but to avoid overwriting any other container services just named 'redis' in our systemd folder, we need to rename the systemd file.

```bash
mv container-redis.service container-myinstance-redis.service
```

then we need to update the configurations to refer to myinstance-redis instead of redis.

in `pod-myinstance.service` change

```
Wants=container-redis.service container-myinstance-celery.service container-myinstance-misago.service container-myinstance-nginx.service
Before=container-redis.service container-myinstance-celery.service container-myinstance-misago.service container-myinstance-nginx.service
```

to be

```
Wants=container-myinstance-redis.service container-myinstance-celery.service container-myinstance-misago.service container-myinstance-nginx.service
Before=container-myinstance-redis.service container-myinstance-celery.service container-myinstance-misago.service container-myinstance-nginx.service
```

Then in `container-myinstance-redis.service` change

```
Description=Podman container-redis.service
```

to be 

```
Description=Podman container-myinstance-redis.service
```

(repeat for `# container-redis.service` at the top of the file for tidyness, not required for functionality)

This will still start the container as `redis` in the `myinstance` pod, but it will stop other container-redis systemd files from conflicting with `myinstance`, in the event that you have other services running that also need redis containers simply named as `redis`.

The systemd configuration files are now prepared to enable the service

### enable systemctl autostart

Copy these files to your `/etc/systemd/system/` directory, to enable the pod as a service.

```
cp pod-myinstance.service /etc/systemd/system/pod-myinstance.service
cp container-myinstance-nginx.service /etc/systemd/system/container-myinstance-nginx.service
cp container-myinstance-redis.service /etc/systemd/system/container-myinstance-redis.service
cp container-myinstance-misago.service /etc/systemd/system/container-myinstance-misago.service
cp container-myinstance-celery.service /etc/systemd/system/container-myinstance-celery.service
```

Now we can control our misago service using systemctl

Stop and delete the containers that you have built in your `myinstance` pod, and delete the `myinstance` pod. The data will be persisted in the database and in the volume mounts. Now re-enable the service using systemctl.

```bash
systemctl enable pod-myinstance.service
systemctl start pod-myinstance.service
```

You should now see that the `myinstance` pod, and the containers, have rebuilt, and it will automatically restart on reboot.

as well as `enable` we can use `start`,`stop`,`status`, and `disable`. 

Note: If you do want to stop the containers you need to do so with systemctl (not with podman), or it will auto-restart them. Which is what enabling them as a service is for.

## Auto-updating containers

Because we have labelled the containers with `--label io.containers.autoupdate=local` To update the container images, all we need to do is pull, or build, the new images, and tag them as with the `:latest` tag for the image in question. Then use

```bash
podman auto-update --dry-run
```

if you are happy to proceed, then just run

```bash
podman auto-update
```

## Note on backups

To backup an instance there are two aspects that we need to consider. One is the database, and one the static files. 

The database should be achieved using a dump of the database. [borgmatic](https://torsion.org/borgmatic/) has database hooks that can automate this as part of the backup process. Depending on the nature of your postgresql provision, there are various options for that, which are not in scope here.

The static files should be backed up when there is no chance of a conflicting write during the process. It may be possible to snapshot the storage, and thus perform an uninterrupted backup. This will depend on the storage provision on your individual setup. If this is not possible, then it is best to temporarily stop the service, backup, and then restart the service. For example you might use a script like this (which could be run by `cron` or `systemctl timer`):

```bash
#!/bin/bash
/bin/systemctl stop pod-myinstance
rsync -avzh /[app] /backup/[app]
/bin/systemctl start pod-myinstance
```

== We're Using GitHub Under Protest ==

This project is currently hosted on GitHub.  This is not ideal; GitHub is a
proprietary, trade-secret system that is not Free and Open Souce Software
(FOSS).  We are deeply concerned about using a proprietary system like GitHub
to develop our FOSS project.  We have an open issue 
([Alternative to github](https://github.com/tetricky/misago-image/issues/3) where the
project contributors are actively discussing how we can move away from GitHub
in the long term.  We urge you to read about the
[Give up GitHub](https://GiveUpGitHub.org) campaign from
[the Software Freedom Conservancy](https://sfconservancy.org) to understand
some of the reasons why GitHub is not a good place to host FOSS projects.

Any use of this project's code by GitHub Copilot, past or present, is done
without our permission.  We do not consent to GitHub's use of this project's
code in Copilot.

![Logo of the GiveUpGitHub campaign](https://sfconservancy.org/img/GiveUpGitHub.png)
