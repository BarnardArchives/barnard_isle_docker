Barnard's Islanadora ISLE Migration Walk-through
=
status: in development

## Definitions

  - "the host" always refers to the server or machine that is running Docker and ultimately the ISLE stack, whether real or virtual.
  - "volume" a Docker-daemon controlled place to hold data on the local file system.  Used to persist data across containers.
  - "network" refers to a defined Docker network that is controlled by docker.  They have powerful implications in production.

## General Migration Overview

To test the ISLE stack my host is a CentOS Virtual Machine. The Virtual Machine is running on Windows.  Windows _is not the host_.

You may run Docker directly on your machine if you would like, however ISLE will not work on Windows natively.  Mac users are free to Docker on.

So *my* development environment from above appears like: 

Windows Workstation -> VirtualBox running CentOS 7 (the host) -> Docker daemon -manages-> ISLE STACK == repository bliss.

## Docker networking

A quick and important note about networking and Docker.

Services `fedora, solr, apache, mysql, proxy` communicate using an internal _private_ stack network.  The service `proxy` **also** joins an insecure network that is accessible to the WAN (or for testing "WAN" likely means a smaller internal network).  Why two networks?  Swarms, scaling, replicating.

## Diagrams
  - Coming shortly?

## The Migration Must Haves

  - SQLDumps of the Fedora and Drupal databases.

  - Usernames/Passwords for key parts of your stack which are used **for** the migration.
    - Drupal SQL information: username, password, database name can be obtained from your original `www/sites/default/settings.php`
    - Fedora SQL information: username, password, database name can be obtained from your original `fedora/server/config/fedora.fcfcg`
    - Fedora users: please have a copy of your `fedora-users.xml`
    - Tomcat users: please have a copy of your `tomcat-users.xml` OR use the default login: admin,ild_tc_adm_2018 (for both fedora and solr)
  
  - The host **must** have access to your Fedora data, Solr data and schema, FGS transforms, and www folders.  I'm using a network share.


## Host Preparations

  - Docker CE installed  (https://docs.docker.com/install/)
  - Docker-compose installed (https://docs.docker.com/compose/install/)
  - Mounted Read/Write access to the copies of your original fedora/solr/www!
  - **Add your user to the Docker group so you can run `docker` without needing to `sudo`.**  `usermod -aG docker <your username>` adds a user to the docker group.


## Migration Guide

THIS IS ALL DONE ON THE HOST.  If you are ever to connect to a docker container I will explicitly state so, and why.

### Phase 1: docker-compose.yml

  1. Clone this repository so you have the normalized configs and docker-compose.yml.  No need to do this as root, nor in any special directory.
  2. Rename the `bcol` to something meaningful to you.  Additionally you can duplicate this folder to have different sets of configs for testing.
  3. Open our docker-compose.yml in your *favorite* text editor **and** change the config paths to the new paths.
  4. Modify the the remaining volume definitions to point to the correct datastores.
  5. Add some FQDNs to apache.  We're making apache aware of the vhosts it will serve connections.
  6. Change the mysql root password.  This isn't safe for production and should be removed when you move to production.
  7. Save and close _your_ docker-compose.yml. 


### Phase 2: migration configuration changes.

 
  1. **ALL MIGRATORS** need to modify _one line_ in your Drupal settings.php.  In the webroot folder that will be mounted to the container, traverse to `./sites/default/` and open `settings.php`.  Look for your $database connection and change the `host` to `mysql`.  Save and close.  
  2. In the `config\fedora`  folder: 
     - place your fedora-users.xml here or update the existing to match your users. 
     - update filter-drupal.xml with your mysql username, password, and database name.  Do **not** change the host from `mysql`.
     - update fedora.fcfg, searching for and updating:
```
    <param name="dbPassword" value="superSecretPassword">
      <comment>The database password.</comment>
    </param>

    <param name="dbUsername" value="fedoraDBuser">
      <comment>The database user name.</comment>
    </param>

    <param name="jdbcURL" value="jdbc:mysql://mysql:3306/{DatabaseName:fedora3}?useUnicode=true&amp;amp;characterEncoding=UTF-8&amp;amp;autoReconnect=true">
      <comment>The JDBC connection URL.</comment>
    </param>
```
  3. In the `config\fgsconfigFinal` folder:
     - Search and update all instances of "superSecretPassword". I know this is painful, the password should be updated with your fedoraAdmin\fgsAdmin users (defined in `fedora-users.xml`).
     - Please do not modify any host settings!  "solr" "mysql" "fedora" are valid hosts on the docker network!
     - Have XSLTS? You're free to mount as much or as little as you want here - I just did our foxml because it's SLIGHTLY unique, but everything else was the same in terms of the actual transforms. Add an additional volume entry and point it to `./bcol/config/fgsconfigFinal/our_transforms:/usr/local/tomcat/webapps/fedoragsearch/WEB-INF/classes/fgsconfigFinal/index/FgsIndex/islandora_transforms`

  NB: do not use the default that are provided here they will not work by default (they're here for learning and are key files).

### Phase 3: Initialize MySQL.

This step is designed to focus on importing our existing Drupal and Fedora SQL tables into a persisted Docker _volume_.

You may import your SQL databases from the CLI or using [phpMyAdmin](https://www.phpMyAdmin.net/) a web interface for managing MySQL servers.

  1. Place your SQL dumps in the `mysql\data` folder.  This folder is only used for import and can be deleted when we're finished.
  2. Open a terminal and `cd` to the folder with your docker-compose.yml
  3. Start MySQL
     - CLI: run `docker-compose up -d mysql` 
     - phpMyAdmin: run `docker-compose up -d myadmin` (this will start the server as well.) 
  4. Import your data 
     - CLI: run `docker exec -it isle-a2-mysql bash` to connect to your container.  Import your sql dumps: `cd /sql_databases_for_import` `mysql -uroot -p<MYSQLROOTPASS> < *.sql`
     - phpMyAdmin: to connect phpMyAdmin visit http://<host_ip>:8081/. To login, the host is `mysql` username `root` password from the MySQL ENV. Click on import and select the users.sql first. Now go to your Fedora database, hit import (find correct file and import) and the same for your Drupal db.  
  5. Exit the container OR stop phpMyAdmin
     - CLI: `exit;`
     - phpMyAdmin: `docker-compose stop myadmin`
  5. Your imports are complete! Your SQL data is now persisted in a Docker-controlled volume.  
  6. Please comment out the line in docker-compose for the mysql import folder.  Do not remove the SQL files until you're done testing and satisfied. 

### Phase 4: Launch the rest of the stack.
  1. Start the remainder of the stack with `docker-compose up -d proxy`
  2. If phpMyAdmin is running and you no longer need it: `docker-compose stop myadmin`

### Phase 5: Update your Islandora Drupal settings and HACK:
  0. Website look funky?  Check your VHOSTS in the compose AND review the NGINX proxy conf file.  
     - There are so many way to handle VHOSTs: you have your pick to do it at `proxy` or at `apache`.  You could even run multiple `apache` containers to serve only ONE host each if you desired. Regardless, more to write about this new `proxy` container (new as of 17 FEB 2018).
  1. Visit your site at http://<host_ip>/.  If you ran are local to the instance (i.e., you're on the `host`) you can try (http://0.0.0.0:80).
  2. Login normally
  3. Settings -> Islandora -> Fedora, change the host to read `fedora` (i.e., `http://fedora:8080/fedora`)
  4. Settings -> Islandora -> Solr Index, change the host to read `solr` (e.g., `http://solr:8080/solr/collection1`)

Visit your site and test!  It will be at whatever the EXTERNAL IP of your host is.  Port 80, 443, 8080, 8081, 8091 are the cool ones... please read the docker-compose for port forwards.

### Phase 6: Issue reporting
   If you've followed this and have questions or would like to report issues please report to this repo.
   
