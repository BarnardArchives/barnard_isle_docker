## Generic Overview

To test the ISLE stack I'm using a CentOS Virtual Machine.  The main OS is Windows, and virtual machines are used to keep the machine sand-boxed (clean).

You may run Docker directly on your machine if you desire.  ISLE will not work on Windows hence the need to get a real OS running.  Mac users are fine.

Our development environment from above looks like: Windows Computer -> Virtualbox running CentOS 7 -> Docker daemon -(magic)->ISLE STACK

Vocab
=
From this point "the host" always refers to the server that is running Docker and ultimately the ISLE stack, whether real or virtual.


Must Haves
=
  - SQL Dumps of Fedora and Drupal databases.
  - Usernames/Passwords: can be obtained from your original www/sites/default/settings.php and fedora/server/config/fedora.fcfcg!
  - The host must have access to your copied fedora data, solr data/schema, and www folders.  I suggest using a network share.  Please do not connect to your production data stores.


## Host Preparations

  - Docker CE installed  (https://docs.docker.com/install/)
  - Docker-compose installed (https://docs.docker.com/compose/install/)
  - Mounted Read/Write access to the copies of your original fedora/solr/www!
  - Add your user to the Docker group so you can run `docker` without needing to `sudo`.  `usermod -aG docker <your username>` this adds your user to the docker group.

Do not edit /etc/hosts at this time.

## Migration Guide

Have your backup data handy because you will use it to locate any values noted here for change!

### Phase 1:
Clone this repo so you have the normalized configs and docker-compose.yml.  

For this demonstration we're going to directly mount our webroot files.  In production we'd likely put this on a volume.

Irrespective of this we need to modify _one file_ in our webroot if we're migrating: Drupal's settings.php.

In the webroot folder and traverse to sites/default and open settings.php.  Look for your $database connection and change the `host` to `mysql`.  Save and close.  Please also take note of your sql database, username, and password.

### Phase 2:
1. RENAME THE `bcol` FOLDER
2. and adjust all references in the docker-compose.yml.
3. Additionally change the MOUNT locations of:

`- /mnt/barnard/islandora/fedora/data:/usr/local/fedora/data # Barnard's Fedora data folder and all subfolders from bare metal instance.` to point to your fedora data.

`- /mnt/barnard/islandora/fedora/solr/collection1/data:/usr/local/solr/collection1/data  # A COMPLETE OVERWRITE WITH BC'S DATA` to point to your Solr collection DATA

`- /mnt/barnard/islandora/fedora/solr/collection1/conf/schema.xml:/usr/local/solr/collection1/conf/schema.xml  # A COMPLETE OVERWRITE WITH BC'S DATA` to point to your SCHEMA.xml.

`- /mnt/barnard/www/html/islandora:/var/www/html:/var/www/html` to point to your webroot!

Change the mysql root password!


### Phase 3:
In the mysql data folder:
Place your drupal.sql and fedora.sql.  The included users.sql must be edited with the values from your existing instance.  We have the DRUPAL MYSQL information from the above.  In the next section we'll find our Fedora SQL username and password to fill in!

In the fedora folder:  
1. `fedora-users.xml` please update `<fedoraPass>` with your own.  You may also change any other passwords you like.
2. `fedora.fcfg` we need to update our mysql connection parameters. Check your jbdcURL to ensure it's using the correct database name (fedora3).  Do not edit the host.

    <param name="dbPassword" value="superSecretPassword">
      <comment>The database password.</comment>
    </param>

    <param name="dbUsername" value="fedoraDBuser">
      <comment>The database user name.</comment>
    </param>

    <param name="jdbcURL" value="jdbc:mysql://mysql:3306/fedora3?useUnicode=true&amp;amp;characterEncoding=UTF-8&amp;amp;autoReconnect=true">
      <comment>The JDBC connection URL.</comment>
    </param>

3. `filter-drupal.xml` edit `dbname="<your_database_name>" user="<dbUsername>" password="<dbPassword>"` with your mysql drupal user/pass

Finally, FGS folder:

Check for all instances of "superSecretPassword" and CHANGE IT!  Do not modify any host settings!  "solr" "mysql" "fedora" are KNOWN on the docker network!

You're free to mount as much or as little as you want here - I just did our foxml because it's unique, but everything else was the same. 

### Phase 4: PREPARE TO FIRE.
1. From the folder with the docker-compose.yml please type `docker-compose up -d mysql` OR
if you would like to use PHPMyAdmin you just need to run `docker-compose up -d myadmin` and it will start both. 

2. Connect to your sql instance `docker exec -it isle-a2-mysql bash` and import your sql dumps located in /sql_databases_for_import.  Alternatively see the docker-compose on how to connect to PHPMyAdmin and get importing that way. 

3. Once you have that done that is, your imports are complete.  

4. Please start the remainder of the stack with `docker-compose up -d`.

Visit your site and do things.  It will be at whatever the EXTERNAL IP of your host is. Port 80, 8080, 8081, 8091 are the important ones?  In Drupal you will need to change your fedora URL to read to be "fedora" and your solr host to be "solr" - you can use container names if that fails.
