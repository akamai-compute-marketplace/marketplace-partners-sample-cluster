# Marketplace Cluster Development Guidelines

A Marketplace Cluster leverages the Linode API and Ansible to deploy and configure a multi-node service such as Redis with replicas. The end-user’s Linode API token must be scoped with appropriate permissions to add and remove the necessary compute resources. A requirement of clustered deployments is that the “provisioner” (the initial Linode deployed by the Stackscript) configures itself as a member of the cluster. This is generally achieved by adding a specific delegation of `$NODE1` to localhost. In additon, please adhere to the following guidelines for clustered apps:
 
- Marketplace Clusters should use fully supported Linode Ubuntu 22.04 LTS images when possible. 
- The deployment of the service should be “hands-off,” requiring no command-line intervention from the user before reaching its initial state. The end user should provide all necessary details via User Defined Variables (UDF) defined in the StackScript, so that Ansible can fully automate the deployment.
- There is not currently password strength validation for StackSctript UDFs, therefore, whenever possible credentials should be generated and provided to the end-user.
- At no time should billable services that will not be used in the final deployment be present on the customer’s account.
- To whatever extent possible all Marketplace Clusters should use fully maintained packages and automated updates. 
- All Marketplace Clusters should include Linux server best practices such as a sudo-user, SSH hardening and basic firewall configurations. 
- All Marketplace Clusters should generally adhere to the best practices for the deployed service, and include security such as SSL whenever possible. 
- All Marketplace installations should be minimal, providing no more than dependencies and removing deployment artifacts. 

## Deployment Scripts

All Bash files, including the deployment Stackscript for each Marketplace cluster is kept in the `scripts` directory. Deployment Stackscripts should adhere to the following conventions.

- The StackScript should implement the [Linux trap command](https://man7.org/linux/man-pages/man1/trap.1p.html) for error handling.
- The primary purposes of the Stackscript is to set up the Ansible enviroment. This includes assigning variables, creating a working directory, cloning the correct Marketplace cluster repo, creating the Python virtual environment, and executing the Ansible playbooks.
  - Installations and configurations that are not necessary for supporting the Ansible environment (i.e. Python or Git dependencies) should be performed with Ansible playbooks, and not included in the Stackscript. The StackScript should be as slim as possible, letting Ansible do most of the heavy lifting.
  - All working directories should be cleaned up on successful completion of the Stackscript.
  - A public deployment script must conform to the [Stackscript requirements](https://www.linode.com/docs/guides/writing-scripts-for-use-with-linode-stackscripts-a-tutorial/) and we strongly recommend including a limited number of [UDF variables](https://www.linode.com/docs/guides/writing-scripts-for-use-with-linode-stackscripts-a-tutorial/#user-defined-fields-udfs).

## Ansible Playbooks 

All Ansible playbooks should generally adhere to the [sample directory layout](https://docs.ansible.com/ansible/latest/user_guide/sample_setup.html#sample-ansible-setup) and best practices/recommendations from the latest Ansible [User Guide](https://docs.ansible.com/ansible/latest/user_guide/index.html).

- All Ansible playbooks for Marketplace Clusters should include common [`.ansible-lint`](../.ansible-lint), [`.yamllint`](../.yamllint), [`ansible.cfg`](../ansible.cfg) and [`.gitignore`](../.gitignore) files. 
- All Ansible playbooks should use Ansible Vault for initial secrets management. Generated credentials should be provided to the end-user in a standard `.deployment-secrets.txt` file located in the sudo user’s home directory. 
- Whenever possible Jinja should be leveraged to populate a consistent variable naming convention during [node provisioning.](../provision.yml)
- It is recommended to import service specific tasks as modular `.yml` files under the application’s `main.yml.` 

Marketplace Cluster playbooks should align with the following sample directory trees. There may be certain clustered applications that require deviation from this structure, but they should follow as close as possible. To use a custom FQDN see [Configure your Linode for Reverse DNS](https://www.linode.com/docs/guides/configure-your-linode-for-reverse-dns/).

```
$CLUSTER_APP/
  ansible.cfg
  collections.yml
  provision.yml
  requirements.txt
  site.yml
  hosts
  LICENSE
  .ansible-lint
  .yamllint
  .gitignore

  group_vars/
    $CLUSTER_APP/
      vars 
      secret_vars
  
  scripts/
    ss.sh
    run.sh

  roles/
    $CLUSTER_APP/
      handlers/
        main.yml
      tasks/
        main.yml
      templates/
        $FILE.j2
    common/ 
      handlers/ 
        main.yml
      tasks/ 
        main.yml
        firewall.yml
        ssl.yml
        hostname.yml
        $CLUSTER_APP.yml
    post/ 
     handlers/ 
        main.yml
      tasks/ 
        main.yml 
        creds.yml
        sshkeys.yml
        sudouser.yml
```  

As general guidelines: 
  - The `secret_vars` file should be encrypted with Ansible Vault
  - The `roles` should general and conform to the following standards:
    - `common` - including preliminary configurations and Linux best practices.
    - `$cluster_app` - including all necessary plays for service/cluster deployment and configuration.
    - `post` - any post installation tasks such as clean up operations and generating additonal user credentials. 


For more information on roles please refer to the [Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#using-roles-at-the-play-level).


### Cluster Development Tips and Tricks 

A requirement of clustered deployments is that the **"provisioner"** (the initial Linode deployed by the Stackscript) configures itself as a member of the cluster. This is generally achieved by adding a specific delegation of `$NODE1` to localhost. We recommend assigning the provisioner into a unique group and creating a delegation to localhost in the `hosts` file.

```
[APP_provisioner]
        localhost ansible_connection=local user=root
```

Tasks can then be delegated to the provisioner with the `delegate_to` argument:

```
delegate_to: "{{ groups['APP_provisioner'][0] }}"
```

Additionally, a dynamic number of cluster members can be provisioned and populated into the Ansible `hosts` file with Jinja loops in the `provision.yml` playbook. See [provision.yml](../provision.yml) to see an example of how we utilize Jinja loops.

Marketplace Cluster Stackscripts require a `cluster_size` UDF to estimate billing for the cluster correctly.

```
# <UDF name="cluster_size" label="Redis cluster size" default="3" oneof="3,5" />
```

Here are some additonal examples of UDFs your deployment can use in the `ss.sh` StackScript: 

```
## Example Cluster Settings
## Deployment Variables
# <UDF name="token_password" label="Your Linode API token" />
# <UDF name="sudo_username" label="The limited sudo user to be created in the cluster" />

## Linode/SSH Security Settings
# <UDF name="sslheader" label="SSL Information" header="Yes" default="Yes" required="Yes">
# <UDF name="country_name" label="Details for self-signed SSL certificates: Country or Region" oneof="AD,AE,AF,AG,AI,AL,AM,AO,AQ,AR,AS,AT,AU,AW,AX,AZ,BA,BB,BD,BE,BF,BG,BH,BI,BJ,BL,BM,BN,BO,BQ,BR,BS,BT,BV,BW,BY,BZ,CA,CC,CD,CF,CG,CH,CI,CK,CL,CM,CN,CO,CR,CU,CV,CW,CX,CY,CZ,DE,DJ,DK,DM,DO,DZ,EC,EE,EG,EH,ER,ES,ET,FI,FJ,FK,FM,FO,FR,GA,GB,GD,GE,GF,GG,GH,GI,GL,GM,GN,GP,GQ,GR,GS,GT,GU,GW,GY,HK,HM,HN,HR,HT,HU,ID,IE,IL,IM,IN,IO,IQ,IR,IS,IT,JE,JM,JO,JP,KE,KG,KH,KI,KM,KN,KP,KR,KW,KY,KZ,LA,LB,LC,LI,LK,LR,LS,LT,LU,LV,LY,MA,MC,MD,ME,MF,MG,MH,MK,ML,MM,MN,MO,MP,MQ,MR,MS,MT,MU,MV,MW,MX,MY,MZ,NA,NC,NE,NF,NG,NI,NL,NO,NP,NR,NU,NZ,OM,PA,PE,PF,PG,PH,PK,PL,PM,PN,PR,PS,PT,PW,PY,QA,RE,RO,RS,RU,RW,SA,SB,SC,SD,SE,SG,SH,SI,SJ,SK,SL,SM,SN,SO,SR,SS,ST,SV,SX,SY,SZ,TC,TD,TF,TG,TH,TJ,TK,TL,TM,TN,TO,TR,TT,TV,TW,TZ,UA,UG,UM,US,UY,UZ,VA,VC,VE,VG,VI,VN,VU,WF,WS,YE,YT,ZA,ZM,ZW" />
# <UDF name="state_or_province_name" label="State or Province" example="Example: Pennsylvania" />
# <UDF name="locality_name" label="Locality" example="Example: Philadelphia" />
# <UDF name="organization_name" label="Organization" example="Example: Akamai Technologies"  />
# <UDF name="email_address" label="Email Address" example="Example: user@domain.tld" />
# <UDF name="ca_common_name" label="CA Common Name" default="Redis CA" />
# <UDF name="common_name" label="Common Name" default="Redis Server"  />

## Cluster Settings
# <UDF name="clusterheader" label="Cluster Settings" default="Yes" header="Yes">
# <UDF name="add_ssh_keys" label="Add Account SSH Keys to All Nodes?" oneof="yes,no"  default="yes" />
# <UDF name="cluster_size" label="Redis cluster size" default="3" oneof="3,5" />
```
### UDF Tips and Tricks 

- UDFs without a `default` are required. 
- UDFs with a `default` will write that default if the customer does not enter a value. Non printing characters are treated literally in defaults.
- UDFs containing the string `password` within the `name` will display as obfuscated dots in the Cloud Manager. They are also encrypted on the host and do not log.
- A UDF containing `label="$label" header="Yes" default="Yes" required="Yes"` will display as a header in the Cloud Manager, but does not affect deployment.

## Testing

Due to limitations involved with the control/provisoner node configuring itself as a member of the cluster, the traditional Molecule testing framework will only work for testing certain roles, and not for testing the whoile deployment. To test the deployment, you will need to instead provision and test against Linodes on your account. Note that billing will occur for any Linode instances deployed.

To test your Marketplace Cluster on Linode infrastucutre, copy and paste `ss.sh` into a new Stackscript on your account. Then substitute the provided Stackscript ID into our example [API calls](https://github.com/linode-solutions/marketplace-partners-sample-app/blob/main/apps/linode-marketplace-wordpress/README.md#use-our-api). Logging output can be viewed in `/var/log/stackscript.logs.`
