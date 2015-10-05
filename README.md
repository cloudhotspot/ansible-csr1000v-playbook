# Sample Playbook for Automating Deployment of the Cisco CSR 1000V on VMWare Fusion

This is an example playbook that can be used to automate deployment of the Cisco Cloud Services Router (CSR) 1000V on VMWare Fusion, leveraging the `mixja.csr1000v` role I have published on Ansible Galaxy.

The `mixja.csr1000v` role manually creates an OVF environment (VMWare Fusion does not support OVF environment creation by itself) to allow the CSR 1000V to provision and configure itself upon deployment.  

This is controlled via an OVF environment file, which is deployed based upon the <a href="https://github.com/cloudhotspot/ansible-csr1000v-role/blob/master/templates/ovf-env.xml.j2" target="_blank">`templates/ovf-env.xml.j2` template</a>.  

The `mixja.csr1000v` packages this file into an ISO image that is then attached to the virtual machine, allowing the CSR 1000V guest to bootstrap its configuration based upon the OVF environment file.

See <a href="http://pseudo.co.de/ansible-cisco-csr1000v/" target="_blank">my blog post</a> for further details on this role and playbook.

This playbook has been tested on the following system:

- OS X 10.10.5/10.11 (Yosemite/El Capitan)
- VMWare Fusion 7.1.2
- Ansible 1.9.3
- Cisco CSR 1000v IOS XE 3.14(1)  (csr1000v-universalk9.03.14.01.S.155-1.S1-std)

## Prerequisites

- Mac OS X
- <a href="http://www.vmware.com/products/fusion" target="_blank">VMWare Fusion</a> 7.x or higher
- <a href="https://www.vmware.com/support/developer/ovf/" target="_blank">VMWare OVF Tools</a> 4.1 or higher (VMWare account may be required)
- <a href="http://www.ansible.com/" target="_blank">Ansible 1.9.x or higher (`brew install ansible`)
- <a href="https://software.cisco.com/download/release.html?mdfid=284364978&softwareid=282046477&release=3.14.1S&relind=AVAILABLE&rellifecycle=ED&reltype=latest" target="_blank">Cisco CSR 1000v OVA Image</a> (CCO login required)

And of course the `mixja.csr1000v` role:
```bash
$ ansible-galaxy install mixja.csr1000v
- downloading role 'csr1000v', owned by mixja
- downloading role from https://github.com/mixja/ansible-csr1000v-role/archive/v0.5.tar.gz
- extracting mixja.csr1000v to /usr/local/etc/ansible/roles/mixja.csr1000v
- mixja.csr1000v was installed successfully
```

This `mixja.csr1000v` role also relies on the Ansible Galaxy <a href="https://github.com/yaegashi/ansible-role-blockinfile" target="_blank">yaegashi.blockinfile module</a>.  You must install this module prior to using the role:

```bash
$ ansible-galaxy install yaegashi.blockinfile
- downloading role 'blockinfile', owned by yaegashi
- downloading role from https://github.com/yaegashi/ansible-role-blockinfile/archive/v0.5.tar.gz
- extracting yaegashi.blockinfile to /usr/local/etc/ansible/roles/yaegashi.blockinfile
- yaegashi.blockinfile was installed successfully
```

## Quick Start

First, clone this repository:

```bash
$ git clone https://github.com/cloudhotspot/ansible-csr1000v-playbook.git
$ cd ansible-csr1000v-playbook
``` 

Next, you must define the following variables for the `mixja.csr1000v` role:

- `csr_ova_source`: The path to the CSR 1000V OVA image
- `csr_vm_root`: The root path where you want the virtual machine deployed (e.g. `/Users/myname/Virtual Machines/`)

Futher configuration variables can optionally be configured as described in the <a href="#configuration-variables">Configuration Variables</a> section.  

To run the playbook for a new environment, use the `ansible-playbook site.yml` command.  You will be prompted for your password for sudo access:

```bash
$ ansible-playbook site.yml
SUDO password: *******

PLAY [Verify ovftool is installed] ********************************************

TASK: [Check for ovftool] *****************************************************
ok: [localhost]
...
...
TASK: [Wait for CSR1000V to provision] ****************************************
ok: [localhost -> 127.0.0.1]

TASK: [Remove DHCP reservation] ***********************************************
skipping: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=40   changed=24   unreachable=0    failed=0
```

This will deploy a virtual machine to the following location:

`{{ csr_vm_root }}/{{ csr_vm_name }}.vmwarevm/`

For example, if `csr_vm_root` is **/Users/alice/guests** and `csr_vm_name` is **csr01** (default) the virtual machine will be deployed to **/Users/alice/guests/csr01.vmwarevm**.

### Overwriting an Existing Virtual Machine

If an existing virtual machine exists at the target location, by default the playbook will fail with the following error:

```bash
TASK: [Fail if VM path exists] ************************************************
failed: [localhost] => {"failed": true}
msg: VM already exists.  Please set csr_vm_overwrite variable to any value to overwrite the existing VM
``` 

To overwrite a previous deployment, you can set the `csr_vm_overwrite` variable to any value, specifying this either in your playbook or by passing it as an extra variable via the command line (recommended):

`$ ansible-playbook site.yml --extra-vars csr_vm_overwrite=true`

## <a name="configuration-variables"></a>Configuration Variables

The following variables are available to configure:

```yaml
# Name of the Cisco CSR 1000V virtual machine that will be created
csr_vm_name: "csr01"

# Last Octet of IP address assigned to CSR 1000V management interface.  This value should be between 3 and 127.
csr_vm_mgmt_ip_octet: "120"

# Management Interface - 0 = Ethernet0/GigabitEthernet1, 1 = Ethernet1/GigabitEthernet2, 2 = Ethernet2/GigabitEthernet2
csr_vm_mgmt_interface: 2

# Keep the DHCP reservation used for provisioning
csr_vm_persist_dhcp_reservation: yes

# CSR 1000V configuration variables
csr_name: csr01
csr_admin_username: admin
csr_admin_password: Pass1234
csr_domain_name: cloudhotspot.co

# Set to 'True' or 'False'
csr_enable_scp: False

# Set to 'ax' or 'appx'
csr_license_level: appx
```

## Management Interface Configuration

The default intended use case is to configure GigabitEthernet2 as a management interface that is attached to the `vmnet8` interface on the host.  This playbook configures a DHCP reservation based upon `vmnet8` interface IP address and uses the `csr_vm_mgmt_ip_octet` value to configure the desired last octet of the management interface IP address.

By default this playbook also provisions the management IP address and `csr_vm_name` value into the local machines `/etc/hosts` file.

For example, if the `vmnet8` interface subnet is **192.168.100.0/24** (this will vary for each VMWare Fusion environment) and the `vm_mgmt_ip_octet` value is **120**, then the management IP address assigned to the CSR 1000V virtual machine will be **192.168.100.120**.  

If you only wish to temporarily use the dynamic management interface IP address (e.g. because you are overwriting the management interface with your own configuration using a `csr_config_file`), you should remove the DHCP reservation after provisioning by setting the `csr_vm_persist_dhcp_reservation` variable to **no**.  Setting this variable to **no** will remove the DHCP reservation and also will not provision the management IP address and VM name into the local `/etc/hosts` file.

## Bring your own CSR 1000V configuration

You can deploy your own configuration file to the deployed CSR 1000V virtual machine by configuring the `csr_config_file` variable:

`$ ansible-playbook site.yml --extra-vars csr_config_file=/path/to/your/config/file`

The configuration file should have all blank lines, spacing lines and comments removed.  This is because the configuration file is inserted line by line into the OVF environment file that configures the CSR 1000v during bootstrapping.

For example, a typical Cisco configuration file snippet might look like:

```
hostname csr01
!
boot-start-marker
boot-end-marker
!
!
!
aaa new-model
!
!
aaa group server radius TEST_SUBSCRIBER_AAA
 server-private 192.168.1.201 auth-port 1812 acct-port 1813 key pass1234
!
aaa authentication login default local
```
You should modify the configuration file to look like this:

```
hostname csr01
boot-start-marker
boot-end-marker
aaa new-model
aaa group server radius TEST_SUBSCRIBER_AAA
 server-private 192.168.1.201 auth-port 1812 acct-port 1813 key pass1234
aaa authentication login default local
```

The modified configuration file snippet above will result in OVF environment settings that look like this:

```xml
<Property oe:key="com.cisco.csr1000v.ios-config-0005.1" oe:value="hostname csr01"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0006.1" oe:value="boot-start-marker"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0007.1" oe:value="boot-end-marker"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0008.1" oe:value="aaa new-model"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0009.1" oe:value="aaa group server radius TEST_SUBSCRIBER_AAA"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0010.1" oe:value=" server-private 192.168.1.201 auth-port 1812 acct-port 1813 key pass1234"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0011.1" oe:value="aaa authentication login default local"/>
```

## Known Issues

- https://github.com/cloudhotspot/ansible-csr1000v-role/issues/2