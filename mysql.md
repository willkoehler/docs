# MySQL

Starting and stopping from the command line

    sudo /usr/local/mysql/support-files/mysql.server start
    sudo /usr/local/mysql/support-files/mysql.server stop
    sudo /usr/local/mysql/support-files/mysql.server restart

Helpful MySQL commands for creating databases and users and granting privileges

    SHOW DATABASES;
    SHOW TABLES;
    CREATE DATABASE rxmeditrend;
    USE rxmeditrend;                                    # switch to a database
    
    SELECT User, Host FROM mysql.user;                  # List all the users in the system
    CREATE USER 'username' IDENTIFIED BY 'password';    # Create a user
    GRANT ALL ON rxmeditrend.* TO 'username';           # Grant privileges to a user
    SHOW GRANTS FOR 'username'@'host';                  # Show privileges for a user
    SET PASSWORD FOR 'username'@'host' = PASSWORD('password');    # Change user password
    DROP USER 'username'@'host';                        # Delete a user
    FLUSH PRIVILEGES;                                   # Flush privileges after manually manipulating the user table
    
Show MySQL table sizes

    SHOW TABLE STATUS; # shows size of table used and amount of free space than can be reclaimed in Data_free

    SELECT table_name, table_rows, round(((data_length) / 1024 / 1024), 2) `Size in MB`
    FROM information_schema.TABLES
    WHERE table_schema="yourdatabase"
    ORDER BY table_rows DESC;


# MyISAM Tables

MyISAM data is stored in database-specific directories under the root MySQL data directory. Each table has
a set of three files:

* **tbl\_name.frm**: Table definition
* **tbl\_name.MYD**: Table data
* **tbl\_name.MYI**: Table indexes

Commands to check and repair a MyISAM table. (Often used to repair the RxMedi-Trend events table
when it's marked as crashed.)

    CHECK TABLE tbl_name QUICK;
    REPAIR TABLE tbl_name EXTENDED;

# InnoDB Tables

InnoDB data is stored in the following files

* **ibdata1**: By default all data and indexes for all tables/databases is stored in a single, shared file:
ibdata1. idbata1 does not shrink. If rows are deleted, space will be reclaimed by new rows, but the file
will only grow in size.
* **ib\_logfile0** and **ib\_logfile1**: Redo log files to recover lost data if MySQL is not shutdown properly.
* **tablename.frm**: Each table has a small *.frm file in the database-specific directory which stores the
table definition.

MySQL does a check and recovery of InnoDB tables automatically when MySQL starts.
CHECK TABLE and REPAIR TABLE don't do anything for InnoDB tables.

You can use OPTIMIZE TABLE x on InnoDB tables. It's just an alias for ALTER TABLE x ENGINE=InnoDB

### Shared tablespace (default)

All data and indexes are stored in a single file, ibdata1, in the MySQL data root directory

To shrink the size of ibdata1, or make any changes to the ibdata configuration (such as enabling per-table
tablespace), you must clear out the database, make the desired changes and then rebuild the database from a
backup.

1. Backup all data with

        mysqldump --all-databases --extended-insert --add-drop-database --disable-keys --flush-privileges --quick --routines --triggers -u root > backup.sql

2. Stop MySQL.

3. Delete all contents of the MySQL data directory except the data/mysql directory

4. Make any desired changes to my.cnf (such as enabling per-table tablespace)

5. Start MySQL 
    
6. Restore data with the following commands from the mysql console

        SET FOREIGN_KEY_CHECKS=0;
        SOURCE backup.sql;
        SET FOREIGN_KEY_CHECKS=1;

### Per-table tablespace

Enable per-table tablespace with the `innodb_file_per_table` flag in my.cnf. Data for each table is store in
individual `tbl_name.idb` files (similar to MyISAM file structure). The `tbl_name.idb` files will shrink when the table is defragmented with
ALTER TABLE x ENGINE=InnoDB

* More details: <http://dev.mysql.com/doc/refman/5.0/en/innodb-multiple-tablespaces.html>
* There may be a performance penalty to per-table tablespace <http://umangg.blogspot.com/2010/02/innodbfilepertable.html>
* Another perspective: <http://code.openark.org/blog/mysql/reasons-to-use-innodb_file_per_table>

Seems like the default shared tablespace is best unless there is a specific need for per-table tablespace
(such as a table where records are frequently deleted)

# Infoquest IMS

The ibdata1 file on the Infoquest server had grown to 2.9Gbyte. After rebuilding the database on 8/8/2011,
the ibdata1 file shrunk to 141Mbyte. It's not clear why the ibdata1 file has grown so large.

One thought is that the Inv_log table has lots of deletes (all those "No activity is observed" events?)
which causes fragmentation of ibdata. However Infoquest claims they don't delete that many log entries.
Another possibility is this happened during development when the data was transferred from the old system.
Data may have been imported and cleared several times to test the process leaving large amounts of empty
table space.
