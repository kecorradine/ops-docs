# Packaging

The [jdeb](https://github.com/tcurdt/jdeb) Maven plugin is used to create .deb packages for
deployment to Ubuntu systems.

The package produced needs to contain enough to get the application running with little, if any further, configuration. To that end, a typical package will contain:

* A self-contained jar file that listens to a port for input from the network. These ports are standarised. Please add your service to the [document](service_ports.md).
* Configuration to create a user for the service to run as.
* Configuration files, configured with sane defaults.
* A start script to start the service.

The resulting packages are design to be use in either a typical Ubuntu or Debian environment, or in a Docker environment where supervisord is used as the entry point. Note that at the time of writing (May 2014) only Ubuntu 12.04 and 14.04 deploy has been tested.

# Application proxy

Generally speaking, where an application is exposed to end users via a web browser Nginx should be used as a proxy for the application. A package containing required assets can be produced and deployed. This package can contain a default nginx configuration to be placed in /etc/nginx/conf.d/<pkg_name>.conf. The package control should _not_ restart Nginx or manage any other Nginx configuration. That should be done via Ansible.

Context should be served from /srv/http/<pkg_name>.

The [SNOMED Release Service web component](https://github.com/IHTSDO/snomed-release-service/tree/master/web) can serve as an example.

# The application user

Priviledge seperation is used to enhance security. A user and group specifically for running the application is created at install time. This should be a system user. This user and group should only be able to write files in /var/opt/<app_name>.

The naming convention for this user is to use than same value as the package name, e.g. the snomed-release-service-api application runs as the snomed-release-service-api user.

# Installation locations

The follow installations must be followed to avoid confusion. This follows the [Linux File System Hierarchy](http://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard).

* /opt/<app_name> - the application itself, typically with a jar in /opt/<app_name>/lib and any control tools in /opt/<app_name>/bin or /opt/<app_name>/sbin, depending on purpose. This tree must be root:root owner with no other write bits set.
* /etc/opt/<app_name> - configuration files. The location of these files is to be specified on the command line. Typically this will contain authentication information for 3rd party services perhaps via a properties file and logging configuration (e.g. a log4j.xml file, configure for production use). Configuration files containing sensitive information should be root:<app_user> owned with a file mode of 0640.
* /var/opt/<app_name> - logs, temporary files etc. This is the only location to which the application can write.
* /var/log/<app_name> - a symlink pointing to /var/opt/<app_name/logs, optional

Note that supervisor by default will log stdout and stderr to /var/log/supervisor and can be configured to rotate logs.

# Producing a jar file

The jar file should be able to be run as a free-standing daemon. Tomcat and Jetty are both able to do this and either is a fine choice. The configuration of maven to produce this file is beyond the scope of this document, but examples can be see in the [snomed-release-service](https://github.com/IHTSDO/snomed-release-service) api, web and builder POM files.

The naming using through out the pom hierarchy should be consistent and terse. Do not postfix the root pom artifactId with -parent as it limits its reuse.

```xml
  <parent>
    <groupId>org.ihtsdo.otf.mapping</groupId>
    <artifactId>oft-mapping-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>
```

In module pom.xml files, define the parent in the same form.

```
  <groupId>org.ihtsdo.otf.mapping</groupId>
  <artifactId>oft-mapping-service</artifactId>
  <version>0.0.1-SNAPSHOT</version>
```

Set properties for `execFinalName` (the name of the resulting self container jar) and `packageName` (the resulting name for the package). The following can pasted into your modules pom.xml directly.

```
  <properties>
    <execFinalName>exec-${project.build.finalName}.jar</execFinalName>
    <packageName>${project.parent.artifactId}-${project.artifactId}</packageName>
  </properties>
```

Be sure to set the modules build `finalName` to a static value, to ensure the same name is used for the resulting war. Again, this can be pasted directly.

```xml
  <build>
    <finalName>${project.artifactId}</finalName>
  ...
```

The creation of the tomcat jar. You may wish to edit the `path` that your app is exposed via.

```xml
      <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <version>2.1</version>
        <executions>
          <execution>
            <id>tomcat-run</id>
            <goals>
              <goal>exec-war-only</goal>
            </goals>
            <phase>package</phase>
            <configuration>
              <path>/api</path>
              <finalName>${execFinalName}.jar</finalName>
              <enableNaming>true</enableNaming>
            </configuration>
          </execution>
        </executions>
      </plugin>
```


# Producing the package

## Tree layout

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

This can be copied from an existing module and editted accordingly. In particular, the files directly under deb/ are not templated and so will need to be manually altered. Note the naming convention of using the `${parent.artifactId}-${artifactId}` for naming of items related to the build. This is sometime automatically derived so case should be take to use the same form everywhere it is needed.

## Application configuration

Application configuration files should exist that allow the specification of environmental configuration such as database host, username, password and so on. This file can be in any text format. It must live in /etc/opt/<app_name> and the application can be told of its location via a command line parameter in the applications supervisor conf file

In addition to the application configuration, logging should be configured. This can be in the form of a log4j.xml file in /etc/opt/<app_name> or via stdout and redirection with a suitable logrotate configuration contained in the software package. Logs should be written to a file and rotated to avoid the disk filling up.

## Control files

The following control files manage how the package is install, removed and upgraded. As a general guide the control file for an application must:

* Create a user and group for the application to run as (in preinst)
* Create /var/opt/<app_name> and any directories under that that are required (postinst)
* Handle the restarting of the application during an upgrade (postinst)
* Stop the application before removal or purge (prerm)
* Remove the /var/opt/<app_name> on purge

Note that jdeb supports a limited template format for the control files and Maven properties can be injected using the [[propertyName]] format.

See the [SNOMED Release Service API control sub tree](https://github.com/IHTSDO/snomed-release-service/tree/feature/config_logs/api/src/deb/control) for examples.

In general, the jdeb section of the pom.xml will contain sections for each file or directory to be included in the package. Empty directories can either be created in the pom.xml or via the postinst control file.

## Start script

[Supervisord](http://supervisord.org/) is used to managed the starting and stopping of the application. See the [release service supervisor.conf](https://github.com/IHTSDO/snomed-release-service/blob/develop/api/src/deb/supervisor.conf) for an example.

Note that the -resetExtract  -extractDirectory options must be specified. They seem to avoid problems which badly extracted, and hence corrupted, files.

## jdeb configuration

Typical jdeb configuration will look like the following. This example is using log4j rather that
simply taking stdout and stderr via supervisord and letting it managed the log files.

```xml

      <plugin>
        <groupId>org.vafer</groupId>
        <artifactId>jdeb</artifactId>
        <version>1.1.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>jdeb</goal>
            </goals>
            <configuration>
              <deb>${project.build.directory}/${packageName}-${project.version}-all.deb</deb>
              <controlDir>${basedir}/src/deb/control</controlDir>
              <snapshotExpand>true</snapshotExpand>
              <snapshotEnv>BUILD_NUMBER</snapshotEnv>
              <verbose>true</verbose>
              <classifier>all</classifier>
              <signPackage>false</signPackage>
              <dataSet>
                <data>
                  <src>${project.build.directory}/${execFinalName}</src>
                  <dst>app.jar</dst>
                  <type>file</type>
                  <mapper>
                    <type>perm</type>
                    <prefix>/opt/${packageName}/lib/</prefix>
                  </mapper>
                </data>
                <data>
                  <src>${basedir}/src/deb/supervisor.conf</src>
                  <dst>/etc/supervisor/conf.d/${packageName}.conf</dst>
                  <type>file</type>
                  <conffile>true</conffile>
                </data>
                <data>
                  <src>${basedir}/src/deb/config.properties</src>
                  <dst>/etc/opt/${packageName}/config.properties</dst>
                  <type>file</type>
                  <conffile>true</conffile>
                  <mapper>
                    <type>perm</type>
                    <group>${packageName}</group>
                    <filemode>0640</filemode>
                  </mapper>
                </data>
                <data>
                  <src>${basedir}/src/deb/log4j.xml</src>
                  <dst>/etc/opt/${packageName}/log4j.xml</dst>
                  <type>file</type>
                  <conffile>true</conffile>
                  <mapper>
                    <type>perm</type>
                    <group>${packageName}</group>
                    <filemode>0640</filemode>
                  </mapper>
                </data>
              </dataSet>
            </configuration>
          </execution>
        </executions>
      </plugin>
```

# Testing in a VM

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

# Ansible

Ansible is used to deploy and configure applications. In particular it:

* Configure which repository to use (IHTSDO.repository role)
* Installs the package from the Nexus based apt repository
* Configure the application from a template
* Configure ufw firewall either via the IHTSDO.nginx module if Nginx is used, otherwise manually.

In a normal run it also updates user accounts, system time zone and SSH configuration. 

When packaging up a new application, copy an existing role that is similar in layout and edit accordingly.

Testing can be done via Vagrant by adding the IP address of your VM to the vagrant.ini and refering to that.

The intention of the Ansible repository is that is it is self documenting. To understand the configuraition of a role, study the tasks/main.yml and the defaults/main.yml file and any templates/. These show what is configured by the role and
the variables needed to change that configuration.

Variables should be placed in the appropriate inventory file. In general, passwords live in host_vars/ files, broad configuration in group_vars/. It is worth spending time studying these files to understand how this data is used in the configuration of applications. It is the inventory that sets the configuration for a particular environment.



