Example
-------

```yaml
- name: Download wrapper
    ansible.builtin.include_role:
    name: runsap.sap.download_software
    vars:
    software_share: "{{ host_data.vm.tags.Software_share }}"
    downloads: "{{ sap_launchpad_downloads }}"
    suser_id: "{{ vault.suser_id }}" 
    suser_password: "{{ vault.suser_password }}"
    sap_launchpad_downloads:
    - download: MAXDB7911_6-20009119.SAR
        location: Sap_maxdb/7.9.11.6        
    when: 
    - download_sap_software
    - sap_launchpad_downloads is defined 
    - sap_launchpad_downloads | length > 0
```