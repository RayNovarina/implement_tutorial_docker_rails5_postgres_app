# README - AtomProjects/Dockerizing-Rails/NickJanetakis

Based on Dockerize a Rails 5, Postgres, Redis, Sidekiq and Action Cable
Application with Docker Compose at:
https://nickjanetakis.com/blog/dockerize-a-rails-5-postgres-redis-sidekiq-action-cable-app-with-docker-compose

After this tutorial, you’ll understand what the benefits of using Docker are and
will be able to:

Install Docker on all major platforms in 5 minutes or less
Run an example Rails 5+ app that uses a bunch of best practices
Know how to write a Dockerfile
Know how to run multiple Docker containers manually
Run multiple Docker containers with Docker Compose

=============================================================================================================================

1) Install orats
What is orats? at: https://github.com/nickjj/orats
It stands for opinionated rails application templates.
The goal is to provide you an excellent base application that you can use on
your next Rails project. You're meant to generate a project using orats and then
build your custom application on top of it.

It also happens to use Docker so that your app can be ran on any major
platform -- even without needing Ruby installed. Since you’re very likely a
Ruby developer, you have Ruby installed on your work station so installation is
as simple as 'gem install orats'.
  $ gem install orats

-----------------------------------------------------------------------------------------

2) Generate a new project
# Feel free to generate the project anywhere you want.
$ orats new my_dockerized_app
Open the project in your favorite code editor

# Move into the project's directory
cd my_dockerized_app


Dockerize the Rails application
There’s a few things we need to do to Dockerize the application.

-----------------------------------------------------------------------------------------

3) Logging
In order for logs to function properly, Docker expects your application or
process to log to STDOUT. This is a very good idea because the concept of
managing log files will now be de-coupled from your Rails application.

You can choose to have Docker write those log entries to syslog or another
local service running on your server, or you can ferry the log output over to a
third party service such as Loggly.

In either case, you need to make a small adjustment to your Rails app.

# config/application.rb : Lines 21-24

logger           = ActiveSupport::Logger.new(STDOUT)
logger.formatter = config.log_formatter
config.log_tags  = [:subdomain, :uuid]
config.logger    = ActiveSupport::TaggedLogging.new(logger)
In the orats base project, I’ve decided to set up logging in the application.rb
file because it will be common to all environments.

We’re just setting Rails up to log to STDOUT and then we also set up a custom
formatter to include both the subdomain and uuid in each log entry.

I like doing this because it makes filtering logs very simple – especially when
your application grows in size and has many moving parts.

-----------------------------------------------------------------------------------------

4) Dockerfile
You can think of this file as your Docker image blueprint or recipe. When you
run the docker build command it will execute each line from top to bottom.

It’s going to run all of these commands in the context of the Docker image,
look at the lines below:
    ENV INSTALL_PATH /my_dockerized_app
    RUN mkdir -p $INSTALL_PATH

This command is not going to be executed on your workstation. Instead, that
folder is going to be created inside of the Docker image.

At the end of the day, when you build this Dockerfile, it’s going to create a
Docker image that has a Debian Jessie base, all of the Rails code and by default
it will run the Puma app server.

-------------------------------------
    FROM ruby:2.3-slim
    # Docker images can start off with nothing, but it's extremely
    # common to pull in from a base image. In our case we're pulling
    # in from the slim version of the official Ruby 2.3 image.
    #
    # Details about this image can be found here:
    # https://hub.docker.com/_/ruby/
    #
    # Slim is pulling in from the official Debian Jessie image.
    #
    # You can tell it's using Debian Jessie by clicking the
    # Dockerfile link next to the 2.3-slim bullet on the Docker hub.
    #
    # The Docker hub is the standard place for you to find official
    # Docker images. Think of it like GitHub but for Docker images.

    MAINTAINER Nick Janetakis <nick.janetakis@gmail.com>
    # It is good practice to set a maintainer for all of your Docker
    # images. It's not necessary but it's a good habit.

    RUN apt-get update && apt-get install -qq -y --no-install-recommends \
          build-essential nodejs libpq-dev
    # Ensure that our apt package list is updated and install a few
    # packages to ensure that we can compile assets (nodejs) and
    # communicate with PostgreSQL (libpq-dev).

    ENV INSTALL_PATH /my_dockerized_app
    # The name of the application is my_dockerized_app and while there
    # is no standard on where your project should live inside of the Docker
    # image, I like to put it in the root of the image and name it
    # after the project.
    #
    # We don't even need to set the INSTALL_PATH variable, but I like
    # to do it because we're going to be referencing it in a few spots
    # later on in the Dockerfile.
    #
    # The variable could be named anything you want.

    RUN mkdir -p $INSTALL_PATH
    # This just creates the folder in the Docker image at the
    # install path we defined above.

    WORKDIR $INSTALL_PATH
    # We're going to be executing a number of commands below, and
    # having to CD into the /my_dockerized_app folder every time would be
    # lame, so instead we can set the WORKDIR to be /my_dockerized_app.
    #
    # By doing this, Docker will be smart enough to execute all
    # future commands from within this directory.

    COPY Gemfile Gemfile.lock ./
    # This is going to copy in the Gemfile and Gemfile.lock from our
    # work station at a path relative to the Dockerfile to the
    # my_dockerized_app/ path inside of the Docker image.
    #
    # It copies it to /my_dockerized_app because of the WORKDIR being set.
    #
    # We copy in our Gemfile before the main app because Docker is
    # smart enough to cache "layers" when you build a Docker image.
    #
    # You see, each command we have in the Dockerfile is going to be
    # ran and then saved as a separate layer. Docker is smart enough
    # to only re-build pieces that change, in order from top to bottom.
    #
    # This is an advanced concept but it means that we'll be able to
    # cache all of our gems so that if we make an application code
    # change, it won't re-run bundle install unless a gem changed.

    RUN bundle install --binstubs
    # We want binstubs to be available so we can directly call sidekiq and
    # potentially other binaries as command overrides without depending on
    # bundle exec.
    # This is mainly due for production compatibility assurance.

    COPY . .
    # This might look a bit alien but it's copying in everything from
    # the current directory relative to the Dockerfile, over to the
    # /my_dockerized_app folder inside of the Docker image.
    #
    # We can get away with using the . for the second argument because
    # this is how the unix command cp (copy) works. It stands for the
    # current directory.

    RUN bundle exec rake RAILS_ENV=production DATABASE_URL=postgresql://user:pass@127.0.0.1/dbname ACTION_CABLE_ALLOWED_REQUEST_ORIGINS=foo,bar SECRET_TOKEN=dummytoken assets:precompile
    # Provide a dummy DATABASE_URL and more to Rails so it can pre-compile
    # assets. The values do not need to be real, just valid syntax.
    #
    # If you're saving your assets to a CDN and are working with multiple
    # app instances, you may want to remove this step and deal with asset
    # compilation at a different stage of your deployment.

    VOLUME ["$INSTALL_PATH/public"]
    # In production you will very likely reverse proxy Rails with nginx.
    # This sets up a volume so that nginx can read in the assets from
    # the Rails Docker image without having to copy them to the Docker host.

    CMD puma -C config/puma.rb
    # This is the command that's going to be run by default if you run the
    # Docker image without any arguments.
    #
    # In our case, it will start the Puma app server while passing in
    # its config file.

-----------------------------------------------------------------------------------------

5) docker-compose.yml

Docker Compose is an official tool supplied by Docker. At its core, it’s a
utility that lets you “compose” Docker commands.

version: '2'

services:
  postgres:
    image: 'postgres:9.5'
    environment:
      POSTGRES_USER: 'my_dockerized_app'
      POSTGRES_PASSWORD: 'yourpassword'
    ports:
      - '5432:5432'
    volumes:
      - 'postgres:/var/lib/postgresql/data'

  redis:
    image: 'redis:3.2-alpine'
    command: redis-server --requirepass yourpassword
    ports:
      - '6379:6379'
    volumes:
      - 'redis:/var/lib/redis/data'

  website:
    depends_on:
      - 'postgres'
      - 'redis'
    build: .
    ports:
      - '3000:3000'
    volumes:
      - '.:/my_dockerized_app'
    env_file:
      - '.env'

  sidekiq:
    depends_on:
      - 'postgres'
      - 'redis'
    build: .
    command: sidekiq -C config/sidekiq.yml.erb
    volumes:
      - '.:/my_dockerized_app'
    env_file:
      - '.env'

  cable:
    depends_on:
      - 'redis'
    build: .
    command: puma -p 28080 cable/config.ru
    ports:
      - '28080:28080'
    volumes:
      - '.:/my_dockerized_app'
    env_file:
      - '.env'

volumes:
  redis:
  postgres:

We’re using version: 2 because as of Docker Compose 1.6+, Docker Compose
introduced a new syntax that is going to be used moving forward.

Then we have a services namespace that lets us define our services. They are the
same services we ran manually.

As you can see, the properties we define match up with the command line flags
that we passed in to various Docker commands earlier.

The only new thing is depends_on which is a Docker Compose feature that will
start the depended on services before starting the service that depends on it.

In other words, postgres and redis containers will be started before the
website, sidekiq and cable containers get started.

Lastly, we have a volumes namespace for our named volume(s).


-----------------------------------------------------------------------------------------

6) .env

This file isn’t technically part of Docker, but it’s partly used by Docker
Compose and the Rails application.

By default Docker Compose will look for an .env file in the same directory as
your docker-compose.yml file.

We can set various environment variables here, and you can even add your custom
environment variables here too if your application uses ENV variables.

COMPOSE_PROJECT_NAME=my_dockerized_app
By setting the COMPOSE_PROJECT_NAME to my_dockerized_app, Docker Compose will
automatically prefix our Docker images, containers, volumes and networks with
mydockerizedapp.

It just so happens that Docker Compose will strip underscores from the name.

There’s plenty of other values in this .env file but most of them are custom to
the Rails application. We’ll go over a few Docker related values in a bit.

-----------------------------------------------------------------------------------------

7) Run the Ruby on Rails application
You can run everything by typing:

  docker-compose up --build

Docker Compose has many different sub-commands and flags. You’ll definitely
want to check them out on your own.

docker-compose up --build
Creating network "mydockerizedapp_default" with the default driver
Creating volume "mydockerizedapp_redis" with default driver
Creating volume "mydockerizedapp_postgres" with default driver
Pulling redis (redis:3.2-alpine)...
3.2-alpine: Pulling from library/redis
Digest: sha256:9cd405cd1ec1410eaab064a1383d0d8854d1eef74a54e1e4a92fb4ec7bdc3ee7
Status: Downloaded newer image for redis:3.2-alpine
Pulling postgres (postgres:9.5)...
9.5: Pulling from library/postgres
 ......
Status: Downloaded newer image for ruby:2.3-slim

---> b8244f7bc643
Step 2/12 : MAINTAINER Nick Janetakis <nick.janetakis@gmail.com>
---> Running in 67ee4b36c49a
---> 5888fb67b69d
Removing intermediate container 67ee4b36c49a
Step 3/12 : RUN apt-get update && apt-get install -qq -y --no-install-recommends build-essential nodejs libpq-dev
---> Running in 1634d78a7ae3
Get:1 http://security.debian.org jessie/updates InRelease [63.1 kB]
Ign http://deb.debian.org jessie InRelease
  ...
Setting up nodejs (0.10.29~dfsg-2) ...
update-alternatives: using /usr/bin/nodejs to provide /usr/bin/js (js) in auto mode
Processing triggers for libc-bin (2.19-18+deb8u7) ...

 ---> d21102e4c653
Removing intermediate container 1634d78a7ae3
Step 4/12 : ENV INSTALL_PATH /my_dockerized_app
 ---> Running in a1536b53fb12
 ---> c85fcef7b358
Removing intermediate container a1536b53fb12
Step 5/12 : RUN mkdir -p $INSTALL_PATH
 ---> Running in 7d44c7ddf208
 ---> f0ad991ea7a3
Removing intermediate container 7d44c7ddf208
Step 6/12 : WORKDIR $INSTALL_PATH
 ---> 1ce149ff10cd
Removing intermediate container 79adc36c73f7
Step 7/12 : COPY Gemfile Gemfile.lock ./
 ---> 424955e54a7f
Removing intermediate container a6003979a897
Step 8/12 : RUN bundle install --binstubs
 ---> Running in b1a92194ec16
Fetching gem metadata from https://rubygems.org/.........
    ....
Installing redis-rails 5.0.1
Bundle complete! 20 Gemfile dependencies, 73 gems now installed.
Bundled gems are installed into /usr/local/bundle.
 ---> 9e9008be5447
Removing intermediate container b1a92194ec16
Step 9/12 : COPY . .
 ---> fd3b053e873e
Removing intermediate container e2badbd225c1
Step 10/12 : RUN bundle exec rake RAILS_ENV=production
                                  DATABASE_URL=postgresql://user:pass@127.0.0.1/dbname
                                  ACTION_CABLE_ALLOWED_REQUEST_ORIG
INS=foo,bar SECRET_TOKEN=dummytoken assets:precompile
 ---> Running in 61079ab790e0
 .......
 ---> 775175718b2b
 Removing intermediate container 61079ab790e0
 Step 11/12 : VOLUME $INSTALL_PATH/public
  ---> Running in 81209c234e38
  ---> ed0570b9aa8e
 Removing intermediate container 81209c234e38
 Step 12/12 : CMD puma -C config/puma.rb
  ---> Running in a72b26fab33b
  ---> 3663a86e69c7
 Removing intermediate container a72b26fab33b
 Successfully built 3663a86e69c7

 Building cable
 Step 1/12 : FROM ruby:2.3-slim
  ---> b8244f7bc643
 Step 2/12 : MAINTAINER Nick Janetakis <nick.janetakis@gmail.com>
  ---> Using cache
  ---> 5888fb67b69d
 Step 3/12 : RUN apt-get update && apt-get install -qq -y --no-install-recommends build-essential nodejs libpq-dev
  ---> Using cache
  ---> d21102e4c653
 Step 4/12 : ENV INSTALL_PATH /my_dockerized_app
  ---> Using cache
  ---> c85fcef7b358
 Step 5/12 : RUN mkdir -p $INSTALL_PATH
  ---> Using cache
  ---> f0ad991ea7a3
 Step 6/12 : WORKDIR $INSTALL_PATH
  ---> Using cache
  ---> 1ce149ff10cd
 Step 7/12 : COPY Gemfile Gemfile.lock ./
  ---> Using cache
  ---> 424955e54a7f
 Step 8/12 : RUN bundle install --binstubs
  ---> Using cache-from
  ---> 9e9008be5447
  Step 9/12 : COPY . .
   ---> Using cache
   ---> fd3b053e873e
  Step 10/12 : RUN bundle exec rake RAILS_ENV=production
                                    DATABASE_URL=postgresql://user:pass@127.0.0.1/dbname
                                    ACTION_CABLE_ALLOWED_REQUEST_ORIG
  INS=foo,bar SECRET_TOKEN=dummytoken assets:precompile
   ---> Using cache
   ---> 775175718b2b

  Step 11/12 : VOLUME $INSTALL_PATH/public
   ---> Using cache
   ---> ed0570b9aa8e
  Step 12/12 : CMD puma -C config/puma.rb
   ---> Using cache
   ---> 3663a86e69c7
  Successfully built 3663a86e69c7
  Creating mydockerizedapp_postgres_1
  Creating mydockerizedapp_redis_1
  Creating mydockerizedapp_sidekiq_1
  Creating mydockerizedapp_cable_1
  Creating mydockerizedapp_website_1
  Attaching to mydockerizedapp_postgres_1, mydockerizedapp_redis_1,
               mydockerizedapp_website_1, mydockerizedapp_cable_1, mydockerizedapp_sidekiq_1
  postgres_1  | The files belonging to this database system will be owned by user "postgres".
  postgres_1  | This user must also own the server process.
  postgres_1  |
  redis_1     |                 _._
  redis_1     |            _.-``__ ''-._
  postgres_1  | The database cluster will be initialized with locale "en_US.utf8".
  postgres_1  | The default database encoding has accordingly been set to "UTF8".
  postgres_1  | The default text search configuration will be set to "english".
  postgres_1  |
  redis_1     |       _.-``    `.  `_.  ''-._           Redis 3.2.8 (00000000/0) 64 bit
  website_1   | [7] Puma starting in cluster mode...
  redis_1     |   .-`` .-```.  ```\/    _.,_ ''-._
  website_1   | [7] * Version 3.6.2 (ruby 2.3.3-p222), codename: Sleepy Sunday Serenity
  postgres_1  | Data page checksums are disabled.
  redis_1     |  (    '      ,       .-`  | `,    )     Running in standalone mode
  website_1   | [7] * Min threads: 1, max threads: 1
  redis_1     |  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
  website_1   | [7] * Environment: development
  postgres_1  | fixing permissions on existing directory /var/lib/postgresql/data ... ok
  redis_1     |  |    `-._   `._    /     _.-'    |     PID: 1
  website_1   | [7] * Process workers: 1
  redis_1     |   `-._    `-._  `-./  _.-'    _.-'
  website_1   | [7] * Phased restart available
  postgres_1  | creating subdirectories ... ok
  redis_1     |  |`-._`-._    `-.__.-'    _.-'_.-'|
  website_1   | [7] * Listening on tcp://0.0.0.0:3000
  redis_1     |  |    `-._`-._        _.-'_.-'    |           http://redis.io
  website_1   | [7] Use Ctrl-C to stop
  redis_1     |   `-._    `-._`-.__.-'_.-'    _.-'
  redis_1     |  |`-._`-._    `-.__.-'    _.-'_.-'|
  redis_1     |  |    `-._`-._        _.-'_.-'    |
  postgres_1  | selecting default max_connections ... 100
  redis_1     |   `-._    `-._`-.__.-'_.-'    _.-'
  website_1   | [9] + Gemfile in context: /my_dockerized_app/Gemfile
  redis_1     |       `-._    `-.__.-'    _.-'
  postgres_1  | selecting default shared_buffers ... 128MB
  postgres_1  | selecting dynamic shared memory implementation ... posix
  redis_1     |           `-._        _.-'
  redis_1     |               `-.__.-'
  redis_1     |
  postgres_1  | creating configuration files ... ok
  redis_1     | 1:M 09 Mar 06:30:19.895 # WARNING: The TCP backlog setting of 511
                cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
  redis_1     | 1:M 09 Mar 06:30:19.895 # Server started, Redis version 3.2.8
  redis_1     | 1:M 09 Mar 06:30:19.895 # WARNING you have Transparent Huge Pages
                (THP) support enabled in your kernel. This will create latency
                and memory usage issues with Redis. To fix this issue run the
                command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
                as root, and add it to your /etc/rc.local in order to retain the
                setting after a reboot. Redis must be restarted after THP is disabled.
  redis_1     | 1:M 09 Mar 06:30:19.895 * The server is now ready to accept connections on port 6379
  postgres_1  | creating template1 database in /var/lib/postgresql/data/base/1 ... ok
  postgres_1  | initializing pg_authid ... ok
  postgres_1  | initializing dependencies ... ok
  cable_1     | [1] Puma starting in cluster mode...
  cable_1     | [1] * Version 3.6.2 (ruby 2.3.3-p222), codename: Sleepy Sunday Serenity
  cable_1     | [1] * Min threads: 1, max threads: 1
  cable_1     | [1] * Environment: development
  cable_1     | [1] * Process workers: 1
  cable_1     | [1] * Phased restart available
  cable_1     | [1] * Listening on tcp://0.0.0.0:28080
  cable_1     | [1] Use Ctrl-C to stop
  cable_1     | [8] + Gemfile in context: /my_dockerized_app/Gemfile
  postgres_1  | creating system views ... ok
  postgres_1  | loading system objects' descriptions ... ok
  postgres_1  | creating collations ... ok
  postgres_1  | creating conversions ... ok
  postgres_1  | creating dictionaries ... ok
  postgres_1  | setting privileges on built-in objects ... ok
  sidekiq_1   | WARN: Unresolved specs during Gem::Specification.reset:
  sidekiq_1   |       minitest (~> 5.1)
  sidekiq_1   |       rake (>= 0.8.7)
  sidekiq_1   | WARN: Clearing out unresolved specs.
  sidekiq_1   | Please report a bug if this causes problems.
  postgres_1  | creating information schema ... ok
  postgres_1  | loading PL/pgSQL server-side language ... ok
  postgres_1  | vacuuming database template1 ... ok
  postgres_1  | copying template1 to template0 ... ok
  postgres_1  | copying template1 to postgres ... ok
  postgres_1  | syncing data to disk ...
  postgres_1  | WARNING: enabling "trust" authentication for local connections
  postgres_1  | You can change this by editing pg_hba.conf or using the option -A, or
  postgres_1  | --auth-local and --auth-host, the next time you run initdb.
  postgres_1  | ok
  postgres_1  |
  postgres_1  | Success. You can now start the database server using:
  postgres_1  |
  postgres_1  |     pg_ctl -D /var/lib/postgresql/data -l logfile start
  postgres_1  |
  postgres_1  | waiting for server to start....LOG:  database system was shut down at 2017-03-09 06:30:22 UTC
  postgres_1  | LOG:  MultiXact member wraparound protections are now enabled
  postgres_1  | LOG:  database system is ready to accept connections
  postgres_1  | LOG:  autovacuum launcher started
  sidekiq_1   | 2017-03-09T06:30:23.864Z 1 TID-5hmho INFO: Booting Sidekiq 4.2.7
                with redis options {:url=>"redis://:REDACTED@redis:6379/0"}
  postgres_1  |  done
  postgres_1  | server started
  website_1   | [7] - Worker 0 (pid: 9) booted, phase: 0
  postgres_1  | CREATE DATABASE
  postgres_1  |
  sidekiq_1   | 2017-03-09T06:30:24.570Z 1 TID-5hmho INFO: Running in ruby 2.3.3p222 (2016-11-21 revision 56859) [x86_64-linux]
  sidekiq_1   | 2017-03-09T06:30:24.571Z 1 TID-5hmho INFO: See LICENSE and the LGPL-3.0 for licensing details.
  sidekiq_1   | 2017-03-09T06:30:24.571Z 1 TID-5hmho INFO: Upgrade to Sidekiq Pro for more features and support: http://sidekiq.org
  postgres_1  | CREATE ROLE
  postgres_1  |
  postgres_1  |
  sidekiq_1   | 2017-03-09T06:30:24.608Z 1 TID-5hmho INFO: Starting processing, hit Ctrl-C to stop
  postgres_1  | /usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
  postgres_1  |
  postgres_1  | LOG:  received fast shutdown request
  postgres_1  | LOG:  aborting any active transactions
  postgres_1  | LOG:  autovacuum launcher shutting down
  postgres_1  | waiting for server to shut down....LOG:  shutting down
  postgres_1  | LOG:  database system is shut down
  postgres_1  |  done
  postgres_1  | server stopped
  postgres_1  |
  postgres_1  | PostgreSQL init process complete; ready for start up.
  postgres_1  |
  postgres_1  | LOG:  database system was shut down at 2017-03-09 06:30:24 UTC
  postgres_1  | LOG:  MultiXact member wraparound protections are now enabled
  postgres_1  | LOG:  autovacuum launcher started
  postgres_1  | LOG:  database system is ready to accept connections
  cable_1     | [1] - Worker 0 (pid: 8) booted, phase: 0

-----------------------------------------------------------------------------------------

After the up command finishes, open up a new terminal tab and check out what was
created on your behalf.

8) Docker images

Run docker images:

mydockerizedapp_cable     latest              ...         579 MB
mydockerizedapp_sidekiq   latest              ...         579 MB
mydockerizedapp_website   latest              ...         579 MB
postgres                  9.5                 ...         265.8 MB
redis                     3.2-alpine          ...         29.02 MB
ruby                      2.3-slim            ...         285.1 MB
Docker Compose automatically pulled down Redis and Ruby for you, and then built
the website, sidekiq and cable images for you.

I get:
mydockerizedapp_cable                   latest               3663a86e69c7        2 minutes ago       577 MB
mydockerizedapp_sidekiq                 latest               3663a86e69c7        2 minutes ago       577 MB
mydockerizedapp_website                 latest               3663a86e69c7        2 minutes ago       577 MB

-----------------------------------------------------------------------------------------

9) Docker containers

Run docker-compose ps:

Name                        State   Ports
-------------------------------------------------------------------
mydockerizedapp_cable_1      ...   Up      0.0.0.0:28080->28080/tcp
mydockerizedapp_postgres_1   ...   Up      0.0.0.0:5432->5432/tcp
mydockerizedapp_redis_1      ...   Up      0.0.0.0:6379->6379/tcp
mydockerizedapp_sidekiq_1    ...   Up
mydockerizedapp_website_1    ...   Up      0.0.0.0:3000->3000/tcp
Docker Compose automatically named the containers for you, and it appended
a _1 because it’s running 1 instance of the Docker image.

Docker Compose supports scaling but that goes beyond the scope of this tutorial.

I get:
Name                         Command               State            Ports
----------------------------------------------------------------------------------------------
mydockerizedapp_cable_1      puma -p 28080 cable/config.ru    Up      0.0.0.0:28080->28080/tcp
mydockerizedapp_postgres_1   docker-entrypoint.sh postgres    Up      0.0.0.0:5432->5432/tcp
mydockerizedapp_redis_1      docker-entrypoint.sh redis ...   Up      0.0.0.0:6379->6379/tcp
mydockerizedapp_sidekiq_1    sidekiq -C config/sidekiq. ...   Up
mydockerizedapp_website_1    /bin/sh -c puma -C config/ ...   Up      0.0.0.0:3000->3000/tcp

We can also see which ports the services are using.

-----------------------------------------------------------------------------------------

10) Docker volumes

Run docker volume ls:

DRIVER              VOLUME NAME
local               mydockerizedapp_postgres
local               mydockerizedapp_redis
Docker Compose automatically created the named volume for Postgres and Redis.

Run docker volume --help to see what else you can do on your own.

I get:
DRIVER              VOLUME NAME
local               mydockerizedapp_postgres
local               mydockerizedapp_redis

-----------------------------------------------------------------------------------------

12) Docker networks

Run docker network ls:

NETWORK ID          NAME                      DRIVER
...                 bridge                    bridge
...                 host                      host
...                 none                      null
...                 mydockerizedapp_default   bridge
Docker Compose created the network for you when you ran up. I recommend you run
docker network --help to see what else you can do on your own.

I get:
NETWORK ID          NAME                            DRIVER              SCOPE
ccb1912fe4e4        bridge                          bridge              local
55de3be4b34e        host                            host                local
2222e2dc3dea        mydockerizedapp_default         bridge              local
910de74fcb19        none                            null                local
-----------------------------------------------------------------------------------------

13) Redis
If you were to look at the .env file and check out this line:

REDIS_CACHE_URL=redis://:yourpassword@redis:6379/0/cache
You’ll notice that we’re referencing redis as the actual Redis host name.

We can do this because of the Docker network that was created for us. Our Redis
service is defined as redis in the docker-compose.yml file, so we’re able to reference it by name as a legit host name.

This works because Docker is managing a DNS server for you on your behalf.

-----------------------------------------------------------------------------------------

14) Viewing the site
(Note: At this point you’ll notice that it throws an error saying the database
does not exist. No worries, that’s expected right? We haven’t reset our database
yet.)

If you’re running Docker natively then you can
access
  http://localhost:3000
right now.

-----------------------------------------------------------------------------------------

15) Interacting with the Rails application

At this point you’ll notice that it throws an error saying the database does not
exist. No worries, that’s expected right? We haven’t reset our database yet.

This section of the blog post will serve 2 purposes. It will show you how to run
Rails commands through Docker, and also shows you how to initialize a Rails
database.

Run both of the commands below in a new Docker-enabled terminal tab

-----------------------------------------------------------------------------------------

16) Reset the database
docker-compose exec website rails db:reset
      (0.2ms)  DROP DATABASE IF EXISTS "my_dockerized_app_development"
  Dropped database 'my_dockerized_app_development'
      (0.3ms)  DROP DATABASE IF EXISTS "my_dockerized_app_test"
  Dropped database 'my_dockerized_app_test'
      (423.6ms)  CREATE DATABASE "my_dockerized_app_development" ENCODING = 'utf8'
  Created database 'my_dockerized_app_development'
      (414.7ms)  CREATE DATABASE "my_dockerized_app_test" ENCODING = 'utf8'
  Created database 'my_dockerized_app_test'
      /my_dockerized_app/db/schema.rb doesn't exist yet. Run `rails db:migrate` to create it,
      then try again. If you do not intend to use a database, you should instead
      alter /my_dockerized_app/config/application.rb to limit the frameworks that will be loaded.

Migrate the database
docker-compose exec website rails db:migrate
    (61.9ms)  CREATE TABLE "schema_migrations" ("version" character varying PRIMARY KEY)
    (63.5ms)  CREATE TABLE "ar_internal_metadata" ("key" character varying PRIMARY KEY,
                "value" character varying, "created_at" timestamp NOT NULL, "updated_at" timestamp NOT NULL)
    (0.4ms)  SELECT pg_try_advisory_lock(8429416894839021480);
    ActiveRecord::SchemaMigration Load (0.5ms)
        SELECT "schema_migrations".* FROM "schema_migrations"
    ActiveRecord::InternalMetadata Load (0.7ms)
        SELECT  "ar_internal_metadata".* FROM "ar_internal_metadata" WHERE "ar_internal_metadata"."key
                " = $1 LIMIT $2  [["key", :environment], ["LIMIT", 1]]
    (0.2ms)  BEGIN SQL (0.5ms)
        INSERT INTO "ar_internal_metadata" ("key", "value", "created_at", "updated_at")
               VALUES ($1, $2, $3, $4) RETURNING "key"  [["key", "environment"],
                  ["value", "development"], ["created_at", 2017-03-09 06:55:14 UTC],
                  ["updated_at", 2017-03-09 06:55:14 UTC]]
    (14.9ms)  COMMIT
    (0.4ms)  SELECT pg_advisory_unlock(8429416894839021480)
    ActiveRecord::SchemaMigration Load (0.3ms)
        SELECT "schema_migrations".* FROM "schema_migrations"

What’s going on with the above commands?

Docker Compose has an exec command which lets you execute commands on an already
running container. We’re running commands on the website container.

The --user "$(id -u):$(id -g)" flag ensures that any files being generated by
the exec command end up being owned by your user rather than root.

You only need to add this flag if you’re running Docker natively.

Common theme between the above commands?

After defining which container will run the command, all we have to do is pass
in the actual command we want to run.

In this case, we’re running standard rails commands. With Rails 5+, you can
perform rake tasks through the rails binary.

-----------------------------------------------------------------------------------------

17) Viewing the site again:
  http://localhost:3000

I get error:
  Rack::Timeout::RequestTimeoutException in PagesController#home

  Extracted source (around line #12):
  10  <%= action_cable_meta_tag %>
  11
  12  <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
  13  <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
  14
  15  <%= csrf_meta_tags %>

  Rails.root: /my_dockerized_app

  app/views/layouts/application.html.erb:12:in `_app_views_layouts_application_html_erb__1970674373493421856_36503580'

Orats github issues:
https://github.com/nickjj/orats/issues/15
Fix: Increased timeout from 5 secs to 30 secs.
.env file:
REQUEST_TIMEOUT=30

-----------------------------------------------------------------------------------------

18) More rails commands:

Every Rails command that you would normally run without Docker can be ran in
the above way. For example if you wanted to generate a new controller:

Add a new controller

docker-compose exec --user "$(id -u):$(id -g)" website rails g controller Hi

Remove that controller

docker-compose exec --user "$(id -u):$(id -g)" website rails d controller Hi

-----------------------------------------------------------------------------------------

19) A live development environment

Since we have volumes set up in docker-compose.yml you’ll be able to actively
develop your application as if it were running natively on your OS.

Try going to app/views/pages/home.html.erb, then make a change to the file. All
you have to do is save the file and reload your browser to see the changes.

-----------------------------------------------------------------------------------------

20) Shutting things down
You’ll want to goto your Docker Compose terminal tab and press CTRL+C. Then for
good measure, type docker-compose stop. Sometimes Compose bugs out and won’t
stop all of your containers automatically.

You could also optionally run docker-compose rm -f to clean up your stopped
containers. I tend to do this all the time.

-----------------------------------------------------------------------------------------

21) General house keeping

If you keep building images, Docker is going to accumulate a lot of cruft.
You may notice that you even run out of disk space but you can easily prevent that.

You can remove dangling Docker images by running:
docker rmi -f $(docker images -qf dangling=true)

You can also remove dangling Docker volumes by running:
docker volume rm $(docker volume ls -qf dangling=true)

I recommend you run these commands at least once a week, or more if you’re
really actively using Docker. I personally run them on a daily cronjob.

---------------------------------------------------------------------------------

22) Deploy to Heroku by Ray:
per: https://devcenter.heroku.com/articles/container-registry-and-runtime

Make sure you have a working Docker installation (eg. docker ps) and that
you’re logged in to Heroku (heroku login).

a) Install the container-registry plugin by running:
$ heroku plugins:install heroku-container-registry

b) Log in to the Heroku container registry:
$ heroku container:login

c) Navigate to the app’s directory and create a Heroku app:
$ heroku create
Creating salty-fortress-4191... done, stack is cedar-14
https://salty-fortress-4191.herokuapp.com/ | https://git.heroku.com/salty-fortress-4191.git

d) Pushing an existing image
To push an image to Heroku, such as one pulled from Docker Hub, tag it and push it according to this naming template:
  $ docker tag <image> registry.heroku.com/<app>/<process-type>
  $ docker push registry.heroku.com/<app>/<process-type>

After the image has successfully been pushed, open your app:
$ heroku open -a <app>
