#Rubber
Various notes on using the Rubber gem (v1.15)

##Haproxy
By default Rubber sets up HTTP ports for haproxy, even if haproxy is not present. In particular
`passenger_listen_port` and `passenger_listen_ssl_port` are set to 7000 and 7001 respectively.
It is assumed that haproxy will listen on `web_port` (80) and `web_ssl_port` (443) and forward
the requests to ports 7000 and 7001. If haproxy is not installed, you need to manually edit
rubber-passenger_nginx.yml and set `passenger_listen_port` to 80 and `passenger_listen_ssl_port`
to 443.

##Crontab notification emails
If a command in crontab fails, cron will send an email containing the output of the command to the owner
of the crontab. Rubber adds "MAILTO=you@mail.com" to crontab to override this and send the notification
directly to the "admin_email" email configured in rubber.yml

##Don't deploy an app with .rvmrc
The first time RVM sees an .rvmrc file, it asks if you want to trust it. During deployment, this will
cause the server to sit forever waiting for you to press ENTER. Rubber addresses this by adding
.rvmrc to `:copy_exclude` in `deploy.rb`. However it could still be a problem when deploying from git.

##Crontab and system gem dependency
Rubber places all its cron jobs in a user-specific crontab file for the root user. (User specific
crontabs are stored separately from the standard system `/etc/crontab` file and can be manipulated
with the `crontab` command.) You can see the contents of the crontab with the `"crontab -l"` command.

Originally rubber defined the environment for crontab to run in by dumping all relevant environment
variables at the top of the crontab file. This was done with the following erb code at the top of the
crontab file in the base template.

    <%- ENV.select {|k, v| k =~ /rvm|ruby|bundler|gem|path/i }.each do |k, v| -%>
    <%= k %>='<%= v %>'
    <%- end -%>

The result looked something like this

    # cron clears out enviroment variables, so we need to add these in
    # to make sure we are running the correct ruby (rvm env vars, plus paths)
    GEM_HOME='/mnt/cms-production/shared/bundle/ruby/1.9.1'
    MY_RUBY_HOME='/usr/local/rvm/rubies/ruby-1.9.2-p180'
    GEM_PATH=''
    BUNDLE_BIN_PATH='/usr/local/rvm/gems/ruby-1.9.2-p180/gems/bundler-1.0.15/bin/bundle'
    BUNDLE_GEMFILE='/mnt/cms-production/releases/20110711162435/Gemfile'
    PATH='/mnt/cms-production/shared/bundle/ruby/1.9.1/bin:/usr/local/rvm/gems/ruby-1.9.2-p180/bin:...'
    RUBY_VERSION='ruby-1.9.2-p180'
    RUBYOPT='-I/usr/local/rvm/gems/ruby-1.9.2-p180/gems/bundler-1.0.15/lib -rbundler/setup rubygems'
    rvm_gemsets_path='/usr/local/rvm/gemsets'
    rvm_scripts_path='/usr/local/rvm/scripts'
    rvm_bin_path='/usr/local/rvm/bin'
    
    ... and whole bunch more rvm variables

crontab was built during execution of `"rake rubber:config"`, which is run on the server each time
you run `"cap deploy"`. The command run on the server looks like this:

    cd /mnt/cms-production/current && RUBBER_ENV=\"production\" RAILS_ENV=\"production\" bundle exec rake rubber:config

Because `"bundle exec"` was used, the environment captured in crontab was the environment created by
Bundler. Notice that `GEM_PATH` was empty and `GEM_HOME` pointed to the shared location where Bundle
installed gems for the app. **So cron jobs could not see system gems.**

Unfortunately Rubber installs several system gems that are needed by the cron jobs. These gems
are listed in rubber.yml

    # OPTIONAL: The gems to install on all instances
    # You can install a specific version of a gem by using a sub-array of gem, version
    # For example, gem: [[rails, 2.2.2], open4, aws-s3]
    gems: [open4, aws-s3, bundler, [rubber, "#{Rubber.version}"]]

open4 in particular causes problems. It's required by `cron-sh` but it's not installed in the app's gemset.

In a nutshell: `cron-sh` must be able to see to system gems, but the Rake tasks run from `cron-sh` must
run in the Rails app environment, provided by Bundler, and thus will not see system gems.

The solution is to remove all the environment variables from the crontab, leaving only the PATH variable.

    # cron clears out enviroment variables. We need to put our path back in so we can find RVM.
    # 'rvm exec' and 'bundle exec' in cron-rake will take care of setting up the rest of the
    # environment needed by Ruby / Rails
    PATH='/mnt/cms-production/shared/bundle/ruby/1.9.1/bin:/usr/local/rvm/gems/ruby-1.9.2-p180/bin:...'

Then use `"rvm exec"` and `"bundle exec"` to build the appropriate environment for `cron-sh` and `rake`.
This is done with the following code in `cron-rake`

    Dir.chdir(RAILS_ROOT)
    args = %W{-l #{log} -- bundle exec rake -t} + ARGV
    system "rvm", "exec", "script/cron-sh", *args

Note we use `"rvm exec"` to run `cron-sh`. This gives `cron-sh` visibility to the system gems so it can find open4.
We then use `"bundle exec"` to run `rake`. This gives `rake` visibility to the app-specific gemset.

##Sample munin plugins
The sample munin plugins `"example_mysql_query.rb"` and `"example_simple.rb"` present some
interesting possibilities - for example adding a munin chart showing number of registered users
over time.

##Some Rubber-related Capistrano tasks are defined in the app
Rubber adds several Capistrano tasks to the app by adding various deploy-* files during the
vulcanization step. These tasks are made available to the app with the following code in
config/deploy.rb.

    # load in the deploy scripts installed by vulcanize for each rubber module
    Dir["#{File.dirname(__FILE__)}/rubber/deploy-*.rb"].each do |deploy_file|
      load deploy_file
    end

##Capistrano tasks are also defined in the Rubber gem
Capistrano provides a method for gems to share Capistrano tasks - often called recipes. This is done
with `Capistrano::Configuration.instance.load()`. Without going into full details, Rubber uses this
mechanism to share all the recipes in the lib/recipes/rubber folder. This gives the app access to
all the Capistrano tasks defined in various files under /recipes.

##Global Rubber environment in Capistrano
Rubber environment is made available to all Capistrano tasks and made available within the ERB config
templates (.conf files) through several Capistrano configuration variables and global constants.

* **rubber\_env**: Contains hash built from combination of all config files (rubber.yml, rubber-*.yml,
rubber-(environment)-env.yml, rubber-secret.yml) without any role/host overrides.
* **rubber\_instances**: Contains hash built from instance file associated with the environment (i.e. instance-production.yml)
* **rubber\_cfg**: Typically used in statements like rubber\_cfg.environment.bind(role, host) to get
full configuration with role/host overrides.
* **RUBBER_ENV**: "`production`", "`staging`", etc. Can also be a custom name to build a developer-specific
environment. Value taken from `RUBBER_ENV` environment variable.
* **RUBBER_ROOT**: App root folder on target server (also used for app root on dev system?).

##Functions used in Rake and Capistrano tasks
* **rsudo()**: Wrapper around cap's run command to run a command as root on the target server
* **sudo_script()**: Uses cap put() to put contents of script into temp file on target server and then uses
cap run() to run the script
* **capture()**: cap command to run a command on the target server and capture output
* **send()**: Ruby method to make a "function call". Rubber creates temporary cap tasks and uses send() to
execute them so that the task name appears in the console output (I think)
* **required\_task**: Rubber function to override `allow_optional_tasks` and allow cap to generate error
* **set() / fetch()**. Cap methods to set and fetch configuration variables
* **load()**. Loads a Ruby file and evaluates the code inline.

##Rubber Rake tasks
Rubber defines Rake tasks (`config`, `rotate_logs`, `backup`, `backup_db`, `restore_db_s3`) in
rubber/tasks/rubber.rb. rubber.rb is included in the app by a reference in tasks/rubber.rake

##Global Rubber environment in Rake
The Rubber environment is made available to Rubber Rake tasks through functions defined at the
beginning of rubber/tasks/rubber.rb. These functions are defined in the rubber namespace

* **rubber\_env**: Equivalent to cap configuration variable rubber\_env
* **rubber_instances**: Equivalent to cap configuration variable rubber\_instances
* **cloud_provider**: Subset of rubber\_env. Cloud provider (AWS) configuration

##Transforming .conf files
During deployment, rubber transforms config (.conf) files and places the resulting output on the
target server. The .conf files are basically erb templates with special @ variables defined at
the top that control what happens when the template is transformed. These @ variables tell Rubber
where to place the file, commands to execute after transforming the file, etc. These @ variables
are documented in the [Rubber Wiki](https://github.com/wr0ngway/rubber/wiki/Configuration) and,
more completely, in the source code in lib/rubber/generator.rb

