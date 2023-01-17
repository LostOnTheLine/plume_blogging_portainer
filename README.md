# Plume Blogger in Portainer
1st I should note that this is NOT in any way related to or similar to Plume Creator. I was referred to it by something saying it was a Docker Interface for Plume Creator, & after much effort to get it working, discovered it was not what I wanted at all.

But since I went through the trouble of getting it to work, I wanted to share it so others could avoid the troubles I went through.

Plume has instructions for creating via a Docker-Compose... but it doesn't work with Portainer. Here's how to get around that

The instructions for creating a Plume Docker-Compose are... not great...
The Official Instructions are as follows:

### Official Docker-Compose Instructions
> You can use docker and docker-compose in order to manage your Plume instance and have it isolated from your host.
> 
> If you don’t have docker and docker-compose installed yet, here are their respective installation documentation: docker and docker-compose.
> 
> Then use these commands:
> ```
> mkdir plume
> cd plume
> ```
> #Get the docker-compose configuration and Plume's configuration<br>
> ```
> curl https://docs.joinplu.me/docker-compose.sample.yml > docker-compose.yml
> ```
> #If you are on an ARM machine, use the image from the Lollipop Cloud project instead<br>
> ```
> curl https://docs.joinplu.me/docker-compose.sample.arm32v7.yml > docker-compose.yml
> ```
> #Or<br>
> ```
> curl https://docs.joinplu.me/docker-compose.sample.arm64v8.yml > docker-compose.yml
> ```
> 
> ```
> curl https://docs.joinplu.me/docker.sample.env > .env
> ```
> You should edit the freshly created .env file as it contains the configuration of your Plume instance. The options at the top especially should be modified.<br>
> 
> Once it’s done, you can finalize the installation.<br>
> 
> #Download the images<br>
> ```
> docker-compose pull
> ```
> #Launch the database container<br>
> ```
> docker-compose up -d postgres
> ```
> #Wait for postgres init (user docker-compose logs to get postgres output)<br>
> #Database setup, first migration run<br>
> ```
> docker-compose run --rm plume plm migration run
> ```
> #Setup your instance<br>
> ```
> docker-compose run --rm plume plm search init
> docker-compose run --rm plume plm instance new -d 'domain.name' -n 'instance name' -l 'default licence'
> docker-compose run --rm plume plm users new -n 'admin' -N 'name' -b 'bio' -e 'admin@domain.name' -p 'pass' --admin
> ```
> #Launch your instance for good<br>
> ```
> docker-compose up -d
> ```

This leaves us with a few problems if we are using Portainer to manage our Docker
1. We don't need to curl the docker-compose.yaml, we're building it ourselves
1. You need 
    1. to build the stack
    1. run only part of it
    1. wait for it to startup
    1. run commands on the other part before it can be in a running state

This creates a problem. I have 2 ways around this, using Portainer, both essentially the same thing, just done in different ways.


---
---


### Method 1 involves making the container hang & BASHing into the CLI to run the initialization commands.

To do this we simply need to add a `command:` to the Docker-Compose so the container will be running. I accomplish this by using a sleep command. I have mine set to 9999 seconds, which is roughly 2¾ hours, more than enough time to get everything done. `command: sleep 9999 && mkdir -p /end` I added a simple directory creation command as it's my default command I use to have a container bypass it's default CMD & exit, but it shouldn't be necessary.

Once the container is up wait a couple minutes for the database to be created, Portainer makes it easy to watch the logs until they say `database system is ready to accept connections`

Once the database is up use Portainer to CLI into the `plume` Container. If the container is running you should be able to connect with the default `/bin/bash`

In the CLI you need to run the same 4 commands
```ash
plm migration run
plm search init
plm instance new -d '<<you.domain.ext>>' -n "<<instance name>>" -l '<<default license>>'
plm users new -n 'admin' -N 'AdminDisplayName' -b 'bio' -e 'admin@example.ext' -p '<<YOURPASSWORD>>' --admin
```


---


### Method 2 involves having the Docker-Compose run the commands without having to use the CLI. This has the benefit of being simpler in execution, but requires you to put the password you intend to use in plain text in the compose which, even if only there for a single run, may not be what many wish to do.

To do this I run a `command:` as well, but I wait 2 minutes for the database to fully startup, then run the commands. `sh -c "sleep 120 && plm migration run && plm search init && plm instance new -d '<<you.domain.ext>>' -n '<<instance name>>' -l '<<default license>>' && plm users new -n 'admin' -N 'AdminDisplayName' -b 'bio' -e 'admin@example.ext' -p '<<YOURPASSWORD>>' --admin`

For both of these methods I would recommend having the `restart:` policy commented out so that it does not try to restart

---
---

## Here is an example of a Docker-Compose with the lines commented out. You can add the one you like, then remove or re-comment it out after you finish.

```yaml
---
version: '3'

services:
  plume-postgres:
    image: postgres:10.5
    container_name: postgres
    #env_file: ./.env
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Cancun
      - BASE_URL=https://YOUR.DOMAIN.EXT/
      - ROCKET_SECRET_KEY=randomstringhere    # generate one with openssl rand -base64 32
                  # Mail settings
      - MAIL_SERVER=smtp.example.org
      - MAIL_USER=example
      - MAIL_PASSWORD=123456
      - MAIL_HELO_NAME=example.org
                  # DATABASE SETUP
      - POSTGRES_PASSWORD=passw0rd
      - POSTGRES_USER=plume
      - POSTGRES_DB=plume
                  # you can safely leave those defaults
      - DATABASE_URL=postgres://plume:passwOrd@plume-postgres:5432/plume
      - MIGRATION_DIRECTORY=migrations/postgres
      - USE_HTTPS=1
      - ROCKET_ADDRESS=0.0.0.0
      - ROCKET_PORT=7878
    volumes:
      - /docker/books/novels/plume/db/postgres:/var/lib/postgresql/data
      - /docker/log/var/log:/var/log:rw
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
 ##################################################
  plume:
    image: plumeorg/plume:latest
    container_name: plume
    #env_file: ./.env
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Cancun
      - BASE_URL=https://YOUR.DOMAIN.EXT/
      - ROCKET_SECRET_KEY=randomstringhere    # generate one with openssl rand -base64 32
                  # Mail settings
      - MAIL_SERVER=smtp.example.org
      - MAIL_USER=example
      - MAIL_PASSWORD=123456
      - MAIL_HELO_NAME=example.org
                  # DATABASE SETUP
      - POSTGRES_PASSWORD=passw0rd
      - POSTGRES_USER=plume
      - POSTGRES_DB=plume
                  # you can safely leave those defaults
      - DATABASE_URL=postgres://plume:passw0rd@plume-postgres:5432/plume
      - MIGRATION_DIRECTORY=migrations/postgres
      - USE_HTTPS=1
      - ROCKET_ADDRESS=0.0.0.0
      - ROCKET_PORT=7878
    ports:
      - 7878:7878
    volumes:
      - /docker/books/novels/plume/static/media:/app/static/media
      - /docker/books/novels/plume/.env:/app/.env
      - /docker/books/novels/plume/search_index:/app/search_index
      - /docker/log/var/log:/var/log:rw
      - /etc/localtime:/etc/localtime:ro
    #command: sleep 9999 && mkdir -p /end >> /var/log/novels/plume-cli.log
  #  command: >
  #    sh -c "sleep 120 
  #    && plm migration run
  #    && plm search init
  #    && plm instance new -d '<<you.domain.ext>>' -n "<<instance name>>" -l '<<default license>>'
  #    && plm users new -n 'admin' -N 'AdminDisplayName' -b 'bio' -e 'admin@example.ext' -p '<<YOURPASSWORD>>' --admin
    restart: unless-stopped
    depends_on:
      - plume-postgres

```
I have the 2 separate options commented on different columns to make it easier to tell that they are separate.

---
---

Obviously you will want to change your mapping to your directories.

I add a mapping for `/var/log` to every container as well as `/etc/localtime:ro` & the `TZ=` Environment variable. I have left those in, but they are, of course, not necessary<br>
I have also added the `.env` Variables to the Docker-Compose. You can make a copy of the `.env` file & put it in the Portainer compose directory for that stack or import the `.env` file, But the `.env` file ***needs*** to be mapped to `/app/.env` & with Portainer that will require there to be 2 copies of it as Portainer cannot map to the file unless it's within Portainer's Volume or Bind Mounts & the container cannot access it inside Portainer's Volume/Bind Mounts.
