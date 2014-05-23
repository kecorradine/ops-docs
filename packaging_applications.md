# Packaging via Maven

The [jdeb](https://github.com/tcurdt/jdeb) Maven plugin is used to create .deb packages for
deployment to Ubuntu systems.

The package produced needs to contain enough to get the application running with little, if any
further, configuration. To that end, a typical package will contain:

* A self-contained jar file that listens to a port for input from the network.
* Configuration to create a user for the service to run as.
* Configuration files, configured with sane defaults.
* A start script to start the service.

## The application user

Priviledge seperation is used to enhance security. A user and group specifically for running the application is created at install time. This should be a system user. This user and group should only be able to write files in /var/opt/<app_name>.

The naming convention for this user is to use than same value as the package name, e.g. the snomed-release-service-api application runs as the snomed-release-service-api user.

## Installation locations

The follow installations must be followed to avoid confusion. This follows the [Linux File System Hierarchy](http://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard).

* /opt/<app_name> - the application itself, typically with a jar in /opt/<app_name>/lib and any control tools in /opt/<app_name>/bin or /opt/<app_name>/sbin, depending on purpose. This tree must be root:root owner with no other write bits set.
* /etc/opt/<app_name> - configuration files. The location of these files is to be specified on the command line. Typically this will contain authentication information for 3rd party services perhaps via a properties file and logging configuration (e.g. a log4j.xml file, configure for production use). Configuration files containing sensitive information should be root:<app_user> owned with a file mode of 0640.
* /var/opt/<app_name> - logs, temporary files etc. This is the only location to which the application can write.
* /var/log/<app_name> - a symlink pointing to /var/opt/<app_name/logs

## Producing the jar file

The jar file should be able to be run as a free-standing daemon. Tomcat and Jetty are both able to
do this and either is a fine choice. The configuration of maven to produce this file is beyond
the scope of this document, but examples can be see in the [snomed-release-service](https://github.com/IHTSDO/snomed-release-service) api, web and builder POM files.

In general, the jdeb section of the pom.xml will contain sections for each file or directory to be included in the package. Emptry directories can either be created in the pom.xml or via the postinst control file.

## Producing the package

# Tree layout

Within the projects tree, the follow structure should exist

```
src/
  deb/
    control/
      control
      postinst
      postrm
      preinst
      prerm
    supervisor.conf
    otherdefault.properties
    log4j.xml
    defaults
```

### Application configuration

Application configuration files should exist that allow the specification of environmental configuration such as database host, username, password and so on. This file can be in any text format. It must live in /etc/opt/<app_name> and the application can be told of its location via a command line parameter in the applications supervisor conf file

In addition to the application configuration, logging should be configured. This can be in the form of a log4j.xml file in /etc/opt/<app_name> or via stdout and redirection with a suitable logrotate configuration contained in the software package. Logs should be written to a file and rotated to avoid the disk filling up.

### Control files

The following control files manage how the package is install, removed and upgraded. As a general guide the control file for an application must:

* Create a user and group for the application to run as (in preinst)
* Create /var/opt/<app_name> and any directories under that that are required (postinst)
* Handle the restarting of the application during an upgrade (postinst)
* Stop the application before removal or purge (prerm)
* Remove the /var/opt/<app_name> on purge

Note that jdeb supports a limited template format for the control files and Maven properties can be injected using the [[propertyName]] format.

See the [SNOMED Release Service API control sub tree](https://github.com/IHTSDO/snomed-release-service/tree/feature/config_logs/api/src/deb/control) for examples.

### Start script

[Supervisord](http://supervisord.org/) is used to managed the starting and stopping of the application. See the [release service supervisor.conf](https://github.com/IHTSDO/snomed-release-service/blob/master/api/src/deb/supervisor.conf) for an example.

### Testing in a VM

The ihtsdo-ansible repository contains a suitable Vagrant file for testing.  To use:

```
$ cd ihtsdo-ansible
$ vagrant up
```

Copy the  package into the ihtsdo-ansible directory

```
$ vagrant ssh
$ sudo dpkg -i /vagrant/<pkgname>
$ apt-get update
$ apt-get -f install
```

The application should then be installed and running. The VM listens on 192.168.33.10

