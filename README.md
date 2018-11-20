# postgres-datacollector-ansible

## Ansible Playbook to Collect Postgres Tables as CSV Aggregates

This playbook is desgined to capture database tables as CSV from a running Postgres DB docker image. It fetches the queries as a CSVs and then aggregates them all together and uploads them to a running webserver. 

Setup a simple webserver and configure the SSH keys beforehand. 

![demo](https://github.com/mascij/postgres-datacollector-ansible/blob/master/ansible-postgres.gif?raw=true)


```python

---
- hosts: localhost
  vars:
    diag_date: "{{ ansible_date_time.iso8601_basic }}"
    diag_path: "/tmp/HCP_Diag_{{ ansible_hostname }}-{{ diag_date }}"
    app_config_file: "appconfig-{{ diag_date }}.csv"
    alerts_file: "alerts-{{ diag_date }}.csv"
    users_file: "users-{{ diag_date }}.csv"
    vms_file: "vms-{{ diag_date }}.csv"
    blueprint_file: "blueprint-{{ diag_date}}.csv"
    docker_clusters_file: "data_center-{{ diag_date }}.csv"
    syslog_file: "syslog-{{ diag_date }}.log"
    blank_line: "echo ' ' >> {{ diag_path}}/{{ hcp_diag_file }};"
    hcp_diag_file: "HCP_Diag_{{ ansible_hostname }}-{{ diag_date }}.csv"
    postgres_ip: "172.17.0.4"

  tasks:

    - name: "Create Directory"
      file:
        path: "{{ diag_path }}"
        state: directory

    - name: "Grab App Config"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from app_config) to '{{ diag_path }}/{{ app_config_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab Alerts"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from inbox_message limit 100) to '{{ diag_path }}/{{ alerts_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab Users"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from users) to '{{ diag_path }}/{{ users_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab VMs"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from docker_server) to '{{ diag_path }}/{{ vms_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab Blueprints"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from blueprint) to '{{ diag_path }}/{{ blueprint_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab Docker Clusters"
      command: psql -h {{ postgres_ip }} -p 5432 -U dchq dchq -c "\copy (select * from data_center) to '{{ diag_path }}/{{ docker_clusters_file }}' DELIMITER ',' CSV HEADER"

    - name: "Grab Syslog"
      shell: tail -n 100 /var/log/syslog > {{ diag_path }}/{{ syslog_file }}

    - name: "Create Header"
      shell: echo "HCP_Diag_{{ diag_date }}" > {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Sys Settings"
      shell: echo "System Settings" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ app_config_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Alerts"
      shell: echo "Alerts" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ alerts_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join VMs"
      shell: echo "VMs" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ vms_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Users"
      shell: echo "Users" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ users_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Blueprints"
      shell: echo "Blueprints" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ blueprint_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Docker Clusters"
      shell: echo "Docker Clusters" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ docker_clusters_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Join Syslog"
      shell: echo "Syslog" >> {{ diag_path}}/{{ hcp_diag_file }}; cat {{ diag_path }}/{{ syslog_file }} >> {{ diag_path}}/{{ hcp_diag_file }}; {{ blank_line }}

    - name: "Upload File"
      shell: scp {{ diag_path }}/{{ hcp_diag_file }} hf@10.0.8.17:/home/hf/webserver

```
