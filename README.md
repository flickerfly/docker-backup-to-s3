# flickerfly/backup-to-s3

Docker container that periodically backups files to Amazon S3 using awscli and cron.
All files will be tgz'd and encrypted with AES 256 CBC.

**Always test to restore the files from the backup, before relying on it.**


To decrypt resulting s3 object 2016-04-11T07:25:30Z.tgz.aes:
 
 ```
openssl enc -aes-256-cbc -md sha512 -pbkdf2 -iter 100100 -pass "pass:${AES_PASSPHRASE}" -in 2016-04-11T07:25:30Z.tgz.aes -out restore.tgz -d
tar xf restore.tgz
```

## Usage

```
docker run -d [options] flickerfly/backup-to-s3 backup-once|schedule|restore
```

* Backup: Make a single backup and exit.
* Schedule: Schedule backups with using cron. 
* Restore: Restore a backup, 

## Options

| Name                                                | Operation        | Required | Description |
| -------------------------------------------------  | ----------------- | --------- | --------------------------- |
| -e AWS_ACCESS_KEY_ID=eu-central-1                  | all                   | yes |  Endpoint region (ideally where bucket is located)  |
| -e AWS_ACCESS_KEY_ID=&lt;AWS_KEY&gt;               | all                   | yes |  Your AWS key  |
| -e AWS_SECRET_ACCESS_KEY=&lt;AWS_SECRET&gt;        | all                   | yes | Your AWS secret |
| -e S3_PATH=s3://&lt;BUCKET_NAME&gt;/&lt;PATH&gt;/  | all                   | yes | S3 Bucket name and path. Should end with trailing slash. | 
| -e AES_PASSPHRASE=&lt;PASSPHRASE&gt;               | all                   | yes | Passphrase to generate AES-256-CBC encryption keys with. 
| -e WIPE_TARGET=false                               | restore               | no | Delete contents of target directory before restoring.
| -e POST_RESTORE_COMMAND=cmd                        | restore               | no | Command to run (in the container) after successfully restoring.
| -e VERSION=&lt;VERSION_TO_RESTORE&gt;              | restore               | yes | The version to restore, must be the full s3 object name without the `tgz.aes` suffix. | 
| -e PARAMS="--dry-run"                              | all                   | no  | Parameters to pass to the s3 command. [(full list here)](http://docs.aws.amazon.com/cli/latest/reference/s3/cp.html) | 
| -e DATA_PATH=/data/                                | all                   | no  | Container's data folder. Default is `/data/`. Should end with trailing slash. | 
| -e PREFIX=prefix                                   | backup-once, schedule | no  | Prefix to encrypted tgz file name. The basename is a date stamp with a tgz.aes suffix | 
| -e PRE_BACKUP_COMMAND=cmd                          | backup                | no | Command to run (in the container) before starting the zip and encryption process.
| -e CRON_SCHEDULE='5 3 \* \* \*'                    | schedule              | no  | Specifies when cron job runs, see [format](http://en.wikipedia.org/wiki/Cron). Default is 5 3 \* \* \*, runs every night at 03:05 |
| -v /path/to/backup:/data:ro                        | backup-once, schedule | yes | Mount target local folder to container's data folder. Content of this folder will be tar:ed, encrypted and uploaded to the S3 bucket. | 
| -v /path/to/restore:/data                          | restore               | yes | Mount target local folder to container's data folder. The restored files from the S3 bucket will overwrite all files in the /path/to/restore folder. Note that the folder will not be emptied first, leaving any no overwritten files as is. |


## Examples
 
Backup to S3 everyday at 12:00:

```bash
docker run -d \
  -e AWS_DEFAULT_REGION=eu-central-1 \
  -e AWS_ACCESS_KEY_ID=myawskey \
  -e AWS_SECRET_ACCESS_KEY=myawssecret \
  -e S3_PATH=s3://my-bucket/backup/ \
  -e AES_PASSPHRASE=secret \
  -e CRON_SCHEDULE='0 12 * * *' \
  -v /home/user/data:/data:ro \
  flickerfly/backup-to-s3 schedule
```


Backup once and then delete the container:

```bash
docker run --rm \
  -e AWS_DEFAULT_REGION=eu-central-1 \
  -e AWS_ACCESS_KEY_ID=myawskey \
  -e AWS_SECRET_ACCESS_KEY=myawssecret \
  -e S3_PATH=s3://my-bucket/backup/ \
  -e AES_PASSPHRASE=secret \
  -v /home/user/data:/data:ro \
  flickerfly/backup-to-s3 backup-once
```

Restore the backup from `2016-04-11T07:25:30Z` and then delete the container:

```bash
docker run --rm \
  -e AWS_DEFAULT_REGION=eu-central-1 \
  -e AWS_ACCESS_KEY_ID=myawskey \
  -e AWS_SECRET_ACCESS_KEY=myawssecret \
  -e S3_PATH=s3://my-bucket/backup/ \
  -e AES_PASSPHRASE=secret \
  -e VERSION=2016-04-11T07:25:30Z \
  -v /home/user/data:/data \
  flickerfly/backup-to-s3 restore
```

Create a simple backup of a single Kubernetes persistent volume claim:

```bash
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-to-s3
  labels:
    role: backup
data:
  CRON_SCHEDULE: '0 12 * * *'
  S3_PATH: s3://mybucketname/bedrock/
  AWS_DEFAULT_REGION: us-east-1
  # Pass this to the S3 command
  PARAMS: ""
  PREFIX: bedrock
  DATA_PATH: /data/

---
# Be sure to Base64 encrypt your secrets before entering
# Example: echo -n "secret" | base64
apiVersion: v1
kind: Secret
metadata:
  name: backup-to-s3
  labels:
    role: backup
data:
  AES_PASSPHRASE: XXXXXXXX
  AWS_ACCESS_KEY_ID: XXXXXXX
  AWS_SECRET_ACCESS_KEY: XXXXXXXX

---
# Setup the Deployment of a pod that will do the work.
# Note: We mount the pvc read only, so this can not do a restore or otherwise corrupt your volume.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bds-pvc-backup
  labels:
    role: backup
    app: bedrock
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      role: backup
      app: bedrock
  template:
    metadata:
      labels:
        app: bedrock
        role: backup
    spec:
      containers:
        - name: backup-to-s3
          image: jritchie/backup-to-s3
          args: 
            - schedule
          volumeMounts:
            - name: bds
              mountPath: /data
          envFrom:
            - configMapRef:
                name: backup-to-s3
            - secretRef:
                name: backup-to-s3
      volumes:
        - name: bds
          persistentVolumeClaim:
            claimName: bds
            readOnly: true
```
