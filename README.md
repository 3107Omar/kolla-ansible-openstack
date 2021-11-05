# kolla-ansible-openstack all-in-one
Deploy Opentstack Platform using Kolla-Ansible all-in-one 1 host ( Services on Containers)
done on VM with specs (CPU:2 RAM:8G Drive:30G OS: Centos 8 Stream minimal Networking: 2 NIC configured as bridge on VirtualBox)
##install Python build dependencies
sudo dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux
##Install pip
sudo dnf install python3-pip
##Ensure the latest version of pip is installed:
sudo pip3 install -U pip
##Install Ansible. Kolla Ansible requires at least Ansible 2.9 and supports up to 2.10.
sudo dnf install ansible
or
sudo pip3 install 'ansible=2.10'
##Install Kolla-ansible for deployment
##Install kolla-ansible and its dependencies using pip
sudo pip3 install kolla-ansible
##Create the /etc/kolla directory.
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
##Copy globals.yml and passwords.yml to /etc/kolla directory.
## global is a file that includes all needed services for openstack like nova and neutron
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
##Copy all-in-one and multinode inventory files to the current directory.
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
##Configure Ansible
##add the following options to the Ansible configuration file /etc/ansible/ansible.cfg
##if ansible directory and ansible.cfg are not found creat them
[defaults]
host_key_checking=False
pipelining=True
forks=100
##Kolla passwords
##Passwords used in our deployment are stored in /etc/kolla/passwords.yml file. All passwords are blank in this file and have to be filled ##either manually or by running random password generator
kolla-genpwd
##Kolla globals.yml
##globals.yml is the main configuration file for Kolla Ansible. There are a few options that are required to deploy Kolla Ansible:
##Image options
##User has to specify images that are going to be used for our deployment. In this guide DockerHub provided pre-built images are going to be ##used
##Kolla provides choice of several Linux distributions in containers:
##CentOS Stream (centos)
##Ubuntu (ubuntu)
##Debian (debian)
##RHEL (rhel, deprecated)
##For newcomers, we recommend to use CentOS Stream 8 or Ubuntu 20.04.
kolla_base_distro: "centos"
kolla_install_type: "source"
##Networking
##Kolla Ansible requires a few networking options to be set. We need to set network interfaces used by OpenStack.
##First interface to set is “network_interface”. This is the default interface for multiple management-type networks.
network_interface: "enp0s3"
##Second interface required is dedicated for Neutron external (or public) networks, can be vlan or flat, depends on how the networks are ##created. This interface should be active without IP address. If not, instances won’t be able to access to the external networks.
neutron_external_interface: "enp0s8"
##Next we need to provide floating IP for management traffic. This IP will be managed by keepalived to provide high availability, and should ##be set to be not used address in management network that is connected to our network_interface.
kolla_internal_vip_address: "10.0.0.240" ## note this address should be in the same network of your router or host
##Enable additional services
##By default Kolla Ansible provides a bare compute kit, however it does provide support for a vast selection of additional services. To ##enable them, set enable_* to “yes”. For example, to enable Block Storage service:
##Kolla now supports many OpenStack services, there is a list of available services. For more information about service configuration, Please ##refer to the Services Reference Guide.
enable_openstack_core: "yes"
enable_glance: "{{ enable_openstack_core | bool }}"
enable_hacluster: "no"
enable_haproxy: "yes"
enable_keepalived: "{{ enable_haproxy | bool }}"
enable_keystone: "{{ enable_openstack_core | bool }}"
enable_mariadb: "yes"
enable_memcached: "yes"
enable_neutron: "{{ enable_openstack_core | bool }}"
enable_nova: "{{ enable_openstack_core | bool }}"
enable_rabbitmq: "{{ 'yes' if om_rpc_transport == 'rabbit' or om_notify_transport == 'rabbit' else 'no' }}"
#enable_outward_rabbitmq: "{{ enable_murano | bool }}"

##Deployment
##Kolla Ansible provides a playbook that will install all required services in the correct versions.
##The following assumes the use of the multinode inventory. If using a different inventory, such as all-in-one, replace the -i argument ##accordingly.
##For deployment or evaluation, run:
1-Bootstrap servers with kolla deploy dependencies:
kolla-ansible -i ./all-in-one bootstrap-servers
2- Do pre-deployment checks for hosts
kolla-ansible -i ./all-in-one prechecks
3- Finally proceed to actual OpenStack deployment:
kolla-ansible -i ./all-in-one deploy
##When this playbook finishes, OpenStack should be up, running and functional!
##Using OpenStack
1- Install the OpenStack CLI client:
pip install python3-openstackclient
or
pip install python-openstackclient
2- OpenStack requires an openrc file where credentials for admin user are set. To generate this file:
kolla-ansible post-deploy
. /etc/kolla/admin-openrc.sh
## to access horizon after deployment we need to access horizon container then change ServerName to localhost
vi /etc/httpd/conf/httpd.conf
ServerName localhost
exit
##then exit the container and restart it
docker container restart <container ID of horizon>
##to access horizon get ip address with assigned on kolla_internal_vip_address from global.yml file
## to get credentials run
  cat /etc/kolla/admin-openrc.sh
  and use admin as username and get the value of export OS_PASSWORD= and use it on password field
