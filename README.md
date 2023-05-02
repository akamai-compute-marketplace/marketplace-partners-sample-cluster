# Linode Marketplace Apps

The Linode Marketplace is designed to make it easier for developers and companies to share [One-Click Apps](https://www.linode.com/marketplace/) with the Linode community. One-Click Apps are portable and modular solutioning tools written as Ansible playbooks. The Linode Marketplace allows users to quickly deploy services and perform essential configurations on a Linode Compute Instance's first boot. 

A Marketplace deployment refers to an application (single service on a single node) or a cluster (multi-node clustered service such as Galera). A combination of StackScripts and Ansible playbooks give the Marketplace a one-click installation and delivery mechanism for deployments. The end user is billed just for the underlying cloud resources (compute instances, storage volumes, etc) in addition to any applicable BYOLs.

## Marketplace App Development Guidelines.

A Marketplace application consists of three major components: a Stackscript, Ansible playbooks, and Git repository to clone from.

### Stackscript

A [Stackscript](https://www.linode.com/docs/products/tools/stackscripts/guides/write-a-custom-script) is a BASH script adhering to industry best practices that is stored on Linode hosts and is accessible to all customers.

### Ansible Playbook

All Ansible playbooks should generally adhere to the [sample directory layout](https://docs.ansible.com/ansible/latest/user_guide/sample_setup.html#sample-ansible-setup) and best practices/recommendations from the latest Ansible [User Guide](https://docs.ansible.com/ansible/latest/user_guide/index.html).

## Creating Your Own One-Click Cluster

For more information on creating and submitting a Partner App for the Linode Marketplace please see [Contributing](CONTRIBUTING.md) and [Development](DEVELOPMENT.md).

