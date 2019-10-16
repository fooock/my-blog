+++
title = "Manage libvirt with Python"
date = "2019-10-16T19:46:11+02:00"
author = "Javi"
tags = ["libvirt", "python", "kvm", "virtualization"]
showFullContent = false
description = "In this post I will show you how to install the python package to manage libvirt and connect to the libvirtd RPC interface."
draft = false
+++

If you don't know what the `libvirt` is, then the [official](https://libvirt.org/) web page can clarify some things:

>The libvirt project is a toolkit to manage virtualization platforms. It is used by many applications and support multiple hypervisors like `KVM`, `QEMU`, `Xen`, `VMWare` and [others](https://libvirt.org/drivers.html).

In this post I will show you how to interact with libvirt using Python and the `KVM` hypervisor, but first things first.

### What do you need to start?

If you are in Ubuntu like me, you need to install the `libvirt-dev` package. Using apt is easy:

```
$ sudo apt-get install libvirt-dev
```

Also, you will need [Python](https://www.python.org/) and [pip](https://pypi.org/project/pip/) installed in your system. We will create a new virtual environment using `Python 3.7`. For this purpuse I use [Conda](https://docs.conda.io/en/latest/). The next command creates a virtual environment called `libvirt`

```
$ conda create --name libvirt python=3.7
$ conda install --name libvirt pip
```

When done, you need to activate this environment using `conda activate libvirt`. Now, we will proceed to install the [libvirt-python](https://pypi.org/project/libvirt-python/) package, version `5.7.0` using pip (at this time, this is the latest release).

```
(libvirt) $ pip install libvirt-python==5.7.0
```

If all is ok, then you are ready to go.

### Connect to the KVM hypervisor driver

As I commented previously, we'll use the `KVM` hypervisor. As this hypervisor is `QEMU` compatible, the driver will be `qemu:///system`. If you want to see other options, see [this](https://libvirt.org/docs/libvirt-appdev-guide-python/en-US/html/libvirt_application_development_guide_using_python-Connections-URI_Formats.html).

>Note that this is a privileged URI.

To open a connection to the RPC server exposed by the *libvirt daemon*, it must be running. You can check it's state using `systemctl status libvirtd`. In case that the daemon is not available, you will start it using `systemctl start libvirtd`.

>For default, this daemon will only be listening for connection on a local UNIX domain socket. As it is only accessible on the local machine, it's **unencrypted**. 

There are two types of sockets:

* Full management capabilities,  `/var/run/libvirt/libvirt-sock`.
* Read only operations, `/var/run/libvirt/libvirt-sock-ro`.

We will open the full read-write socket.

```
>>> import libvirt
>>> conn = libvirt.open('qemu:///system')
>>> conn.getVersion()
3001000
```

If you want to use the read only socket, then you need to specify it when `open()` method is called:

```
>>> conn = libvirt.open('qemu:///system?socket=/var/run/libvirt/libvirt-sock-ro')
>>> conn.getVersion()
3001000
```

When no hostname is provided in the URI, by default it uses `unix` transport (same reduntant URI is `qemu+unix:///system?socket=...`).

The `conn` object is of type `libvirt.virConnect`. All connections should be closed using the `close()` method:

```
>>> conn.close()
0
>>> conn.getVersion()
libvirt: Domain Config error : invalid connection pointer in virConnectGetVersion
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/javi/miniconda3/envs/libvirt/lib/python3.7/site-packages/libvirt.py", line 4016, in getVersion
    if ret == -1: raise libvirtError ('virConnectGetVersion() failed', conn=self)
libvirt.libvirtError: invalid connection pointer in virConnectGetVersion
```

As you can see, after closing the connection, no method calls are allowed. 

Using this connection object you can define new virtual machines, list host domains, filter by name etc, but I will explain it in a new blog posts.