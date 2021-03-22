Chuẩn bị
Deployment host chính là controller01

cat <<EOF >> /etc/hosts
172.16.14.121 controller01
172.16.14.122 controller02
172.16.14.123 controller03
172.16.14.124 compute01
172.16.14.125 compute02
EOF

apt install -y chrony
chronyc sources

timedatectl set-timezone Asia/Ho_Chi_Minh

Cập nhật rồi reboot.
sudo apt update && sudo apt -y upgrade && sudo systemctl reboot


Cài đặt trên deployment host - controller01
Install additional software packages if they were not installed during the operating system installation:

apt install -y build-essential git python3-dev sudo


Install the source and dependencies¶
Clone the latest stable release of the OpenStack-Ansible Git repository in the /opt/openstack-ansible directory:
git clone -b 22.1.0 https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible

Change to the /opt/openstack-ansible directory, and run the Ansible bootstrap script:
cd /opt/openstack-ansible
scripts/bootstrap-ansible.sh

ssh-keygen -t Ed25519
ssh-copy-id root@controller01
ssh-copy-id root@controller02
ssh-copy-id root@controller03
ssh-copy-id root@compute01
ssh-copy-id root@compute02


Initial environment configuration
Copy the contents of the /opt/openstack-ansible/etc/openstack_deploy directory to the /etc/openstack_deploy directory.
cp -R /opt/openstack-ansible/etc/openstack_deploy /etc/openstack_deploy

Change to the /etc/openstack_deploy directory.
cd /etc/openstack_deploy

Copy the openstack_user_config.yml.example file to /etc/openstack_deploy/openstack_user_config.yml.
cp openstack_user_config.yml.example openstack_user_config.yml

Review the file and make changes to the deployment of your OpenStack environment.
vi openstack_user_config.yml

Review the user_variables.yml file to configure global and role specific deployment options. The file contains some example variables and comments but you can get the full list of variables in each role’s specific documentation.
vi user_variables.yml 

Configuring service credentials
cd /opt/openstack-ansible
 ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
