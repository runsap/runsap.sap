Documentation: System Copy
==========================

Introduction
------------
This documentation is part of the system copy procedure that can be used to refresh a SAP ABAP system and reuse certain tables.

The refresh procedure has the following steps:

1. pre processing
2. backup sap hana tenant database
3. stop sap system
4. restore sap hana backup
5. post processing 

Using the workflow you can run the steps individual or all steps sequential using the **full** option.

Running the pre and post configuration parts you need to provide a `refresh_tables` and `truncate_tables` variable.

In the example below a `sap_table_groups` variable holds all defined tables that are grouped into groups. From this variable the tables are then collected.

Implementation
--------------
In your `ansible/requirements.yml` include this repo:

```yaml
  - name: https://github.com/runsap/runsap.core.git
    version: vx.x.x.
    type: git
```

Add the playbook and the workflow to your github repository.

Example playbook and workflow
-----------------------------

Here you see an example playbook that is used in a github workflow.

**Playbook**

```yaml
---

- name: Construct the inventory
  import_playbook: construct-inventory-aws-play.yml

- hosts: ec2hosts
  gather_facts: False
  become: true
  vars_files:
    - vault.yml
  tasks:
  - block:

    - name: Set fact with combined tables (get from host_data later on)
      set_fact:
        refresh_tables: "{{ refresh_tables | default([]) + (sap_table_groups[item] | map(attribute='name') | list) }}"
      with_items: "{{ host_data.sap.system_refresh.refresh_table_groups }}"
      when: >
        (cli_refresh_action == "preprocessing") or
        (cli_refresh_action == "postprocessing") or
        (cli_refresh_action == "full")

    - name: Set fact with truncate table
      set_fact:
        truncate_tables: "{{ truncate_tables | default([]) + (sap_table_groups[item] | map(attribute='name') | list) }}"
      with_items: "{{ host_data.sap.system_refresh.truncate_table_groups }}"
      when: >
        (cli_refresh_action == "preprocessing") or
        (cli_refresh_action == "postprocessing") or
        (cli_refresh_action == "full")

    #
    # Stoppen SAP target instance
    #
    - include_role:
        name: runsap.sap.control_instance
      vars:
        control_action: StopSystem
        control_sid: "{{ cli_refresh_sap_env }}"
        control_instance_nr:  10 # "{{ instance_nr }}"
        control_waittimeout: 600    # - name: Stop SAP Instance 
      when: >
        (cli_refresh_action == 'stop_system') or
        (cli_refresh_action == "full")
    #
    # Backup
    #    
    - include_role: 
        name: runsap.hana.hdb_backup
      vars: 
        secure_connection: "{{ host_data.ec2.tags.Sap_secure_db_connection  }}"
        backup_path: "/backup/{{ hdb_db | upper }}/{{ cli_backup_name | default(hdb_db ~ '_' ~ lookup('pipe', 'date +\"%Y-%m-%d_%H%M%S\"')) }}"
        hdb_db: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
        hdb_sid: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
        backup_type: file # backint # or file
      when:
        - "'hdb' in host_data.ec2.tags.Sap_type"
        - (cli_refresh_action == 'backup') or (cli_refresh_action == "full")

    #
    # Pre-processing
    #
    - name: Perform pre processing activities
      include_role: 
        name: runsap.sap.system_copy
      vars:
        source_sap_sid: "{{ cli_refresh_sap_env }}"
        target_sap_sid: "{{ cli_refresh_sap_env }}"
        scripts_dir: "/software/Sap_refresh/{{ target_sap_sid }}"
        system_copy_action: preprocessing
      when: >
        (cli_refresh_action == 'preprocessing') or
        (cli_refresh_action == "full")       
    #
    # Restore
    #
    - name: Restore SAP HANA database
      include_role: 
        name: runsap.hana.hdb_restore
      vars: 
        secure_connection: "{{ host_data.ec2.tags.Sap_secure_db_connection | default('False') }}"
        backup_path: "{{ cli_refresh_backup_location }}" # | default("/backup/" ~ hdb_db | upper) }}"
        hdb_db: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
        hdb_sid: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
      when:
        - "'hdb' in host_data.ec2.tags.Sap_type"
        - (cli_refresh_action == 'restore') or (cli_refresh_action == "full")
    #
    # Post-processing
    #
    - include_role: 
        name: runsap.sap.system_copy
      vars:
        scripts_dir: "/software/Sap_refresh/{{ target_sap_sid }}"
        source_sap_sid: "{{ cli_refresh_sap_env }}"
        target_sap_sid: "{{ cli_refresh_sap_env }}"
        target_database: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
        hdbsql_key: DEFAULT
        client: '000'
        #truncate_tables: "{{ truncate_tables }}" 
        secure_connection: "{{ host_data.ec2.tags.Sap_secure_db_connection }}"
        #refresh_tables: "{{ refresh_tables }}"
        system_copy_action: postprocessing

      when: >
        (cli_refresh_action == 'postprocessing') or
        (cli_refresh_action == "full")
    #
    # Starten SAP
    #
    - include_role:
        name: runsap.sap.control_instance
      vars:
        control_action: StartSystem
        control_sid: "{{ host_data.ec2.tags.Sap_abap_sid }}"
        control_instance_nr:  10 # "{{ instance_nr }}"
        control_waittimeout: 600       
      when: >
        (cli_refresh_action == 'start_system') or
        (cli_refresh_action == "full")

    #
    # dit moet je gaan doen dmv cli_sap_env filter
    #
    when: host_data.ec2.tags.Sap_env == cli_refresh_sap_env
```

**Workflow**

```yaml
name: Refresh Workflows 
run-name: Action ${{ github.event.inputs.cli_refresh_action }} is running 🚀

on:
  workflow_dispatch:
    inputs:
      cli_refresh_action:
        description: 'Choose a Playbook to run'
        required: true
        type: choice
        options:
        - full
        - stop_system
        - backup
        - preprocessing
        - restore
        - start_system
        - postprocessing
      cli_refresh_backup_location:
        description: 'Specify the Backup location for restore'
        required: false
        default: '/backup/H4F/h4f_2024-09-10_130531/'
        type: string
      cli_refresh_sap_env:
        description: 'Specify the SAP environment'
        required: false
        default: 's4f'
        type: string
jobs:

  build:
    runs-on: runsap
    name: 'Execute playbook'
  
    steps:
      - name: Checkout code
        uses: actions/checkout@v4.1.1

      - name: Set up secrets
        run: |
          echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > secure_area/ansible_vault_password.txt
          echo "${{ secrets.ID_RSA_TEMPER }}" > secure_area/id_rsa_temper
          chmod 600 secure_area/*
          ln -s vault-runsap.yml ansible/vault.yml

      - name: Install dependencies
        run: |
          ansible-galaxy collection install -r ansible/requirements.yml

      - name: Running playbook ${{ github.event.inputs.playbook }}
        run: >
          ansible-playbook -i ansible/inventory-runsap 
          ansible/sap-system-refresh.yml 
          -e cli_refresh_action="${{ github.event.inputs.cli_refresh_action }}"
          -e cli_refresh_sap_env="${{github.event.inputs.cli_refresh_sap_env}}"
          -e cli_refresh_backup_location="${{github.event.inputs.cli_refresh_backup_location}}"

```
