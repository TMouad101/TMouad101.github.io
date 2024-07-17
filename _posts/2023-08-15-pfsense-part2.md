---
title: "Configuration of the Captive Portal on pfSense with FreeRADIUS - PART 2 : Integration of MySQL Database"
date: 2024-07-16 12:00:00 -500
categories: [Network Security]
tags: [Captive Portal,FreeRADIUS,pfSense, Authentication Systems, MySQL]
---

**Configuration of the Captive Portal on pfSense with FreeRADIUS - PART 2**

> ✋ This report is a follow-up to the previous report about configuring a captive portal in pfSense using FreeRADIUS. This time, I've added some new elements to make it more interesting.🙂

# Overview
![Overview](/media/PFSENSE2/overviewplus.png)
## 📋 Table of Contents

- [Introduction](#introduction)
- [Configuration](#configuration)
  - [Step 1: Installing FreeRADIUS](#step-1-installing-freeradius)
  - [Step 2: Setting up mysql database](#step-2-setting-up-mysql-database)
  - [Step 3: Linking FreeRADIUS with MySQL database](#step-3-linking-freeradius-with-mysql-database)
  - [Step 4: Configuring Captive Portal](#step-4-configuring-captive-portal)
- [Conclusion](#conclusion)

## Introduction
In the previous report, I covered a step-by-step tutorial on how to link a captive portal with a FreeRADIUS server to manage access to the network using pfSense. It is very useful for controlling who can access your network.

Last time, the setup was based on using a text file that the FreeRADIUS server fetches to find the usernames and passwords. For a small infrastructure, this works quite well. However, for larger infrastructures, managing a vast number of modifications with a text file can become complicated in terms of data loss or speed. 

![netstate](/media/PFSENSE2/batman-thinking.gif)

That's why I worked on optimizing it. Instead of using a text file, we will use a MySQL database, which will be much more efficient, especially for this task.

this time i'll be using 3 virtual machines 
1. Ubuntu 23.04 as a FreeRADIUS server and MySQL server.
2. Ubuntu 20.04 as a client in the network (for tests)
3. pfsense

The steps for installing pfSense and configuring it remain the same for this project as well. I recommend reviewing the previous report by clicking [here](/posts/pfsense/).

So let's get started.

## Configuration
### Step 1: Installing FreeRADIUS
At this step, I'll be using two virtual machines. In the first one, I'll install the FreeRADIUS server using the following commands:

```console
# Update the package manager
sudo apt update

# Install FreeRADIUS
sudo apt install freeradius freeradius-mysql freeradius-utils -y

# Start the FreeRADIUS service
sudo systemctl start freeradius
```


To check if everything went well, execute this command to see the status of FreeRADIUS:
```console
sudo systemctl status freeradius
```
You should get a result similar to this one:

![status](/media/PFSENSE2/freeradiusinstalled.png)

Things are fine here for now; we'll leave it as it is and move on to the next step, which is configuring the MySQL database.

### Step 2: Setting up mysql database
Let's start by configuring MySQL in our VM. I'll be using an Ubuntu machine. Let's begin by installing the MySQL packages. 

```console
# Update the package manager
sudo apt update

# Install MySQL
sudo apt install mysql-server mysql-client -y
```

Now, let's create our database. It will be named radius.

First, connect to mysql as root:
```console
mysql -u root -p
```
Now let's create the database:

```sql
CREATE DATABASE radius;
exit
```
![status](/media/PFSENSE2/createdb.png)

Let's add a user that we will use from now on. Using the root user to access the database is not quite secure.

To do so :

```sql
CREATE USER 'radius'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'radius'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
![status](/media/PFSENSE2/sqluser.png)

![status](/media/PFSENSE2/sqlradius.png)

Now the database and the user are created, let's import the schemas provided by FreeRADIUS, using the following commands :

```console
cd /etc/freeradius/3.0/mods-config/sql/main/mysql/
 
mysql -uroot -p radius < schema.sql
```
After importing the schemas, you should see tables like this:

![status](/media/PFSENSE2/schema.png)

Now this phase is completed successfully. Let's link the previous phases together.

### Step 3: Linking FreeRADIUS with MySQL database

Now, to link FreeRADIUS and MySQL, let's take a look at how things work.

![status](/media/PFSENSE2/takenote.gif)

In FreeRADIUS, modules are stored in the `mods-available` directory, and only those that are enabled are linked in the `mods-enabled` directory. By creating a symbolic link to the `sql` module in the `mods-enabled` directory, you enable the SQL module, allowing FreeRADIUS to use SQL databases for authentication, authorization, and accounting.

To do so, let's navigate to the mods-enabled directory where enabled FreeRADIUS modules are linked.

```console
cd /etc/freeradius/3.0/mods-enabled
```

Now, we'll enable the SQL module by creating a symbolic link to it from the mods-available directory.

```console
ln -s ../mods-available/sql sql
```
Like this :

![status](/media/PFSENSE2/link.png)

Ok, let's start configuring FreeRADIUS to use MySQL.

```console
cd /etc/freeradius/3.0/sites-available/
nano default
```

By editing the default configuration file, we can modify the settings of the default configuration in FreeRADIUS.

You should modify these file sections to get something like this : 

```text
authorize {
.....
sql
....
}
accounting {
......
sql
....
}
post-auth {
......
sql
....
}
session{
......
sql
.....}
```
Make sure to remove any `-` before sql in theses sections.

Now, let's configure freeradius to connect the mysql database.

```console
cd /etc/freeradius/3.0/mods-available
nano sql
```
make sure the config file has the following lines :

```text
driver = "rlm_sql_mysql"
dialect = "mysql"
server = "localhost"
port = 3306
login = "radius"
password = "password"
radius_db = "radius"
read_clients = yes
```

Make sure you have the username and password correct. You can use the root user or any other user that has permission to access the database.

![status](/media/PFSENSE2/sql1.png)

Make sure to comment out all the lines in the MySQL TLS section. It creates a lot of conflicts, especially since we are the ones configuring this, so we are confident about who can use it.

![status](/media/PFSENSE2/sql2.png)

![status](/media/PFSENSE2/sql3.png)

Now we're almost there, just a little bit more.

Restart the FreeRADIUS service.

![restart](/media/PFSENSE2/restart.png)

Let's check the status of the port that FreeRADIUS uses to listen:

```console
netstat -lnp | grep 1812
```
![netstate](/media/PFSENSE2/netstate.png)

Now that everything is working fine, let's add some data to our database.

All the users will be stored in the table named `radcheck`.

Login as the user `radius`, and add some data to the database:

```sql
USE radius;

INSERT INTO radcheck (username, attribute, op, value) VALUES ('testuser', 'Cleartext-Password', ':=', 'testpassword');
```
I'm using `testuser` as the username and `testpassword` as the password.

![netstate](/media/PFSENSE2/test1.png)

Let's see if it works:

```console
radtest testuser testpassword localhost 1812 testing123
```
![netstate](/media/PFSENSE2/accept.png)

![netstate](/media/PFSENSE2/spongebob-dancing.gif)

Let's test it out from another machine:

To do so, we need to configure something in the `clients.conf` file to allow requests from other machines.

```console
nano /etc/freeradius/3.0/clients.conf
```

![netstate](/media/PFSENSE2/clients.png)

Restart FreeRADIUS, and now you are good to go.

the ip adress of our FreeRADIUS server is `10.0.0.37`, the request will be the following : 

```console
radtest testuser testpassword 10.0.0.37 1812 testing123
```
![netstate](/media/PFSENSE2/networkclient.png)

Now with a wrong password :

![netstate](/media/PFSENSE2/networkclient2.png)

Awesooome!! Let's get to configure the captive portal with our server.

### Step 4: Configuring Captive Portal

These steps will be similar to the previous report, except for some changes in the IP addresses.

![Architecture](/media/PFSENSE/architecture%202.png)

This is the architecture we'll be working with right now.

![pfsense](/media/PFSENSE/14.png)

Let's proceed to link pfSense with the FreeRADIUS server we configured.

Let's start by creating the FreeRADIUS user in the user manager.

![usermanager](/media/PFSENSE/15.png)



In the authentication server section, we will add a new user. I have already created the user under the name 'freeradius' :
<br>

![add user](/media/PFSENSE2/auth.png)

Let's create the FreeRADIUS user by providing the necessary information such as IP address, ports, and shared secret.
In my case: the IP address of the Ubuntu machine is 192.168.1.101.

![freeradius settings](/media/PFSENSE2/details.png)

After configuring the FreeRADIUS user, we will now proceed to configure the settings in the CaptivePortal section.

![captiveportal](/media/PFSENSE/18.png)

![add config](/media/PFSENSE/19.png)

Make sure you have all the necessary details and have configured the settings properly.

![captive settings 1](/media/PFSENSE/20.png)
![captive settings 2](/media/PFSENSE/21.png)
![pfsense settings 3](/media/PFSENSE/22.png)

Now, everything is properly configured.

To connect to the Internet, valid credentials configured in the FreeRADIUS users file must be provided. For this :

![firefox login](/media/PFSENSE/23.png)

If the browser does not offer the option to open the authentication interface, you can access it via the following link:


> http://192.168.1.1:8002/index.php?zone=myzone

192.168.1.1 is the address of pfSense, and myzone is the name of the Captive Portal configuration.

You should get this :

![captive portal](/media/PFSENSE/24.png)

Let's try using the credentials 'usertest' and 'testpassword'.

![creds](/media/PFSENSE/25.png)
![google works](/media/PFSENSE/26.png)

<div style="text-align: center;">
    <img src="/media/PFSENSE/tuco.gif" alt="Gif" style="width:400px;"> <br>
    IT'S WORKING ... AGAIIIN
</div>

![sucess](/media/PFSENSE/27.png)

Now, I will add more usernames to the database :

![adding users](/media/PFSENSE2/addusers.png)
let's test them.
![verify users](/media/PFSENSE2/test2.png)

Regarding the status of the Captive Portal in pfSense:

![verify users](/media/PFSENSE2/lastcheck.png)

In the case of incorrect credentials, it returns this.

![verify users](/media/PFSENSE/last.png)

To add another measure to ensure everything works well, be sure to configure the `radcheck` table to accept usernames only once to prevent any confusion or disruptions in the working solution.

```sql
ALTER TABLE radcheck ADD CONSTRAINT uc_username UNIQUE (username);
```

![update database](/media/PFSENSE2/updatedb.png)

## Conclusion

By following these steps, you can successfully link pfSense's Captive Portal with a FreeRADIUS server that uses a MySQL database as a reference for usernames and passwords.

This solution greatly simplifies and stabilizes the management of network access in a large infrastructure.

Overall, implementing a Captive Portal with FreeRADIUS on pfSense enhances network security and control while providing flexibility and ease of management.

✌️ Thanks for reading once again.

See you in another report.

![see ya](/media/PFSENSE2/see_ya.gif)
