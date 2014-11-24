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
  end
  ```

- [ ] Generate Capistrano default configurations via `cap install`

- [ ] Set correct repository path and app name in `config/deploy.rb`
  ```ruby
  set :application, 'my_app_name'
  set :repo_url, 'git@example.com:me/my_repo.git'
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

- [ ] Install additional libraries on server (using apt as an example):
  ```
  sudo apt-get update
  sudo apt-get install libmysql-ruby libmysqlclient-dev libmagickwand-dev
  sudo apt-get install imagemagick
  ```

- [ ] Push all changes to your Git repository and run capistrano deploy for the first time:
  ```
  cap deploy production
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