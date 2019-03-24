# Docker CLI docs:
https://docs.docker.com/engine/reference/commandline/docker/

# Starting a new container
`$ docker run [options] <image> <command>`

This command starts a new container based on `<image>` and executes `<command>` inside the container

example:
`$ docker run ruby:2.6 ruby -e "puts :hello"`

Variatie: throwaway container met --rm option
`$ docker run --rm <image> <command>`
This deletes the container after it completes

# Run and show containers
running containers:
`$ docker ps`

all containers including stopped ones:
`$ docker ps -a`

removing container:
`$ docker rm <container_id> [<contaner_id2>...]`

# Bash shell in container

move into new directory

`docker run -i -t --rm -v ${PWD}:/usr/src/app ruby:2.6 bash`

`-v` mounts a volume: shaing a porting of our local filesystem with the container
`-v ${PWD}:/usr/src/app` mounts current dir inside the container at `user/src/app`
docker creates the `user/src/app`
works two ways if you change something

`-i` for input. sluist input door naaar docker deamon als je op windows of mac bent
waar docker in een vs draait

`-t` for terminal. Opent een terminal

`-it` InTeractive. If you need and long-lived, interactive session, you need both -i 
and -t, shorthand: -it

bash  Starting bash

- exiting bash and stopping the container:
`exit`

This gives you this prompt (shell in the container). nb we are root:
root@05a29b49a35d:/#

### Moving to our new dir in the containter:
root@05a29b49a35d:/# cd /usr/src/app

# Generate a new Rails project using a container
- start an interactive bash shell running inside a container
`docker run -i -t --rm -v ${PWD}:/usr/src/app ruby:2.6 bash`

- installing rails gem:
`root@05a29b49a35d:/usr/src/app# gem install rails`

- making a new rails app:
`rails new myapp --skip-test --skip-bundle`

- exiting bash and stopping the container:
`exit`

nb: if new rails app change Gemfile:
gem 'sqlite3' 
to
gem 'sqlite3', '~> 1.3.6'

# Dockerfile for rails

create your Dockerfile in the root of the app

- `FROM ruby:2.6` starting image
- `RUN apt-get update -yqq` get updated package list
- `RUN apt-get install -yqq --no-install-recommends nodejs` 
   y = yes to any prompts, qq = quiet mode, 
   --no-install-recommends = do not install nonessential packages
   Adding these together in one command (&&) will also update
   package list if only intall changes
- `COPY . /usr/src/app/` copy all files from our local current (.)
   dir to /usr/src/app/  Nb current= wherever the dockerfile is
   You will want to split this in a update Gemfile part and an 
   update files part to avoid re-installing all gems when changing
   a file.
- `WORKDIR /usr/src/app` cd into /usr/src/app
- `RUN bundle install` which we want to do from inside our app dir
- `CMD ["bin/rails", "s", "-b", "0.0.0.0"]` default command to run when
   a container is started form the image. Here we start the rails server
   and bind to all ip adressed (to give access to local host that rails
   listens to)

```
FROM ruby:2.6

RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends \
   nodejs

COPY Gemfile* /usr/src/app/
WORKDIR /usr/src/app
RUN bundle install

COPY . /usr/src/app/

CMD ["bin/rails", "s", "-b", "0.0.0.0"]
```

# C hange file ownership on container files (linux only)
`sudo chown <your-user>:<your-group> -R myapp/`
so on AWS:
`sudo chown ec2-user:ec2-user -R myapp/`

# Building our image

`$ docker build [options] path/to/build/directory`

when you are in the root of your app you can use current dir (.)
`$docker build . `

build lots of throwaway intermediate images per step
and one final one. It is not a file. 

See image with `$ docker images`

# Running our image as a container
on droplet:
`docker run -p 3000:3000 <image_id> bin/rails s -b 0.0.0.0`
then go to `http://<droplet_ip_adress>:3000`

on aws
`docker run -p 8080:3000 <image_id> bin/rails s -b 0.0.0.0`

`-p` port localport:containterport
`-b` binding container to an ip adress. 0.0.0.0 is a special
     adress that means 'all IPv4 address on this machine'. 
     You need this because rails listens to localhost that is not
     accessible from inside the container which listens to 
     [ip addres of container]:3000 instead of localhost:3000
     
running a named/taged image:
`docker run -p <local port>:<containter_port> <image_name: tag> bin/rails s -b 0.0.0.0`

running something else in your container (not the rails sever) like listing
our Rake tasks:
`docker run -p 8080:3000 railsapp bin/rails -T`

# Tagging existing image

Adding a image name
`$ docker tag <image id> <new_image_name>` nb image name = repository
example:
`docker tag be99ae8996a0 railsapp`

Adding a tag (other than the default 'latest') to image
`$ docker tag <image_name> <image_name:tag>`
example:
`docker tag railsapp railsapp:1.0`

note same image id:
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
railsapp            1.0                 be99ae8996a0        20 hours ago        1.02GB
railsapp            latest              be99ae8996a0        20 hours ago        1.02GB
ruby                2.6                 2d6f7a467f2c        9 days ago          870MB
```

# Tagging new image when building with -t

`$ docker build -t <image_name> -t <image_name:tag>`
example
`docker build -t railsapp -t railsapp:1.0`


# Dockerignore
add a .dockerignore file to your project

```
# Git
.git
.gitignore

# Logs
log/*
```

# Actions:

- change dockerfile ->
  - rebuild image `docker build -t railsapp .`
  - run new image `docker run -p 8080:3000 railsapp`

# Docker compose config file

NB: docker compose needs to be installed seperatly
https://docs.docker.com/compose/install/

docker-compose.yml file. Note de dash instead of underscore.

we are using version 3 of docker-compose. Find latest on:
docs.docker.com/compose/compose-file/compose-versioning/

The 'web' service
We are calling our rails server 'web', it needs to be build
(since it's a custom image) and we are running on AWS (8080 instead
of port 3000)
We are also mounting (`-v`) our local directory to the /usr/src/app
in the container

```
version: '3'

services:

  web:
    build: .
    ports:
      - "8080:3000"
    volumes:
      - .:/usr/src/app
```

note:
switch of Ruby's output buffering -> add to top config/boot:
`$stdout.sync = true`

# Running docker compose

Compose up:
`$ docker-compose up`  exit with ^C

or in detached mode:
`$ docker-compose up -d`

Check running services:
`docker-compose ps`

Stop compose:
- `docker-compose stop`
- `docker-compose stop <service_name>`

Start individual service
- `docker-compose start <service_name>`
- `docker-compose restart <service_name>` picks up config changes

Stop and cleanup
`docker-compose down`

# Diverse docker-compose commands

View container logs
`docker-compose logs -f <service_name>` ^C to quit

Run an one-off command
`docker-compose run --rm <service_name> <command>` in a new 
 seperate container.
 example: `docker-compose run --rm web echo 'a random command'>`

`docker-compose exec <service_name> <command>` in a running container

Rebuilding images
`docker-compose build <service_name>`

Cleaning up dangling images:
`docker image prune`

Cleaning up dangling containers:
`docker image prune`

Cleaning up all:
`docker system prune`
