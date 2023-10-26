# Manual build instructions

If the pre-built binaries don't work on your system for any reason, it's possible to build PhotoPrism yourself by following the below instructions.

## Prerequisites

If you haven't done so already, ensure your server's packages are up-to-date:

```shell
$ sudo apt update
$ sudo apt upgrade
```

Next, a few core packages need to be installed, these are mostly various helpers for installing PhotoPrism:

```shell
$ sudo apt install -y gcc g++ git gnupg make zip unzip ffmpeg
```

> **Note:** If running in an environment where you're root by default, like in an LXC container, make sure sudo is installed, it'll be needed in a later step.

Optionally, the following packages can be installed to enable better metadata extraction (exiftool) and RAW image conversion (darktable, imagemagick):

```shell
$ sudo apt install -y exiftool darktable libpng-dev libjpeg-dev libtiff-dev imagemagick
```

### Node.js

[Debian 12 ships Node.js v18](https://packages.debian.org/bookworm/nodejs), which as of the writing of this is recent enough, so it can be installed from the default repos:

```shell
$ sudo apt install -y nodejs npm
```

For distros that ship an older version, or in the case that PhotoPrism starts requiring a newer version of Node.js, it should be installed from [Nodesource](https://github.com/nodesource/distributions#debian-and-ubuntu-based-distributions) instead:

```shell
$ wget https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key
$ sudo mv nodesource-repo.gpg.key /etc/apt/keyrings/nodesource.asc
$ echo "deb [signed-by=/etc/apt/keyrings/nodesource.asc] https://deb.nodesource.com/node_18.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
$ sudo apt update
$ sudo apt install -y nodejs
```

### Golang

While [Golang](https://go.dev/) is available in Debian (and Ubuntu) repos, the version is usually outdated (even in Debian backports). For the most up-to-date version, to keep up with what PhotoPrism uses, Golang needs to be downloaded and installed manually. The latest version as of the writing of this is 1.21.1, but [check the website](https://go.dev/dl/) and change the URLs below if necessary:

```shell
$ wget https://go.dev/dl/go1.21.1.linux-amd64.tar.gz
$ sudo rm -rf /usr/local/go
$ sudo tar -C /usr/local -xzf go1.21.1.linux-amd64.tar.gz
$ sudo ln -s /usr/local/go/bin/go /usr/local/bin/go
$ rm go1.21.1.linux-amd64.tar.gz
```

This downloads and extracts Golang to `/usr/local/go` (deleting old installation, if it exists), and creates a symlink to the `go` binary in `/usr/local/bin` (so it's in the $PATH).

> **Note:** For ARM-based devices like the Raspberry Pi, download the arm64 or armv6l version instead, depending on whether you have a 64-bit or 32-bit OS.

### Tensorflow

[Tensorflow](https://www.tensorflow.org/) is an AI library developed by Google. PhotoPrism uses it to classify photos and detect faces. The necessary version (1.15, as of the writing of this) can be downloaded from the PhotoPrism website.

Choose the appropriate Tensorflow build based on whether your CPU supports the [AVX or AVX2 instruction sets](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions). You can check by running `lscpu | grep -Eo 'avx2?\W'`.

If you have [a reasonably recent CPU](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#CPUs_with_AVX2), you'll want the "avx2" version:

```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-avx2-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-avx2-1.15.2.tar.gz
$ sudo ldconfig
$ rm libtensorflow-linux-avx2-1.15.2.tar.gz
```

For [older CPUs](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#CPUs_with_AVX), use the "avx" version:

```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-avx-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-avx-1.15.2.tar.gz
$ sudo ldconfig
$ rm libtensorflow-linux-avx-1.15.2.tar.gz
```

If your CPU supports neither instruction set, use the "cpu" version:

```shell
$ wget https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-cpu-1.15.2.tar.gz
$ sudo tar -C /usr/local -xzf libtensorflow-linux-cpu-1.15.2.tar.gz
$ sudo ldconfig
$ rm libtensorflow-linux-cpu-1.15.2.tar.gz
```

If an unsupported version is chosen, the PhotoPrism service will fail to start with an "Illegal operation" error. To repair, just run the above steps again with the corrected archive path.

See <https://dl.photoprism.org/tensorflow> for download URLs for other platforms (like ARM).

## Download and install PhotoPrism

Create a directory where the compiled PhotoPrism code will be stored:

```shell
$ sudo mkdir -p /opt/photoprism/bin
```

Now download the PhotoPrism source code:

```shell
$ git clone https://github.com/photoprism/photoprism.git
$ cd photoprism
$ git checkout release
```

Then run the following commands to download the various dependencies for Tensorflow, the Node.js front-end and the Golang back-end, and install PhotoPrism in `/opt/photoprism`:

```shell
$ sudo make all
$ sudo ./scripts/build.sh prod /opt/photoprism/bin/photoprism
$ sudo cp -a assets/ /opt/photoprism/assets/
```

The dependencies step can produce errors when running in shells like ZSH. Ensure you're using Bash if this happens.

Building the front-end can take more than 1 GB of RAM, and the build might crash with Javascript running out of memory. If using a virtual machine, allocate at least 2 GB. Alternatively, you can try limiting Node's memory usage as follows (adjust the number based on available RAM on your system):

```shell
$ NODE_OPTIONS=--max_old_space_size=1024 make all
```

If you're still having problems, consult [the PhotoPrism makefile](https://github.com/photoprism/photoprism/blob/release/Makefile#L34) for the steps that `make all` executes, and try running them individually to isolate the problem.
