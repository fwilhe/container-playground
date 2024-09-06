
# Variants of building Debian / Garden Linux based container images

## official debian image

https://docker.debian.net

```
docker.io/library/debian                    testing-slim  0b408ca8d54e  3 days ago      108 MB
docker.io/library/debian                    testing       1992e8a88300  3 days ago      152 MB
```

## debuerreotype

https://github.com/debuerreotype/debuerreotype

Has shell scripts in [/examples](https://github.com/debuerreotype/debuerreotype/tree/master/examples).

Example usage (where `$PWD` is `debuerreotype/examples`):

```
mkdir my-debian-image
sudo ./debian.sh my-debian-image testing 2024-05-21T03:00:22Z
./oci-image.sh my-testing.tar my-debian-image/20240521/arm64/testing/
./oci-image.sh my-testing-slim.tar my-debian-image/20240521/arm64/testing/slim/
cat my-testing.tar | podman image load
cat my-testing-slim.tar | podman image load
```

```
$ podman images
localhost/arm64v8/debian                         testing-slim  14593c4b9df0  8 hours ago  108 MB
localhost/arm64v8/debian                         testing       9bda03d9d419  8 hours ago  152 MB
```

## mmdebstrap

https://gitlab.mister-muffin.de/josch/mmdebstrap

```
mmdebstrap --variant=essential testing | podman import - debian-essential-`date +%s`
```

Can use `mmtarfilter` to filter certain parts

```
mmdebstrap --variant=essential testing | mmtarfilter --path-exclude='/dev/*' --path-exclude='/usr/share/man/*'  | podman import - debian-essential-`date +%s`
```

Resulting image:

```
localhost/debian-essential-1715946830       latest      2ff5b4e151e3  2 hours ago     170 MB
```


Provided the following gardenlinux-sources.list

```
deb [trusted=yes] https://packages.gardenlinux.io/gardenlinux today main
```

You can run:

```
mmdebstrap --variant=essential < gardenlinux-sources.list | podman import - gardenlinux-essential-`date +%s`
```

Resulting image:

```
localhost/gardenlinux-essential-1715953167  latest      4ce6a509beb7  18 minutes ago  170 MB
```

```
mmdebstrap --variant=apt < gardenlinux-sources.list | podman import - gardenlinux-apt-`date +%s`
```

**FIXME**: Proper handling of gpg key


```
./diffoci-v0.1.4.linux-arm64 diff --semantic docker://localhost/debian-essential-1715946830 docker://localhost/gardenlinux-essential-1715953167
```



Experimenting with smallest feasible container:

```
mmdebstrap --variant=extract --include=base-files,bash,coreutils,hostname,libc-bin,tar,util-linux,ca-certificates testing tiny-testing
sudo tar cf ctr.tar --directory=tiny-testing/ .
sudo cat ctr.tar | sudo podman import -
```

```
mmdebstrap --variant=extract --include=base-files,bash,coreutils,hostname,libc-bin,tar,util-linux < gardenlinux-sources.list > gl-tiny.tar
mmdebstrap --variant=extract --include=base-files,bash,coreutils,hostname,libc-bin,tar,util-linux,ca-certificates < gardenlinux-sources.list | podman import - gardenlinux-exact-`date +%s`
```


## Relevant Podman Commands

The `load` and `import` command look similar, but they do different things.
Which one to use depends on the input.

[podman load](https://docs.podman.io/en/latest/markdown/podman-load.1.html) can import oci archives

This are tar files (might also end in `oci`) that contain container layers and metadata like in this example:

```
$ tar tvf .build/slimifiedContainer-arm64-1443.3-d4e015db.oci
drwxr-xr-x root/root         0 2024-05-21 13:59 blobs/
drwxr-xr-x root/root         0 2024-05-21 13:59 blobs/sha256/
-rw-r--r-- root/root  46935186 2024-05-21 13:59 blobs/sha256/f3e6fb9771ea3035359e43d2d1aca86c87e475788e7e8787c6701d21f0bcce1a
-rw-r--r-- root/root       257 2024-05-21 13:59 blobs/sha256/6e729b165081b7eea0ad8e49be01effdbd1bebc69d8506384c90cd1853815ece
-rw-r--r-- root/root       482 2024-05-21 13:59 blobs/sha256/ccc5fd0ef4bb813857649828e1b082422de822f61e4c1168e8a4e2c7fb79ea74
-rw-r--r-- root/root       233 2024-05-21 13:59 index.json
-rw-r--r-- root/root        36 2024-05-21 13:59 oci-layout
```

[podman import](https://docs.podman.io/en/latest/markdown/podman-import.1.html) can import plain root-fs directories or tarballs, i.e. something produced by debootstrap or similar tools like mmdebstrap.
