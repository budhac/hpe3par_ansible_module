# Using the HPE 3PAR Ansible Module to manage Virtual Domains

This guide will walk through the steps to manage Virtual Domains within a 3PAR using a mixture of the SSMC and the Ansible module for 3PAR. This guide targets those who want to setup and manage multi-tenancy on a 3PAR.

## Assumptions
* HPE 3PAR configured and zoned correctly
* WSAPI enabled
* Super user access to 3PAR
* Ansible installed on workstation

## Configuring Virtual Domains

Domains allow an administrator to create up to 1,024 domains, or spaces, within a system, where each domain is dedicated to a specific application. A subset of the system users has assigned rights over the domains. Domains can be useful in scenarios where a single system is used to manage data from several different independent applications.
For more information, refer to the **HPE 3PAR Storeserv Storage Concepts Guide**.

The first step to setting up your 3PAR for multi-tenancy is to create a new virtual domain. Currently the configuration of Domains/Users can only done within the SSMC or via the 3PAR CLI. The example shown below will be done within the SSMC.

1. Login into the SSMC with Super user access.  

![SSMC Login](/demo/virtual_domains/img/ssmc_login.jpg)

2. In the mega menu, click Show All > Domains

![Domains](/demo/virtual_domains/img/3par_domains.jpg)

3. Click Create domain

![Create Domain](/demo/virtual_domains/img/3par_domains_create.jpg)

4. Enter the name of the domain. In this example, **bob_domain**. Then click **Add systems**, specify the 3PAR where the domain will be created. Once complete, click **Create**.

![Create Domain](/demo/virtual_domains/img/3par_domains_create_bob.jpg)

5. In the mega menu, click **Show All > Users**

![Users](/demo/virtual_domains/img/3par_users.jpg)

6. Click **Create User**

![Create User](/demo/virtual_domains/img/create_user.jpg)

7. Specify the **NEW** user name and password.

![Create bob_user](/demo/virtual_domains/img/create_user_bob.jpg)

8. Click **Add Authorizations**, choose the domain created previously (**bob_domain** on **virt-3par** system). Choose the **edit** Role for the user. Click Add.

![Authorizations](/demo/virtual_domains/img/add_authorization.jpg)
