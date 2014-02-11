These are maintenance procedures for apps deployed with [Using Rubber to deploy a Rails app on AWS](aws_deploy.md).  

# Update Packages

Run this on a regular basis

    cap rubber:upgrade_packages

If a reboot is needed, you will be asked after the packages are updated if you want to reboot

# Update Gems

To update the system gems used by Rubber (doesn't upgrade the application gems)

    cap rubber:upgrade_gems

In a typical configuration the gems that would be updated are open4, aws-s3, and bundler.
Passenger and Rubber are locked at versions defined in the config.

# Upgrade Ruby

Edit `config/rubber/rubber-ruby.yml` and update `ruby_build_version` to the latest version (which
can be found here <https://github.com/sstephenson/ruby-build/blob/master/CHANGELOG.md>) and update
Ruby to whatever version you wish

    ruby_build_version: 20140110
    ruby_version: 2.0.0-p353

Bootstrap the server. This will detect the version of ruby-build and Ruby have changed and install
the new versions. Rubygems will also be updated by ruby-build. All systems gems (including Bundler)
will be rebuilt and updated to the latest version (unless they are version-locked in the templates).

    cap rubber:bootstrap

Redeploy the app. This will rebuild the application gems if needed.

    cap deploy

# Upgrade Rubber

Upgrading Rubber can be tricky, especially if there have been changes to the templates. There
is no definitive procedure. However some basic steps will always be needed.

Update the gem

    bundle update rubber

Re-vulcanize each of the roles and overwrite your current files as needed For example:

    bundle exec rubber vulcanize minimal_passenger_nginx
    bundle exec rubber vulcanize mysql
    bundle exec rubber vulcanize munin
    bundle exec rubber vulcanize monit

Remove extra rubber and open4 lines that were added to the bottom of your gemfile.

Using a diff tool, go through all the changed files and back out changes that you don't want.
No easy way around this. It's a time-consuming process. Commit changes. Bootstrap servers and
deploy the app

    cap rubber:bootstrap
    cap deploy

# Changing AWS instance types

It's easy to switch between t1.micro, m1.small, and c1.medium instance types on AWS.

Using AWS web-based console:

  * Stop the instance and wait for it to stop
  * Choose "Change Instance Type" from the Instance Actions menu and select the desired instance type
  * Start the instance
  * Re-associate the elastic IP with the new instance
  
Although, the AWS-id of the new virtual machine will be the same, it will have a new private IP Address,
and DNS name. You will need to update the server configuration to use the new private IP address.
Get the server's private IP address and private DNS from the AWS console. Edit `instance-production.yml`
and update the appropriate fields (shown below)

    - !ruby/object:Rubber::Configuration::InstanceItem 
      domain: collagewall.com
      external_host: ec2-50-17-81-168.compute-1.amazonaws.com
      external_ip: 50.16.189.180
      image_id: ami-61be7908
      image_type: c1.medium
      instance_id: i-17c7a376
      internal_host: ip-10-66-102-94.ec2.internal   <<-- Put new Private DNS here
      internal_ip: 10.214.130.113                   <<-- Put new Private IP address here
      .
      .
      .

Push new configuration to the server and restart appropriate services to apply the new private IP.

      bundle exec cap rubber:setup_remote_aliases
      bundle exec cap deploy

If you're using Munin, you might want to reset the munin graphs (if the server memory size
has changed for example): <http://serverfault.com/questions/189014/how-to-reset-munin-graphs>

# Recovering from failures

## Recovering if an instance crashes

If an instance crashes, Rubber allows you to create a new instance and remount your EBS volumes on the new
instance. All assets on the EBS volumes will be preserved. The static IP associated with the instance will
also be preserved and reused when you recreate the instance.

Tell Rubber the instance has crashed by destroying it.

    bundle exec cap rubber:destroy
    
IMPORTANT: Make sure to say "No" when asked "Instance has persistent volumes, do you want to destroy them? [y/N]?:"    
    
Recreate and redeploy the a new instance

    bundle exec cap rubber:create         # creates the instance on AWS
    bundle exec cap rubber:bootstrap      # sets up the instance, installs all require packages, ruby, etc
    bundle exec cap rubber:mysql:restart  # Restart MySQL so it looks in the right place for the data folder (why is this needed?)
    bundle exec cap deploy                # deploy the app

Rubber will attach and mount the existing EBS volumes during the bootstrap. Rubber will not format or overwrite
data on an existing EBS volume.

After rebuilding the server, you may get nightly errors from `/etc/cron.daily/logrotate`. In this case you need
to fix the password for the `debian-sys-maint` user in MySQL. Get the correct password from `/etc/mysql/debian.cnf`.
Update the password by running MySQL on the server as root and changing the password.

    mysql -u root
    ...
    mysql> SET PASSWORD FOR 'debian-sys-maint'@'localhost' = PASSWORD('ThePassword');
    mysql> quit

## Recovering if the EBS volume is corrupted or both the instance and the EBS volume are lost

This is a little heavy-handed if the instance has not been lost. But it's simpler than recreating and
attaching an EBS volume to a running instance.

Shut down the instance (or tell Rubber the instance has crashed) by destroying it.

    bundle exec cap rubber:destroy

Say "No" when asked "Instance has persistent volumes, do you want to destroy them? [y/N]?:"
This leaves the volume setting in instance-production.yml which we can use later.

Using AWS web-based console

* Delete the corrupted EBS volume
* Create a new EBS volume from the most recent nightly snapshot. Make a note of the new volume ID

Edit instance-production.yml and put the new volume ID under the "volumes:" section

    - volumes: 
        cwa2_/dev/sdh: vol-46de282c     <<-- NEW VOLUME ID GOES HERE
      static_ips: 
        cwa2: 50.16.189.180

Create a new instance using the steps in "Recovering if an instance crashes"
