# OpenShift DIY cartridge for GlassFish 4

As people have been asking me how I got GlassFish 4 running on OpenShift, I decided to write down everything I learned in the process. I started out by following <a href="http://www.openshift.com/blogs/running-java-apps-in-the-cloud-with-glassfish-and-a-paas">these</a> <a href="https://github.com/openshift-quickstart/openshift-glassfish3-sample">examples</a>, but ran into a lot of trouble along the way. I hope this post can make the process a lot easier for you.

## Prerequisites

First of all: don't even bother trying this on Windows. Use OS X or Linux. Start by creating an <a href="https://www.openshift.com/">OpenShift</a> account. Using their web interface, create a new application with the DIY cartridge and add a MySQL cartridge to it as well. I will refer to this application as `yourapp` from now on. Lastly, you will need the OpenShift Client Tools, so <a href="https://www.openshift.com/developers/rhc-client-tools-install">install</a> those as well.

Next, download this repository. This is the official GlassFish 4 release, modified in the following ways:

- The `domain.xml` configuration file has been modified to run on OpenShift.
- It already includes the MySQL driver in `domain1/lib`.

The archive also contains the `start` and `stop` hooks you need.

## Setting up the server

Start by cloning your repository to your local machine:

```
rhc git-clone yourapp
```

This will create a `yourapp` directory. Move the files you downloaded (or cloned) into `yourapp` as follows:

- Move the `glassfish4` directory (the entire directory, not just its contents) into `yourapp/diy`.
- Move the `start` and `stop` hooks into `yourapp/.openshift/action_hooks` and check to make sure they're executable.

Finally, add, commit and push the changes you made back to OpenShift:

```
cd yourapp
git add .
git status
git commit -m "Added GlassFish"
git push
```

The push will cause your application to restart and execute the new start hook. Give it a few minutes just to be sure, then check if the server is running at <a href="http://yourapp-youraccount.rhcloud.com">http://yourapp-youraccount.rhcloud.com</a>.

I added the `git status` command in there so you can verify if Git picked up all the changes you made. If not, use `git add` to add the files Git missed.

If this is your first time using Git from the command-line, I recommend you set up your name and email address and change your default editor to something other than `vi`:

```
git config --global user.name "My Name"
git config --global user.email my@email.com
git config --global core.editor nano
```

## Deploying applications

To deploy an application, simply copy its **war** into `domain1/autodeploy` and run the previous commands again to add, commit and push the changes. GlassFish will automatically deploy your application after launch. If you think something went wrong, use `ssh` to log into your application and look for a
`your.war_deployed` file in the `autodeploy` directory:

```
rhc app ssh yourapp
cd $OPENSHIFT_REPO_DIR/
cd diy/glassfish4/glassfish/domains/domain1/autodeploy/
ls
```

If there is no such file, take a look at the server logs to find out what went wrong:

```
cd ../logs/
cat server.log | tail -n100
```

## Setting up resources

You cannot use the GlassFish administration console with OpenShift. Neither does `glassfish-resources.xml` seem to be supported. If you need to set up JDBC resources, connection pools, security realms, ... on your server, I recommend you use the following approach:

- Set up the resources you need on a local GlassFish server.
- Take a look at that server's `domain.xml` to see what changes were made.
- Make the same changes to `domain1/config/domain.xml` in `yourapp`. Usually, this just means adding one or two extra elements.
- As usual: add, commit and push the changes.

## Connecting to your MySQL server

Working with your MySQL server on OpenShift is pretty easy once you figure out how to connect to it from MySQL Workbench.

Launch Workbench and add a new connection. As you can't connect to MySQL on OpenShift directly, you need to use a TCP/IP over SSH connection. The settings I used for my application can be seen in the following screenshot:

![](images/glassfish-openshift-1.png)

To find the settings you need, use the `rhc app show yourapp` command. This is what I get for my `glassfish` application:

```
MacBook-Air:OpenShift Steven$ rhc app show glassfish
glassfish @ http://glassfish-svanimpe.rhcloud.com/ (uuid: 5232bc5f500446f41000014c)
-----------------------------------------------------------------------------------
Domain:  svanimpe
Created: Sep 13  9:18 AM
Gears:   1 (defaults to small)
Git URL: ssh://5232bc5f500446f41000014c@glassfish-svanimpe.rhcloud.com/~/git/glassfish.git/
SSH:     5232bc5f500446f41000014c@glassfish-svanimpe.rhcloud.com

diy-0.1 (Do-It-Yourself 0.1)
----------------------------
Gears: Located with mysql-5.1

mysql-5.1 (MySQL Database 5.1)
------------------------------
Gears:          Located with diy-0.1
Connection URL: mysql://$OPENSHIFT_MYSQL_DB_HOST:$OPENSHIFT_MYSQL_DB_PORT/
Database Name:  glassfish
Password:       supersecret
Username:       adminHWvBmju
```

The output will give you all of the values you need except for the MySQL hostname and port, as the command only
shows the environment variables and not the actual values. To find out those values, `ssh` into your application and
use the `env` and `grep` commands:

```
[glassfish-svanimpe.rhcloud.com 5232bc5f500446f41000014c]\> env | grep MYSQL
OPENSHIFT_MYSQL_DIR=/var/lib/openshift/5232bc5f500446f41000014c/mysql/
OPENSHIFT_MYSQL_DB_PORT=3306
OPENSHIFT_MYSQL_DB_HOST=127.9.226.130
OPENSHIFT_MYSQL_DB_PASSWORD=supersecret
OPENSHIFT_MYSQL_IDENT=redhat:mysql:5.1:0.2.2
OPENSHIFT_MYSQL_DB_USERNAME=adminHWvBmju
OPENSHIFT_MYSQL_DB_SOCKET=/var/lib/openshift/5232bc5f500446f41000014c/mysql//socket/mysql.sock
OPENSHIFT_MYSQL_DB_URL=mysql://adminHWvBmju:asIQ9W3wc26y@127.9.226.130:3306/
OPENSHIFT_MYSQL_DB_LOG_DIR=/var/lib/openshift/5232bc5f500446f41000014c/mysql//log/
```

That's it! You should now be up and running with both GlassFish 4 and MySQL.
