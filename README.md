# Duplicity to S3 Glacier Guide

## Overview
Many teams want to backup large datasets directly to S3 Glacier or Glacier Deep Archive for a low-cost DR strategy. The first consideration for doing this is to use `aws s3 sync` or `rclone` to do this. While this approach *could* work, it's important to understand the drawbacks. 

Specifically, ***the cost of PUT operations***. If your dataset contains many files, a simple copy could be ***very, very*** expensive. 

Another issue is that each object written to Glacier has an overhead of ~32KB in Glacier **AND** 8KB of metaata **stored in S3**. This means that if you have many small files, the cost could actually be **more expensive** to write them to Glacier than keeping them in S3. 

To give an overly simplistic and dramatic scenario, let's say you have 10,000,000 1KB files. That's ~ 10GB of files total. To write this raw to Glacier, it woudl cost:

`10,000,000 PUT operations / ($0.05/1,000 PUT operations) = $500`

`10,000,000 * 8KB = 8 GB * $0.023 = $0.184 per GB-Month S3 (for metdata)`

`(10,000,000 * 1KB + 10,000,000 * 32 KB)/(1024*1024) = 33 GB * $0.004 per GB-Month for Glacier (non-DA)  = $0.16 per GB-month`

Even though the total size was only 10Gb, it cost $500 for the initial upload plus $0.344 per month. For comparison, cost to upload the same files to S3 would be a tenth of that and the monthly cost would be roughly $0.23. The problem is amplified if there is a high change rate with more frequent backups/syncs.

In order to reap the benefits of using S3 Glacier of a low-cost backup target, backups need to be sent in a way that optimizes the number of PUT operations and file sizes. Many commercial backup/archival software products address this, but what if you're looking for an open and free alternative? This is where Duplicity comes in.

Duplicity is an open source command line backup utility that uses `librsync` to perform full+incremental backups of files, written to a variety of targets including S3 and S3 Glacier. Further, it optimizes backups by combining files into tar archives with (optional) compression and encryption. Using Duplicity, we can create full backups and send them directly to glacier as consolidated tar files in S3 and S3 Glacier, then capture incremental backups of changes. 

Revising the 10,000,000 1KB file example above, with Duplicity a full backup with a conservative 20% compression ratio and volume size of 1GB could result in as little as 8 1GB objects written to Glacier, costing just $0.0004 for PUT operations and $0.032 per month.

The guide below descibes how to setup and use a containerized version of Duplicity. It leverages the  [Tecnativa repo](https://github.com/Tecnativa/docker-duplicity) on Github, which contains a Dockerfile that builds a Duplicity container along with helper utilities to simplify its use. Using the sample commands or docker-compose below, you can stand up a Duplicity backup strategy to efficiently backup files to Glacier or Glacier Deep Archive on a regular basis. 

## Execution Options
The container provided in the [Tecnativa repo](https://github.com/Tecnativa/docker-duplicity) could be used in two ways:
- Call the container to run individual duplicity commands and immediately exit
- Run the container continuously and use the built in CRON/scheduler functionality to run jobs

## Running individual commands and exiting

If you want to schedule and run Duplicity operations through on-demand executions, you can do so by using `docker run`. This option is most useful if you do not want to continuously run the container.

First you'll need to clone the repo and build the docker image: 
```
git clone https://github.com/Tecnativa/docker-duplicity.git
docker build --target latest -t docker-duplicity:latest ./docker-duplicity
```
**Note:** The repo is available on Docker Hub as well, but we suggest you clone the git repo and build/push the image into your own repo.

With the image built and tagged, we can run individual commands. Below are examples:

### Run a backup:
```bash
docker run -it --rm --hostname 'backupname' \
-e OPTIONS="--full-if-older-than 4W --s3-use-deep-archive --no-compression --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \
-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \
-v /tmp/testfiles:/mnt/backup/src/data:ro \
-v "$PWD/config:/mnt/backup/src/config" \
-v "$PWD/logs:/logfiles" \
docker-duplicity:latest dup \
/mnt/backup/src/data \
boto3+s3://bucketname/backup-folder
```
Breaking this down:
| | |
| --- | ------------ |
| `docker run -it --rm --hostname 'backupname' \` | Run the container using hostname 'backupname'. This shoudl be a unique name that you use to describe the backup. It will be used in the name of the manifest file |
| `-e OPTIONS="--full-if-older-than 4W --s3-use-deep-archive --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \` | [Duplicity configuration options](http://duplicity.nongnu.org/vers8/duplicity.1.html#sect5) that will be used. <br /> The options in this example indicate to: <br /> <ul><li>Generate a full backup if the last full backup was more than 4 weeks ago, otherwise incremental</li><li>Upload to S3 Deep Archive</li><li>No encryption</li><li>Show progress</li><li>Store the archive manifest files (backup config) in a separate location</li><li>Use S3 multiprocessing</li><li>Combine (i.e. glob) files into achives up to 1024 MB - Important to reduce the number of PUT operations to Glacier</li><li>Log INFO</li><li>Log to file</li></ul> <br /> Feel free to customize these as needed.| 
| `-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \` | These are additional Duplicity configuration options that are used. In general, these don't need to be changed and are separated for convenience. |
| `-v /tmp/testfiles:/mnt/backup/src/data:ro \` | Mounts `/tmp/testfiles` from the host and uses it as the backup source |
| `-v "$PWD/config:/mnt/backup/src/config" \` | Mounts `./config` from to host as a volume in the container. This is where the duplicity manifest snd sig files are stored to track backup sets/chains. <br /> *Note:* Though these files are backed up to S3/Glacier, they should be stored persistently locally and used. Otherwise, you will need to restore the .sig file from Glacier in order to perform incrementals. |
| `-v "$PWD/logs:/logfiles" \` | Mounts `./logs` so that Duplicity can write its log files there and persist them. |
| `docker-duplicity:latest dup \` | The container and the command to execute (`dup`) |
| `/mnt/backup/src/ \` | The backup source - in this case the mount point |
| `boto3+s3://bucketname/backup-folder` | The target to write to - in this case an S3 bucket (Note: the `--s3-use-deep-archive` option above will cause the backups to be written to the Glacier Deep Archive tier in S3) |

### Cleanup backup sets 
The command below will cleanup backup sets (fulls and incrementals) that are older than the *7* most recent incrementals.

In other words, if full backups are taken once every 4 weeks, this will ensure fulls and incrementals older than `28 * 4 * 7 = 180`days are retained 
```bash
docker run -it --rm --hostname 'backupname' \
-e OPTIONS="--full-if-older-than 4W --s3-use-deep-archive --no-compression --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \
-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \
-v /tmp/testfiles:/mnt/backup/src/data:ro \
-v "$PWD/config:/mnt/backup/src/config" \
-v "$PWD/logs:/logfiles" \
docker-duplicity:latest dup \
remove-all-but-n-full 7 \
boto3+s3://bucketname/backup-folder
```

### List Collection Sets
List all backup chains - fulls and incrementals
```bash
docker run -it --rm --hostname 'backupname' \
-e OPTIONS="--full-if-older-than 47W --s3-use-deep-archive --no-compression --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \
-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \
-v /tmp/testfiles:/mnt/backup/src/data:ro \
-v "$PWD/config:/mnt/backup/src/config" \
-v "$PWD/logs:/logfiles" \
docker-duplicity:latest dup \
collection-status \
boto3+s3://bucketname/backup-folder
```

### List Files in a Backup
List all files in the latest backup. To list files from a specific point in time, use the `--time TIME` option. See [TIME](http://duplicity.nongnu.org/vers8/duplicity.1.html#sect8) format for more information.
```bash
docker run -it --rm --hostname 'backupname' \
-e OPTIONS="--full-if-older-than 47W --s3-use-deep-archive --no-compression --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \
-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \
-v /tmp/testfiles:/mnt/backup/src/data:ro \
-v "$PWD/config:/mnt/backup/src/config" \
-v "$PWD/logs:/logfiles" \
docker-duplicity:latest dup \
list-current-files \
boto3+s3://bucketname/backup-folder
```

## Using the container's built-in scheduler
The container from the Tecnativa repo has a built-in schduler option where the container can run continously and execute Duplcity commands on a schedule. The arguments are similar. 

The following is an example of a `docker-compose` file that will provision container and schedule a daily backup of a source to S3. Similar to the `docker run` commands above, it is configured to create a full backup every 4 weeks and incrementals otherwise. It also cleans up backup sets (fulls and associated incrementals) that are older than the last 7 fulls.


```yaml
version: "3.7"
services:
  backup:
    build: 
      context: https://github.com/Tecnativa/docker-duplicity.git
      target: latest
    image: docker-duplicity:latest
    hostname: srcname.backup
    container_name: srcname
    privileged: true # To speak with host's docker socket
    #context: ./
    volumes:
      - ./config:/mnt/backup/src/config
      - /tmp/testfiles://mnt/backup/src/data:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./logs:/logfiles
    environment:
      # JOB_200: Run backup daily. Based on $OPTIONS below, this will create a full every 4 weeks and incrementals otherwise
      JOB_200_WHAT: dup $$SRC $$DST
      JOB_200_WHEN: daily
      # JOB_300: Cleanup backups. This will delete backup files that are older than the last 6 fulls. 4 weeks per full x 7 fulls = 180 week retention
      # NOTE: Without --force this only lists chains to be deleted
      JOB_300_WHAT: dup remove-all-but-n-full 7 $$DST --force
      JOB_300_WHEN: daily

      # SPECIFY an AWS ACCESS_KEY to use. DO NOT store the key and ID docker-compose file itself to avoid accidentally committing credentials to a public repo! It will be compromised in minutes! Use environment vars or secrets instead
      # If you run this container on ECS, you can automatically use the task execution role's permissions instead of specifying them here.
      #AWS_ACCESS_KEY_ID: example amazon s3 access key
      #AWS_SECRET_ACCESS_KEY: example amazon s3 secret key
      DST: boto3+s3://bucket/backup-path
      SMTP_HOST: localhost
      EMAIL_FROM: user@nobody.com
      EMAIL_TO: user@nobody.com
      # Duplicity options: http://duplicity.nongnu.org/vers8/duplicity.1.html#sect5 
      OPTIONS: --s3-use-deep-archive --no-encryption --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$$(date "+%Y%m%d-%H%M%S").log --full-if-older-than 4W --asynchronous-upload
      # Generally, these shouldn't change
      OPTIONS_EXTRA: --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$$(hostname -f)- --file-prefix-manifest manifest-$$(hostname -f)- --file-prefix-signature signature-$$(hostname -f)- 
```
**Note:** This `docker-compose` will pull and build containers directly from the Tecnative repo. It is suggested that you clone the repo, review the code, and then point docker-compose to your repo to buld the image.

## Restores
If backing up to Glacier or Glacier Deep Archive, restorations will be a two-step process:
- First, you will need to [restore the Duplicity backup archives from Glacier into S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/restoring-objects.html).
- Once the files have been restored to S3, you can run this Duplicity command to recover files. The command below recovers a specific path, `/smallfiles`, in the backup. The recovered files are written to the local path `/mnt/recovered`
```bash
docker run -it --rm --hostname 'backupname' \
-e OPTIONS="--full-if-older-than 47W --s3-use-deep-archive --no-compression --no-encryption --progress --archive-dir /mnt/backup/src/config --s3-use-multiprocessing --volsize 1024 --verbosity i --log-file /logfiles/log-$(date "+%Y%m%d-%H%M%S").log" \
-e OPTIONS_EXTRA="--asynchronous-upload --s3-european-buckets --s3-multipart-chunk-size 10 --s3-use-new-style --metadata-sync-mode partial --file-prefix-archive archive-$(hostname -f)- --file-prefix-manifest manifest-$(hostname -f)- --file-prefix-signature signature-$(hostname -f)-" \
-v /tmp/testfiles:/mnt/backup/src/data:ro \
-v "$PWD/config:/mnt/backup/src/config" \
-v "$PWD/logs:/logfiles" \
-v "$PWD/recovered:/mnt/recovered" \
docker-duplicity:latest-s3 dup \
restore \
--file-to-restore smallfiles/ \
boto3+s3://bucketname/backup-folder
/mnt/recovered
```

## Resources
Duplicity combines files and diffs into tar archives and (by default) compresses them. As such, it's important to ensure you have enough local storage for tempoary files (generally, at least 2 times the value of `volsize` from experimentation but more is recommended). You should also ensure enough CPU, memory, and networking resources are available to perform backups as quickly as possible. 


