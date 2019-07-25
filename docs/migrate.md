---
id: migrate
title: Migrating an Open EdX site (Single Server)
---

These instructions will help you migrate an Open EdX website. This migration requires a fresh installation on a target machine. Essentially, the two main databases (Mongo & SQL) will be moved from the old machine to the new machine, and then the data migration tasks will be run. 

This is the process most commonly used when upgrading a site. But it can also be used to move a site from one server to another, for example if changing hosting providers. 

## Pre-step: Use the my-passwords.yml file from the old machine when performing the fresh install for the target machine. 
When you do the [fresh installation](https://openedx.atlassian.net/wiki/spaces/OpenOPS/pages/146440579/Native+Open+edX+Ubuntu+16.04+64+bit+Installation) for your target machine, skip the step to generate a new my-passwords.yml file. Instead, copy the my-passwords.yml file from your old machine. This will make sure you don't need to keep track of two sets of passwords throughout this tutorial (one for the old machine and one for the target machine), and will undoubtedly avoid other credential headaches. 

## 1. Save all your passwords from the my-passwords.yml file. 
Commonly, this file is at /home/ubuntu/my-passwords.yml. 
You will need the credentials for your Mongo and SQL databases. 
At a minimum, you will need:

	MONGO_ADMIN_PASSWORD
	EDXAPP_MONGO_PASSWORD
	EDXAPP_MONGO_USER
	FORUM_MONGO_USER
	FORUM_MONGO_PASSWORD

> Note: If you have sudo access on a single server install, then you don't need a MySQL password from my-passwords.yml. 


## 2. Back up the databases on your target machine, just in case
If you have done a fresh install, these databases should essentially be blank. You don't want anything worth keeping on them, since they will be replaced by the data on the old machine. 

**On your target instance:**

Dump the SQL tables and structure into a file called mysql-fresh.sql

	cd /home/ubuntu  # any directory will do
	sudo mysqldump --add-drop-database --no-data --databases analytics-api dashboard discovery ecommerce edxapp edxapp_csmh reports xqueue > mysql-fresh.sql

Dump the content of those SQL databases into the same file. 

	sudo mysqldump --no-create-info --databases analytics-api dashboard discovery ecommerce edxapp edxapp_csmh reports xqueue >> mysql-fresh.sql


Optionally, stop the mongo service


	sudo service mongod stop
> Note: Stopping the mongo database may or may not be necessary. I've experienced both. 

Dump the edxapp mongo collection into a folder. 

	mongodump --username [EDXAPP_MONGO_USER] --password [EDXAPP_MONGO_PASSWORD] --db edxapp --out mongo-fresh-edxapp
> Note: [EDXAPP_MONGO_USER] will commonly be *edxapp* and [EDXAPP_MONGO_PASSWORD] will commonly be a long string of random characters

Dump the cs_comments_service mongo collection into a folder. 

	mongodump --username [FORUM_MONGO_USER] --password [FORUM_MONGO_PASSWORD] --db cs_comments_service --out mongo-fresh-cs_comments_service
> Note: [FORUM_MONGO_USER] will commonly be *cs_comments_service* and [FORUM_MONGO_PASSWORD] will commonly be a long string of random characters

> Note: If forums are totally empty, this may not output anything. 

Dump the admin mongo collection into a folder. 

	mongodump --username admin --password [MONGO_ADMIN_PASSWORD] --db admin --out mongo-fresh-admin

If you stopped the mongo service, start it again

	sudo service mongod start

## 3. Backup the old machine, to be moved to the target machine

**On your old instance:**

Dump the SQL tables and structure into a file called mysql-old.sql

	cd /home/ubuntu #any directory will do 
	sudo mysqldump --add-drop-database --no-data --databases analytics-api dashboard discovery ecommerce edxapp edxapp_csmh reports xqueue > mysql-old.sql

Dump the content of those SQL databases into the same file. 
	
	sudo mysqldump --no-create-info --databases analytics-api dashboard discovery ecommerce edxapp edxapp_csmh reports xqueue >> mysql-old.sql

Optionally, stop the mongo service


	sudo service mongod stop
> Note: Again, this step may or may not be necessary.

Dump the edxapp mongo collection into a folder. 

	cd /home/ubuntu #any directory will do
	mongodump -u admin -p [MONGO_ADMIN_PASSWORD] --host localhost --authenticationDatabase admin --db edxapp --out mongo-old-edxapp
	mongodump -u admin -p [MONGO_ADMIN_PASSWORD] --host localhost --authenticationDatabase admin --db cs_comments_service --out mongo-old-cs_comments_service

> Note the slightly modified approach here. We are using the mongo *admin* username & password to authenticate, as opposed to the db-specific user. I believe both will work. If you are having trouble with one, you can try the other. 

If you stopped the mongo service, start it again

	sudo service mongod start

## 4. Move the database files from the old instance to the target instance.
I find scp to be the fastest. You can use alternative methods, such as FTP. 

Mongo dumps are generated as a collection of folders and files. Zip them up into individual .zip files

	cd /home/ubuntu
	zip -r mongo-old-edxapp.zip mongo-old-edxapp
	zip -r mongo-old-cs_comments_service.zip mongo-old-cs_comments_service

Use scp to transfer your files to the target instance. The commands below will move the appropriate files onto the target instance in the /home/ubuntu directory

	scp -i my_target_key.pem mysql-old.sql ubuntu@1.23.456.78:/home/ubuntu
	scp -i my_target_key.pem mongo-old-edxapp.zip ubuntu@1.23.456.78:/home/ubuntu
	scp -i my_target_key.pem mongo-old-cs_comments_service.zip ubuntu@1.23.456.78:/home/ubuntu

> Be sure to replace the following: 
> 
> * **my_target_key.pem** with the key that can authenticate into the target server. It should reside on the old server in the same directory as your backup files (e.g. /home/ubuntu)
>
> * **ubuntu** with the name of the user that can authenticate into the target server, if it is different. 
> 
> * **1.23.456.78** with the IP address of the target server

## 5. Unzip the files on the target server
**On the target server:**

	cd /home/ubuntu
	unzip mongo-old-cs_comments_service.zip
	unzip mongo-old-edxapp.zip


## 6. Import the databases on the target machine
In this step, we take our database backups that we've moved from the old machine onto the target machine and we unpack them into SQL and Mongo. This obviously blows away the (mostly blank) content that exists in the target databases from the fresh installation, and replaces it with the content from our existing site. 

Import the SQL file

	sudo mysql < mysql-old.sql

Import the mongo collections. Make sure your folders reside in /home/ubuntu for these commands to work as written.
	
	cd /home/ubuntu
	sudo service mongod stop
	mongorestore --username [FORUM_MONGO_USER] --password [FORUM_MONGO_PASSWORD] --drop --db cs_comments_service /home/ubuntu/mongo-old-cs_comments_service/cs_comments_service
	mongorestore --username [EDXAPP_MONGO_USER] --password [EDXAPP_MONGO_PASSWORD] --drop --db edxapp /home/ubuntu/mongo-old-edxapp/edxapp

## 7. Run Database Migrations

	sudo su edxapp -s /bin/bash
	cd ~
	source edxapp_env
	sudo -u edxapp -E /edx/bin/python.edxapp /edx/bin/manage.edxapp lms migrate --settings=production --fake-initial
	sudo -u edxapp -E /edx/bin/python.edxapp /edx/bin/manage.edxapp cms migrate --settings=production --fake-initial

> Note: This is an alternate command to run the migrations. It has been recommended by Open EdX but I've had less luck with it: 
>	```
>	python /edx/app/edxapp/edx-platform/manage.py lms syncdb --settings=production
>	python /edx/app/edxapp/edx-platform/manage.py cms syncdb --settings=production
>	```




