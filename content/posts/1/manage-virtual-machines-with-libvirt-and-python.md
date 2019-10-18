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

Also, you will need [Python](https://www.python.org/) and [pip](https://pypi.org/project/pip/) installed in your system. We will create a new virtual environment using `Python 3.7`. For this purpose I use [Conda](https://docs.conda.io/en/latest/). The next command creates a virtual environment called `libvirt`

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

```python
>>> import libvirt
>>> conn = libvirt.open('qemu:///system')
>>> conn.getVersion()
3001000
```

If you want to use the read only socket, then you need to specify it when `open()` method is called, or use the short way, calling `openReadOnly()`:

```python
>>> conn = libvirt.open('qemu:///system?socket=/var/run/libvirt/libvirt-sock-ro')
>>> # Or call, conn = libvirt.openReadOnly('qemu:///system')
>>> conn.getVersion()
3001000
```

When no hostname is provided in the URI, by default it uses `unix` transport (same reduntant URI is `qemu+unix:///system?socket=...`).

These two methods to create a connection are deprecated in favour of the method `openAuth(driver, credentials, flags)`. Apart from the driver URI, it takes two more arguments, one with a Python list with the client credentials, and a flags parameter to allows the application request a read only or other type of connection.

I will show you how to configure `libvirt` with authentication using [`SASL`](https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer). There are several points to note:

* `SASL` doesn't require real accounts on the server. It uses its own database to store names and passwords.
* Connections using `SASL` are encrypted. The current data encryption mechanism in the new versions of `libvirt` is `GSSAPI`, but we will use `TLS`. Since we will use `SASL` an top of `TLS`, we can deactivate session encryption to avoid overhead.

Lets go. First, we need to edit the file `/etc/default/libvirtd` and add this option to start listening on TCP: `libvirtd_opts="-l"`

>You can use `-l` or `--listen`

Next, if you restart the `libvirtd` using `systemctl restart libvirtd`, it will fail. Why?. Well, by default when this option is enabled, it is necesary to set up a CA and issue server certificates.

We need to generate server certificates. For this purpose we will use `openssl`.

```bash
$ # First we create a random passphrase that is written to the passphrase file
$ echo `openssl rand -base64 8` > passphrase.txt
$ # Generate CA certificates
$ openssl genrsa -out cakey.pem -passout file:passphrase.txt 2048
$ openssl req -new -x509 -key cakey.pem -out cacert.pem -passin file:passphrase.txt
$ # Generate new CSR
$ openssl genrsa -out serverkey.pem 2048
$ openssl req -new -key serverkey.pem -out server.csr
$ # Now we create a certificate using the CA
$ openssl x509 -req -in server.csr -CA cacert.pem -CAkey cakey.pem -CAcreateserial -out servercert.pem
```

Now, we need to copy these files to the corresponding folder:

* The `cacert.pem` certificate will be copied to the `/etc/pki/CA/` folder.
* The `servercert.pem` certificate will be copied to the `/etc/pki/libvirt/` folder.
* The `serverkey.pem` private key will be vopied to the `/etc/pki/libvirt/private/` folder.

>Remember that server certificates must be readable to QEMU processes. Adjust read access to those files.

When done, you need to enable `TLS` in the `/etc/libvirt/libvirtd.conf` file. By default, secure TLS connections are enabled by default, but you can add `listen_tls = 1` to make configuration more readable. Restart `libvirtd`. Done.

To test this configuration, you need to create a new certificate for your client, and call one command using `TLS` transport in the hypervisor driver connection. We will use `virsh` for this test:

```bash
$ sudo virsh -c qemu+tls://localhost/system list --all
Id   Name   State
--------------------
```

The last step is to configure `SASL`. We need to add to the file `/etc/libvirt/libvirtd.conf` this two lines:

* `auth_tcp = "sasl"` to enable `SASL` for tcp connections.
* `auth_tls = "sasl"` to enable `SASL` for TLS connections.

Now we need to configure `SASL` users. Install the package `sasl2-bin`, from ubuntu you can:

```bash
$ sudo apt-get install sasl2-bin
```

Add a new user and restart the `libvirtd`. If you try to execute the last command to list domains in the host, the result will be:

```
$ sudo virsh -c qemu+tls://localhost/system list --all
error: failed to connect to the hypervisor
error: authentication failed: authentication failed
```

Create the file `auth.conf` in `/etc/libvirt`, and put the user and password created in the last step using `saslpasswd2`.

```toml
[credentials-sasl]
authname=
password=

[auth-libvirt-localhost]
credentials=sasl
```
>Remeber restart `libvirtd`.

We go to write a Python script to test if all works correctly.

```python
import libvirt

SASL_USER = "YOUR_USER"
SASL_PASS = "YOUR_PASSWORD"

def request_cred(credentials, user_data):
  for credential in credentials:
    if credential[0] == libvirt.VIR_CRED_AUTHNAME:
      credential[4] = SASL_USER
    elif credential[0] == libvirt.VIR_CRED_PASSPHRASE:
      credential[4] = SASL_PASS
  return 0

auth = [[libvirt.VIR_CRED_AUTHNAME, libvirt.VIR_CRED_PASSPHRASE], request_cred, None]
conn = libvirt.openAuth('qemu+tls://localhost/system', auth, 0)
print(conn.getVersion())
conn.close()
```

If you execute the script, then:

```bash
(libvirt) $ python virt.py
3001000
```

All works fine!.

### Release resources

The `conn` object is of type `libvirt.virConnect`. All connections should be closed using the `close()` method:

```python
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
