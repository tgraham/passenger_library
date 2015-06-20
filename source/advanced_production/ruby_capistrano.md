---
title: Automating deployments of Ruby application updates through Capistrano
section: advanced_production
sidebar: sidebar
---
# Automating deployments of Ruby application updates through Capistrano

<figure><a href="http://capistranorb.com/"><img src="CapistranoLogo.png" alt="Capistrano"></a></figure>

If you have followed the [Ruby deployment walkthrough](../walkthroughs/deploy/ruby/), then you know that deploying application updates takes multiple steps. Performing all these steps every time you want to deploy application updates is time-consuming and error-prone.

This guide teaches you how to automate the deployment of application updates through [Capistrano](http://capistranorb.com/). Capistrano is a popular task automation tool among Ruby developers. Once Capistrano is set up, deploying further application updates only takes a single command.

**Note:** this guide assumes that you know how to manually deploy Ruby applications and how to manually deploy updates. Capistrano only makes sense if you have that prior knowledge. If you are not experienced in deploying manually, please read the [Ruby deployment walkthrough](../walkthroughs/deploy/ruby/).

**Table of contents**

<ol class="toc-container"><li>Loading...</li></ol>

## Fundamental concepts

### Basic Capistrano functionality

Capistrano is a tool for automating the execution of commands on remote servers through SSH. In its most basic form, Capistrano expects that you supply it with several files:

 * A Capistrano script, which not only defines the commands that Capistrano should execute, but also defines which servers those commands should execute on. This script is in Ruby syntax.
 * One or more configuration files which tell Capistrano which servers exist, and how to login to them.

Capistrano runs locally -- on your computer -- and logs in to one or more remote servers via SSH to execute commands. Because the Capistrano script is in Ruby syntax, and because Capistrano runs locally, the Capistrano script can use arbitrary Ruby code to process the result of each command execution, for example to determine what command to execute next.

Capistrano can also run commands in parallel on multiple servers. This is especially useful in scenarios where you have scaled out your application to multiple servers, and want to deploy to each one of them simultaneously.

### Recipes

While Capistrano's basic functionality is useful, its real power lies in pre-written recipes that the community has made available.

For example, by using the `capistrano/deploy` recipe, it will automatically run `git clone` or `git pull` for you. It will also manage directories in such a way as to make rolling back to previous versions easy ([the Capistrano-style directory structure](#capistrano-style-directory-structure)). The only thing you have to do is to fill in some blanks, e.g. you need to tell it the URL of your Git repository.

As another example, by using the [capistrano-rails](https://github.com/capistrano/rails/) recipe, it will automatically take care of running `bundle install` for you, compiling Rails assets for you, running Rails database migrations for you, etc.

### Workflow

The Capistrano workflow is as follows.

 1. First, you need to install and configure Capistrano.
 2. Every time you are ready to deploy a new version of your application, you need to:
    - Commit and push all changes to your Git repository.
    - Run the Capistrano deploy command.

Both of these are covered in this guide.

### Capistrano-style directory structure

In the [deployment walkthrough](../walkthroughs/deploy/ruby/), we simply cloned our application code to `/var/www/myapp/code` and instructed Passenger serve the app from there. But if you use Capistrano's `capistrano/deploy` recipe to clone/pull from Git, it will use [a strictly defined directory structure](http://capistranorb.com/documentation/getting-started/structure/) on the server. Here is an example of how `/var/www/myapp` -- an example application directory -- will look like when Capistrano is used:

<pre class="highlight">
myapp
 ├── <span class="label label-info">1</span> releases
 │       ├── <span class="s">20150080072500</span>
 │       ├── <span class="s">20150090083000</span>
 │       ├── <span class="s">20150100093500</span>
 │       ├── <span class="s">20150110104000</span>
 │       └── <span class="m">20150120114500</span>
 │            ├── &lt;checked out files from Git repository&gt;
 │            └── config
 │                 ├── <span class="label label-info">5</span> <span class="o">database.yml</span> -&gt; /var/www/myapp/shared/config/database.yml
 │                 └── <span class="label label-info">6</span> <span class="o">secrets.yml</span>  -&gt; /var/www/myapp/shared/config/secrets.yml
 │
 ├── <span class="label label-info">2</span> <span class="o">current</span> -&gt; /var/www/myapp/releases/<span class="m">20150120114500</span>/
 ├── <span class="label label-info">3</span> repo
 │       └── &lt;VCS related data&gt;
 └── <span class="label label-info">4</span> shared
         ├── &lt;linked_files and linked_dirs&gt;
         └── config
              ├── database.yml
              └── secrets.yml
</pre>

~~~ruby
class Bar < Baz
  def foo
    puts 123 + "456"
  end

  %Q(hello world)
  %w(hello world)
end
~~~

 1. `releases` holds all deployments in a timestamped folder. Every time you instruct Capistrano to deploy, Capistrano makes clones the Git repository to a new subdirectory inside `releases`.
 2. `current` is a symlink pointing to the latest release inside the `releases` directory. This symlink is updated at the end of a successful deployment. If the deployment fails in any step the current symlink still points to the old release.
 3. `repo` holds a cached copy of the Git repository, for making subsequent Git pulls faster.
 4. `shared` is meant to contain any files that should persists across deployments and releases.

    Because a Capistrano deploy works by cloning the Git repository into a `releases` subdirectory, only files that are version controlled will survive deployments. But sometimes you want more files to survive, for example configuration files (e.g. Rails's config/database.yml and config/secrets.yml), log files, and persistent user storage handed over from one release to the next.

    You are supposed to put those kinds of files in `shared`, while instructing Capistrano to symlink them into a release directory on every deploy. This is done through the `linked_files` and `linked_dirs` configuration options, which we will cover in this guide.

    (5) and (6) show this mechanism in action.

The advantage over the simple `/var/www/myapp/code` approach in the deployment walkthrough is twofold:

 1. It makes deployments atomic. If a deployment fails, the currently running version of the application is not affected. Users also never get to see a state in which an update is half-deployed.
 2. It makes rolling back to previous releases dead-simple. Simply change the `current` symlink, tell Passenger to restart the app, and done.

## Initializing Capistrano

Now that you understand Capistrano's fundamental concepts, it is time to get started. The first thing you need to do is to install Capistrano into your Ruby project. Open your Gemfile and add:

~~~ruby
group :development do
  gem 'capistrano'
  gem 'capistrano-bundler'
  gem 'capistrano-passenger'

  # Remove the following if your app does not use Rails
  gem 'capistrano-rails'

  # Remove the following if your server does not use RVM
  gem 'capistrano-rvm'
end
~~~

Then install the gem bundle and initialize Capistrano:

<pre class="highlight">
<span class="prompt">$ </span>bundle install
<span class="output">...</span>
<span class="prompt">$ </span>bundle exec cap install
<span class="output">mkdir -p config/deploy
create config/deploy.rb
create config/deploy/staging.rb
create config/deploy/production.rb
mkdir -p lib/capistrano/tasks
create Capfile
Capified</span>
</pre>

## Editing Capfile

`Capfile` is the Capistrano entry point. It defines what recipes to load. You must edit it to load the recipes you need.

### Anatomy

If you open Capfile you will see:

~~~ruby
# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/deploy'

# Include tasks from other gems included in your Gemfile
#
# For documentation on these, see for example:
#
#   https://github.com/capistrano/rvm
#   https://github.com/capistrano/rbenv
#   https://github.com/capistrano/chruby
#   https://github.com/capistrano/bundler
#   https://github.com/capistrano/rails
#   https://github.com/capistrano/passenger
#
# require 'capistrano/rvm'
# require 'capistrano/rbenv'
# require 'capistrano/chruby'
# require 'capistrano/bundler'
# require 'capistrano/rails/assets'
# require 'capistrano/rails/migrations'
# require 'capistrano/passenger'

# Load custom tasks from `lib/capistrano/tasks' if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
~~~

I have covered the `capistrano/deploy` recipe in the introduction. It is loaded by default. But a few other recipes that you may want are not loaded by default (because they are uncommented).

### Editing instructions

We will want Capistrano to automatically run `bundle install`, and we will want Capistrano to automatically tell Passenger to restart our app. These are taken care of by the [capistrano-bundler](https://github.com/capistrano/bundler) and [capistrano-passenger](https://github.com/capistrano/passenger) recipes. So make sure that the following lines are uncommented:

~~~ruby
require 'capistrano/bundler'
require 'capistrano/passenger'
~~~

If your app is a Rails app, then uncomment the following lines to load the [capistrano-rails](https://github.com/capistrano/rails) recipe:

~~~ruby
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
~~~

If your server uses RVM, then uncomment the following to activate the [capistrano-rvm](https://github.com/capistrano/rvm) recipe:

~~~ruby
require 'capistrano/rvm'
~~~

Capistrano also has recipes for rbenv and chruby support. Those are outside the scope of this document, so if you want to use them, be sure to check the [capistrano-rbenv](https://github.com/capistrano/rbenv) and [capistrano-chruby](https://github.com/capistrano/chruby) websites for documentation.

## Editing config/deploy.rb

The next step is to edit config/deploy.rb. This file contains configuration values that control how the loaded recipes should do their jobs. It also defines additional commands to be executed on servers. You must edit it according to your situation.

### Anatomy

If you open the file, you should see this:

~~~ruby
# config valid only for current version of Capistrano
lock '3.3.5'

set :application, 'my_app_name'
set :repo_url, 'git@example.com:me/my_repo.git'

# Default branch is :master
# ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }.call

# Default deploy_to directory is /var/www/my_app_name
# set :deploy_to, '/var/www/my_app_name'

# Default value for :scm is :git
# set :scm, :git

# Default value for :format is :pretty
# set :format, :pretty

# Default value for :log_level is :debug
# set :log_level, :debug

# Default value for :pty is false
# set :pty, true

# Default value for :linked_files is []
# set :linked_files, fetch(:linked_files, []).push('config/database.yml')

# Default value for linked_dirs is []
# set :linked_dirs, fetch(:linked_dirs, []).push('bin', 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system')

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for keep_releases is 5
# set :keep_releases, 5

namespace :deploy do

  after :restart, :clear_cache do
    on roles(:web), in: :groups, limit: 3, wait: 10 do
      # Here we can do anything such as:
      # within release_path do
      #   execute :rake, 'cache:clear'
      # end
    end
  end

end
~~~

There are a few things you see here.

#### Variables

The various `set` calls set [configuration values](http://capistranorb.com/documentation/getting-started/configuration/). These configuration values are used by the recipes that are loaded in the Capfile. Here are a few of the configuration variables explained:

 * `application`: used by the `capistrano/deploy` recipe to determine which directory to deploy to. Closely related to this is the `deploy_to` variable: its default value depends on the `application` variable. So if you set `application` to `hello`, Capistrano will deploy to `/var/www/hello` unless you further customize `deploy_to`.
 * `repo_url`: used by the `capistrano/deploy` recipe to determine where to clone/pull code from.
 * `linked_files` and `linked_dirs`: used by the `capistrano/deploy` recipe to link files and directories from the `shared` directory into a release directory.

#### Tasks

Inside the `namespace` block, you can define additional commands to execute during a deployment process. For simple apps, no additional commands need to be executed, but for more complex apps this flexibility is welcome.

This works by defining tasks that hook into the various steps that the `capistrano/deploy` recipe executes. Roughly speaking, a typical Capistrano script that has a few popular recipes loaded, perform the following steps:

 1. Cloning code into a release directory
 2. Setting up linked files and directory
 3. Running bundle install, running database migrations, etc
 4. Changing the `current` symlink
 5. Restarting the application server (Passenger, in our case)

The `after :restart, :clear_cache` block you see means "define a new task named :clear_cache, and run it after the restart step (step 5)". Inside the block you can do anything you want, such as clearing the Rails cache with `rake cache:clear` as per the example.

### Editing instructions

 1. Please remove the `lock` statement. The statement is meant to lock down the Capistrano script to a certain Capistrano version. But Bundler [already serves as a mechanism to lock down gem versions](http://bundler.io/rationale.html), so the `lock` statement is quite redundant here.
 2. Set `application` to your application's name. Please note that this affects which directory Capistrano deploys to as explained earlier.
 3. Set `repo_url` to your Git repository's URL.
 4. If your app is a Rails app, set `linked_files` and `linked_dirs`:

    ~~~ruby
    set :linked_files, fetch(:linked_files, []).push('config/database.yml', 'config/secrets.yml')
    set :linked_dirs, fetch(:linked_dirs, []).push('log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system', 'public/uploads')
    ~~~
 5. If your server uses RVM, set `rvm_ruby_version` to the Ruby version that your application uses. For example:

    ~~~ruby
    set :rvm_ruby_version, '2.2.1'
    ~~~
 6. Depending on how Passenger was installed, you may need to set some configuration values.

     * If you installed Passenger through source tarball (i.e. not through APT, YUM, RubyGems, etc), then set `passenger_environment_variables` and `passenger_restart_command` so that Capistrano can find Passenger:

       ~~~ruby
       set :passenger_environment_variables, { :path => '/path-to-passenger/bin:$PATH' }
       set :passenger_restart_command, '/path-to-passenger/bin/passenger-config restart-app'
       ~~~

       Replace `/path-to-passenger` with the actual path that Passenger is installed in. You can find the correct value for `/path-to-passenger` by logging into your server and running `passenger-config --root`.

     * If you are using Passenger Standalone, and you have `passenger` as an entry in your Gemfile, set:

       ~~~ruby
       set :passenger_in_gemfile, true
       ~~~

## Editing config/deploy/production.rb

The next step is to edit config/deploy/production.rb. This file defines the servers that Capistrano should deploy to, in the form of SSH login information.

### Anatomy

If you open the file, you should see this:

~~~ruby
# Simple Role Syntax
# ==================
# Supports bulk-adding hosts to roles, the primary server in each group
# is considered to be the first unless any hosts have the primary
# property set.  Don't declare `role :all`, it's a meta role.

role :app, %w{deploy@example.com}
role :web, %w{deploy@example.com}
role :db,  %w{deploy@example.com}


# Extended Server Syntax
# ======================
# This can be used to drop a more detailed server definition into the
# server list. The second argument is a, or duck-types, Hash and is
# used to set extended properties on the server.

server 'example.com', user: 'deploy', roles: %w{web app}, my_property: :my_value


# Custom SSH Options
# ==================
# You may pass any option but keep in mind that net/ssh understands a
# limited set of options, consult[net/ssh documentation](http://net-ssh.github.io/net-ssh/classes/Net/SSH.html#method-c-start).
#
# Global options
# --------------
#  set :ssh_options, {
#    keys: %w(/home/rlisowski/.ssh/id_rsa),
#    forward_agent: false,
#    auth_methods: %w(password)
#  }
#
# And/or per server (overrides global)
# ------------------------------------
# server 'example.com',
#   user: 'user_name',
#   roles: %w{web app},
#   ssh_options: {
#     user: 'user_name', # overrides user setting above
#     keys: %w(/home/user_name/.ssh/id_rsa),
#     forward_agent: false,
#     auth_methods: %w(publickey password)
#     # password: 'please use keys'
#   }
~~~

There are two syntaxes for defining servers. The first one is the simple role-based syntax, while the second one is the extended server syntax. _You should only use one of the two syntaxes, not both of them._

What is this "role" thing and why does it exist? Well, it exists mainly to support scenarios that involve multiple servers. Some Capistrano recipes only execute certain commands on servers with a specific role.

Here is a concrete example of how roles are useful. As I mentioned earlier, the `capistrano-rails` recipe automatically runs database migrations. But the thing about database migrations is that it should only be run from one server. If you have scaled your app across 5 servers, then you will want some way to tell Capistrano "run database migrations only on server #1". To solve this problem, you define all 5 servers in production.rb, but only designate the "db" role to server #1. `capistrano-rails` only executes database migrations on servers with the "db" role, and since there is only one, everything goes well.

### Editing instructions

In this guide, we use the simple role syntax. Comment out the code that uses the extended server syntax:

~~~ruby
# Make sure the following is commented out!

# server 'example.com', user: 'deploy', roles: %w{web app}, my_property: :my_value
~~~

Then edit the `role` code. Specify your server's (or servers') host names. As username, you should specify the OS user account that your application uses. (If you do not yet have an OS user account for your application, then that will be covered in the next step, [Preparing the server(s)](#preparing-the-servers).)

For example, if you have one server:

~~~ruby
# Single server example
role :app, %w{myapp@myserver.com}
role :web, %w{myapp@myserver.com}
role :db,  %w{myapp@myserver.com}
~~~

If you have multiple servers, simply add them to the `app` and `web` roles (but *not* the `db` role, because that role dictates which server Rails database migrations are run on). For example:

~~~ruby
# Multi-server example
role :app, %w{myapp@myserver.com myapp@myserver2.com}
role :web, %w{myapp@myserver.com myapp@myserver2.com}
role :db,  %w{myapp@myserver.com}
~~~

If your server is on Amazon EC2, be sure to uncomment the `set :ssh_options` call and point the `keys` option to your Amazon EC2 key file.

~~~ruby
# Global options
# --------------
set :ssh_options, {
  keys: %w(/path-to-your-ec2-key.pem),
  forward_agent: false,
  auth_methods: %w(password)
}
~~~

## Preparing the server(s)

You are now done configuring Capistrano. But before Capistrano can do its work, you need to setup a basic directory structure on the server, create initial configuration files and configure Passenger.

### Install Git

Install Git on your server if you haven't already:

<table class="table table-bordered table-striped">
  <tr>
    <td>Debian, Ubuntu</td>
    <td><pre class="highlight">sudo apt-get install git</pre></td>
  </tr>
  <tr>
    <td>Red Hat, CentOS, Fedora, Scientific Linux, Amazon Linux</td>
    <td><pre class="highlight">sudo yum install git</pre></td>
  </tr>
  <tr>
    <td>Other operating systems</td>
    <td>Please install Git from <a href="https://git-scm.com/">git-scm.com</a>.</td>
  </tr>
</table>

### Setting up a basic directory structure

If you have followed the [deployment walkthrough](../walkthroughs/deploy/ruby/), then you have already setup a user account for your application (`myappuser` in our example), and you have already setup a directory in which your application code lives (`/var/www/myapp/code` in our example). If not, you must set them up now. Login to your server(s), and run the following commands on each one:

<pre class="highlight"><span class="c"># Add user account</span>
<span class="prompt">$ </span>sudo adduser <span class="o">myappuser</span>

<span class="c"># Setup initial SSH key</span>
<span class="prompt">$ </span>sudo mkdir -p ~<span class="o">myappuser</span>/.ssh
<span class="prompt">$ </span>sudo sh -c <span class="s">"cat $HOME/.ssh/authorized_keys >> ~<span class="o">myappuser</span>/.ssh/authorized_keys"</span>
<span class="prompt">$ </span>sudo chown -R <span>myappuser:</span> ~<span class="o">myappuser</span>/.ssh
<span class="prompt">$ </span>sudo chmod <span>700</span> ~<span class="o">myappuser</span>/.ssh
<span class="prompt">$ </span>sudo sh -c <span class="s">"chmod <span>600</span> ~<span class="o">myappuser</span>/.ssh/*"</span>

<span class="c"># Create application directory</span>
<span class="prompt">$ </span>sudo mkdir -p /var/www/<span class="o">myapp</span>/shared
<span class="prompt">$ </span>sudo chown <span class="o">myappuser</span>: /var/www/<span class="o">myapp</span> /var/www/<span class="o">myapp</span>/shared</pre>

Of course, replace `myappuser` with the username that you desire.

### Create initial configuration files

_You can skip this step if your app is not a Rails app._

If your app is a Rails app, then it expects a `config/database.yml` and a `config/secrets.yml`. The contents of these configuration files are identical on each server, are only known to the server(s) and persist across application releases. The `shared` directory is the perfect place to place them. And as configured before in `config/deploy.rb`, Capistrano will automatically create symlinks within the release directory to those files.

Create the `shared/config` directory and add `database.yml` and `secrets.yml` inside:

<pre class="highlight"><span class="prompt">$ </span>sudo mkdir -p /var/www/<span class="o">myapp</span>/shared/config
<span class="prompt">$ </span>nano /var/www/<span class="o">myapp</span>/shared/config/database.yml
<span class="prompt">$ </span>nano /var/www/<span class="o">myapp</span>/shared/config/secrets.yml</pre>

Then fix and tighten permissions:

<pre class="highlight"><span class="prompt">$ </span>sudo chown -R <span class="o">myappuser</span>: /var/www/<span class="o">myapp</span>/shared/config
<span class="prompt">$ </span>chmod 600 /var/www/<span class="o">myapp</span>/shared/config/database.yml
<span class="prompt">$ </span>chmod 600 /var/www/<span class="o">myapp</span>/shared/config/secrets.yml</pre>

### Configure Passenger

As configured in `config/deploy.rb`, Capistrano will deploy app releases to `/var/www/myapp/releases`, with `/var/www/myapp/current` pointing to the latest release. So Passenger must be configured to serve the app from `/var/www/myapp/current/public`.

Passenger + Apache
: Open the Apache configuration file in which your app's virtual host is defined. Modify the `DocumentRoot` directive, and point it to `/var/www/myapp/current/public`. Notice the `public` part after `/var/www/myapp/current`. When done, restart Apache.

Passenger + Nginx
: Open the Nginx configuration file in which your app's virtual host is defined. Modify the `root` directive, and point it to `/var/www/myapp/current/public`. Notice the `public` part after `/var/www/myapp/current`. When done, restart Nginx.

Passenger Standalone
: <p>Stop your Passenger Standalone instance for now. Also, if you created a Passengerfile.json on the server (as per the deployment walkthrough), then it is a good idea to copy that to the `shared` directory:</p>

      cp /var/www/myapp/code/Passengerfile.json /var/www/myapp/shared/

  Modify your `config/deploy.rb` and add Passengerfile.json to `linked_files` so that it is persisted across deployments.

  ~~~ruby
  set :linked_files, fetch(:linked_files, []).push(...previous value..., 'Passengerfile.json')
  ~~~

## Deploying a new release

You are now ready to deploy a new release using Capistrano!

Make a random change in your application, add the various Capistrano config files, then commit and push your changes.

<pre class="highlight"><span class="prompt">$ </span>nano app/somefile.rb
<span class="prompt">$ </span>git add Capfile config/deploy.rb config/deploy lib/capistrano
<span class="prompt">$ </span>git commit -a -m "Test Capistrano"
<span class="prompt">$ </span>git push</pre>

Next, run Capistrano to start the deployment:

<pre class="highlight"><span class="prompt">$ </span>bundle exec cap production deploy</pre>

## Conclusion

Congratulations, you have learned how to automate deployments using Capistrano! To learn more about Capistrano, please [visit its website](http://capistranorb.com/).