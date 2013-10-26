# PostgreSQL

## Installing and setting up database

OS X comes with client-side PostgreSQL tools, but not the server. Use Homebrew to install
the server

    brew install postgresql

Edit /etc/paths and make sure '/usr/local/bin/' is in the path and comes before '/usr/bin'
so that the Homebrew version of the client-side tools are used instead of the older OS X
versions.

Initialize the database with the current logged in user as the superuser account. In my
case the logged in user is "will". For a development database, a single superuser account
is all we need. Note the superuser account is the username parameter that should be used
in database.yml

    initdb /usr/local/var/postgres

If you see an error like `"FATAL:  could not create shared memory segment: Cannot allocate memory"`
edit `/usr/local/var/postgres/postgresql.conf and change these values.

    max_connections = 20			  # (change requires restart)
    shared_buffers = 1600kB			# min 128kB

## Starting / Stopping

Start/stop PostgreSQL server using pg_ctl

    pg_ctl start -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log
    pg_ctl stop -D /usr/local/var/postgres

Launch PostgreSQL at login. This starts up PostgreSQL using the database at
`/usr/local/var/postgres` and stores the log file in that directory.

    ln -sfv /usr/local/opt/postgresql/*.plist ~/Library/LaunchAgents

Then start/stop PostgreSQL using launchctl

    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist

## Common commands

Command-line commands

    psql -l                   # list all databases
    ps -ef | grep postgres    # get list of current postgres processes
    psql db_name              # connect to database and open database console

DB console commands

    \q                  # quit
    \d                  # list all tables in the databse
    \d users            # list table schema
    \?                  # list of Posgres commands
    \h                  # list of SQL commands
    \h SELECT           # help on specific SQL command
