# Using the HPE 3PAR Ansible Module to manage Virtual Domains

This guide will walk through the steps to manage Virtual Domains within a 3PAR using a mixture of the SSMC and the Ansible module for 3PAR. This guide targets those who want to setup and manage multi-tenancy on a 3PAR.

## Assumptions
* HPE 3PAR configured and zoned correctly
* WSAPI enabled
* Super user access to 3PAR
* Ansible installed on workstation
* Knowledge of Ansible and playbooks

## Configuring Virtual Domains

Domains allow an administrator to create up to 1,024 domains, or spaces, within a system, where each domain is dedicated to a specific application. A subset of the system users has assigned rights over the domains. Domains can be useful in scenarios where a single system is used to manage data from several different independent applications.
For more information, refer to the **HPE 3PAR Storeserv Storage Concepts Guide**.

The first steps to setting up your 3PAR for multi-tenancy is to create a new virtual domain. Currently the configuration of Domains/Users can only done within the SSMC or via the 3PAR CLI. The example shown below will be done within the SSMC.

1. Login into the SSMC with Super user access.  

![SSMC Login](/demo/virtual_domains/img/ssmc_login.jpg)

2. In the mega menu, click **Show All > Domains**

![Domains](/demo/virtual_domains/img/3par_domains.jpg)

3. Click **Create domain**

![Create Domain](/demo/virtual_domains/img/3par_domains_create.jpg)

4. Enter the name of the domain. In this example, **bob_domain**. Then click **Add systems**. Specify the **3PAR** where the domain will be created. Once complete, click **Create**.

![Create Domain](/demo/virtual_domains/img/3par_domains_create_bob_65.jpg)

5. In the mega menu, click **Show All > Users**

![Users](/demo/virtual_domains/img/3par_users_menu.jpg)

6. Click **Create User**

![Create User](/demo/virtual_domains/img/create_user.jpg)

7. Specify the **NEW** user name and password.

![Create bob_user](/demo/virtual_domains/img/create_user_bob_65.jpg)

8. Click **Add Authorizations**, choose the domain created previously (**bob_domain** on **virt-3par** system). Choose the **edit** Role for the user. Click Add.

![Authorizations](/demo/virtual_domains/img/add_authorization.jpg)

#### Repeat these steps as necessary to configure additional Domains and Users within your 3PAR.

&nbsp;  

&nbsp;  


## Using Ansible to configure CPGs, Hosts, Volumes and more.

The following section will demonstrate the process to configure resources like CPGs, 3PAR hosts, etc and assign them to the newly created domains. Remember that the domain users will only be able to edit resources they have access to and will be unable to see resources from other domains unless authorized to do so.

Also everything else from this point will be able to be done via the HPE 3PAR Ansible Storage Module on a Linux (RHEL, CentOS, Ubuntu) system with Ansible (ver. 2.5 or later) installed.

#### Let's get started.

Clone the repo to get access to the virtual domain demo.

```
https://github.com/budhac/hpe3par_ansible_module
```

#### Generic Ansible housekeeping

1. Configure **ansible.cfg** to know about the 3PAR Ansible Storage Modules.

>If you have other modules already installed on this system, you can move the **Modules** folder from this repo to that directory.

```
vi /etc/ansible/ansible.cfg
```

2. Under the **[defaults]** section, Edit **library** to point to your Modules directory.

```
[defaults]

# some basic default values...

inventory      = /etc/ansible/hosts
library        = /root/workspace/hpe3par_ansible_module/Modules
```

#### Understanding the 3PAR Ansible playbooks

1. Navigate to the **hpe3par_ansible/demo/virtual_domains** folder. Here we will find two Ansible playbooks and the **properties** folder.
    * **virtual_domains_demo_3par_admin.yml**
    * **virtual_domains_demo_3par_user.yml**
    * **properties/storage_system_properties.yml** (This is configuration files containing the 3PAR IP address, Storage admin username and password for the 3PAR array)
    * **properties/storage_system_properties_bob.yml** (This is configuration files containing the 3PAR IP address, Storage admin username and password for the 3PAR array)

```
cd ~/workspace/hpe3par_ansible/demo/virtual_domains

[root@ansible-host virtual_domains]# ls -la
total 12
drwxr-xr-x. 4 root root  137 Sep 26 13:00 .
drwxr-xr-x. 3 root root   29 Sep 26 09:23 ..
drwxr-xr-x. 2 root root  254 Sep 26 13:00 img
drwxr-xr-x. 2 root root  116 Sep 26 09:23 properties
-rw-r--r--. 1 root root 2219 Sep 26 13:00 README.md
-rw-r--r--. 1 root root 2065 Sep 26 09:23 virtual_domains_demo_3par_admin.yml
-rw-r--r--. 1 root root 1847 Sep 26 09:23 virtual_domains_demo_3par_user.yml
```

2. Edit the **properties/storage_system_properties.yml** and configure the 3PAR IP address. Enter the **Storage Admin** username and password. Save the file.

```
vi **properties/storage_system_properties.yml**
```

```
storage_system_ip: "192.168.1.50"
storage_system_username: "3paradm"
storage_system_password: "3pardata"
```

3. Edit the **properties/storage_system_properties_bob.yml** and configure the 3PAR IP address. Enter the **bob_user** username and password. Save the file.

```
vi **properties/storage_system_properties_bob.yml**
```

```
storage_system_ip: "192.168.1.50"
storage_system_username: "bob_user"
storage_system_password: "Password"
```

4. Next, we need to encrypt those files so that they aren't stored plain text. Enter a unique password for each file.

```
ansible-vault encrypt properties/storage_system_properties.yml

ansible-vault encrypt properties/storage_system_properties_bob.yml
```

5. Check to verify they are encrypted. You should see something similar to below.

```
$ head -2 properties/storage_system_properties.yml
$ANSIBLE_VAULT;1.1;AES256
33636137356335
```

>**Note:** If you need to edit the encrypted file, you can run `ansible-vault edit file.yml`, enter the vault password and perform the edits. Alternatively, if you need to decrypt the file, run `ansible-vault decrypt file.yml` and enter the vault password.


Now let's get on to the fun stuff.

We will be working in the **virtual_domains_demo_3par_admin.yml** playbook. This playbook is ran by the Storage Admin to create **CPGs** and assign **Hosts** to the domain we created previously.

>**Note:** These are very simple examples to help you understand the capabilities of the Virtual Domains within the HPE 3PAR system. You can expand these to add multiple CPGs and multiple Hosts within the same playbook without ever having to log into the SSMC. This is the power automating the configuration of the HPE 3PAR Storage System using the Ansible Storage Modules.

When we open the file, we will see multiple sections. Again we are assuming that you are familiar with YAML and Ansible playbooks to understand the layout and structure.

```yaml
---
- name: Virtual Domains on 3PAR Ansible Demo playbook - Admin
  hosts: localhost
  become: no

  vars:
    cpg_name: 'bob_cpg_FC_r6'
    host_name: 'scom.virtware.co'
    domain: 'bob_domain'
    iscsi_names: 'iqn.1991-05.com.microsoft:scom.virtware.co'

  tasks:
    - name: Load Storage System Vars
      include_vars: 'properties/storage_system_properties.yml'

    - name: Create CPG "{{ cpg_name }}"
      hpe3par_cpg:
        # 3PAR CPG options found here: https://github.com/HewlettPackard/hpe3par_ansible_module/blob/master/Modules/hpe3par_cpg.py
        storage_system_ip: "{{ storage_system_ip }}"
        storage_system_username: "{{ storage_system_username }}"
        storage_system_password: "{{ storage_system_password }}"
        state: present
        domain: "{{ domain }}"
        cpg_name: "{{ cpg_name }}"
        growth_increment: 32.5
        growth_increment_unit: GiB
        growth_limit: 100
        growth_limit_unit: GiB
        growth_warning: 90
        growth_warning_unit: GiB
        raid_type: R6
        set_size: 6
        high_availability: MAG
        disk_type: FC

    - name: Create Host "{{ host_name }}"
      hpe3par_host:
        # 3PAR Host options found here: https://github.com/HewlettPackard/hpe3par_ansible_module/blob/master/Modules/hpe3par_host.py
        storage_system_ip: "{{ storage_system_ip }}"
        storage_system_username: "{{ storage_system_username }}"
        storage_system_password: "{{ storage_system_password }}"
        state: present
        host_name: "{{ host_name }}"
        host_domain: "{{ domain }}"
        host_persona: WINDOWS_SERVER

    - name: Add iSCSI paths to Host "{{ host_name }}"
      hpe3par_host:
        storage_system_ip: "{{ storage_system_ip }}"
        storage_system_username: "{{ storage_system_username }}"
        storage_system_password: "{{ storage_system_password }}"
        state: add_iscsi_path_to_host
        host_name: "{{ host_name }}"
        host_iscsi_names: "{{ iscsi_names }}"

```  

#### Variables section

There are several sections where you can specify variables allowing maximum flexibility when creating playbooks. They can be specified at the playbook level (Global), in external file (properties/storage_system_properties.yml), or at the task level.

In the **vars** section, you can modify the CPG/Host names to be added to the **bob_domain**.

**Note:** In order to assign the new CPGs and Hosts to the domain, you must specify a domain in the `domain: 'new_domain'` variable. This variable will then be used within each of the tasks (**domain:**, **host_domain:**), where required to assign the CPG or Host to the domain. If the domain is **not** specified, the CPG or Host will **not** be assigned to a domain and will not be accessible to the Domain user when they log into the array.

> Modify the **host_name** and **iscsi_names** to match a host and iSCSI iqns you want to add from your environment.

In the **tasks** section, for example in the **Create CPG** task, you can add/modify the variables (growth_limit, raid_type, etc) in the tasks as well move them into **vars** section if needed.

> In the case of Multi-Tenancy, creating growth limits, defining disk characteristics on CPGs in critical in order to enforce boundaries per tenant (so one tenant doesn't consume the entire storage array), as well as to ensure all tenants get the appropriate resources and performance per their needs.

Also you can specify an external variables file like the **storage_system_properties.yml**. This gave us the ability to encrypt the external file without affecting the main playbook.

#### Tasks sections

We have 4 main tasks in this example.

These tasks are taken from the main associated (CPG, Host, Volume, etc) playbooks found in the [https://github.com/HewlettPackard/hpe3par_ansible_module/tree/master/playbooks](https://github.com/HewlettPackard/hpe3par_ansible_module/tree/master/playbooks).

Please refer to the [Modules README](https://github.com/HewlettPackard/hpe3par_ansible_module/blob/master/Modules/readme.md) for detailed information on each Module including optional parameters.

1. Load Storage System Vars (load the encrypted storage system IP, username/password)
2. Create CPG (create CPGs per the provided specifications)
3. Create Host (create a basic 3PAR host)
4. Add iSCSI paths to Host (modify the host and add iSCSI IQNs or FC WWNs)

#### Running the Playbook

Now that we know what is going on within the admin playbook, we can run it so that it creates the CPGs, Host resources within the **bob_domain**.

We will run this playbook with the `ansible-playbook --ask-vault-pass` option in order to decrypt the **storage_system_properties.yml** file.

```yaml
$ ansible-playbook --ask-vault-pass virtual_domains_demo_3par_admin.yml
Vault password:

PLAY [Virtual Domains on 3PAR Ansible Demo playbook - Admin] **************************

TASK [Gathering Facts] ****************************************************************
ok: [localhost]

TASK [Load Storage System Vars] *******************************************************
ok: [localhost]

TASK [Create CPG "bob_cpg_FC_r6"] *****************************************************
ok: [localhost]

TASK [Create Host "scom.virtware.co"] *************************************************
ok: [localhost]

TASK [Add iSCSI paths to Host "scom.virtware.co"] ***************************************************************************************
ok: [localhost]

PLAY RECAP ****************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0

```

#### Success.
Now our Domain and Users have been configured, CPGs and Hosts have been created for the users to consume.
