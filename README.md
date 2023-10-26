# Installing PhotoPrism on Debian

## Background

### What is PhotoPrism?

[PhotoPrism](https://photoprism.app/) is a self-hosted web application for managing and organising a photo collection. It aims to provide many of the popular features of cloud services like Google Photos.

### Why this guide?

The [PhotoPrism documentation](https://docs.photoprism.app/getting-started/) recommends Docker as the installation method. However, not everyone can, or wants to, use Docker. The purpose of this guide, therefore, is to provide instructions for setting up a usable PhotoPrism installation on [Debian](https://www.debian.org/). The guide has been written for, and tested on, Debian 12 "Bookworm", but should also work on older versions like Debian 11 "Bullseye", as well as derivatives like Ubuntu and Raspbian.

## DISCLAIMER

This guide is provided in good faith and for informational purposes only. No claims are made or guarantees given that it will work on any particular combination of hardware and software, or that it will be kept up-to-date with new releases of PhotoPrism. You will assume all responsibility for managing your PhotoPrism server, including the prevention of unauthorised access and safeguards against data loss.

## Installing PhotoPrism

### Download PhotoPrism

PhotoPrism provides [pre-built Linux packages](https://dl.photoprism.app/pkg/linux/) on their website. Download the latest build and extract it to `/opt/photoprism`:

```shell
$ wget https://dl.photoprism.app/pkg/linux/amd64.tar.gz
$ sudo mkdir /opt/photoprism
$ sudo tar xzf amd64.tar.gz -C /opt/photoprism/
$ rm amd64.tar.gz
```

If you're using an ARM-based system like a Raspberry Pi, you'll need to download the ARM64 build instead, so change the URLs and filenames accordingly.

You can run `/opt/photoprism/bin/photoprism -v` to verify the version.

If the pre-built package does not work on your system for some reason, it's possible to [build PhotoPrism manually](MANUAL_BUILD.md).

### Configure PhotoPrism

Create a separate user account for running PhotoPrism:

```shell
$ sudo useradd --system photoprism
```

Create a directory where PhotoPrism will store files like metadata, thumbnails, database (if using SQLite) and so on:

```shell
$ sudo mkdir /var/lib/photoprism
```

Ensure all relevant directories are owned by the newly created user:

```shell
$ sudo chown -R photoprism:photoprism /var/lib/photoprism /opt/photoprism
```

Go to the newly added directory and create a file for PhotoPrism configuration parameters:

```shell
$ cd /var/lib/photoprism
$ sudo nano .env
```

This opens the file in the [Nano](https://www.nano-editor.org/) text editor. Feel free to use another editor if you have a preference, but this guide will assume Nano.

The full list of configuration options is available [here](https://docs.photoprism.app/getting-started/config-options/), but you can use the following as a starting point:

```
# Initial password for the admin user
PHOTOPRISM_AUTH_MODE="password"
PHOTOPRISM_ADMIN_PASSWORD="photoprism"

# PhotoPrism storage directories
PHOTOPRISM_STORAGE_PATH="/var/lib/photoprism"
PHOTOPRISM_ORIGINALS_PATH="/var/lib/photoprism/photos/Originals"
PHOTOPRISM_IMPORT_PATH="/var/lib/photoprism/photos/Import"

# Uncomment below if using MariaDB/MySQL instead of SQLite (the default)
# PHOTOPRISM_DATABASE_DRIVER="mysql"
# PHOTOPRISM_DATABASE_SERVER="MYSQL_IP_HERE:PORT"
# PHOTOPRISM_DATABASE_NAME="DB_NAME"
# PHOTOPRISM_DATABASE_USER="USER_NAME"
# PHOTOPRISM_DATABASE_PASSWORD="PASSWORD"
```

Press `Ctrl+O` and `Enter` to save, then `Ctrl+X` to exit Nano. Now enter the following commands:

```shell
$ sudo chown photoprism:photoprism .env
$ sudo chmod 640 .env
```

This ensures that the file cannot be read by other users on the system, as it contains sensitive details.

### System service

The last step is setting up a system service so PhotoPrism can run automatically in the background.

Create a file for the service definition:

```shell
$ sudo nano /etc/systemd/system/photoprism.service
```

Add the following contents:

```
[Unit]
Description=PhotoPrism service
After=network.target

[Service]
Type=forking
User=photoprism
Group=photoprism
WorkingDirectory=/opt/photoprism
EnvironmentFile=/var/lib/photoprism/.env
ExecStart=/opt/photoprism/bin/photoprism up -d
ExecStop=/opt/photoprism/bin/photoprism down

[Install]
WantedBy=multi-user.target
```

Now run the following commands to start the service and to have it start automatically on every boot:

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now photoprism
```

If all went well, you should be able to open `http://YOUR-IP-HERE:2342` in a web browser and see the PhotoPrism interface. Log in as "admin" with the password set in the `.env` file.

### Automatic background tasks

It's possible to have PhotoPrism automatically run background tasks, like importing any photos that have been added to the Imports directory, using a cron job.

Create a file for the cron job:

```shell
$ sudo nano /etc/cron.d/photoprism
```

Enter the following contents:

```
0 * * * * photoprism export $(grep -v ^# /var/lib/photoprism/.env | xargs) && /opt/photoprism/bin/photoprism import >/dev/null 2>&1
```

This runs the PhotoPrism `import` command every hour. If you want to run it more (or less) frequently, change the time expression at the beginning accordingly. Use a helper like <https://crontab.cronhub.io> if needed.

It's also possible to run other commands. For example, if you add photos directly to the Originals directory and just need PhotoPrism to index them, change `import` to `index`. Run `/opt/photoprism/bin/photoprism` to get a full list of commands that can be executed.

For logging, replace `/dev/null` with the name of a log file (make sure the photoprism user can write to it). This can be helpful for troubleshooting.

#### Alternative method

It is also possible to use a systemd service and timer to run the background tasks. See [this comparison against cron](https://wiki.archlinux.org/title/Systemd/Timers#As_a_cron_replacement) for pros and cons.

Create a file for the service definition:

```shell
$ sudo nano /etc/systemd/system/photoprism-bg.service
```

Add the following contents:

```
[Unit]
Description=PhotoPrism background tasks
After=network.target

[Service]
Type=oneshot
User=photoprism
Group=photoprism
WorkingDirectory=/opt/photoprism
EnvironmentFile=/var/lib/photoprism/.env
ExecStart=/opt/photoprism/bin/photoprism import

[Install]
WantedBy=multi-user.target
```

Again, change `import` to whatever command you need to execute.

Since it is a oneshot service, a systemd timer will be used to run it automatically, in this example every hour.

Create a file for the timer definition:

```shell
$ sudo nano /etc/systemd/system/photoprism-bg.timer
```

Add the following contents:

```
[Unit]
Description=PhotoPrism background tasks

[Timer]
OnCalendar=*:0:0

[Install]
WantedBy=timers.target
```

If you want to run the timer more (or less) frequently, change the `OnCalendar` parameter accordingly. You can use [`systemd-analyze calendar`](https://www.freedesktop.org/software/systemd/man/systemd-analyze.html#systemd-analyze%20calendar%20EXPRESSION...) to verify the syntax.

Run the following commands to enable and start the timer:

```shell
$ sudo systemctl enable photoprism-bg.timer
$ sudo systemctl start photoprism-bg.timer
```

For troubleshooting, run the following command to check the service status:

```shell
$ systemctl status photoprism-bg.service
```

And run the following commands to check the timer status:

```shell
$ systemctl status photoprism-bg.timer
$ systemctl list-timers photoprism-bg
```

## Updating PhotoPrism

New versions of PhotoPrism are published to their Github repository: https://github.com/photoprism/photoprism/releases

If a new version is published, the following steps need to be done.

First, update your system:

```shell
$ sudo apt update
$ sudo apt upgrade
```

Also check for new Golang version and install manually using [the instructions from above](#golang).

Then, stop the PhotoPrism service:

```shell
$ sudo systemctl stop photoprism
```

Then, navigate to the PhotoPrism source code directory and pull the latest version from Github:

```shell
$ cd photoprism
$ git pull --force
```

Upgrade dependencies and re-run the build steps from above.

```shell
$ sudo make upgrade
$ sudo make all
$ sudo ./scripts/build.sh prod /opt/photoprism/bin/photoprism
$ sudo rm -rf /opt/photoprism/assets/
$ sudo cp -a assets/ /opt/photoprism/assets/
$ sudo chown -R photoprism:photoprism /opt/photoprism
```

Finally, restart the PhotoPrism service:

```shell
$ sudo systemctl start photoprism
```

## Troubleshooting

Run the following command to check the service status:

```shell
$ systemctl status photoprism
```

Also check the [PhotoPrism troubleshooting checklists](https://docs.photoprism.app/getting-started/troubleshooting/). Some of the information there is Docker-specific, but a lot is useful even with non-Docker setups.

If all else fails, you can try deleting `~/photoprism` (where you cloned the source code) and `/opt/photoprism` (where the built files were copied) and re-installing PhotoPrism. As long as you don't delete `/var/lib/photoprism`, your data and settings won't be lost.

In some cases you might run into a bug in PhotoPrism that has been fixed but the fix is not yet released. In that case, you can try switching to the "preview" branch (or even the bleeding-edge "develop" branch), although note that this might introduce other issues compared to the stable "release" version.

To switch branches (e.g. to "preview"), go to the directory where you downloaded the PhotoPrism source code and enter the following command:

```shell
$ git checkout preview
```

Then re-run the steps to build and install PhotoPrism.
