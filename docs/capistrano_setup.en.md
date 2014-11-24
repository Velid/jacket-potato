### Basic Capistrano & unicorn configuration for Rails

This step-by-step guide is intended to be be used as a Github Issue description, feel free to copy everything below this line - it will show up as a checklist on your issue. 

Just click that `Raw` button above to view markdown source.

---
<!-- copy everything below this line --> 

- [ ] Add Capistrano gems and whenever to your Gemfile, `bundle` it afterwards:
  ```ruby
  group :development do
    gem 'capistrano'
    gem 'capistrano-rails'
    gem 'capistrano-bundler'
    gem 'capistrano-rvm'
  end

  group :production do
    gem 'unicorn'
    gem 'mysql2'
    gem 'capistrano3-unicorn'
    gem 'whenever'
    gem 'execjs'
    gem 'therubyracer',  platforms: :ruby
  end
  ```

- [ ] Generate Capistrano default configurations via `cap install`

- [ ] Set correct repository path, app name and other parameters in `config/deploy.rb`. Example:
  ```ruby
  # config valid only for Capistrano 3.1
  lock '3.2.1'

  set :application, 'app_name'

  # setup repo details
  set :scm, :git
  set :repo_url, 'git@github.com:path/to_repository.git'

  # how many old releases do we want to keep
  set :keep_releases, 5

  # Default deploy_to directory is /var/www/my_app
  set :deploy_to, '/application/app_name'
  set :deploy_user, 'deployer'

  # files we want symlinking to specific entries in shared.
  set :linked_files, %w{config/database.yml}

  # dirs we want symlinking to shared
  set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system public/ckeditor_assets}

  # what specs should be run before deployment is allowed to
  # continue, see lib/capistrano/tasks/run_tests.cap
  set :tests, []

  # which config files should be copied by deploy:setup_config
  # see documentation in lib/capistrano/tasks/setup_config.cap
  # for details of operations
  set(:config_files, %w(
    nginx.conf
    log_rotation
    monit
    unicorn.rb
    unicorn_init.sh
  ))

  # which config files should be made executable after copying
  # by deploy:setup_config
  set(:executable_config_files, %w(
    unicorn_init.sh
  ))

  namespace :deploy do

    desc 'Restart application'
    task :restart do
      on roles(:app), in: :sequence, wait: 5 do
        # Your restart mechanism here, for example:
        # execute :touch, release_path.join('tmp/restart.txt')
      end

      invoke 'unicorn:restart'
    end

    after :publishing, :restart

    after :restart, :clear_cache do
      on roles(:web), in: :groups, limit: 3, wait: 10 do
        # Here we can do anything such as:
        # within release_path do
        #   execute :rake, 'cache:clear'
        # end
      end
    end

  end
  ```

- [ ] Set correct server address and username in `config/deploy/production.rb`:
  ```ruby
  # Prepare stages
  # ==============
  set :stage, :production
  set :rails_env, :production

  # Extended Server Syntax
  # ======================
  # This can be used to drop a more detailed server definition into the
  # server list. The second argument is a, or duck-types, Hash and is
  # used to set extended properties on the server.

  server 'example.com', user: 'deploy', roles: %w{web app}, my_property: :my_value
  ```

- [ ] Require libraries in the Capfile, it should look like this:
  ```ruby
  # Load DSL and Setup Up Stages
  require 'capistrano/setup'

  # Includes default deployment tasks
  require 'capistrano/deploy'

  # Unicorn integration
  require 'capistrano3/unicorn'

  # Includes tasks from other gems included in your Gemfile
  #
  # For documentation on these, see for example:
  #
  #   https://github.com/capistrano/rvm
  #   https://github.com/capistrano/rbenv
  #   https://github.com/capistrano/chruby
  #   https://github.com/capistrano/bundler
  #   https://github.com/capistrano/rails
  #
  require 'capistrano/rvm'
  # require 'capistrano/rbenv'
  # require 'capistrano/chruby'
  require 'capistrano/bundler'
  require 'capistrano/rails/assets'
  require 'capistrano/rails/migrations'

  # Whenever integration
  # Used by default to reload your unicorn on reboots
  require "whenever/capistrano"

  # Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
  Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
  Dir.glob('lib/capistrano/**/*.rb').each { |r| import r }
  ```

- [ ] Create new unicorn configuration under `config/unicorn/production.rb`, correct application paths if needed:
  ```ruby
  worker_processes 2

  working_directory "/application/APP_NAME/current" # available in 0.94.0+

  # listen on both a Unix domain socket and a TCP port,
  # we use a shorter backlog for quicker failover when busy
  listen "/application/APP_NAME/shared/tmp/sockets/APP_NAME.unicorn.sock", :backlog => 64
  listen 8080, :tcp_nopush => true

  # nuke workers after 30 seconds instead of 60 seconds (the default)
  timeout 30

  # feel free to point this anywhere accessible on the filesystem
  pid "/application/APP_NAME/shared/tmp/pids/unicorn.pid"

  # By default, the Unicorn logger will write to stderr.
  # Additionally, ome applications/frameworks log to stderr or stdout,
  # so prevent them from going to /dev/null when daemonized here:
  stderr_path "/application/APP_NAME/shared/log/unicorn.stderr.log"
  stdout_path "/application/APP_NAME/shared/log/unicorn.stdout.log"

  # combine Ruby 2.0.0dev or REE with "preload_app true" for memory savings
  # http://rubyenterpriseedition.com/faq.html#adapt_apps_for_cow
  preload_app true
  GC.respond_to?(:copy_on_write_friendly=) and
    GC.copy_on_write_friendly = true

  # Enable this flag to have unicorn test client connections by writing the
  # beginning of the HTTP headers before calling the application.  This
  # prevents calling the application for connections that have disconnected
  # while queued.  This is only guaranteed to detect clients on the same
  # host unicorn runs on, and unlikely to detect disconnects even on a
  # fast LAN.
  check_client_connection false

  before_fork do |server, worker|

    # the following is highly recomended for Rails + "preload_app true"
    # as there's no need for the master process to hold a connection
    
    defined?(ActiveRecord::Base) and
      ActiveRecord::Base.connection.disconnect!

    # The following is only recommended for memory/DB-constrained
    # installations.  It is not needed if your system can house
    # twice as many worker_processes as you have configured.
    #
    # # This allows a new master process to incrementally
    # # phase out the old master process with SIGTTOU to avoid a
    # # thundering herd (especially in the "preload_app false" case)
    # # when doing a transparent upgrade.  The last worker spawned
    # # will then kill off the old master process with a SIGQUIT.
    old_pid = "#{server.config[:pid]}.oldbin"
    if old_pid != server.pid
      begin
        sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
        Process.kill(sig, File.read(old_pid).to_i)
      rescue Errno::ENOENT, Errno::ESRCH
      end
    end
    #
    # Throttle the master from forking too quickly by sleeping.  Due
    # to the implementation of standard Unix signal handlers, this
    # helps (but does not completely) prevent identical, repeated signals
    # from being lost when the receiving process is busy.
    # sleep 1
  end

  after_fork do |server, worker|
    # per-process listener ports for debugging/admin/migrations
    # addr = "127.0.0.1:#{9293 + worker.nr}"
    # server.listen(addr, :tries => -1, :delay => 5, :tcp_nopush => true)

    # the following is *required* for Rails + "preload_app true",
    defined?(ActiveRecord::Base) and
      ActiveRecord::Base.establish_connection


    # if preload_app is true, then you may also want to check and
    # restart any other shared sockets/descriptors such as Memcached,
    # and Redis.  TokyoCabinet file handles are safe to reuse
    # between any number of forked children (assuming your kernel
    # correctly implements pread()/pwrite() system calls)
  end
  ```

- [ ] Add SSH key to the server:
  ```
  ssh-copy-id -i .ssh/id_rsa.pub username:password@remotehost
  ```

- [ ] Create database on the server (MySQL used as an example):
  ```sql
  CREATE DATABASE 'my_db' CHARACTER SET utf8 COLLATE utf8_general_ci;
  ```

  or running standard rake task on your server:
  ```
  bundle exec rake db:create RAILS_ENV=productio
  ```

- [ ] Install additional libraries on server (using apt as an example):
  ```
  sudo apt-get update
  sudo apt-get install libmysql-ruby libmysqlclient-dev libmagickwand-dev
  sudo apt-get install imagemagick
  ```

- [ ] Push all changes to your Git repository and run capistrano deploy for the first time:
  ```
  cap production deploy
  ```

- [ ] On server create `shared/config/database.yml` file containing production credentials. Run deploy for the second time afterwards:
  ```yml
  # Example database configuration
  production:
    adapter: mysql2
    encoding: utf8
    database: potato
    pool: 5
    timeout: 5000
    username: root
    reconnect: true
  ```

- [ ] Use `rails secret` command to generate new production secret for your application, put it at the start of the server's `~/.bashrc` file in the following way:
  ```
  # Secret key for rails application
  export SECRET_KEY_BASE="your_generated_secret_key"
  ```

- [ ] point Nginx configuration to the unicorn socket, assign domain names:
  ```
  upstream application {
      server unix:/application/app_name/shared/tmp/sockets/app_name.unicorn.sock;
  }

  # Define separate server rule, as suggested in http://nginx.org/en/docs/http/converting_rewrite_rules.html
  server {
      listen       80;

      server_name www.example.com;
      return 301 http://example.com$request_uri;
  }

  server {
      listen 80;

      server_name example.com;

      root /application/app_name/current/public;
      try_files $uri @application;

      location @application {
          proxy_pass http://hostname:8080;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $http_host;
          proxy_redirect off;
      }
  }
  ```
