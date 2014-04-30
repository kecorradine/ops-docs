The build environment consists of the following services:

* Jenkins
* SonarQube
* Sonatype Nexus

All are provisioned using Ansible in a hands off manner.

## Provisioning

Broad configuration is given in the inventory/ folder of the [ihtsdo-ansible](https://github.com/IHTSDO/ihtsdo-ansible)
repository. To provision a new server, edit the inventory accordingly and run:

ansible-playbook -i inventory/live.ini build_environment.yml -u <username> 

On a new build, username will be root, otherwise your normal username.

## SonarQube set up

Login to the new SonarQube install, username admin, password admin. Change the password!

Create a jenkins user, with a suitable password

## Jenkins server set up

1. Update plugins
2. Install the following plugins
* Gravatar plugin
* GitHub OAuth plugin
* Jenkins Sonar plugin
* Config File Provider plugin
* Git plugin
* GitHub plugin

### Configure GitHub authentication

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
* Select Github Commiter Authorization Strategy under Auhorization
* Add the Github usernames of admin users to the Admin User Names
* Insert IHTSDO (or your Github organisations name) to Participant in Organisation
* Select Great READ options accordingly. Save.
* Select Prevent Cross Site Request Forgery exploits and select Default Crumb Issuer

### Configure Jenkins

* JDK
Name: System OpenJDK 7
JAVA_HOME: /usr/lib/jvm/java-7-openjdk-amd64

* Maven
Name: System Maven 3
MAVEN_HOME: /usr/share/maven

* Sonar
Name: Local SonarQube install
Sonar Account Login/Password: Whatever is set in Sonar
Database URL: jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true
Database username: sonar unless changed in inventory
Database password: from the Ansible inventory
Database driver: com.mysql.jdbc.Driver

* E-mail notification
Set to a suitable email provider.
Set Jenkins Location, System Admin to a suitable email address

* Git
Name: System git
Path to Git executable: /usr/bin/git

* Managed files

#### Global Maven settings.xml
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
#### Maven settings.xml

Add credentials for the Nexus Sonatype Repository via the usual Jenkins Credentials mechanism

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" 
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <id>deployment</id>
    <username>jenkins</username>
    <password>password</password>
  </servers>
</settings>
```

Add Credentials with a a ServerId of deployment.

These values can be set as defaults in the main Jenkins configuration page

* Credentials

Add SSH key, username jenkins. If using the official IHTSDO ansible inventory, the ssh key is in ~/.ssh
