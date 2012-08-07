#Bundler

###New way of thinking
With Bundler, you need to shift your thinking from "what gems are installed on my system" to
"what gems does my app use". With Bundler, Rails apps don't see, or care, what gems are installed
on the system. The gems installed on the system are essentially irrelevant. Instead, Bundler
provides a "sandbox" environment for each app containing a well-defined gemset specific to the app.
The gems are visible to that app when it executes.

###Bundler provides a well-defined gemset for each Rails app
When you run a Rails 3 app, or run an executable within the scope of the app with `"bundle exec"`,
Bundler creates an environment containing the specific gems described in the Gemfile. Bundler
defaults to using `--disable-shared-gems` which means that the Rails app and commands run with
`"bundle exec"` only see gems that are specifically called out in the Gemfile (plus dependencies
of these gems). They will not see gems installed in the system.

Bundler is integrated into Rails 3. No extra work is required. The Rails app sees the environment
Bundler provides, and thus only sees gems specified in the Gemfile.

IMPORTANT: Any executable you want to run within the scope of the app (`rake`, `cap`, etc) must be run with
`"bundle exec"`

    bundle exec some-command
  
This means you will use `"bundle exec"` all the time when working within your rails app.

###Bundler in stand-alone Ruby apps
To use Bundler in a Ruby app, add a require statement to your main application file.

    require "bundler"

This allows Bundler to control the load path and give the application access to the gems/versions
called out in the Gemfile.

When you run your app with `bundle exec`, Bundler automatically includes itself in the
application. In this case, the `require "bundler"` line is redundant.

To have Bundler automatically include all the gems in the Gemfile, add a Bundler.require
statement to your main application file.

    Bundler.require(:default)

###Bundler doesn't always install gems in the standard system location
When you install gems with `"bundle install"`, Bundler figures out the best place to put the
gems. The location is based on several factors including whether Bundler is running on a
development system or production system. For the most part, you don't need to worry about
how it works. You can trust that Bundler is making good decisions.

On development systems, bundler installs gems in the standard system location so they are shared
between applications, minimizing redundant gems. On production systems, Bundler should be called
with the `--deployment` flag. Among other things this tells Bundler to install gems in a directory
within the app itself. This duplicates gems, but there are good reasons to do it. See "Deploying
Your Application" section [http://gembundler.com/rationale.html](http://gembundler.com/rationale.html).

Bundler controls what gems Ruby sees through several mechanisms, which vary based on the environment.
Mechanisms used by Bundler are:

* Set GEM_HOME
* Clear GEM_PATH
* Add an -I flag to RUBYOPT (see "Bundler has special rules for itself" section below).

###Bundler with Capistrano
Capistrano improves on Bundler's default `--deployment` behavior and creates a gem directory that is
shared between all versions of a given app (no reason to reinstall gems every time you deploy).
Capistrano uses the command below to install the gems.

    bundle install --gemfile /mnt/your-app/releases/20110708214336/Gemfile --path /mnt/your-app/shared/bundle --deployment --quiet --without development test"

###Bundler has special rules for itself
Bundler only stores one copy of itself, in the standard system location. Bundler never installs
itself in an app-specific gem folder. Bundler adds a flag to RUBYOPT so Ruby can find it.
For example, on RVM installations, this flag is `"-I/usr/local/rvm/gems/ruby-1.9.2-p180/gems/bundler-1.0.15/lib"`
In that case RUBYOPT looks like this:

    RUBYOPT='-I/usr/local/rvm/gems/ruby-1.9.2-p180/gems/bundler-1.0.15/lib -rbundler/setup'

Bundler adds these flags in addition to what's already in RUBYOPT. Without the -I flag,
`"require 'bundler/setup'"` may not be able to find Bundler (for example on a production system
where all gems are stored in an app-specific gem folder)

I discovered this because cron jobs on a server were failing with the error:

    *** Process exited with non-zero error code, full output follows
    *** Command was: rake --trace rubber:backup_db

    rake aborted!
    no such file to load -- bundler/setup
    <internal:lib/rubygems/custom_require>:29:in `require'
    <internal:lib/rubygems/custom_require>:29:in `require'
    /mnt/cms-production/releases/20110714204822/config/boot.rb:8:in `<top (required)>'
    <internal:lib/rubygems/custom_require>:29:in `require'
    <internal:lib/rubygems/custom_require>:29:in `require'
    ...and so on

This error was caused because Rubber added a `"RUBYOPT=rubygems"` line to crontab which overwrote
the Bundler settings in RUBYOPT. I've since updated Rubber to prevent it from adding the
redundant RUBYOPT line, but wanted to document the behavior here because it was difficult
to figure out.

###Special locations for gems retrieved from git
On production systems, Bundle stores gems retrieved from git in a separate folder from gems installed
from http://rubygems.org. For example, in systems setup by Rubber, the path for gems installed from
http://rubygems.org is

    /mnt/your-app/shared/bundle/ruby/1.9.1/gems
    
while gems retrieved from git are stored in

    /mnt/your-app/shared/bundle/ruby/1.9.1/bundler/gems/
    
The "...bundler/gems/" directory is not included in the `GEM_PATH` or `GEM_HOME`. Somehow Ruby is
still able to find these gems, with Bundler's help. I'm not sure how it works. But if you remove
the `"require 'bundler/setup'"` statement from your app, Ruby cannot find gems retrieved from git.

###Helpful Commands
Check to see if all app dependencies are met and install gems that are needed to meet the dependencies.

    bundle install

Check to see if all app dependencies are met.

    bundle check

Show where a bundled gem is installed (similar to "bundle exec gem which gemname")

    bundle show gemname
    
Update a gem to the latest version - within the limits specified in the gem file. In particular, this
is needed to force Bundler to update a gem pulled from git. If changes have been pushed to the repo,
you must use `"bundle update`" to force Bundler to bring down the latest. The commit hash is stored in
Gemfile.lock so that when you deploy, Bundler can ensure the deployed app is using the same commit.

    bundle update gem-name

Update all gems to the latest version - within the limits specified in the gem file.

    bundle update

Bundle stores its current state in a config file in the app (frozen, path, disable\_shared\_gems, without, etc).
See "Grouping Your Dependencies" [http://gembundler.com/rationale.html](http://gembundler.com/rationale.html)
To view the current configuration use the command.

    bundle config
    
Open the contents of a gem in the text editor specified in your shell ($EDITOR or $BUNDLE_EDITOR)

    bundle open gem-name

#RubyGems
###Helpful Commands
See which specific gem is being used and where it's located.

    gem which gemname
    
In the context of a Rails app this becomes.

    bundle exec gem which gemname
    
Cleanup all versions of a gem, except for the latest one. If no gemname is given in the command
line, cleanup will be run agains all installed gems.

    gem cleanup gemname
    
Display environment that RubyGems is running in

    gem environment

### Requiring 'rubygems' no longer necessary
Ruby 1.9 includes RubyGems by default so you do not need a `require 'rubygems'` statement in
order to load gem libraries. Just use `require 'gemname'` statements as needed.


#Use RSpec for testing
Use RSpec instead of Test::Unit in a Rails app. (Make sure to add rspec-rails gem to your Gemfile.)

    rails new app_name -T       # create Rails app without Test::Unit
    cd sample_app
    rails g rspec:install       # generate RSpec files

Remove test for views and helpers.

    rm -rf spec/views
    rm -rf spec/helpers
    
Run tests: all tests, specify subdirectory, specify a spec file, specify specific specs

    bundle exec rspec spec/
    bundle exec rspec spec/controllers/
    bundle exec rspec spec/controllers/your_spec.rb
    bundle exec rspec spec/controllers/your_spec.rb -e "partial text from spec description"

###Use Guard to run tests automatically
<http://railscasts.com/episodes/264-guard>

Run Guard in its own terminal tab.

    bundle exec guard

If we use Spork to preload the Rails environment (see below), RSpec will simply be connecting
to a pre-loaded Rails environment. As long as the system version of RSpec matches the version of
RSpec in our Gemfile, we don't need to use `"bundle exec"` when running RSpec. We can edit
our Guardfile and add `:bundler => false` to the list of options. This saves a few seconds each
time Guard runs our test suite.

    guard 'rspec', :version => 2, :bundler => false do
      watch(%r{^spec/.+_spec\.rb})
      watch(%r{^lib/(.+)\.rb})     { |m| "spec/lib/#{m[1]}_spec.rb" }
      .
      .
      .

###Use Spork to speed up tests
<http://railscasts.com/episodes/285>

"Spork improves the loading time of your test suite by starting up your Rails application once
in the background. Use it with Guard for the ultimate combo in fast feedback while doing TDD"

#Create New Rails App with MySQL and RSpec

    rails new app_name -T -d mysql   # create Rails app without Test::Unit
    
Edit Gemfile and add a line for rspec-rails.

    group :development, :test do
      gem 'rspec-rails'
      ...
    end

Run Bundler, setup the database and generate the RSpec files

    cd app_name
    bundle
    rake db:migrate             # setup the initial database
    rails g rspec:install       # generate RSpec files

#Generate Stuff
Generate a controller

    rails g controller Users new
    rails g controller Pages home contact about
    rails g controller Sessions new
    
Generate a controller without specs. NOTE: Ryan Bates doesn't use controller specs
because the requests specs (i.e. integration tests) test the controller and view
logic well enough. If logic is too complex for the request spec then it should
probably go in the model.

    rails g controller Sessions new --no-test-framework

Generate a model

    rails g model User field1:string field2:string
    rails g model Micropost content:string user_id:integer

Create a migration

    rails g migration add_email_uniqueness_index
    rails g migration add_password_to_users encrypted_password:string
    rails g migration add_salt_to_users salt:string
    rails g migration add_admin_to_users admin:boolean

Generate an integration test (i.e. request spec). Integration test will be created as spec/requests/some_name_spec.

    rails g integration_test some_name


#Misc
Show list of current routes

    bundle exec rake routes

Database commands

    rake db:migrate             # Runs migrations that have not run yet.
    rake db:create              # Creates the database
    rake db:drop                # Deletes the database
    rake db:schema:load         # Creates tables and columns within the database following schema.rb
    rake db:setup               # Does db:create, db:schema:load, db:seed
    rake db:reset               # Does db:drop, db:setup  

Rollback the last db migration / last 3 migrations

    bundle exec rake db:rollback            # just the last migration
    bundle exec rake db:rollback STEP=3     # last three migrations

Rollback and redo the last db migration (typically after you've corrected a problem with
the migration

    bundle exec rake db:migrate:redo            # redo the last migration
    bundle exec rake db:migrate:redo STEP=3     # rollback and re-apply the last three migrations

Prepare the test database after adding new migrations.

    bundle exec rake db:test:prepare

Refresh the thumbnails generated by paperclip. You must specify the name of the class
that contains the paperclip attachments. There are currently problems with this.
Paperclip updates the created_at date for the model which causes it to save a new
version of the attachments in the CMS app.

    bundle exec rake paperclip:refresh:thumbnails CLASS=Contract --trace
