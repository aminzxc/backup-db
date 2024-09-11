### Connect to one of the MongoDB instances in the cluster
```
mongo --host 127.0.0.1 --port 27021
```

### Authenticate as a user with backup privileges
```
use admin
db.auth("backupUser", "backupUserPassword")
```
### Create the `backup`
```
mongodump --host 127.0.0.1 --port 27021 --gzip --archive=/path/to/backup/backup.gz --authenticationDatabase admin --username backupUser --password backupUserPassword
```
or
```
mongodump --db database_name --collection collection_name
```
### `Restore` the backup (if needed)
```
mongorestore --drop --gzip --archive=/path/to/backup/backup.gz --authenticationDatabase admin --username backupUser --password backupUserPassword
```
### Copy the backup file to the host machine
```
docker cp mongo-backup:/backup/backup.gz /path/on/host/backup.gz
```
### Backup and restore collection
```
mongodump --db=EntityDb --collection=brandModelTypes --out=/path/to/backup/directory
```
```
mongorestore --db=EntityDb --collection=brandModelTypes --dir=/path/to/backup/EntityDb/test.bson
```
### Export collection json
```
mongoexport --db=EntityDb --collection=brandModelTypes --out=/mnt/test.json
```
### delete collection 
```
use DBname
db.collectionName.drop()
```
