# ansible
---
  - name: Provision an EC2 Instance
    hosts: localhost
    connection: local
    gather_facts: no
    vars_files:
      - vpc_info

    vars:
      instance_type: t2.micro
      ami: ami-xxxxxxxx
      region: us-west-1
      aws_keypair: xxxx
      vpc_id: "{{ private_subnet }}"
      count: 1 #3
      node: primary
      HOST_COUNT: "{{ groups['primary_mongo'] | length }}" # counting number of IP in the group

    tasks:
      - debug:
          msg: "{{ HOST_COUNT }}" # printing previously counted number

      - name: Launch the new EC2 Instance
        ec2:
          group: "sec_gr_mongo"
          instance_type: "{{ instance_type }}"
          image: "{{ ami }}"
          wait: true
          region: "{{ region }}"
          keypair: "{{ aws_keypair }}"
          vpc_subnet_id: "{{ vpc_id }}"
          count: "{{ count }}"
          instance_tags:
            Name: mongodb_node
        register: ec2

      - name: Checking for count number
        set_fact:
          node: secondary
        when: HOST_COUNT | int >= 1 # if HOST_COUNT=0 then node="primary"; else set node="secondary"

      - name: Add the newly created EC2 instance(s) to the local host group (located inside the directory)
        lineinfile:
          dest: ./hosts
          regexp: '^[{{ node }}_mongo]'
          insertafter: '^\[{{ node }}_mongo\]'
          line: "{{ item.private_ip }}"
        with_items: "{{ ec2.instances }}"
