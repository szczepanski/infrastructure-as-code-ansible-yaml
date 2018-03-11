```yml yaml
#run 1x ec2 ubuntu nano server (playbook 1) and then playbook 2  - install docker repo, run sample container - simple web server accessible via http://54.173.255.8:8000/ or other public IP
#ansible-playbook deploy-hellonode.yml  
#run 10 ec2  servers 
#ansible-playbook -e server_count=10 deploy-hellonode.yml 

#stop the server 
#ansible-playbook -e server_count=0 deploy-hellonode.yml 

# chalanges I had here: 
# don't change region to eu-west-2 as the image used here works for the us-east-1 refion only
# run it as a root make sure initial aws cogugurtion is done a a root - specify 
# python-bot is updated to ver 3



#on my ec2 instance serving as asnible server - Red Hat distro make sure following config is done:
#sudo pip install awscli
#rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
#yum -y install ansible
#yum install -y python python-dev python-pip
#sudo yum install python-boto [needed to run ansible playbooks ]






# 1st Part / 1st play - create the server - ec2 instance

- hosts: localhost #  tasks to be run on local machine
  name: Create infrastructure on AWS #for information only  
  connection: local # do not connect anywhere 
  gather_facts: False # when enabled ansible can extract variables and facts such as os version or interface details
  vars: # here server count var is hardcoded to 1
    server_count: 1

  tasks: #defined by name followed by module name (shell) and module arguments
    - name: Generate SSH key if it doesn't exist  # 1st task  
      shell: ssh-keygen -b 2048 -t rsa -f ./hellonode_key -q -N "" #generate ssh key pair
      args:
        creates: ./hellonode_key #create it as the specified file, if it already exists, skip the step 

    - name: Ensure key pair exists in EC2 # 2nd taks 
      ec2_key:  #module name
        name: hellonode_key #ec2_key module 'name 'argument
        key_material: "{{ item }}" #ec2_key module "key argument"
        region: "us-east-1"
      with_file: ./hellonode_key.pub #with_file key word - run this module for every file specified and save file contents in variable "item" - here it runs once - one file specified

    - name: Ensure security group exists #3rd task 
      ec2_group:
        name: hellonode_sg # if it doesn't exist the create one
        description: Security group for hellonode server
        rules: # implies ingress and egress routes 
          - proto: tcp #open ports 8000 (for hellonode server)and 22(ssh needed for ansible to config the server)
            from_port: 8000 
            to_port: 8000
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
        rules_egress:
          - proto: all
            from_port: -1
            to_port: -1
            cidr_ip: 0.0.0.0/0
        region: "us-east-1"

    - name: Ensure {{ server_count }} server(s) are running  #4th task 
      ec2: #create ec2 instance
        key_name: hellonode_key # define ssh key pair name
        group: hellonode_sg # define security group
        instance_type: t2.micro # define inst type
        image: "ami-da05a4a0" # id of ubuntu 1604 image
        wait: true # wait until the instance is created as I will be connecting to it in later tasks
        exact_count: "{{ server_count }}" # hardcoded previously to 1 - checks if any instance named hellonode exists, if no creates on else it will not. Count passed in as varibale as it can be then changed when run from command line. If set to 0 then this task will terminate/ kill any running hellonode instances
        count_tag:
          Name: Hellonode Server
        instance_tags:
          Name: Hellonode Server
        user_data: | # update OS, install python as this is required for ansible to run
             #!/bin/sh
             sudo apt-get -y update
             sudo apt-get -y --force-yes install python python-pip
        region: "us-east-1"
      register: ec2 # save output of this module into var ec2 as it will be needed in the next task

    - name: Add instance public IP to host group #5th task
      add_host: #add public IP of the just created instance into host group called ec2hosts
        hostname: "{{ item.public_ip }}"
        groups: ec2hosts
        ansible_ssh_private_key_file: ./hellonode_key #definning ssh key pair file argument - relative path - this tells ansible to use the ssh key file when ssh-ing into this host later
      when: item.public_ip != None # conditional - only run this task if True - here: if public ip is not none => it exists / it is defined
      with_items: "{{ ec2.instances }}" #takes a list as an argument and runs above task for every list's item and exposing current item in var item

# 2nd part / 2nd play configure the server

- hosts: ec2hosts # apply to host just saved in previous/above play in var ec2hosts 
  name: Configure Hellonode server # conf the name
  user: ubuntu #default username for the OS 
  become: true # become root
  gather_facts: False

  tasks: # 1st task 
    - name: Wait 300 seconds for target connection to become reachable/usable 
      wait_for_connection:
        timeout: 300

# below task is while loop that will block until the debian package manager lock file is released
# this is to make sure that apt-get command specified in user_data has finished as it needs to be run again in next tasks
    - name: Wait for provisioning script to finish # 2nd task: run shell script snippet
      become: yes 
      shell: while sudo fuser /var/lib/dpkg/lock >/dev/null 2>&1; do sleep 1; done;

    - name: Ensure repository key is installed #3rd task: install public key for the docker repository
      apt_key:
        id: "58118E89F3A912897C070ADBF76221572C52609D"
        keyserver: "hkp://p80.pool.sks-keyservers.net:80"
        state: present

    - name: Ensure Docker repository is available # 4th task: install docker repository
      apt_repository: repo='deb https://apt.dockerproject.org/repo ubuntu-xenial main' state=present

    - name: Ensure Docker and dependencies are installed # 5th task check if docker package is installed
      apt: name=docker-engine update_cache=yes # updat_cache argument tells ansible to run apt-get update before trying to install the package defined

    - name: Ensure Docker-py is installed # 6th task check for this package as ansible needs it to run docker containers
      pip: name=docker-py # if not found install it via pip

    - name: Ensure Docker is running #7th task call ansibble service module 
      service: name=docker state=restarted # restarted argument means that if service was running for any reason, then after this step it has been restarted,If it was nor running then this step wil start it.

    - name: Ensure the hellonode container is running # 8th task
      docker_container: # start docker container
        name: hellonode
        state: started
        image: "getintodevops/hellonode:latest" #point the image to docker hub registry 
        pull: true # try to pull new versionof the image even if it already downloaded one with the same tag
        ports: #map port 8000 to the container and 8000 to the host machine - so i can connect to it via web browser.  
          - 8000:8000
          
          
#################### TESTING #################################
root@ip-172-31-27-104 @35.178.1.46# ansible-playbook deploy-hellonode.yml
 [WARNING]: Could not match supplied host pattern, ignoring: all

 [WARNING]: provided hosts list is empty, only localhost is available


PLAY [Create infrastructure on AWS] *************************************************************************************************************

TASK [Generate SSH key if it doesn't exist] *****************************************************************************************************
ok: [localhost]

TASK [Ensure key pair exists in EC2] ************************************************************************************************************
ok: [localhost] => (item=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQClTjQbbqy/abpsnA7TtspMZ+m/4fM1XJWyTnHIV6el7WzkuPV24Hpz6w8zg9lwi4RBk59gaJplWeKZt68rnQMirWHZiXNx/a8EK4PNITkXajd4cevzZFi/9JLBiewyJHLvg9NAUr6MCYbw06BlIkeexFRdrNldkOs8InbP/3fSV1LO8P4egh4R+ZXNdqqmasCZOpkYsOYdO3LvnM9EMrd9O5YpFRiNzMF6PMxGfEhDe9Q+b0Ghm/41xOSM8GDWFFQUpoa/VG6Pn7PYrT2Qbz1CW96IacduUNwxEP6j7g7lUHh5WU/y8BsA9NPsUcLWZ88+F/Et5MpvE0GHkvwZdLgF root@ip-172-31-27-104.eu-west-2.compute.internal)

TASK [Ensure security group exists] *************************************************************************************************************
ok: [localhost]

TASK [Ensure 1 server(s) are running] ***********************************************************************************************************
changed: [localhost]

TASK [Add instance public IP to host group] *****************************************************************************************************
changed: [localhost] => (item={u'kernel': None, u'root_device_type': u'ebs', u'private_dns_name': u'ip-172-31-94-75.ec2.internal', u'public_ip': u'52.54.213.170', u'private_ip': u'172.31.94.75', u'id': u'i-03851a3db554fe67f', u'ebs_optimized': False, u'state': u'running', u'virtualization_type': u'hvm', u'root_device_name': u'/dev/sda1', u'ramdisk': None, u'block_device_mapping': {u'/dev/sda1': {u'status': u'attached', u'delete_on_termination': True, u'volume_id': u'vol-0251d005b382e7b4d'}}, u'key_name': u'hellonode_key', u'image_id': u'ami-da05a4a0', u'tenancy': u'default', u'groups': {u'sg-158f2162': u'hellonode_sg'}, u'public_dns_name': u'ec2-52-54-213-170.compute-1.amazonaws.com', u'state_code': 16, u'tags': {u'Name': u'Hellonode Server'}, u'placement': u'us-east-1c', u'ami_launch_index': u'0', u'dns_name': u'ec2-52-54-213-170.compute-1.amazonaws.com', u'region': u'us-east-1', u'launch_time': u'2018-02-02T09:10:26.000Z', u'instance_type': u't2.micro', u'architecture': u'x86_64', u'hypervisor': u'xen'})

PLAY [Configure Hellonode server] ***************************************************************************************************************

TASK [Wait 300 seconds for target connection to become reachable/usable] ************************************************************************
ok: [52.54.213.170]

TASK [Wait for provisioning script to finish] ***************************************************************************************************
changed: [52.54.213.170]

TASK [Ensure repository key is installed] *******************************************************************************************************
changed: [52.54.213.170]

TASK [Ensure Docker repository is available] ****************************************************************************************************
changed: [52.54.213.170]

TASK [Ensure Docker and dependencies are installed] *********************************************************************************************
changed: [52.54.213.170]

TASK [Ensure Docker-py is installed] ************************************************************************************************************
changed: [52.54.213.170]

TASK [Ensure Docker is running] *****************************************************************************************************************
changed: [52.54.213.170]

TASK [Ensure the hellonode container is running] ************************************************************************************************
changed: [52.54.213.170]

PLAY RECAP **************************************************************************************************************************************
52.54.213.170              : ok=8    changed=7    unreachable=0    failed=0   
localhost                  : ok=5    changed=2    unreachable=0    failed=0  
```
