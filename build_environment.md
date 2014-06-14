The build environment consists of the following services:

* Jenkins
* SonarQube
* Sonatype Nexus

All are provisioned using Ansible in a hands off manner.

# Provisioning

Broad configuration is given in the inventory/ folder of the [ihtsdo-ansible](https://github.com/IHTSDO/ihtsdo-ansible)
repository. See the [README](https://github.com/IHTSDO/ihtsdo-ansible/README.md) for information on the set up on inventory files.

To provision a new build environment, edit the inventory accordingly and run:

```sh
$ ansible-playbook -i inventory/live.ini build_environment.yml -u root
```

On subsquent runs, omit the `-u root` or change `root` to your username.

# SonarQube set up

Login to the new SonarQube install, username admin, password admin. Change the password!

Create a jenkins user, with a suitable password

Under General settings, configure [Email](email.md).

To copy a database from an old instance to new, simplying dump and restore the database directly.

# Sonatype Nexus

## Install Nexus on a server

Assuming a new server on Digital Ocean.

1. Update inventory/live_inventory.ini, adding the servers FQDN in [nexus]
2. Run:
```sh
$ ansible-playbook build-environment.yml -i inventory/live_inventory.ini --limit nexus -u root
``
3. Wait a few minutes for Nexus to start. Your new site is accessable at http://<server_fqdn>

### Install Nexus in a VM:

```sh
$ vagrant up
$ vagrant provision
$ ansible-playbook build-environment.yml -i inventory/vagrant.ini -u username
```

Where `username` is a valid user with root via sudo.

Once finished, wait a few minutes from Nexus to spin up then login in at http://192.168.33.10 as admin/admin123

## Configuring the APT plugin

To configure the apt plugin, go to the admin interface, under Administration, Capabilities. Select New and fill
in the fields as follows:

* Secure keyring location: /var/lib/sonatype-nexus/pkg_signing_gpg.keyring
* Key ID: AA5872B8
* Passphrase for the key: <passphrase>

## Configuring SMTP

On the server configuration page under SMTP Settings. See the [email document](email.md) for specific settings.

# Jenkins server set up

There are two Jenkins servers - the application build server and the Ansible deployment server. 

1. Update plugins
2. Install the following plugins
* Gravatar
* GitHub OAuth
* Jenkins Sonar (Build Server only)
* Config File Provider (Build server only)
* Git plugin
* GitHub plugin
* Log Command
* IRC
* SSH Agent
* Role-based Authorization Strategy (Ansible server only)

## Configure GitHub authentication

* In Github
* Under Account Settings
* IHTSDO
* Application
* Register new application. Callback URL is in the form of
https://jenkins.ihtsdotools.org/securityRealm/finishLogin, editoring according to your sites URL.

Back in Jenkins:

* Under Configure Global Security
* Select Github Authentication Plugin
* Copy & paste the Client ID and Client Secret from the Github application page
* Select Prevent Cross Site Request Forgery exploits
* Select the Default Crumb Issuer

On the build server do the following:
* Select Github Commiter Authorization Strategy under Auhorization
* Add the Github usernames of admin users to the Admin User Names
* Insert IHTSDO (or your Github organisations name) to Participant in Organisation
* Select Great READ options accordingly. Save.

On the Ansible server:
* Select Role-Based Strategy. Save
* Select Manage and Assign Roles, near the bottom of the page.
* Under assign roles, make sure an admin roles exists, with all priviledges.
* Add a developers role, with Overall Read, Job Read and Job Build permissions.
* Under assign roles, add the IHTSDO group to develops
* Add inviduals how will manage the servier s the admins group. Save.


## Configure Jenkins

* JDK
Name: System OpenJDK 7
JAVA_HOME: /usr/lib/jvm/java-7-openjdk-amd64

* Maven
Name: System Maven 3
MAVEN_HOME: /usr/share/maven

* Sonar
Name: Local SonarQube install
Server URL: is whatever the sonarqube_nginx_fqdn is set to. For IHTSDO core it's https://sonarqube.ihtsdotools.org
Sonar Account Login/Password: Whatever is set in Sonar
Database URL: jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true
Database username: sonar unless changed in inventory
Database password: from the Ansible inventory
Database driver: com.mysql.jdbc.Driver

* E-mail notification
Set to a suitable [email provider](email.md).
Set Jenkins Location, System Admin to a suitable email address

* Git
Name: System git
Path to Git executable: /usr/bin/git

* Managed files

### Global Maven settings.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <mirror>
      <id>IHTSDO-sonatype-nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://maven.ihtsdotools.org/content/groups/public</url>
    </mirror>
  </mirrors>
</settings>
```
### Maven settings.xml

Add credentials for the Nexus Sonatype Repository via the usual Jenkins Credentials mechanism

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <id>ihtsdo-nexus-public</id>
    <username>jenkins</username>
    <password>password</password>
  </servers>
</settings>
```

Add Credentials with a a ServerId of ihtsdo-nexus-public.

These values can be set as defaults in the main Jenkins configuration page

* Credentials

Add SSH key, username jenkins. If using the official IHTSDO ansible inventory, the ssh key is in ~/.ssh

## Upgrading Jenkins
 
Every 12 weeks a new stable version of Jenkins is releaseed. To upgrade:

1. Log in to the build server
2. Run apt-get update
3. Run apt-get upgrade. When asked about overriding /etc/default/jenkins, answer N.


