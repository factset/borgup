# Borgup
Borgup – A configuration-based wrapper for [Borg Backup](https://www.borgbackup.org/)

# Main features
* Use Borg Backup's everyday commands with most options read from a configuration.
* Get notifications in your window manager when the backup starts, finishes, or fails.

## Preparation
* Install [Borg Backup](https://www.borgbackup.org/).
  * On CentOS 7 it is available in the repository EL_7_epel_x86_64.
    ```
    yum install borgbackup
    ```

## Quick start
* Create configuration:
  ```
  $ mkdir doc-backup
  $ cat >doc-backup/config
  REPOSITORY=/mnt/backup/documents
  ARCHIVE="doc-A-`/bin/date +\%Y\%m\%d\-\%H\%M\%S`"
  SOURCE_PATHS="/home/my-name/documents"
  PRUNE_PREFIX="doc-A-"
  PRUNE_OPTIONS="--keep-daily 7 --keep-monthly 12 --keep-yearly 100"
  ^D
  ```
* Pick a passphrase, e.g. using `pwgen -nysB 16`, and write it to the configugration:
  ```
  $ cat >doc-backup/.borgup-passphrase
  export BORG_PASSPHRASE='YOUR_BORG_PASSPHRASE_GOES_HERE'
  ^D
  ```
  **Important:** Keep a copy of your passphrase in a safe place.
  Storing it in your backup won't help you, of course.
* Initialize Borg repository:
  ```
  $ borgup init --encryption repokey-blake2 doc-backup
  ```
* Create a backup archive:
  ```
  $ borgup create doc-backup
  ```
* Prune old archives:
  ```
  $ borgup prune doc-backup
  ```
* List existing archives:
  ```
  $ borgup list doc-backup
  ```

## Would you like to know more?
### Full documentation
See `borgup --help` and `borgup --help-config`.

### Enable notifications
This currently works only with the window manager XFCE.
* Make sure the command `notify-send` is installed. (On CentOS 7 is comes from the package `libnotify`.)
* Add
  ```
  NOTIFY_USER=my-name
  NOTIFY_WM=xfce
  ```
  to the `config` file.
You will get notifications for the Borgup commands `create` and `prune` when they start, finish, or fail.
The latter is a persistent notification. That is particularly useful when running Borgup from Cron.

### Run Borgup from Cron
For example:
```
# cat >/etc/cron.daily/borgup
#!/bin/bash
BORGUP=/path/to/borgup
CONFIGDIR=/path/to/doc-backup
LOGDIR=/path/to/log-dir
sudo -u my-name nice -n 19 ionice -c2 -n7 {
  $BORGUP create $CONFIGDIR
  $BORGUP prune $CONFIGDIR   
} >>$LOGDIR/borgup-`/bin/date +\%Y-\%m`.log 2>&1
^D
chmod u+x /etc/cron.daily/borgup
```
Also you probably want to make sure that `anacron` is installed, unless your computer it always on.

### Verify that is really works
* Do you get error notifications? In that case it does not yet work.
* Do you get notification when Borgup starts and finishes? Good!
* Look at the log specified in `/etc/cron.daily/borgup`. There should be one run of `create` and one of `prune` per day.

### Restore files from a backup
Please use the plain Borg command. I am sorry, but Borgup currently supports only Borg's most common everyday commands. However, continuing the example from above, it roughly goes this way:
* Use `borgup list` to determine the name of the archive you want to access, e.g. `doc-A-19501105-061500`.
  ```
  # borgup list doc-backup
  ```
* Use `borg mount` to mount that archive using the Borg repository and archive name and a mount point. You'll need to enter the passphrase from `doc-backup/.borgup-passphrase`.
  ```
  # borg mount /mnt/backup/documents::doc-A-19501105-061500 /mnt/borg
  ```
* Locate the files to restore in the mounted backup and copy them.
  ```
  # cp /mnt/borg/home/my-name/documents/important-thing /home/my-name/documents/important-thing.restored
  ```
* Unmount.
  ```
  # borg umount /mnt/borg
  ```
  
## Troubleshooting
### I do not get notifications
* Did you set `NOTIFY_USER` and `NOTIFY_WM` in Borg's configuration?
* Is `notify-send` installed and working? Try it out:
  ```
  # notify-send --icon=dialog-information --urgency low "Testing notify-send" "It works\!"
  ```
* Are you using the window manager XFCE? If you don't, but still `notify-send` works, find out how to determine the required `DBUS_KEY` from root. Then we can probably patch that into Borgup.
