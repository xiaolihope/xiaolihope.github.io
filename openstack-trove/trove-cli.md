# Trove CLI
2017/05/10

## Datastore
```
trove datastore-list
trove datastore-show
trove datastore-version-list
trove datastore-version-show
```
## Instances

### trove list
### trove create
```
# trove create percon-test 7 --size 1 --datastore percona --datastore_version percona-5.6 --nic net-id=85e26b54-5f15-41d2-9be4-42a2ed8e086d --databases db1 --users user1:user1
+-------------------------+--------------------------------------+
| Property                | Value                                |
+-------------------------+--------------------------------------+
| created                 | 2017-05-24T10:47:12                  |
| datastore               | percona                              |
| datastore_version       | percona-5.6                          |
| encrypted_rpc_messaging | True                                 |
| flavor                  | 7                                    |
| id                      | 7496dd8e-f62e-49b6-bc7c-19b8139ce541 |
| name                    | percon-instance                      |
| region                  | RegionOne                            |
| server_id               | None                                 |
| status                  | BUILD                                |
| updated                 | 2017-05-24T10:47:12                  |
| volume                  | 1                                    |
| volume_id               | None                                 |
+-------------------------+--------------------------------------+

# trove create percon-test-slave 7 --size 1 --datastore percona --datastore_version percona-5.6 --nic net-id=85e26b54-5f15-41d2-9be4-42a2ed8e086d --locality anti-affinity
+-------------------------+--------------------------------------+
| Property                | Value                                |
+-------------------------+--------------------------------------+
| created                 | 2017-05-24T10:56:09                  |
| datastore               | percona                              |
| datastore_version       | percona-5.6                          |
| encrypted_rpc_messaging | True                                 |
| flavor                  | 7                                    |
| id                      | 085e9c5a-7d5a-487e-94c7-d8944519e0eb |
| name                    | percon-test-slave                    |
| region                  | RegionOne                            |
| server_id               | None                                 |
| status                  | BUILD                                |
| updated                 | 2017-05-24T10:56:09                  |
| volume                  | 1                                    |
| volume_id               | None                                 |
+-------------------------+--------------------------------------+
```
### trove show
### trove update
```
# trove update percon-test --name percon
```
### trove upgrade
### trove delete
### trove force-delete
### trove reset-status
### trove resize-instance
### trove resize-volume
### trove restart

## Database and Users
```
trove database-create
trove database-delete
trove database-list
trove user-create
trove user-delete
trove user-grant-access
trove user-list
trove user-revoke-access
trove user-show
trove user-show-access
trove user-update-attributes

trove root-disable
trove root-enable
trove root-show
```
## Configuration
```
trove configuration-attach
trove configuration-create
trove configuration-default
trove configuration-delete
trove configuration-detach
trove configuration-instances
trove configuration-list
trove configuration-parameter-list
trove configuration-parameter-show
```
### trove configuration-patch
```
# trove configuration-patch cfd4eb2c-2e75-4db8-b0bc-6da63d40aa01 '{"max_connections": 300}'
```
trove configuration-show
trove configuration-update

## Cluster
```
trove cluster-create
trove cluster-delete
trove cluster-force-delete
trove cluster-grow
trove cluster-instances
trove cluster-list
trove cluster-modules
trove cluster-reset-status
trove cluster-show
trove cluster-shrink
trove cluster-upgrade
```
## Backup

trove backup-copy
### trove backup-create

```
# trove backup-create  percon-test backup1 --description "backup for percon-test"
+-------------+-----------------------------------------------------------------------------------------------------------+
| Property    | Value                                                                                                     |
+-------------+-----------------------------------------------------------------------------------------------------------+
| created     | 2017-05-24T10:58:50                                                                                       |
| datastore   | {u'version': u'percona-5.6', u'type': u'percona', u'version_id': u'36f40e4d-ab3f-4fd3-ae99-e70c445bc4d3'} |
| description | backup for percon-test                                                                                    |
| id          | 42b4899f-9cd4-4d34-9940-8d80e6f5211b                                                                      |
| instance_id | 0eea7a9d-1895-4ab0-acb9-c60e04940dc0                                                                      |
| locationRef | None                                                                                                      |
| name        | backup1                                                                                                   |
| parent_id   | None                                                                                                      |
| size        | None                                                                                                      |
| status      | NEW                                                                                                       |
| updated     | 2017-05-24T10:58:50                                                                                       |
+-------------+-----------------------------------------------------------------------------------------------------------+
```
```
trove backup-delete
trove backup-list
trove backup-list-instance
trove backup-show
```
## Schedule Backup
```
trove schedule-create
trove schedule-delete
trove schedule-list
trove schedule-show
```
## Replica
```
trove detach-replica
trove eject-replica-source
trove promote-to-replica-source
```
## Execution
```
trove execution-delete
trove execution-list
```
## Flavor
```
trove flavor-list
trove flavor-show
```
## Limit
trove limit-list

## Log

trove log-disable
trove log-discard
trove log-enable
trove log-list
trove log-publish
trove log-save
trove log-show
trove log-tail

## Metadata

trove metadata-create
trove metadata-delete
trove metadata-edit
trove metadata-list
trove metadata-show
trove metadata-update

## Trove Module

trove module-apply
trove module-create
trove module-delete
trove module-instance-count
trove module-instances
trove module-list
trove module-list-instance
trove module-query
trove module-reapply
trove module-remove
trove module-retrieve
trove module-show
trove module-update

## Quota

trove quota-show
trove quota-update

## Security Groups

trove secgroup-add-rule
trove secgroup-delete-rule
trove secgroup-list
trove secgroup-list-rules
trove secgroup-show

## Volume

trove volume-type-list
trove volume-type-show

## Trove-manage

```
usage: trove-manage [-h] [--config-dir DIR] [--config-file PATH] [--debug]
                    [--log-config-append PATH] [--log-date-format DATE_FORMAT]
                    [--log-dir LOG_DIR] [--log-file PATH] [--nodebug]
                    [--nouse-journal] [--nouse-syslog] [--nowatch-log-file]
                    [--syslog-log-facility SYSLOG_LOG_FACILITY]
                    [--use-journal] [--use-syslog] [--version]
                    [--watch-log-file]

                    {db_sync,db_upgrade,
datastore_update,   //usage: trove-manage datastore_update [-h] datastore_name default_version
datastore_version_update,
db_recreate,
db_load_datastore_config_parameters,
datastore_version_flavor_add,
datastore_version_flavor_delete,
datastore_version_volume_type_add,
datastore_version_volume_type_delete,
datastore_version_volume_type_list}
```

### Note

1. There is no trove-manage datastore delete command, as removing a datastore will require removing all the instances using that datastore, otherwise it would cause an integrity error.


