# Chiliproject migration to Redmine

ChiliProject to Redmine migration howto and resources by Christian Daehn (c) ASinteg GmbH.

Source system:
*  Ruby 1.8.7 + Rails 3.2.16
*  ChiliProject 3.9.0
*  MySQL DB
*  Apache2 with Passenger
*  Debian 7 Wheezy
  
Destination System:
*  Ruby 1.8.7 + Rails 3.2.16
*  Redmine 2.4.1
*  MySQL DB
*  Apache2 with Passenger
*  Debian 7 Wheezy
  
To migrate from ChiliProject 3.x back to Redmine 2.1 we used the guide provided by: http://code.google.com/p/chili-to-redmine/

But we faced with several problems which can be solved with this howto and resources provided here.

*Update:* There's a new Ruby script, which seems to do the same sql database migration task as the Java tool chili-to-redmine:
https://gist.github.com/pille/603702cbb8422cc4244c

## Problems

After the database migration the following problems occurred:

*  old/obsolete paths containing `/chiliproject/`
*  viewing issue details caused internal server errors (template parsing errors)
*  plugins broke (ChiliProject uses Rails 2.x, but Redmine 2.1 Rails 3.x)
*  themes broke (CSS ids changed and CP menu structure is incompatible to Redmine)

Further: The wiki history couldn't be migrated the Java tool - so all wiki histories are gone.


## Preparation Steps

For this guide the following file system structure is assumed:

`/var/chiliproject/`  
`/var/redmine/`  
`/var/www/chiliproject/`  -> link to `/var/chiliproject/public/`  
`/var/www/redmine/`       -> link to `/var/redmine/public/`  

The Apache2 with mod passenger runs as user `www-data` and has DocumentRoot `/var/www/`.

To run the Java based **ChiliToRedmine** database migration tool, the openjdk-6-jre is needed under Debian - for other OS any Java JRE of version 6 or higher is needed.


## Redmine 2.1 or 2.4 ???

We strictly followed the *[Chili-to-Redmine guide](https://docs.google.com/document/d/1SPypGY_cBjeXmpDXjFVkrla3a8CsWpps0qIgj-VYJds/)* and used Redmine 2.1.0 and after the migration updated to Redmine 2.4.1 - which worked without erros.

But Redmine 2.1 has some issues with the `mysql` backend - because in Rails 3.x this backend is `mysql2`. So if you use Redmine 2.4, there all database and Gemfile entries already use `mysql2` as database backend and you don't have to struggle with manually editing database.yml and the Gemfile.

But: We can't give any guarantee that it works - but it should do (in theory) ;-)


## Migration Steps

1. Stop the Apache2 server
2. Install Redmine 2.1.0 (or higher),  
   for this guide we use the svn checkout to `/var/redmine-2.4/` and made a symlink to `/var/redmine/`  
   and created the database `redmine` inside MySQL
3. (for Redmine 2.1 only) Edit the Redmine Gemfile `/var/redmine/Gemfile`  
   and change the section with `:mysql` to `:mysql2`, because only `mysql2` runs flawlessly with Rails 3.x -  
   otherwise you'll get errors during `rake db:migrate` etc.
4. Edit the Redmine config `/var/redmine/config/database.yml` and change the databade backend from `mysql` to `mysql2`  
   (same reasons/problems as mentioned in step 3) and create a copy named `database-migration.yml` where the database backend still is `mysql` (needed for the Java migration tool)
5. Finish Redmine installation, change the Apache2/passenger config from `/var/chiliproject/` to `/var/redmine/` etc. so now Apache2 runs Redmine instead of ChiliProject
6. Start Apache2 and test the basic Redmine install, then stop Apache2 again
7. Backup the ChiliProject database and create a sqldump:  
   `mysqldump -u root -p chiliproject >/tmp/dbdump_chiliproject.sql`
8. We had some issue descriptions containing some examples for UTF codes which broke the migration: `\xXX` - so we had to replace these invalid UTF codes e.g. by `XXX (UTF-8 Hex)` in all entried of table `journals` with:  
   `perl -i -pe 's/\XXX (UTF-8 Hex)/XXX (UTF-8 Hex)/g' dump.sql`
9. Replace all `/chiliproject/` urls to `/redmine/` inside the database dump with:  
   `perl -i -pe 's/\/chiliproject\//\/redmine\//g' /tmp/dbdump_chiliproject.sql`
10. Ensure that your Redmine installation still works and has default data loaded (see Redmine install howtos)
11. Import the `/tmp/dbdump_chiliproject.sql` into the redmine database, e.g. by executing these commands inside MySQL:  
   `USE redmine; SOURCE /tmp/dbdump_chiliproject.sql`  
   and run the Redmine database migration:  
   `rake db:migrate RAILS_ENV=production`
12. Run the SQL commands described in the *[Chili-to-Redmine guide](https://docs.google.com/document/d/1SPypGY_cBjeXmpDXjFVkrla3a8CsWpps0qIgj-VYJds/)*:  
  
        ALTER TABLE wiki_contents ADD comments VARCHAR(250) NULL, CHANGE COLUMN lock_version version INTEGER(11);
        
        ALTER TABLE journals
            CHANGE COLUMN journaled_id journalized_id INTEGER(11),
            CHANGE COLUMN activity_type journalized_type VARCHAR(255),
            CHANGE COLUMN created_at created_on DATETIME;
        
        UPDATE journals SET journalized_type='Issue' WHERE    journalized_type='issues';
        ALTER TABLE journals CHANGE COLUMN changes changes_chili TEXT NULL, DROP COLUMN type, DROP COLUMN version;
        ALTER TABLE trackers ADD COLUMN fields_bits INT(11) NULL DEFAULT '0';

        ALTER TABLE repositories
            ADD COLUMN identifier VARCHAR(255) NULL DEFAULT NULL,
            ADD COLUMN is_default TINYINT(1) NULL DEFAULT '0';

        ALTER TABLE workflows
            ADD COLUMN field_name VARCHAR(30) NULL DEFAULT NULL,
            ADD COLUMN type VARCHAR(30) NULL DEFAULT NULL,
            ADD COLUMN rule VARCHAR(30) NULL DEFAULT NULL;

        ALTER TABLE auth_sources
            CHANGE COLUMN custom_filter filter VARCHAR(255) NULL DEFAULT NULL,
            ADD COLUMN `timeout` INT(11) NULL DEFAULT NULL;
12. Change into `/var/redmine` dir and execute the Java database migration tool *[ChiliToRedmine](https://docs.google.com/file/d/0B2rCUFhTgJxARkhYNkNMOTVuNUE/edit?pli=1)*:  
    `java -jar ChiliToRedmine.jar config/database-migration.yml`
13. Fix conversion issues of the migration tool by running the sql command:  
    `UPDATE journal_details SET value='' WHERE value='(undefined)';`
14. Delete the new table `changes_parents` by:  
   `DROP TABLE changes_parents;`  
    otherwise you'll get errors running the database migration
15. Fix bad date anchor fields from Chili not supported in Redmine:  
  
        UPDATE redmine_dev.journal_details SET value=null WHERE (prop_key = 'due_date' OR prop_key = 'start_date') AND (value = '*id001' OR value = '(unknown)');
        UPDATE redmine_dev.journal_details SET value=replace(value, '&id001 ', '') WHERE (prop_key = 'due_date' OR prop_key = 'start_date') AND value like '&id001 %';
16. Change owner of `/var/redmine/` to `www-data`
17. Start Apache2 and test Redmine

## Plugins

Due to the fact that ChiliProject still uses Rails 2.x usually all plugins won't work in Redmine - you have to download/install newer versions of them or port them to Rails 3.x.

For Redmine 2.x all plugins now reside in `/var/redmine/plugins/` and not any more in `../vendor/plugins/` like in ChiliProject.

## Copyright

Christian Daehn (c) ASinteg GmbH, Schwerin (Germany) 2014

You are free to modify & distribute this document under the MIT license.
