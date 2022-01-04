[![GitHub Workflow Status](https://img.shields.io/github/workflow/status/IndrekHaav/photoprism-debian/link-check?label=link-check)](https://github.com/IndrekHaav/photoprism-debian/actions/workflows/link-check.yml)

# Installing PhotoPrism on Debian

## Background

### What is PhotoPrism?

[PhotoPrism](https://photoprism.app/) is a self-hosted web application for managing and organising a photo collection. It aims to provide many of the popular features of cloud services like Google Photos.

### Why this guide?

The [PhotoPrism documentation](https://docs.photoprism.org/getting-started/) only covers Docker as an officially supported installation method. However, not everyone can, or wants to, use Docker. The only [guide that covers installation without Docker](https://docs.photoprism.org/developer-guide/setup-fedora/) is focused on development, and includes steps that are not necessary when one simply wants to run PhotoPrism.

The purpose of this guide, therefore, is to provide instructions for setting up a usable PhotoPrism installation on [Debian](https://www.debian.org/). The guide has been written for, and tested on, Debian 11 "Bullseye", but should also work on older versions like Debian 10 "Buster", as well as derivatives like Ubuntu and Raspbian.

## DISCLAIMER

This guide is provided in good faith and for informational purposes  only. No claims are made or guarantees given that it will work on any  particular combination of hardware and software, or that it will be kept up-to-date with new releases of PhotoPrism. You will assume all  responsibility for managing your PhotoPrism server, including the  prevention of unauthorised access and safeguards against data loss.

## Installing PhotoPrism

### Prerequisites

If you haven't done so already, ensure your server's packages are up-to-date:

```shell
$ sudo apt update
$ sudo apt upgrade
```

Next, a few packages need to be installed, these are mostly various helpers for installing PhotoPrism:

```shell
$ sudo apt install -y gcc g++ git gnupg make zip unzip
```

#### Node.js

While [Node.js](https://nodejs.org/) is available in Debian (and Ubuntu) repos, the version there is pretty old. [Nodesource](https://github.com/nodesource/distributions#deb) provides up-to-date versions. The latest LTS version as of the writing of this is v16, so install that:

```shell
$ wget https://deb.nodesource.com/setup_16.x -O node_setup.sh
$ chmod +x node_setup.sh
$ sudo ./node_setup.sh
$ sudo apt update
$ sudo apt install -y nodejs
```

#### Golang

[Golang](https://golang.org/) needs to be downloaded and installed manually. The latest version as of the writing of this is 1.17.5, but check the website and change the URLs below if necessary:

```shell
$ wget https://golang.org/dl/go1.17.5.linux-amd64.tar.gz
$ sudo tar -C /usr/local -xzf go1.17.5.linux-amd64.tar.gz
$ sudo ln -s /usr/local/go/bin/go /usr/local/bin/go
```

This downloads and extracts Golang to `/usr/local/go`, and creates a symlink to the `go` binary in `/usr/local/bin` (so it's in the $PATH).

#### Tensorflow

[Tensorflow](https://www.tensorflow.org/) is an AI library developed by Google. PhotoPrism uses it to classify photos and detect faces. The necessary version (1.15, as of the writing of this) can be downloaded from the PhotoPrism website. 

Choose the best supported Tensorflow build set your CPU supports by running `lscpu | grep --color -e Flags -e avx` and look for "avx2" or "avx". If neither is present, use "cpu".

If you have [a reasonably recent CPU](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#CPUs_with_AVX2), you'll want the AVX2 version. 

*avx2 flag*
```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-avx2-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-avx2-1.15.2.tar.gz
$ sudo ldconfig
``` 

**OR**

*avx flag*
```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-avx-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-avx-1.15.2.tar.gz
$ sudo ldconfig
```

**ELSE**

```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-cpu-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-cpu-1.15.2.tar.gz
$ sudo ldconfig
```

If an unsupported version is chosen, the Photoprism service will fail to start with output "Illegal operation". To repair, just run the above steps again with the corrected archive path.

See <https://dl.photoprism.org/tensorflow> for download URLs for other platforms (like ARM).

### System setup

Instead of running Photoprism as root or your own user, it is advisable to create a separate user account for it:

```shell
$ sudo useradd --system -m -d /opt/photoprism -s /bin/bash photoprism
```

This will assume that all PhotoPrism-related files go to `/opt/photoprism`. If you would like to use another location, change it in the above and following commands.

#### Optional: storage directory

If you want to use a separate location for files like metadata, thumbnails, database (if using SQLite) and so on, create it now:

```shell
$ sudo mkdir /var/lib/photoprism
$ sudo chown photoprism:photoprism /var/lib/photoprism
```

Change the name of the directory to whatever you prefer to use.

### Download and install PhotoPrism

Now switch to the newly-added user account and download the PhotoPrism source code:

```shell
$ sudo -u photoprism -i
$ git clone https://github.com/photoprism/photoprism.git src
```

Change to the `src` directory and run the following commands to install PhotoPrism:

```shell
$ cd src
$ make dep
$ make build-js
$ make install
```

The first command downloads the various dependencies for Tensorflow, the Node.js front-end and the Golang back-end. The second command builds the front-end. The third command builds the PhotoPrism production binary, copies it to `~/.local/bin/photoprism`, and copies the front-end assets to `~/.photoprism/assets`.

This will take about 1GB of RAM, and the build may crash with Javascript running out of memory, so allocate at least 2GB or prepend the `make` commands with `NODE_OPTIONS=--max_old_space_size=2048 make build-js`.

Check the [Makefile](https://github.com/photoprism/photoprism/blob/develop/Makefile) for all `make` targets.

### Configure PhotoPrism:

Go up a directory, to `/opt/photoprism`, and create a file for PhotoPrism configuration parameters:

```shell
$ cd ..
$ nano .env
```

This opens the file in the [Nano](https://www.nano-editor.org/) text editor. Feel free to use another editor if you have a preference, but this guide will assume Nano.

The full list of configuration options is available [here](https://docs.photoprism.org/getting-started/config-options/), but you can use the following as a starting point:

```
# Initial password for the admin user
PHOTOPRISM_ADMIN_PASSWORD="photoprism"

# Locations for the Import and Originals directories
# Best to keep these on redundant storage
PHOTOPRISM_ORIGINALS_PATH="/mnt/photos/Originals"
PHOTOPRISM_IMPORT_PATH="/mnt/photos/Import"

# PhotoPrism storage directory, if you set it up above
PHOTOPRISM_STORAGE_PATH="/var/lib/photoprism"

# Uncomment below if using MariaDB/MySQL instead of SQLite (the default)
# PHOTOPRISM_DATABASE_DRIVER="mysql"
# PHOTOPRISM_DATABASE_SERVER="MYSQL_IP_HERE"
# PHOTOPRISM_DATABASE_NAME="DB_NAME"
# PHOTOPRISM_DATABASE_USER="USER_NAME"
# PHOTOPRISM_DATABASE_PASSWORD="PASSWORD"
```

Press `Ctrl+O` and `Enter` to save, then `Ctrl+X` to exit Nano. Now enter the following command:

```shell
$ chmod 640 .env
```

This ensures that the file cannot be read by other users on the system, as it contains sensitive details.

The last step is setting up a system service so PhotoPrism can run automatically in the background.

### System service

There's nothing else to do as the PhotoPrism user, so type `exit` to switch back to your own user account. Then create a file for the service definition:

```shell
$ sudo nano /etc/systemd/system/photoprism.service
```

Add the following contents:

```
[Unit]
Description=Photoprism service
After=network.target

[Service]
Type=forking
User=photoprism
Group=photoprism
WorkingDirectory=/opt/photoprism
EnvironmentFile=/opt/photoprism/.env
ExecStart=/opt/photoprism/.local/bin/photoprism up -d
ExecStop=/opt/photoprism/.local/bin/photoprism down

[Install]
WantedBy=multi-user.target
```

Now run the following commands to start the service and to have it start automatically on every boot:

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start photoprism
$ sudo systemctl enable photoprism
```

If all went well, you should be able to open `http://YOUR-IP-HERE:2342` in a web browser and see the PhotoPrism interface and log in as `admin` with the password set in the `.env` file. 

### Troubleshooting

Run the following command to check the service status:

```shell
$ systemctl status photoprism
```
