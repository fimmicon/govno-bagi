// Jenkinsfile (Scripted Pipeline)
node {
    stage('Prepare artifact') {
        sh '''
            # Remove ssh public keys (they can be old and not actual)
            ssh-keygen -f "/home/ubuntu/.ssh/known_hosts" -R "$IP_ADDRESS"
            # Gather actual SSH public keys from server
            ssh-keyscan -H $IP_ADDRESS >> /home/ubuntu/.ssh/known_hosts
        '''

        sh '''
            # Build inventory file for ansible
            cat <<EOF >inventory
node1 ansible_host=$IP_ADDRESS ansible_user=$LINUX_USER ansible_password=$LINUX_PASS
EOF
           '''

        sh '''
            # Build hosts file for artifact installation
            cat <<EOF >hosts
[local]
controller ansible_host=$IP_ADDRESS ansible_user=iadmin ansible_become=true ansible_connection=local


[k8s_masters:children]
local


[k8s_masters:vars]


[k8s_workers]
#k8s-2 ansible_host=10.0.1.19 new_hostname=k8s-2
#k8s-3 ansible_host=10.0.1.234 new_hostname=k8s-3



[k8s_workers:vars]
ansible_ssh_user=$LINUX_USER
ansible_become=true
ansible_connection=ssh
ansible_ssh_pass=$LINUX_PASS
ansible_sudo_pass=$LINUX_PASS

[all:children]
local
k8s_workers


[all:vars]
virtual_ipaddress=$VIRTUAL_IPADDRESS
metallb_ipaddress=$METALLLB_IPADDRESS
metallb_range=$METALLLB_IPADDRESS
dns_name=demo.centerity
virtual_router_id=$VIRTUAL_ROUTER_ID
docker_repository_name=localhost

storage_class=$STORAGE_CLASS
local_path_provisioner_directory=$LOCAL_PATH_PROVISIONER_PATH
EOF
        '''
        sh 'cat hosts'
        sh '''
            # Download artifact on $IP_ADDRESS machine
            ARTIFACT_NAME=$(echo $NEXUS_URL |  awk -F'/' '{print $NF}')
            cat <<EOF >playbook_on_prem_isntallation.yaml
- name: Download artifact from nexus and unpack
  gather_facts: yes
  hosts: node1
  become: yes
  tasks:
    - name: Set authorized key taken from file
      ansible.posix.authorized_key:
        user: $LINUX_USER
        state: present
        key: "{{ lookup('file', '/home/ubuntu/.ssh/id_rsa.pub') }}"
    - name: Download artifact
      ansible.builtin.shell: "curl -u $NEXUS_USER:$NEXUS_PASS -X GET $NEXUS_URL --output /root/$ARTIFACT_NAME"
    - name: Extract artifact into /root/
      ansible.builtin.unarchive:
        src: /root/$ARTIFACT_NAME
        dest: /root/
        remote_src: yes
    - name: Copy hosts file to artifact folder
      ansible.builtin.copy:
        src: hosts
        dest: /root/devops/hosts
        owner: '1000'
        group: '1000'
        mode: '0644'
EOF

        ansible-playbook -i inventory playbook_on_prem_isntallation.yaml

        cat <<EOF >dirty_hack.sh
# Dirty way to prepare openebs block device (sometimes installation of openebs fails without it)
NOT_USED_BLOCK_DEVICES=\\$(lsblk -p | sed -E '/^\\/dev\\/[a-z]/{x;//s/ .*//p;x;};/^[a-z]|\\/|]/h;$!d;x;  /^\\/dev\\/[a-z]/s/ .*//' | grep 'sd[a-z]')
for i in \\$NOT_USED_BLOCK_DEVICES ; do wipefs -fa \\$i ; dd if=/dev/zero of=\\$i bs=512M count=10; done
EOF
        chmod +x dirty_hack.sh
        rsync -av dirty_hack.sh $LINUX_USER@$IP_ADDRESS:/root/
        ssh $LINUX_USER@$IP_ADDRESS /root/dirty_hack.sh
    '''
    }

    stage('init-install.sh') {
        sh '''
            ssh $LINUX_USER@$IP_ADDRESS 'cd /root/devops/ ; ./init-install.sh'
        '''
    }

    stage('k8s-install.sh') {
        sh '''
            ssh $LINUX_USER@$IP_ADDRESS "cd /root/devops/ ; ./k8s-install.sh"
        '''
    }

    stage('skipper.sh') {
        sh '''
            ssh $LINUX_USER@$IP_ADDRESS 'cd /root/devops/ ; ./skipper.sh -i docker-playbook/files/app-infra-images.tar.gz -a docker-playbook/files/app-images.tar.gz'
        '''
    }
}
