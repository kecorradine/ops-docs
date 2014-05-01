# SNOMED Release Service

## In a VM

Clone the repository, and make sure you have the latest Vagrant and Ansible installed.

Edit Vagrantfile and make sure

```ansible.playbook = "snomed_release_service.yml"```

is set the in the provisioning section.

Run vagrant up, and once complete the Release Service is accessible via http://192.168.33.10

