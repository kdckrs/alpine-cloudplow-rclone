# alpine-cloudplow-rclone

[![License: GPL v3](https://img.shields.io/badge/License-GPL%203-blue.svg?style=flat-square)](https://github.com/kdckrs/alpine-cloudplow-rclone/blob/main/LICENSE)
[![Build Status](https://github.com/kdckrs/alpine-cloudplow-rclone/workflows/Build/badge.svg)](https://github.com/kdckrs/alpine-cloudplow-rclone/actions)
[![Docker Pulls](https://img.shields.io/docker/pulls/kdckrs/alpine-cloudplow-rclone)](https://hub.docker.com/r/kdckrs/alpine-cloudplow-rclone)
[![rclone version](https://img.shields.io/github/v/release/rclone/rclone?label=rclone%20version)](https://hub.docker.com/r/rclone/rclone)
[![s6-overlay version](https://img.shields.io/github/v/release/just-containers/s6-overlay?label=s6-overlay%20version)](https://github.com/just-containers/s6-overlay)

Docker image for the [cloudplow](https://github.com/l3uddz/cloudplow) cloud media sync service, using [rclone's official Docker image](https://hub.docker.com/r/rclone/rclone) based on Alpine Linux as a foundation.

## Application

[cloudplow](https://github.com/l3uddz/cloudplow)

[rclone](https://github.com/rclone/rclone)

## Description

Cloudplow is an automatic rclone remote uploader with support for scheduled transfers, multiple remote/folder pairings, UnionFS control file cleanup, and synchronization between rclone remotes.

## Usage

### Cloudplow

```yaml
cloudplow:
  image: kdckrs/alpine-cloudplow-rclone
  container_name: cloudplow
  environment:
    - PUID=1000 # Optionally replace this with the uid of your user
    - PGID=1000 # Optionally replace this with the gid of your user
    - CLOUDPLOW_CONFIG=/config/config.json
    - CLOUDPLOW_LOGFILE=/config/cloudplow.log
    - CLOUDPLOW_LOGLEVEL=DEBUG
    - CLOUDPLOW_CACHEFILE=/config/cache.db
  # uncomment below if you are planning to run the rclone mount command using this container
  #cap_add: 
  #  - SYS_ADMIN
  #devices:
  #  - /dev/fuse
  #security_opt:
  #  - apparmor:unconfined
  volumes:
    #- ./conf/cloudplow/config.json:/config/config.json # Uncomment if you have a local cloudplow config ready
    - /usr/local/etc/rclone:/config/rclone # path to rclone config path, can be /home/<user>/.config/rclone if already installed
    - /usr/local/etc/cloudplow:/config # path where the cloudplow config will be stored 
    - /mnt:/data:shared # the folder where all your media and linux iso's are stored ;)
    - /etc/localtime:/etc/localtime:ro
    # - /home/<user>/google_drive_service_accounts:/service_accounts # optionally if you are using Google drive service accounts
    #- /etc/passwd:/etc/passwd:ro #uncomment below if you are planning to run the rclone mount command using this container
    #- /etc/group:/etc/group:ro #uncomment below if you are planning to run the rclone mount command using this container
    #- /etc/user:/etc/user:ro #uncomment below if you are planning to run the rclone mount command using this container
    #- /etc/fuse.conf:/etc/fuse.conf:ro #uncomment below if you are planning to run the rclone mount command using this container
  restart: unless-stopped
```

Upon first run, the container will generate a sample config.json (if you haven't already linked an existing one) in the container's /config. Edit this config.json to your liking, making sure to set rclone_config_path to the location of the rclone.conf you mapped into the container. Some suggested settings for uploading to a remote, but not synchronizing between remotes, are given below:

```json
{
  "core": {
    "dry_run": true,
    "rclone_binary_path": "/usr/bin/rclone",
    "rclone_config_path": "/config/rclone/rclone.conf"
  },
  "hidden": {
    "/mnt/local/.unionfs-fuse": {
      "hidden_remotes": [
        "google"
      ]
    }
  },
  "notifications": {},
  "nzbget": {
    "enabled": false,
    "url": "https://user:password@nzbget.domain.com"
  },
  "plex": {
    "enabled": false,
    "max_streams_before_throttle": 1,
    "notifications": false,
    "poll_interval": 60,
    "rclone": {
      "throttle_speeds": {
        "1": "50M",
        "2": "40M",
        "3": "30M",
        "4": "20M",
        "5": "10M"
      },
      "url": "http://localhost:7949"
    },
    "token": "",
    "url": "https://plex.domain.com"
  },
  "remotes": {
    "google": {
      "hidden_remote": "google:",
      "rclone_command": "move",
      "rclone_excludes": [
        "**partial~",
        "**_HIDDEN~",
        ".unionfs/**",
        ".unionfs-fuse/**"
      ],
      "rclone_extras": {
        "--checkers": 16,
        "--drive-chunk-size": "64M",
        "--skip-links": null,
        "--stats": "60s",
        "--transfers": 8,
        "--user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36",
        "--verbose": 1
      },
      "rclone_sleeps": {
        "Failed to copy: googleapi: Error 403: User rate limit exceeded": {
          "count": 5,
          "sleep": 25,
          "timeout": 3600
        },
        " 0/s,": {
          "count": 16,
          "sleep": 25,
          "timeout": 62
        }
      },
      "remove_empty_dir_depth": 2,
      "sync_remote": "google:/Media",
      "upload_folder": "/mnt/local/Media",
      "upload_remote": "google:/Media"
    }
  },
  "syncer": {},
  "uploader": {
    "google": {
      "can_be_throttled": true,
      "check_interval": 30,
      "exclude_open_files": true,
      "max_size_gb": 25,
      "opened_excludes": [
        "/downloads/"
      ],
      "schedule": {
        "allowed_from": "04:00",
        "allowed_until": "08:00",
        "enabled": false
      },
      "size_excludes": [
        "downloads/*"
      ]
    }
  }
}
```

Please refer to the official [cloudplow](https://github.com/l3uddz/cloudplow) documentation for additional information.

### rclone auto mount remote during boot (using this image)

Uncomment the docker-compose directives as documented before and mount the below script in your cloudplox container at `/etc/services.d/rclone/run`

```shell
#!/usr/bin/with-contenv sh

uid=${PUID:-1000}
user=$(getent passwd $uid | awk -F: '{print $1}')

# Kill existing mounts
if ls -1qA /data/remote | grep -q .
then ! killall rclone ||trap true && fusermount -u /data/remote || true
fi

exec s6-setuidgid $user /usr/local/bin/rclone mount google:Media /data/remote
```