---
- name: Creando Instancias Google Cloud.
  hosts: localhost
  gather_facts: no
  connection: local

  vars:
    machine_type_tower: "n1-standard-2"
    machine_type: "n1-highmem-2"
    image: "rhel74-gold-v3"
    service_account_email: "service_account"
    credentials_file: "json_file"
    project_id: "Project_Name"
    zone: "Zone_vm"
    external_projects: "External_Projects"
    disk_size: 100
    user: user_name
    user2: user2_name
    #
    
  tasks:
    - name: Creando instancia RHV-Manager, RHV-Hipervisor 1, RHV-Hipervisor 2.
      gce:
          instance_names: "manager{{ user }},hipervisora{{ user }},hipervisorb{{ user }},manager{{ user2 }},hipervisora{{ user2 }},hipervisorb{{ user2 }}"
          machine_type: "{{ machine_type }}"
          image: "{{ image }}"
          service_account_email: "{{ service_account_email }}"
          credentials_file: "{{ credentials_file }}"
          project_id: "{{ project_id }}"
          zone: "{{ zone }}"
          disk_size: "{{ disk_size }}"      
          tags:
            - http-server
            - https-server

      register: gce

    - name: Esperando Conexión con instancia
      wait_for: host={{ item.public_ip }} port=22 delay=10 timeout=60
      with_items: "{{ gce.instance_data }}"

    - name: Agregando host a grupo.
      add_host: hostname={{ item.public_ip }} groupname=new_instances
      with_items: "{{ gce.instance_data }}"

    - name: Creando instancia RH-Tower.
      gce:
         instance_names: "tower{{ user }}"
         machine_type: "{{ machine_type_tower }}"
         image: "{{ image }}"
         service_account_email: "{{ service_account_email }}"
         credentials_file: "{{ credentials_file }}"
         project_id: "{{ project_id }}"
         zone: "{{ zone }}"
         disk_size: "{{ disk_size }}"
         tags:
           - http-server
           - https-server
      register: gce

    - name: Esperando Conexión con instancia
      wait_for: host={{ item.public_ip }} port=22 delay=10 timeout=60
      with_items: "{{ gce.instance_data }}"

    - name: Agregando host a grupo.
      add_host: hostname={{ item.public_ip }} groupname=tower
      with_items: "{{ gce.instance_data }}"
      sleep: 30

- hosts: tower
  remote_user: root
  vars:
    - tower_path: /tmp/ansible-tower-last-release
    - tower_admin_pass: "R3dH@t2017!.-"
    - rhn_username: rhn_user
    - rhn_pass: "rhn_passwd"
    - hostname_full: towermp123456.themike.systems
  tasks:
    - name: Configurando nombre de Servidor Tower.
      hostname:
        name: "{{hostname_full }}"

    - name: Registrando Sistema en Portal de Red Hat.
      redhat_subscription:
          state: present
          username: " {{ rhn_username }} " 
          password: "{{ rhn_pass }}"
          consumer_name: "{{ hostname_full }}"
          autosubscribe: true
          force_register: yes

    - name: Habilitando canales necesarios.
      shell: |
        subscription-manager repos --disable=*
        subscription-manager repos --enable rhel-7-server-rpms --enable rhel-7-server-extras-rpms --enable rhel-7-server-optional-rpms
        
    - name: Actualizando Sistema Operativo
      yum:
        name: '*'
        state: latest
        
    - name: "Limpiando metadata de los repositorios."
      command: "yum clean all"

    - name: Reiniciando nodo
      shell: sleep 2 && shutdown -r now "Ansible reboot"
      async: 1
      poll: 0
      ignore_errors: true

    - name: Esperando conexión con servidor.
      local_action: wait_for
      args:
        host: "{{ inventory_hostname }}"
        port: 22
        state: started
        delay: 30
        timeout: 60

    - name: Descargando Ansible Tower.
      get_url:
        url: https://releases.ansible.com/ansible-tower/setup-bundle/ansible-tower-setup-bundle-latest.el7.tar.gz
        dest: /tmp

    - file:
        path: /tmp/paqueton
        state: directory

    - name: Extrayendo Ansible Tower.
      unarchive:
        src: /tmp/ansible-tower-setup-bundle-latest.el7.tar.gz 
        dest: /tmp/paqueton
        remote_src: yes
      register: tar_reg

    - name: Instalando Tower.
      shell: mv * /tmp/ansible-tower-last-release
      args:
        chdir: /tmp/paqueton

    - name: Configurando Password Admin.
      lineinfile: 
        dest: "{{ tower_path }}/inventory"
        regexp: admin_password
        line: "admin_password='{{ tower_admin_pass }}'"

    - name: Configurando Password Rabbitmq.
      lineinfile:
        dest: "{{ tower_path }}/inventory"
        regexp: rabbitmq_password
        line: "rabbitmq_password='{{ tower_admin_pass }}'"

    - name: Configurando Password PG.
      lineinfile:
        dest: "{{ tower_path }}/inventory"
        regexp: pg_password
        line: "pg_password='{{ tower_admin_pass }}'"

    - name: Instalando Tower.
#      shell: ./setup.sh
      command: ./setup.sh
      args:
        chdir: "{{ tower_path }}"
        executable: /bin/bash
