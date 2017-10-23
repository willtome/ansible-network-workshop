Ansible Network Workshop
=========

This is a set of playbooks used for provisioning workshops in clouds.  It takes a template and creates the architecture specified for each student.

Requirements
------------

These playbooks require that you have your cloud environment setup correctly for authentication.  They also require the cloud roles linked in as sub modules.  In
order to pull down all of the required submodules, run:

```
git clone https://github.com/network-automation/ansible-network-workshop.git --recursive
```

Example
-------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```
ansible-playbook -e @net-ws.yml build-workshop.yml
```

To configure the control node (install Ansible, setup Ansible Inventory, etc)
```
ansible-playbook configure-hosts.yml -i net-ws.hosts
```
where `net-ws` is your workshop_name.  The -i specifies the inventory.

To validate a workshop, run the approrieate playbook (e.g. `validate-workshop1.yml`):
```
ansible-playbook -i smc-ws.hosts -e @smc-ws.yml --private-key=smc-ws_key -u ec2-user validate-workshop1.yml
```

Example vars file
-----------------
```yaml
ansible_python_interpreter: python
workshop_name: "net-ws"
workshop_template: 'network-automation-template1.yml'
workshop_dns_zone: "rhdemo.io"
workshop_provider1: "aws"
workshop_region1: "us-east-2"
workshop_provider2: "aws"
workshop_region2: "us-east-2"
num_students: 1
```

Example template
----------------
```yaml
ansible_ssh_private_key_file: "./{{ workshop_name }}_key"
cloud_user: ec2-user
student_name: "student{{ student_number }}"
inside_octet: "{{ student_number | int * 2 }}"
outside_octet: "{{ inside_octet | int - 1 }}"
acl_dict:
  host-acl:
    - { src_ip: 0.0.0.0/0, dst_ports: 22, proto: tcp }
    - { src_ip: 0.0.0.0/0, proto: icmp }
  rtr-acl:
    - { src_ip: 0.0.0.0/0, proto: all }
ssh_keys: "{ '{{ workshop_name }}': '{{ lookup('file', './{{ workshop_name }}_key.pub') }}' }"
vpc_list:
  - name: "{{ workshop_name }}-vpc1"
    provider: aws
    region: us-east-2
    project: "{{ workshop_name }}"
    cidr: 172.17.0.0/16
    acl_list:
      - host-acl
      - rtr-acl
    networks:
      - { name: "{{ student_name }}-net1", cidr: "172.17.{{ outside_octet }}.0/24", az: us-east-2a }
    instances:
      - { name: "{{ student_name }}-control", size: micro, image: rhel7, acl: host-acl, subnet: "{{ student_name }}-net1", public_ip: true, key_name: "{{ workshop_name }}", tags: {Owner: student, group: control} }
      - { name: "{{ student_name }}-rtr1", size: medium, image: csr-byol, acl: rtr-acl, subnet: "{{ student_name }}-net1", public_ip: true, key_name: "{{ workshop_name }}", tags: {Owner: student, network_os: ios, group: routers}, user_data: 'ios-config-0001=ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 dhcp' }
    routes:
      - { subnet: "{{ student_name }}-net1", cidr: "172.18.{{ inside_octet }}.0/24", instance: "{{ student_name }}-rtr1" }
  - name: "{{ workshop_name }}-vpc2"
    provider: aws
    region: us-east-2
    project: "{{ workshop_name }}"
    cidr: 172.18.0.0/16
    acl_list:
      - host-acl
      - rtr-acl
    networks:
      - { name: "{{ student_name }}-net2-outside", cidr: "172.18.{{ outside_octet }}.0/24", az: us-east-2b }
      - { name: "{{ student_name }}-net2-inside", cidr: "172.18.{{ inside_octet }}.0/24", az: us-east-2b, vnf_instance: "{{ student_name }}-rtr2", inside_ip: "172.18.{{ inside_octet }}.254" }
    instances:
      - { name: "{{ student_name }}-host1", size: micro, image: rhel7, acl: host-acl, subnet: "{{ student_name }}-net2-inside", public_ip: false, key_name: "{{ workshop_name }}", tags: {Owner: student, group: hosts } }
      - { name: "{{ student_name }}-rtr2", size: medium, image: csr-byol, acl: rtr-acl, subnet: "{{ student_name }}-net2-outside", public_ip: true, key_name: "{{ workshop_name }}", tags: {Owner: student, network_os: ios, group: routers}, user_data: 'ios-config-0001=ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 dhcp' }
```

This template yeilds the following architecture:

![Image of workshop](https://github.com/ismc/ansible-networking-workshop/blob/master/images/network-automation-template1.png)

License
-------

GPL-3

---
![Ansible Red Hat Engine](ansible-engine-small.png)

In addition to open source Ansible, there is Red Hat® Ansible® Engine which includes support and an SLA for the nxos_facts module shown above.

Red Hat® Ansible® Engine is a fully supported product built on the simple, powerful and agentless foundation capabilities derived from the Ansible project.  Please visit [ansible.com](https://www.ansible.com/ansible-engine) for more information.
