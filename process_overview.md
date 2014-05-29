The approach taken is to produce a standalone jar bundling either Tomcat or Jetty or similar. From an operations point of view either is fine, but it must run stand alone. Configuration is pulled in an external configuration file which is passed on the command line. When run, the jar should listen on localhost only and on a high port.

A debian .deb package is produced using the maven jdeb plugin which contains this jar, a default configuration file and any addition files or directory structure. A start script creates a user for the service to run as. The scripts with in the .deb file handle application upgrade (e.g. restarting the application, maybe triggering migrations). supervisord is used to control the application running. The .deb has dependencies on required applications such as java.

A seperate .deb is created containing any static assets and an nginx configuration for the application. This pacakge does not control nginx and the configuration only acts as a template.

Ideally, the user of these packages should be able to simply install these packages, restart nginx and away they go. If there is an external database it should only need configurating with username, password and location. Ideally the application would deal with the database content.

In terms of release process, jgitflow is used. This is a maven implementation of the gitflow process and allows for efficient release and branch management. At the very least a develop and master branch must exist, with develop being the snapshot versions and the master being the releases. On pushing to their branches jenkins jobs are triggered which cause a mvn clean install. On success, the resulting packages is uploaded to a maven repository (https://nexus.ihtsdotool.org). This repository also signs and indexes the .deb files. The job also triggers and Ansible job on a seperate Jenkins server which runs the deployment role(s) for the application modules. This makes sure the application gets updated and suitable configuration is applied, depending on the environment.

The snomed-release-service is the furthest along with this approach, https://github.com/IHTSDO/snomed-release-service. There is also the https://github.com/IHTSDO/ihtsdo-ansible repository which contains the various roles and automation.

http://12factor.net is worth reading through to see where this fits together. It's not being followed hard & fast, but provides a general aim.
