This is a sample Rails application from the 'Docker for Rails Developers' book.
It was generated using Docker without Ruby installed on the local machine.

Setting up a Docker-based development environment and running it using Docker Compose
- set up a container running the Rails app bases on a Ruby base image
- set up a container running a Redis database to work with the Rails app
- set up a container running a PostgreSQL database to work with the Rails app
- set up unit tests to run in the container using Rspec
- set up systemtest to run non-Javascript tests with Capybara and Rack-test
- set up a container to run Chrome/Selenium container to run Javascript sytemtests
  (both headless and non-headless)
- debug the Ruby code in running in the Rails container with byebug. 