
# Cloudplow
Automatic rclone remote uploader, with support for multiple remote/folder pairings.  UnionFS Cleaner functionality: Deletion of UnionFS whiteout files  and their corresponding files on rclone remotes. Automatic remote syncer: Sync between different remotes via a Scaleway server instance, that is created and destroyed at every sync.


# Requirements
1. Python 3.5 or higher (`sudo apt install python3 python3-pip`).
2. requirements.txt modules (see below).

# Installation on Ubuntu/Debian

1. `cd /opt`
3. `sudo git clone https://github.com/l3uddz/cloudplow`
4. `sudo chown -R user:group cloudplow` (run `id` to find your user / group)
5. `cd cloudplow`
6. `sudo python3 -m pip install -r requirements.txt`
7. `python3 cloudplow.py` to generate default config.json


# Config


## Sample

```
{
    "core": {
        "dry_run": false
    },
    "hidden": {
        "/mnt/local/.unionfs-fuse": {
            "hidden_remotes": [
                "google"
            ]
        }
    },
    "notifications": {
        "Pushover": {
            "app_token": "",
            "service": "pushover",
            "user_token": ""
        }
    },
    "remotes": {
        "google": {
            "hidden_remote": "google:",
            "rclone_excludes": [
                "**partial~",
                "**_HIDDEN~",
                ".unionfs/**",
                ".unionfs-fuse/**"
            ],
            "rclone_extras": {
                "--checkers": 8,
                "--drive-chunk-size": "64M",
                "--no-traverse": null,
                "--stats": "60s",
                "--transfers": 4,
                "--verbose": 1
            },
            "rclone_sleeps": {
                "Failed to copy: googleapi: Error 403: User rate limit exceeded": {
                    "count": 5,
                    "sleep": 25,
                    "timeout": 300
                }
            },
            "remove_empty_dir_depth": 1,
            "upload_folder": "/mnt/local/Media/",
            "upload_remote": "google:/Media/"
        },
    },
    "uploader": {
        "google": {
            "check_interval": 30,
            "exclude_open_files": true,
            "max_size_gb": 500,
            "opened_excludes": [
                "/downloads/"
            ],
            "size_excludes": [
                "downloads/*"
            ]
          }
    }
}
```


## Core

```
    "core": {
        "dry_run": false
    },
```

`"dry_run": true` will prevent any files being uploaded or deleted - use this to test out your config.


## Hidden
UnionFS Hidden File Cleaner: Deletion of UnionFS whiteout files and their corresponding files on rclone remotes.

```
    "hidden": {
        "/mnt/local/.unionfs-fuse": {
            "hidden_remotes": [
                "google"
            ]
        }
    },
    
```

This is where you specify the location of the unionfs _HIDDEN~ files (i.e. whiteout files) and the rclone remotes where the corresponding files will need to be deleted from. You may specify than one remote here. 

The specific remote path, where those corresponding files are, will be specified in the `remotes` section. 


## Notifications

Notification alerts during tasks. 


```
    "notifications": {
        "Pushover": {
            "app_token": "",
            "service": "pushover",
            "user_token": ""
        }
    },
```

Currently, only Pushover is supported. But more will be added later. 

### Pushover

Retrieve `app_token` and `user_token` from Pushover.net and fill it in. 

Note: The key name (e.g. `"Pushover":`) can be anything, but the `"service":` must be  `"pushover"`,


## Remotes

This is the heart of the configuration, most of the config references this section one way or another (e.g. hidden path references).

You can specify more than one remote here. 


```
    "remotes": {
        "google": {
            "hidden_remote": "google:",

```

Under `"remote"`, you have the name of the remote as the key (in the example above, it is `"google"`). The remote name can be anything (e.g. google1, google2, google3, dropbox1, etc). 

`"hidden_remote"`: is the remote path where the unionfs hidden cleaner will remove files from (if the remote is listed under the `hidden` section). 


```
            "rclone_excludes": [
                "**partial~",
                "**_HIDDEN~",
                ".unionfs/**",
                ".unionfs-fuse/**"
            ],
```
These are the excludes to be used when uploading to this remote.


```
            "rclone_extras": {
                "--checkers": 16,
                "--drive-chunk-size": "64M",
                "--no-traverse": null,
                "--stats": "60s",
                "--transfers": 8,
                "--verbose": 1
            },
```
These are rclone parameters that will be used when uploading to this remote. You may add other rclone parameters. 

Note: a value of null will mean `--no-traverse` instead of `--no-traverse=null`. 



```
            "rclone_sleeps": {
                "Failed to copy: googleapi: Error 403: User rate limit exceeded": {
                    "count": 5,
                    "sleep": 25,
                    "timeout": 300
                }
            },
```
`"rclone_sleeps"` are keywords or phrases that are monitored during rclone tasks that will cause this remote's upload task to abort and go into a sleep for a specified amount of time. When a remote is asleep, it will not do it's regularly scheduled uploads (as definted in `check_intervals`). 

You may list multiple keywords or phrases here.

In the example above, the phrase `"Failed to copy: googleapi: Error 403: User rate limit exceeded"` is being monitored. 

`"count"`: How many times this keyword/phrase has to occur within a specific time period (i.e. `timeout`), from the very first occurrence, to cause the remote to go to sleep. 

`"sleep"`: How many hours the remote goes to sleep. 

`"timeout"`: The time period (in seconds) the phrase is counted in. On it's first occurrence, the time is logged and if count is reached within this many seconds, the upload task will abort and the remote will go into sleep. If the timeout period expires, then the count will restart from 0. 



```
            "remove_empty_dir_depth": 2,
```
This is the depth to min-depth to delete empty folders from relative to `upload_folder`  (1=/Media  ; 2 = /Media/Movies; 3=/Media/Movies/Movies-Kids/)


```
            "upload_folder": "/mnt/local/Media/",
            "upload_remote": "google:/Media/"

```

`"upload_folder"`: is the local path that is uploaded by the `uploader` task, once it reaches the size threshold as specified in `max_size_gb`. 

`"upload_remote"`: is the remote path that `uploader` task will  uploaded to. 




## Uploader

Each entry to `uploader` references a remote inside `remotes`. The remote can only be referenced ONCE inside this list. 

```
    "uploader": {
        "google": {
            "check_interval": 30,
            "exclude_open_files": true,
            "max_size_gb": 500,
            "opened_excludes": [
                "/downloads/"
            ],
            "size_excludes": [
                "downloads/*"
            ]
          }
    }
```

In the example above, the remote `"google"` is being referenced from the `remotes` section. 


`"check_interval"`: how often (in minutes) to check the size of this remotes `upload_folder`. Once it reaches the size threshold as specified in `max_size_gb`, the uploader will start. 
            
`"exclude_open_files"`: when set to `true`, open files will be excluded from the rclone transfer (i.e. transfer will occur without them).

`"max_size_gb"`: maximum size (in gigabytes) before uploading can commence

`"opened_excludes"`: Paths the open file checker will check for when searching for open files. In the example above, any open files with `/downloads/` in it's path, would be ignored. 

`"size_excludes"`: Paths that will not be counted in the total size calculation for `max_size_gb`.		
         
            
<pre>








</pre>


# Temp (only for reference)

```

{
    "core": {
		# dry_run when enabled will prevent any files being deleted (hiddens / local content) - it will always use dry-run when executing rclone so its perfect for testing your config
        "dry_run": false
    },
	# this is where you specify the hidden folder location(s) of unionfs that will be cleaned
    "hidden": {
		# the key is this dict is the hidden folder to clean, followed by its settings
        "/mnt/local/.unionfs-fuse": {
			# hidden_remotes is a list of remote keys from the remotes key below
			# any remote listed inside this list will have the hidden folders/files deleted when hiddens are found for this folder
            "hidden_remotes": [
                "google"
            ]
        }
    },
    "notifications": {
		# each key here is the name of the notification, it can be anything, its a generic key to specify this notification name, e.g. Pushover l3uddz, Pushover desi
        "Pushover": {
			# the app_token to be used for the service below
            "app_token": "",
			# the service key is VITAL, this is where cloudplow determines which notification agent to use
            "service": "pushover",
			# the user_token used for this service 
            "user_token": ""
        }
    },
	# this is the heart of the configuration, most of the config references this section one way or another (e.g. hidden_remotes references the entrys of this section)
    "remotes": {
		# the remote name, this can be anything, e.g. google1, google2, google3, dropbox1 etc...
        "google": {
			# hidden_remote is used by the hidden cleaner, when it finds a hidden file in e.g. /mnt/local/.unionfs-fuse it will replace that with this to generate a path that can be used with rclone delete.
            "hidden_remote": "google:",
			# these are the excludes to be used when uploading to this remote
            "rclone_excludes": [
                "**partial~",
                "**_HIDDEN~",
                ".unionfs/**",
                ".unionfs-fuse/**"
            ],
			# these are parameters that will be used when uploading to this remote (a value of null will mean, e.g. --no-traverse instead of --no-traverse=null). YOu can add other rclone parameters below. 
            "rclone_extras": {
                "--checkers": 16,
                "--drive-chunk-size": "64M",
                "--no-traverse": null,
                "--stats": "60s",
                "--transfers": 8,
                "--verbose": 1
            },
			# these are keywords that will cause this remote to abort,
            "rclone_sleeps": {
                "Error 403: User rate limit exceeded": {
                    "count": 5, # how many times to occur within timeout from first occurrence
                    "sleep": 25,   #After sleep of X hours,  EMOTE will become available for uploads at normal check_intervals. 
                    "timeout": 300 # on first occurrence, log this time and if count is reached within this many seconds, abort.
                }
            },
			# this is the depth to min-depth to delete empty folders from relative to upload_folder  (1=/Media  ; 2 = /Media/Movies; 3=/Media/Movies/Movies-Kids/)
            "remove_empty_dir_depth": 2,
			# this is the folder that is uploaded by the uploader / size checked
            "upload_folder": "/mnt/local/Media",
			# this is the remote that upload_folder will be uploaded too
            "upload_remote": "google:/Media"
        }
    },
    "uploader": {
		# each entry to uploader references a remote inside remotes. 1 remote can only be referenced ONCE inside this list
        "google": {
			# how often to check the size of this remotes upload_folder 
            "check_interval": 30,
			# exclude open files from the rclone transfer if they are open (transfer will occur without them)
            "exclude_open_files": true,
			# maximum size before uploading can commence
            "max_size_gb": 400,
			# ignore anything matching these excludes when it comes to the open file checker, e.g. ignore any open files that have /downloads/ in their path 
            "opened_excludes": [
                "/downloads/"
            ],
			# dont include any files/folders in the total size calculation that match these excludes
            "size_excludes": [
                "downloads/*"
            ]
        }
    }
}

```
