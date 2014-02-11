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
