-- old

| VM Name / Role        | NFS Shares (Synology) | Optional ZFS Disks   | VM Mount Points                  | Docker Volumes                                                       | Notes                                           |
| --------------------- | --------------------- | -------------------- | -------------------------------- | -------------------------------------------------------------------- | ----------------------------------------------- |
| plex-worker           | `media`               | `configs`            | `/mnt/configs`, `/mnt/media`     | `plex-config`, `tautulli-config`, `media`                            | Media is NFS; configs optional ZFS              |
| media-mgmt-worker     | `downloads`           | `configs`            | `/mnt/configs`, `/mnt/downloads` | `radarr-config`, `sonarr-config`, `transmission-config`, `downloads` | Downloads on NFS or optional local              |
| home-assistant-worker | none                  | `configs`            | `/mnt/configs`                   | `ha-config`, `unifi-config`                                          | Only configs on optional ZFS for now            |
| metrics-worker        | none                  | `configs`, `db-data` | `/mnt/configs`, `/mnt/data`      | `grafana-config`, `prometheus-data`, `graylog-data`                  | Metrics DBs can use optional ZFS                |
| db-worker             | none                  | `configs`, `db-data` | `/mnt/configs`, `/mnt/data`      | `influxdb-data`, `mysql-data`                                        | DB storage optional ZFS; future migration ready |
