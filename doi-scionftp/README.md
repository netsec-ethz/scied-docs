# Access DOI at PSI over SCION

This guide shows how to access the [Public Data Repository at PSI](https://doi.psi.ch/) over
[SCION](https://www.scion-architecture.net/) using [scionFTP](https://github.com/elwin/scionFTP).
Thanks to the high-speed bulk data transfer system Hercules, we achieve enormous speedups when compared to the
traditional methods (see below). 
Do note, however, that this still is a proof-of-conecept and, hence, only a limited number of datasets are available
over SCION. 

## Access DOI using scionFTP

### Prerequisites

Hercules only works on Linux and, to exploit high performance features, Hercules requires root permissions.
Make sure, you have `sudo` access on your Linux machine.

Generally, SCION is available in all ETH domains, thanks to the SCI-ED project.
Ask your network/system administrator to get a SCION connection.

#### Verify your SCION Connectivity

To test your SCION connection, ping our server at PSI:
 
```bash
$ scion ping 64-2:0:b,[192.33.125.17] -c 3
```

Its output should look similar to this:

```
Resolved local address:
  (your IP address)
Using path:
  Hops: [(your path)] MTU: 1472, NextHop: (next hop IP address):30042
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=0 time=3.305ms
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=1 time=3.177ms
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=2 time=2.944ms

--- 64-2:0:b,[192.33.125.17:0] statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2.015354s
```

#### Get scionFTP and Hercules

Download scionFTP and Hercules:

```bash
$ wget https://raw.githubusercontent.com/netsec-ethz/scied-docs/master/doi-scionftp/scion-ftp -O ./scion-ftp
$ wget https://raw.githubusercontent.com/netsec-ethz/scied-docs/master/doi-scionftp/hercules -O ./hercules
```

Verify the checksums:

```bash
$ md5sum scion-ftp hercules
7e653299235156ad67294751c476f659  scion-ftp
2264fcece22504e7d41cac57663c8836  hercules
```

And make them executable:

```bash
$ chmod +x scion-ftp hercules
```

Or alternatively, you can build the binaries from the sources.

##### Building scionFTP and Hercules

To build scionFTP, checkout the [SCION Apps](https://github.com/netsec-ethz/scion-apps) repository (or
[this fork](https://github.com/cneukom/scion-apps/tree/cneukom/scionftp)) and follow the build instructions there.
The binary above has been built from commit
[7f4cca](https://github.com/cneukom/scion-apps/commit/7f4cca7645fe697eea9c5bd419069ad8ab3b3d63).

For Hercules, get the source code from the [Hercules repository](https://gitlab.inf.ethz.ch/OU-PERRIG/hercules) and
follow the build instructions there.
The binary above has been built from commit
[4564cd](https://gitlab.inf.ethz.ch/OU-PERRIG/hercules/-/commit/4564cdb9980ad66080b0875ff72f810e894511a5).


### Get started

If you want to use the Hercules subsystem, you need to pass Hercules' path using `-hercules`, e.g.:

```bash
$ ./scion-ftp -hercules ./hercules
```

**Do not run** this command with root permissions, `scion-ftp` will invoke Hercules using `sudo`.

After starting the scionFTP client, connect to the DOI scionFTP server:

```
> connect 64-2:0:b,[192.33.125.17]:2121
```

Since it supports anonymous FTP, you don't need to log in explicitly. You can use `ls`, `cd` and `pwd` to look around
and `get` or `geth` to download files.

#### Configuration for Hercules File Transfers

Most likely, `geth` will not "just work".
However, if you're lucky, it will, so go ahead and try it:

```
> geth datasets/sls/X02DA/Data20/e11218/disk4/ElseMarieFossil/PP43780a.tar /tmp/test
```

If your download does not make progress (completion stays at 0.00%) and it aborts with a message like:

```
hercules.c:rx_trickle_acks:2041: errno: 110/"Connection timed out"
transfer failed: 551 Hercules returned an error
```

This is totally expected.
The reason for this is that Hercules expects the data packets to arrive at a specific network queue and your NIC just
picked the wrong one.

So, we need to instruct Hercules and the NIC to use the same queue.
There are two approaches to this:
you can tell Hercules which queue to use and instruct it to automatically configure the NIC, using a configuration file,
or you can configure the NIC manually using `ethtool`.

##### Option 1: Use a Configuration File for Hercules

You can pass configuration options to Hercules in a `hercules.toml` file.
The ScionFTP client expects this file to be in your current working directory, in `/etc` or in `/etc/scion-ftp`.

Create a `hercules.toml` file with the following contents:

```bash
Direction = "download"
ConfigureQueues = true
Interface = "INTERFACE_NAME"
Queues = [ 1 ] # This line is optional; if you specify it, make sure to use a valid queue number
```

You can find the number of available queues with `ethtool -n INTERFACE_NAME`.

##### Option 2: Configure the NIC Manually

In some cases, Hercules will fail to configure the NIC automatically, or you might just find it inconvenient to pass the
config file to each `geth` call.

If you don't tell Hercules which queue to use, it will always use queue 0.
So, before starting the scionFTP client, set up a flow redirection rule matching the Hercules data packets and
redirecting them to queue 0.
An example looks as follows:

```bash
$ sudo ethtool -N INTERFACE_NAME flow-type udp4 dst-ip YOUR_LOCAL_IP action 0
Added rule with ID RULE_ID
``` 

This is the command Hercules uses internally if you let it configure the NIC automatically.
Hence, if automatic configuration fails, you will need to specify an alternative rule.

When you're done downloading the files, delete the flow redirection rule:

```bash
$ sudo ethtool -N INTERFACE_NAME delete RULE_ID
```

Where `RULE_ID` is the ID obtained from the previous command.

### A Note on Server Resources

For performance reasons, multiple Hercules instances can not share a network queue.
Hence, the number of parallel Hercules transfers is also limited at the server and you might get a temporary error

```
425 All Hercules units busy - please try again later
```

As Hercules transfers are expected to complete quickly, you will probably succeed if you retry after a few minutes.

### A Note on Huge Pages

If you want to download large files (> 10 GiB) at a sustainably high rate, you should use huge pages.
To configure huge pages, please refer to the
[Kernel documentation](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt).
Configure a `hugetlbfs` that is larger than the file you plan to download.
Make sure to use the largest available page size.

Then, in scionFTP, download the file into the `hugetlbfs`, e.g.:

```
> geth file_to_download.tar /dev/hugepages/file_to_download.tar hercules-config.toml
```

You can directly untar it from there:

```bash
tar -xvf /dev/hugepages/file_to_download.tar
```

## Speedups using scionFTP with Hercules

We have measured the completion time for downloading a 
[sample dataset](https://doi2.psi.ch/datasets/sls/X02DA/Data20/e11218/disk4/ElseMarieFossil/) of total size 64.5 GiB
with a client at ETHZ.

The traditional approach using `wget` took 5min 56s to complete, achieving an average transmission rate of 185.12 MiB/s.
ScionFTP took 2min 16s (485.94 MiB/s), including the time to untar the files.
Note, that it took longer to untar the files than to transmit them. 
