This document covers the broad approach of a 'feature complete' application. It's possible to omit or amend sections when different technologies are used, but the closer the system is to the broad principles the less work is required to deploy it.

# Tenents

* The application is a stand alone process that listens on a known port for HTTP requests. (tomcat exec-war) 
* A proxy sits between the application and the end user and serves static assets and provides security via TLS if required. (nginx)
* The application is deploy via a standard operating system package. (.deb package)
* Static assets are also packaged with a stock proxy configuration (.deb package)
* The application runs as its own user. This users can write data but cannot modify itself (a user and group with the same name as the application)
* The application is controlled via a manager (supervisord)
* Logs are produced to stdout and stderr. The start up manager controls where then go (supervisord)
* Git flow is used to manage versioning and release (maven jgitflow)



The approach taken is to produce a standalone jar bundling either Tomcat or Jetty or similar. From an operations point of view either is fine, but it must run stand alone. Configuration is pulled in an external configuration file which is passed on the command line. When run, the jar should listen on localhost only and on a high port.

A debian .deb package is produced using the maven jdeb plugin which contains this jar, a default configuration file and any addition files or directory structure. A start script creates a user for the service to run as. The scripts with in the .deb file handle application upgrade (e.g. restarting the application, maybe triggering migrations). supervisord is used to control the application running. The .deb has dependencies on required applications such as java.

A seperate .deb is created containing any static assets and an nginx configuration for the application. This pacakge does not control nginx and the configuration only acts as a template.

Ideally, the user of these packages should be able to simply install these packages, restart nginx and away they go. If there is an external database it should only need configurating with username, password and location. Ideally the application would deal with the database content.

In terms of release process, jgitflow is used. This is a maven implementation of the gitflow process and allows for efficient release and branch management. At the very least a develop and master branch must exist, with develop being the snapshot versions and the master being the releases. On pushing to their branches jenkins jobs are triggered which cause a mvn clean install. On success, the resulting packages is uploaded to a maven repository (https://nexus.ihtsdotool.org). This repository also signs and indexes the .deb files. The job also triggers and Ansible job on a seperate Jenkins server which runs the deployment role(s) for the application modules. This makes sure the application gets updated and suitable configuration is applied, depending on the environment.

The snomed-release-service is the furthest along with this approach, https://github.com/IHTSDO/snomed-release-service. There is also the https://github.com/IHTSDO/ihtsdo-ansible repository which contains the various roles and automation.

http://12factor.net is worth reading through to see where this fits together. It's not being followed hard & fast, but provides a general aim.
