---
layout: post
title:  "Debian, apt and deb"
categories: [dependency-management]
---

```shell
apt-get update
apt-get install
```

When I was a newbee to Ubuntu. `apt-get` is the most powerful and magic command. With only one line of command, you could get a bunch of output and a brand new software installed on your system.Whoa! That's amazing! 

But it is easy to bump into issues once you begin to use `apt` family tools for a while. These questions came from a experience when I built a docker image for my work.So basically, here are some questions:

- What is the difference between `apt-get` and `apt`
- What did `apt-get install` do under the hood. Is `apt-get install` simply means copy the files you needed and make some `ln`?

# What is the difference between `apt-get` and `apt`?
After some source reading, I conclude that `apt` is a "front-end" to `apt-get`. The two commands eventually call the same function.

If you are calling `apt` from another script
You will get a warning like:
```
WARNING : apt does not have a stable CLI interface. Use with caution in scripts.
```
So better not write some scripts based on `apt`'s interface. Also, according some man-reading, there are some different "front-end" for apt-get, which are some alternative tools if you would like to try. Once I get used to `apt-get`, I found that the difference between `apt`'s interface and `apt-get` is actually minimal. So I stick to `apt-get` for most of the time. 

# What did `apt-get install` do under the hood

## locating the package by reading indices
We demonstrate the answer by reproducing the steps I used to build the docker image. So we begin with a bare debian buster.
```shell
docker run --rm -it debian:buster bash
```
If we try to install something by `apt install`, we will fail:
```
root@0717ec74f22d:/# apt install vim
Reading package lists... Done
Building dependency tree
Reading state information... Done
E: Unable to locate package vim
```
That means we could not find the package. Debian packages are indexed by files called `Packages Indices`. These files store the information about where to download the given package. The files is usually about 10MB, so apt need to cache them.

That's the purpose of `apt update`!

`apt update` essentially download `Packages Indices` from a `Debian Repository` by HTTP so that you could locate some information for downloading packages later. There are many different `Package Indices` in a `Debian Repository` for different debian distribution, different cpu architecture, different component... Usually we only need a few of them.

So we run `apt update`, that would download `Packages Indices` in `Packages Indices`.
```shell
apt update
```

The most important `Packages Indices` is usually for the binary packages in the main component, in our example, it's:
```shell
$ ls -lh /var/lib/apt/lists/deb.debian.org_debian_dists_buster_main_binary-amd64_Packages.lz4
-rw-r--r-- 1 root root 17M Nov 16 08:59 /var/lib/apt/lists/deb.debian.org_debian_dists_buster_main_binary-amd64_Packages.lz4
```

If we decompress it, we see it is merely a plain text file of millions lines. Every package record has a format like:

```
Package: aribas
Version: 1.64-6
Installed-Size: 488
Maintainer: Ralf Treinen <treinen@debian.org>
Architecture: amd64
Depends: libc6 (>= 2.2.5)
Description: interpreter for arithmetic
Homepage: http://www.mathematik.uni-muenchen.de/~forster/sw/aribas.html
Description-md5: 77c3b742edd36fe9a727451a0230f75f
Tag: devel::interpreter, field::mathematics, implemented-in::lisp,
 interface::commandline, role::plugin, role::program, scope::utility,
 suite::emacs
Section: math
Priority: optional
Filename: pool/main/a/aribas/aribas_1.64-6_amd64.deb
Size: 204680
MD5sum: f20a833cc6e7b15393eaa6cb5cb31660
SHA256: 96d8e7bad2eda5b4344e96a9f181ede059543b5e77c15e5229abe1664b4733ea
```

Again, we run:
```shell
apt-get install vim
```

We see that we could locate the package because we already synced the `Package Indices` so `apt-get` could read it.

## download the deb file
By reading the `Package Indices`, we can construct a URL from `Filename`, for instance:
```
http://deb.debian.org/debian/pool/main/a/aribas/aribas_1.64-6_amd64.deb
```
the second step of `apt-get` is actually just download it!

#### about the deb format
So we download a deb file. A deb file is essentially a archive. After some `ar` and `tar`, we can see the structure of a deb file:
```
.
|-- control 
|   |-- control
|   |-- md5sums
|   |-- postinst
|   |-- postrm
|   |-- preinst
|   `-- prerm
|-- data
|   `-- usr
|       |-- bin
|       |   `-- vim.basic
|       `-- share
|           |-- bug
|           |   `-- vim
|           |       |-- presubj
|           |       `-- script
|           |-- doc
|           |   `-- vim
|           |       |-- NEWS.Debian.gz
|           |       |-- changelog.Debian.gz
|           |       |-- changelog.gz
|           |       `-- copyright
|           `-- lintian
|               `-- overrides
|                   `-- vim
`-- debian-binary
```
For a brief summary:
- debian-binary usually has a value of "2.0", that is the version of the deb format
- data directory contains all the filed need to be copied to the system, so when you remove a package, you just `rm` all the files occurred in the data directory
- control/control is the meta data of the package, similar to the format existed in the Package Index file
- control/md5sums is the md5 checksum of all the files occurred in data 
- control/postinst, control/postrm, control/preinst, control/prerm are some scripts executed during the process of installing

Knowing the format of a deb package open the door of using low level debian package manager `dpkg` and its front-end `aptitude`, you can `man` to have a look at it. Generally speaking these tools are old-fashion and overkill for daily development, they are mainly for system administrators.

