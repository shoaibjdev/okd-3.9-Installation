SSH to Master
---
cd $OKD_HOME
ssh -i okd_keypair.pem" centos@master-node

[ALL NODES] - Install Packages
---
sudo su -

yum install -y wget git zile nano net-tools docker-1.13.1 bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct openssl-devel httpd-tools NetworkManager python-cryptography python2-pip python-devel python-passlib java-1.8.0-openjdk-headless "@Development Tools"

yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm  
sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo  
  
systemctl start NetworkManager  
systemctl enable NetworkManager  

Prepare for Installation
---
yum -y --enablerepo=epel install ansible pyOpenSSL

cd /home/centos  
git clone https://github.com/openshift/openshift-ansible.git   
cd openshift-ansible && git fetch && git checkout release-3.9 && cd ..  
  
[ON ALL NODES]  
sudo vi /etc/hosts   
  
<MASTER-IP> master console console.apizone.io   
<NODE-1-IP> node01 node01.apizone.io  
<NODE-2-IP> node02 node02.apizone.io  
  
systemctl stop docker  
systemctl restart docker  
systemctl enable docker  
  

[ON MASTER]  
ssh-keygen  
Copy id_rsa.pub from Master to All Nodes including Master and add in `authorized_keys` entry  

  
##### Update Inventory  
cd /home/centos  
vi inventory.ini  


	[OSEv3:children]
	masters
	nodes
	etcd

	[masters]
	master-1 openshift_ip=master-1 openshift_schedulable=true
	[etcd]
	master-1 openshift_ip=master-1
	[nodes]
	master-1 openshift_ip=master-1  openshift_schedulable=true openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
	node-1 openshift_ip=node-1 openshift_schedulable=true  openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
	node-2 openshift_ip=node-2 openshift_schedulable=true  openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

	[OSEv3:vars]
	debug_level=4
	ansible_ssh_user=root
	enable_excluders=False
	enable_docker_excluder=False
	openshift_enable_service_catalog=False
	ansible_service_broker_install=False

	containerized=True
	os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
	openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

	openshift_node_kubelet_args={'pods-per-core': ['10']}

	deployment_type=origin
	openshift_deployment_type=origin

	openshift_release=v3.9.0
	openshift_pkg_version=-3.9.0
	openshift_image_tag=v3.9.0
	openshift_service_catalog_image_version=v3.9.0
	template_service_broker_image_version=v3.9.0

	osm_use_cockpit=true

	openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

	openshift_public_hostname=console.apizone.io
	openshift_master_default_subdomain=apps.apizone.io



Create Admin User
---
htpasswd -c /etc/origin/master/htpasswd admin  
<Enter Password>  

Execute Ansible Script
---
ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml  
ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml  


