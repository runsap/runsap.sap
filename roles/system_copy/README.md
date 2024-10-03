Role: runsap.sap.system_copy
============================

Introduction
------------
De system copy role is able to refresh a SAP database with the following steps:

- backup database (optional)
- export rsectab 
- export tables
- restore database
- import rsectab
- import tables
- truncate tables

**Pre Processing**
The **export rsectab and tables** are performed on the "to be refreshed" system. You can define a list of tables that must be exported upfront. This role will first create scripts called `export_rsectab.ctl` and `export_tables.ctl` which are then executed, with R3trans, resulting in a datafile called `SID_export_rsectab.dat` and `SID_export_tables.dat`.

**Post Processing**
The post processing step imports the tables. This is done by creating two import control files which will refer to the export dat files. Next the postprocessing will close the client settings. 

Prerequisites
-------------
The role relies on `runsap.hana.hdb_backup` and `runsap.hana.hdb_restore`.

Example playbook
----------------

```yaml
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
#
# Post-processing
#
- include_role: 
    name: runsap.sap.system_copy
    vars:
    scripts_dir: "{{ share }}/Sap_refresh/{{ target_sap_sid }}"
    source_sap_sid: "{{ cli_refresh_sap_env }}"
    target_sap_sid: "{{ cli_refresh_sap_env }}"
    target_database: "{{ host_data.ec2.tags.Sap_hdb_sid }}"
    hdbsql_key: DEFAULT
    client: '000'
    #truncate_tables: "{{ truncate_tables }}" 
    secure_connection: "{{ host_data.ec2.tags.Sap_secure_db_connection }}"
    #refresh_tables: "{{ refresh_tables }}"
    system_copy_action: postprocessing    
```

