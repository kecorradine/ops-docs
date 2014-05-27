To create a new job on Jenkins:

* Job name - generally the same as the Git repository and branch e.g. snomed-release-service-develop
* Build a maven2/3 project
* Select Discard Old Builds, with a suitable number of builds or days.
* Pass the HTTPS URL of the GitHub project.
* Enter a nice display name under the Advanced Project Options (e.g. SNOMED Release Service: Build Snapshot)
* Select source code management as Git and enter the SSH repository URL, e.g git@github.com:IHTSDO/snomed-release-service.git)
* Make sure the right branch is in Branches to build. (e.g. */develop or */master)
* Build Triggers = Poll SCM. Set schedule to H */3 * * *
* Select SSH in Build Environment and select the Jenkin SSH key
* Root POM is pom.xml, Goals and options are 'clean install'
* Post steps, run only if build succeeds
* Add an Execute Shell post build step which calls curl and then buildByToken URL of a suitable deployment job on the Ansible Jenkins server.
* Add a Deploy artifacts to Maven repository Post-build Action
* Add IRC notification


In Github, add add a Jenkins (Git plugin) Webhook to the repository to call https://jenkins.ihtsdotools.org to trigger the job on pushs.
