system_copy
===========

Introduction
------------
De system copy role is able to refresh a SAP database with the following steps:

- export rsectab 
- export tables
- import rsectab
- import tables

Prerequisites
-------------
The role relies on `runsap.hana.hdb_backup` and `runsap.hana.hdb_restore`.

