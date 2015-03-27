# Bundler

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
See "Grouping Your Dependencies" <http://gembundler.com/rationale.html> To view the current configuration use
the command.

    bundle config
    
Open the contents of a gem in the text editor specified in your shell ($EDITOR or $BUNDLE_EDITOR)

    bundle open gem-name

Check for outdated versions of gems. Particularly useful after a major gem update

    bundle outdated

# RubyGems

See which specific gem is being used and where it's located.

    gem which gemname
    
In the context of a Rails app this becomes.

    bundle exec gem which gemname
    
Cleanup all versions of a gem, except for the latest one. If no gemname is given in the command
line, cleanup will be run against all installed gems.

    gem cleanup gemname
    
Display environment that RubyGems is running in

    gem environment

Uninstall all gems on the system (to start fresh if desired). Ignore warnings about
default gems. Uninstalling default gems will fail and be skipped.

    for i in `gem list --no-versions`; do sudo gem uninstall -aIx $i; done

Refresh all gems from their source and rebuilds all extensions and regenerates bin stubs

    gem pristine --all

### Requiring 'rubygems' no longer necessary
Ruby 1.9 includes RubyGems by default so you do not need a `require 'rubygems'` statement in
order to load gem libraries. Just use `require 'gemname'` statements as needed.

# Create New Rails App with MySQL RSpec

    rails new app_name -T -d mysql    # create Rails app without Test::Unit
    
Run Bundler, setup the database

    cd app_name
    bundle --without production   # (--without is a remembered options, only needed the first time)
    rake db:migrate               # setup the initial database

Setup RSpec (must add `gem 'rspec-rails'` to Gemfile)

    rails g rspec:install

# Use Guard to run tests automatically

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

## Use Spork to speed up tests (Outdated. I use Spring now)

<http://railscasts.com/episodes/285>

"Spork improves the loading time of your test suite by starting up your Rails application once
in the background. Use it with Guard for the ultimate combo in fast feedback while doing TDD"

# RSpec Commands

Run tests: all tests, specify subdirectory, specify a spec file, specify specific specs

    bundle exec rspec spec/
    bundle exec rspec spec/controllers/
    bundle exec rspec spec/controllers/your_spec.rb
    bundle exec rspec spec/controllers/your_spec.rb -e "partial text from spec description"

# Generate Stuff
Generate a controller

    rails g controller Users new
    rails g controller Pages home contact about
    rails g controller Sessions new
    
Generate a model

    rails g model User field1:string field2:string
    rails g model Micropost content:string user_id:integer

Create a migration

    rails g migration add_email_uniqueness_index
    rails g migration add_password_to_users encrypted_password:string
    rails g migration add_salt_to_users salt:string
    rails g migration add_admin_to_users admin:boolean

Generate an integration test (i.e. request spec). Integration test will be created as
`spec/requests/some_name_spec.`

    rails g integration_test some_name

# Misc

Open up the db console

    rails dbconsole

Show list of current routes

    bundle exec rake routes

Database commands

    rake db:migrate             # Runs migrations that have not run yet.
    rake db:create              # Creates the database
    rake db:drop                # Deletes the database
    rake db:schema:load         # Creates tables and columns within the database following schema.rb
    rake db:setup               # Does db:create, db:schema:load, db:seed
    rake db:reset               # Does db:drop, db:setup
    rake db:migrate:reset       # Reset db + schema and rebuild using migrations db:drop db:create db:migrate 

Rollback the last db migration / last 3 migrations

    bundle exec rake db:rollback            # just the last migration
    bundle exec rake db:rollback STEP=3     # last three migrations

Rollback and redo the last db migration (typically after you've corrected a problem with
the migration

    bundle exec rake db:migrate:redo            # redo the last migration
    bundle exec rake db:migrate:redo STEP=3     # rollback and re-apply the last three migrations

Prepare the test database after adding new migrations (not needed in Rails 4.1)

    bundle exec rake db:test:prepare

Refresh the thumbnails generated by paperclip. You must specify the name of the class
that contains the paperclip attachments. There are currently problems with this.
Paperclip updates the created_at date for the model which causes it to save a new
version of the attachments in the CMS app.

    bundle exec rake paperclip:refresh:thumbnails CLASS=Contract --trace

# DateTime performance

There are several methods for calculating dates. The most readable `7.months.ago` is very slow.
Several other methods produce the same results with significantly faster performance

## 16.days.ago is very slow

    irb(main):098:0> Benchmark.measure { 100000.times { 16.days.ago } }
    =>   9.300000   0.030000   9.330000 (  9.329083)

## DateTime.now - 16.days is more efficient

    irb(main):099:0> Benchmark.measure { 100000.times { DateTime.now - 16.days } }
    =>   1.750000   0.020000   1.770000 (  1.772659)

## DateTime.now - 16 is the fastest

    irb(main):100:0> Benchmark.measure { 100000.times { DateTime.now - 16 } }
    =>   0.190000   0.000000   0.190000 (  0.198303)

# Troublshooting Apache

If Apache fails silently to start (no error in /var/log/apache2/error_log), try the following
command to start Apache and display error output

    sudo bash -x /usr/sbin/apachectl -k start
