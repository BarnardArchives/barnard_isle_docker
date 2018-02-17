## General Migration Overview

To test the ISLE stack I'm using a CentOS Virtual Machine.  The main OS is Windows.

You may run Docker directly on your machine if you desire.  ISLE will not work on Windows natively.  Mac users are fine.

Our development environment from above looks like: 

Windows Computer -> Virtualbox running CentOS 7 -> Docker daemon -(WORK)-> ISLE STACK

## Docker Vocab

  - "the host" always refers to the server or machine that is running Docker and ultimately the ISLE stack, whether real or virtual.
  - "volume" a Docker-daemon controlled place to hold data on the local file system.  Used to persist data across containers.

## Docker diagram
  - Coming shortly.

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
  7. Save and close your docker-compose.yml. 


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

    <param name="jdbcURL" value="jdbc:mysql://mysql:3306/fedora3?useUnicode=true&amp;amp;characterEncoding=UTF-8&amp;amp;autoReconnect=true">
      <comment>The JDBC connection URL.</comment>
    </param>
```
  3. In the `config\fgsconfigFinal` folder:
    - Search and update all instances of "superSecretPassword". I know this is painful, the password should be updated with your fedoraAdmin\fgsAdmin users (defined in `fedora-users.xml`).
    - Please do not modify any host settings!  "solr" "mysql" "fedora" are valid hosts on the docker network!
    - Have XSLTS? You're free to mount as much or as little as you want here - I just did our foxml because it's SLIGHTLY unique, but everything else was the same in terms of the actual transforms. Add an additional volume entry and point it to `your transforms: ...cant remember the path off the top...`

  NB: do not use the default thats here, it will NOT work.

### Phase 3: Initialize MySQL.

  1. From the folder with the docker-compose.yml please type `docker-compose up -d mysql` OR if you would like to use PHPMyAdmin just run `docker-compose up -d myadmin` and it will start both. 
  2. Connect to your sql instance `docker exec -it isle-a2-mysql bash` and import your sql dumps located in /sql_databases_for_import.  Alternatively see the docker-compose on how to connect to PHPMyAdmin and get importing that way (i.e., just visit http://<host_ip>:8081/. the host is `mysql` username `root` password from the MySQL ENV.).
  3. Once you have that done that is, your imports are complete.  
  4. Comment out the lines in docker-compose for the import folder (if you'd like).
  5. If you've used PHPMyAdmin and are now done with it `docker-compose stop myadmin`

### Phase 4: Launch the rest of the stack.
  1. Start the remainder of the stack with `docker-compose up -d`.

Visit your site and do things.  It will be at whatever the EXTERNAL IP of your host is. Port 80, 8080, 8081, 8091 are the important ones?  In Drupal you will need to change your fedora URL to read to be "fedora" and your solr host to be "solr" - you can use container names if that fails.

