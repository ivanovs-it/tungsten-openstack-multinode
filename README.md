# tungsten-openstack-multinode

Openstack (Train) cluster installation with [Tungsten Fabric](https://tungsten.io/) as a Neutron network backend (using [tf-ansible-deployer](http://github.com/tungstenfabric/tf-ansible-deployer) as deployment tool). Sample example with controllers combined with compute nodes.

Hosted OS is CentOS Linux release 7.9.2009 (Core)

##### Prerequisites
| Host  | Interface (eth0)| Interface (eth1) |
| ------| --------------- | ---------------- |
| Seed  | 10.0.0.10/24    | not required |
| Ctl01 | 10.0.0.21/24    | w/o IP |
| Ctl02 | 10.0.0.22/24    | w/o IP |
| Ctl03 | 10.0.0.23/24    | w/o IP |

##### On the Seed node
* Install required packages
```bash
sudo su
yum -y install epel-release
yum -y install git screen wget tcpdump
```
* Install pip for Python 2.7
```bash
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
chmod a+x get-pip.py
./get-pip.py
```
* Install ansible 2.7.18
```bash
pip install ansible==2.7.18
```
* Clone ansible-deployer reposotory
```bash
mkdir -p /opt/tf/tf-ansible-deployer
git clone http://github.com/tungstenfabric/tf-ansible-deployer /opt/tf/tf-ansible-deployer
cd /opt/tf/tf-ansible-deployer
```
* Generate keypair to run ansible locally
```bash
ssh-keygen -b 2048 -t rsa -f /root/.ssh/id_rsa -q -N ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
* Copy public key to all nodes for root user
* Create LVM Volume group for cinder volumes (for all nodes)
```bash
sudo yum install -y lvm2
sudo pvcreate /dev/vdb
sudo vgcreate cinder-volumes /dev/vdb
```
* Create deployment configuration /opt/tf/tf-ansible-deployer/config/instances.yaml
```
provider_config:
  bms:
    ssh_user: root
    ssh_public_key: /root/.ssh/id_rsa.pub
    ssh_private_key: /root/.ssh/id_rsa
    ntpserver: pool.ntp.org
instances:
  ctl01:
    provider: bms
    ip: 10.0.0.21
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      vrouter:
      openstack:
      openstack_compute:
  ctl02:
    provider: bms
    ip: 10.0.0.22
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      vrouter:
      openstack:
      openstack_compute:
  ctl03:
    provider: bms
    ip: 10.0.0.23
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      vrouter:
      openstack:
      openstack_compute:
global_configuration:
contrail_configuration:
  OPENSTACK_VERSION: train
  CONTRAIL_VERSION: latest
  UPGRADE_KERNEL: no
  VROUTER_GATEWAY: 192.168.10.254
  PHYSICAL_INTERFACE: eth1
kolla_config:
  kolla_globals:
    enable_haproxy: yes
    enable_ironic: no
    enable_swift: no
    enable_cinder: "yes"
    enable_cinder_backend_lvm: "yes"
    kolla_internal_vip_address: "10.0.0.100"
    network_interface: "eth1"
  customize:
    nova.conf: |
      [libvirt]
      virt_type=qemu
      cpu_mode=none
```
* Run playbooks
```bash
ansible-playbook -vvv -e orchestrator=openstack -i inventory/ playbooks/configure_instances.yml
ansible-playbook -vvv -i inventory/ playbooks/install_openstack.yml
ansible-playbook -vvv -e orchestrator=openstack -i inventory/ playbooks/install_contrail.yml
```
To operate with the cloud from Seed node
```bash
pip install --ignore-installed python-openstackclient

# To fix 'ImportError: No module named queue' issue (https://www.codetd.com/en/article/12518649)
sed -i 's/import queue/from multiprocessing import Queue as queue/g' /usr/lib/python2.7/site-packages/openstack/utils.py
sed -i 's/import queue/from multiprocessing import Queue as queue/g' /usr/lib/python2.7/site-packages/openstack/cloud/openstackcloud.py
```
* To check Tungsten Fabric status (on the Control node)
```bash
contrail-status
```
