---
layout: post
author: Richard Peterson
tags: [sql, performance]
---

Bash script to install PostgreSQL with the PostGIS extension. This can be run directly on an Ubuntu virtual machine, including an Amazon EC2 instance.

```bash
# Install PostgreSQl and PostGIS on Ubuntu 20.4 VM

# add sudo user
sudo adduser richard
sudo usermod -aG sudo richard
sudo -i -u richard

# install postgresql
# https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart
sudo apt update
sudo apt install postgresql postgresql-contrib

# connect first time
sudo -i -u postgres
psql 
# or
sudo -u postgres psql

# exit the db by typing "\q"
# change the postgres password
sudo passwd postgres

# allow postgresql to accept remote connections
# https://bigbinary.com/blog/configure-postgresql-to-allow-remote-connection

sudo find / -name "postgresql.conf"
sudo nano /etc/postgresql/12/main/postgresql.conf
# add listen_addresses = '*'


# configure to accept psql connections
sudo find / -name "pg_hba.conf"
sudo nano /etc/postgresql/12/main/pg_hba.conf

# restart postgresql service
sudo systemctl restart postgresql

# add role with password
sudo -u postgres createuser --interactive

# change password
sudo -u postgres psql
ALTER ROLE richard WITH PASSWORD 'password';

# PgAdmin
# Connection 
#	- ipv4 DNS address from VM 
#	- username=richard and pw password
# SSH 
#	- tunnel host= ipv4 DNS
#	- username=ubuntu
#	- authentication= identity file
# 	- Identity file = pem key for VM access

##########################################

# Install PostGIS

# https://computingforgeeks.com/how-to-install-postgis-on-ubuntu-debian/

# check postgres version
locate bin/postgres
/usr/lib/postgresql/12/bin/postgres -V
# we are on version 12

# install postgis for v12
sudo apt install postgis postgresql-12-postgis-3

# connect to datbase from pgadmin
# run command to enable PostGIS
CREATE EXTENSION postgis;

# postgis features
# https://postgis.net/install/
```