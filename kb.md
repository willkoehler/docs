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
Bundler provides, and thus only sees gems specified in the gemfile.

IMPORTANT: Any executable you want to run within the scope of the app (`rake`, `cap`, etc) must be run with
`"bundle exec"`

    bundle exec some-command
  
This means you will use `"bundle exec"` all the time when working within your rails app.

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

#RubyGems

###Helpful Commands
See which specific gem is being used and where it's located.

    gem which gemname
    
In the context of a Rails app this becomes.

    bundle exec gem which gemname
    
Cleanup all versions of a gem, except for the latests one. If no gemname is given in the command
line, cleanup will be run agains all installed gems.

    gem cleanup gemname
    
Display environment that RubyGems is running in

    gem environment
    
#Rubber

###Haproxy
By default Rubber sets up HTTP ports for haproxy, even if haproxy is not present. In particular
`passenger_listen_port` and `passenger_listen_ssl_port` are set to 7000 and 7001 respectively.
It is assumed that haproxy will listen on `web_port` (80) and `web_ssl_port` (443) and forward
the requests to ports 7000 and 7001. If haproxy is not installed, you need to manually edit
rubber-passenger_nginx.yml and set `passenger_listen_port` to 80 and `passenger_listen_ssl_port`
to 443.

###YAML Engine
Some Ruby systems are compiled with the newer Psych engine, which replaced the older Syck parser.
The Psych engine has problems with a few areas of the YAML generated by Rubber. More information
on what's going on [here](http://pivotallabs.com/users/mkocher/blog/articles/1692-yaml-psych-and-ruby-1-9-2-p180-here-there-be-dragons).
To work around this, we need to force Rails to use the older Syck parse by adding the following
lines to config/boot.rb.

    require 'yaml'
    YAML::ENGINE.yamler= 'syck'

Rubber-generated YAML code is parsed in several rake tasks and even in the Rails app itself so this code must
be in a central location, like boot.rb. Examples of tasks that will throw errors without the the syck YAML
engine are `"bundle exec rake RAILS_ENV=production db:migrate"` and `"bundle exec rake rubber:config"`.

###Crontab notification emails
If a command in crontab fails, cron will send an email containing the output of the command to the owner
of the crontab. Rubber adds "MAILTO=you@mail.com" to crontab to override this and send the notification
directly to the "admin_email" email configured in rubber.yml

###Don't deploy an app with .rvmrc
The first time RVM sees an .rvmrc file, it asks if you want to trust it. During deployment, this will
cause the server to sit forever waiting for you to press ENTER. Rubber addresses this by adding
.rvmrc to `:copy_exclude` in `deploy.rb`. However it could still be a problem when deploying from git.

###Crontab and system gem dependency
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

###Sample munin plugins
The sample munin plugins `"example_mysql_query.rb"` and `"example_simple.rb"` present some
interesting possibilities - for example adding a munin chart showing number of registered users
over time.

###Some Rubber-related Capistrano tasks are defined in the app
Rubber adds several Capistrano tasks to the app by adding various deploy-* files during the
vulcanization step. These tasks are made available to the app with the following code in
config/deploy.rb.

    # load in the deploy scripts installed by vulcanize for each rubber module
    Dir["#{File.dirname(__FILE__)}/rubber/deploy-*.rb"].each do |deploy_file|
      load deploy_file
    end

###Capistrano tasks are also defined in the Rubber gem
Capistrano provides a method for gems to share Capistrano tasks - often called recipes. This is done
with `Capistrano::Configuration.instance.load()`. Without going into full details, Rubber uses this
mechanism to share all the recipes in the lib/recipes/rubber folder. This gives the app access to
all the Capistrano tasks defined in various files under /recipes.

###Global Rubber environment in Capistrano
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

###Functions used in Rake and Capistrano tasks
* **rsudo()**: Wrapper around cap's run command to run a command as root on the target server
* **sudo_script()**: Uses cap put() to put contents of script into temp file on target server and then uses
cap run() to run the script
* **capture()**: cap command to run a command on the target server and capture output
* **send()**: Ruby method to make a "function call". Rubber creates temporary cap tasks and uses send() to
execute them so that the task name appears in the console output (I think)
* **required\_task**: Rubber function to override `allow_optional_tasks` and allow cap to generate error
* **set() / fetch()**. Cap methods to set and fetch configuration variables
* **load()**. Loads a Ruby file and evaluates the code inline.

###Rubber Rake tasks
Rubber defines Rake tasks (`config`, `rotate_logs`, `backup`, `backup_db`, `restore_db_s3`) in
rubber/tasks/rubber.rb. rubber.rb is included in the app by a reference in tasks/rubber.rake

###Global Rubber environment in Rake
The Rubber environment is made available to Rubber Rake tasks through functions defined at the
beginning of rubber/tasks/rubber.rb. These functions are defined in the rubber namespace

* **rubber\_env**: Equivalent to cap configuration variable rubber\_env
* **rubber_instances**: Equivalent to cap configuration variable rubber\_instances
* **cloud_provider**: Subset of rubber\_env. Cloud provider (AWS) configuration

###Transforming .conf files
During deployment, rubber transforms config (.conf) files and places the resulting output on the
target server. The .conf files are basically erb templates with special @ variables defined at
the top that control what happens when the template is transformed. These @ variables tell Rubber
where to place the file, commands to execute after transforming the file, etc. These @ variables
are documented in the [Rubber Wiki](https://github.com/wr0ngway/rubber/wiki/Configuration) and,
more completely, in the source code in lib/rubber/generator.rb

#Other

###%st stat in "top"
The %st stat in "top" shows you much much time other VMs are stealing time from your CPU. "st" stands for
‘Steal Time’ and is the amount of real cpu that the Xen Hypervisor has allocated to tasks other than
running your Virtual Machine (such as somebody else’s VM...).

Amazon uses steal time to throttle your CPU back to the rate allocated for your instance, regardless
of what other VMs are running on the hardware. (i.e. Amazon doesn't let you have more CPU than you've
purchased). This is readily apparent on t1.micro instances after your CPU burst time is exceeded. In this
case steal time pegs at 98% for the remainder of the CPU intensive task, effectively crippling your instance.

    top - 21:26:59 up 4 min,  1 user,  load average: 0.82, 1.22, 0.61
    Tasks:  95 total,   1 running,  94 sleeping,   0 stopped,   0 zombie
    Cpu(s):  0.0%us,  0.3%sy,  0.0%ni, 98.3%id,  0.0%wa,  0.0%hi,  0.0%si,  1.3%st
    Mem:   1757264k total,   323224k used,  1434040k free,    13452k buffers
    Swap:   917496k total,        0k used,   917496k free,    97284k cached

###EBS performance and MySQL data
There is mixed information on the performance of the instance (ephemeral) storage vs EBS. It appears that
EBS performance may be better on average but EBS has high variability in random seek times. If EBS
performance becomes a factor, storing the database in instance storage is an option to consider. But if
we place the database in instance storage, backups must be done with mysqldump and s3 buckets. This has
performance implications. mysqldump creates a significant load on MySQL. Plus it's not clear how we
can get a consistent snapshot of the database while it's still active. Backups generated by mysqldump
also can take a long time to restore.

###AWS Micro (t1.micro) instances
While it may be tempting to use a t1.micro instance for a small Rails app, a t1.micro instance is probably
not practical for any true customer-facing Rails server. The problem is the CPU resources are allocated
in bursts on a t1.micro instance. When not bursting, the steady-state CPU capacity of the instance very
low - about 2% of the burst capacity. If your app does anything CPU intensive for more than ~10 seconds,
your instance CPU is scaled **WAY** back - to about 2% of the burst capacity. This effectively cripples
your server until the CPU-intensive task finishes and the load drops. But since your server is now crawling
along, it may take a long time for the task to finish. If new requests come in during this time, it just
compounds the problem.

If several users visit your app simultaneously, or a bot scans your site, your CPU burst limit can easily be
reached. In addition, background tasks such as munin-node updates or mysqldump can exceed the CPU burst limit
causing significant slowdowns for app users.

Another effect of CPU bursting is that t1.micro instances take several hours to bootstrap with Rubber (vs
~20 minutes on a m1.small instance) because of all the CPU intensive work required.

Greg Wilson provides a good illustration of CPU limiting in the real world in [this blog entry](http://gregsramblings.com/2011/02/07/amazon-ec2-micro-instance-cpu-steal/).
There is a good technical writeup of CPU limiting in [Amazon's official documentation](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/index.html?concepts_micro_instances.html)

###AWS mounts secondary instance storage to /mnt
All AWS instances are given secondary instance storage (150GB for m1.small instances). By default this
storage is mounted on `/mnt`. This storage is free, but will go away when the instances fails or is
terminated. So it should not be used for essential assets. Because `/mnt` is already taken, you cannot
easily mount an EBS volume to `/mnt` using Rubber.

Exception: AWS does not provide secondary instance storage for t1.micro instances. In this case the `/mnt`
mount point is available for an EBS volume.