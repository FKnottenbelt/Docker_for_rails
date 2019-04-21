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

# Change file ownership on container files (linux only)
`sudo chown <your-user>:<your-group> -R myapp/` or
`sudo chown <your-user>:<your-group> -R .`
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

The 'redis' service
This is an off the shelf image, so no building.
Since we only use it from inside our app, no ports, no volumes.

The 'database' service
Also an off the shelf image, so no building.
Since we only use it from inside our app, no ports, no volumes.
The image has an build in CMD ["postgres"] which starts the postgres
server. It comes with psql build in. (and with pager set to off :-).

We add env files to host our database variables.

We use a named volume to store our data seperatly from the container.
We define the named volume as db_data. Then we mount this at
the /var/lib/postgresql/data directory.

```
version: '3'

services:

  web:
    build: .
    ports:
      - "8080:3000"
    volumes:
      - .:/usr/src/app
    env_file:
      - .env/development/web
      - .env/development/database
  
  redis:
    image: redis
    
  database:
    image: postgres
    env_file:
      - .env/development/database
    volumes:
      - db_data:/var/lib/postgresql/data
    
volumes:
  db_data:
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

Recreate a services container (when you changed some config)
`docker-compose up -d --force-recreate web`

Workflow after changing anything in your build (that has impact in your dockerfile):
`docker-compose build <service>`
`docker-compose stop <service>`
`docker-compose up -d --force-recreate <service>`

# Diverse docker-compose commands

View container logs
`docker-compose logs -f <service_name>` ^C to quit

Interactive session with access to other containers
Everything is up except for the container you need (web in this case)
`docker-compose run --service-ports web`
usefull for debugging with byebug

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

List networks:
`docker network ls`

# Rails in Docker examples

Generating a welcome controller with an index action
`docker-compose exec web bin/rails g controller welcome index`

Don't forget to chown the new files.

Creating our development and test databases
`docker-compose run --rm web bin/rails db:create`

Generate basic UserContoller
`docker-compose exec web \
   bin/rails g scaffold User \
   first_name:string last_name:string`
   
And running the migration
`docker-compose exec web bin/rails db:migrate`

# Postgresql in Docker examples

`docker-compose run --rm database psql -U postgres -h database`
- `-U` user
- `-h` host

Put your database variables in env files. One per service:
.env/development/web:
```
DATABASE_HOST=database
```
.env/development/database:
```
POSTGRES_USER=postgres
POSTGRES_PASSWORD= secret
POSTGRES_DB= myapp_development
```

Creating our development and test databases
`docker-compose run --rm web bin/rails db:create`

Note: when mounting volume goes wrong and you get password error
not only remove database, but also remove volume
`docker volume rm myapp_db_data`.
List volumes with `docker volume ls`

# JS in docker: rails as an API

If you have a seperate front end:
- Rename you web service, since it's now more of an API
- create a custom image for running your JavaScript front-end app
- create a separate, front-end service in your docker-compose.yml

# Rails JS frontend with Webpacker

NB: I can't get this to work, so might need editing.

Webpacker requires Yarn. Which needs an update to the Dockerfile
```
# Allow apt to work with https-based sources
RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends \
  apt-transport-https

# Ensure we install an up-to-date version of Node
# see https://github.com/yarnpkg/yarn/issues/2888
RUN curl -sL https://deb.nodesource.com/setup_8.x | bash -

# Ensure latest packages for Yarn
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | \
  tee /etc/apt/sources.list.d/yarn.list
```

and add Yarn to the to install packages.

Add webpacker gem to Gemfile

Install webpacker:
`docker-compose run web bin/rails webpacker:install`
and webpacker intergration:
`docker-compose run web bin/rails webpacker:install:react`

Add the webpacker deb server to the docker-compose.yml
```
  web:
    build: .
    ports:
      - "8080:3000"
    volumes:
      - .:/usr/src/app
    env_file:
      - .env/development/web
      - .env/development/database
    environment:
      - WEBPACKER_DEV_SERVER_HOST=webpack_dev_server

  webpack_dev_server:
    build: .
    command: ./bin/webpack-dev-server
    ports:
      - 3035:3035
    volumes:
      - .:/usr/src/app
    env_file:
      - .env/development/web
      - .env/development/database
    environment:
      - WEBPACKER_DEV_SERVER_HOST=0.0.0.0
 ```

# Adding RSpec

In the Gemfile add rspec-rails gem:
```
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
  gem 'rspec-rails', '~> 3.8'
end
```

Install RSpec (after rebuilding):
`docker-compose exec web bin/rails generate rspec:install`
(chown files)

Run tests:
`docker-compose exec web bin/rails spec`

Generate testfiles:
`docker-compose exec web bin/rails generate rspec:model user`

# system tests with capybara
```ruby
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger c$
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
  gem 'rspec-rails', '~> 3.8'
  gem 'capybara', '~> 3.7'
end
```
add a directory spec/sytem
and add a spec.rb file and write test.

Switch to RackTestdriver:
add to spec/railshelper.rb (in RSpec.configure do)
```
config.before(:each, type: :system) do
  driven_by :rack_test
end
```

Run your systemstest:
`docker-compose exec web rspec spec/system/`

# Testing Javascript
To test js, write RSpec test with js: true
add Capybara and Selenium driver gems:
```ruby
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger con$
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
  gem 'rspec-rails', '~> 3.8'
  gem 'capybara', '~> 3.7'
  gem 'selenium-webdriver', '~> 3.14'
end
```

add a Chrome container:
```
  selenium_chrome:
    image: selenium/standalone-chrome-debug
    logging:
      driver: none
    ports:
      - '5900:5900'
```
bring your new service up:
`docker-compose up -d selenium_chrome`

Configure Capybara to use Chrome running in a container by
registering the selenium driver, connecting our host
`selenium_chrome` service with the port and url that Selenium
listens to. ('http://<host>:4444/wd/hub')
in spec/support/capybara.rb
```ruby
Capybara.register_driver :selenium_chrome_in_container do |app|
  Capybara::Selenium::Driver.new app,
    browser: :remote,
    url: 'http://selenium_chrome:4444/wd/hub'
    desired_capabilities: :chrome
end
```
