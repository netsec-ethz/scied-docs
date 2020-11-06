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
$ scmp echo -remote 64-2:0:b,[192.33.125.17] -c 3
```

Its output should look similar to this:

```
Using path:
  Hops: [64-2:0:9 2>6 64-559 10>2 64-2:0:b] MTU: 1472, NextHop: 192.168.53.34:30042
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=0 time=3.305ms
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=1 time=3.177ms
120 bytes from 64-2:0:b,[192.33.125.17] scmp_seq=2 time=2.944ms

--- 64-2:0:b,[192.33.125.17:0] statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2.015354s
```

#### Get scionFTP and Hercules

Download scionFTP and Hercules:

```bash
$ wget https://gitlab.ethz.ch/cneukom/doi-scionftp/-/raw/master/scionftp -O ./scionftp
$ wget https://gitlab.ethz.ch/cneukom/doi-scionftp/-/raw/master/hercules -O ./hercules
```

Verify the checksums:

```bash
$ md5sum scionftp hercules
e632092ae81f8b8a63a0c1e2f41f024a  scionftp
9c62819269f4a7266010911465d252e6  hercules
```

And make them executable:

```bash
$ chmod +x scionftp hercules
```

### Get started

To start the scionFTP client, you need to know your local SCION address and append a free port number.
It should then look something like `64-2:0:b,[192.33.125.17]:10001`.

If you want to use the Hercules subsystem, you need to pass Hercules' path using `-hercules`, e.g.:

```bash
$ ./scionftp -local YOUR_LOCAL_SICON_ADDRESS_WITH_PORT -hercules ./hercules
```

**Do not run** this command with root permissions, `scionftp` will invoke Hercules using `sudo`.

After starting the scionFTP client, connect to the DOI scionFTP server and log in:

```
> connect 64-2:0:b,[192.33.125.17]:2121
> login public readonly
```

Once logged in, you can use `ls`, `cd` and `pwd` to look around and `get` or `geth` to download files.

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

You can pass options to Hercules by specifying a configuration file as an optional third argument to `geth`.

Create a configuration file (e.g. `hercules-config.toml`) with the following contents:

```bash
Direction = "download"
ConfigureQueues = true
Queues = [ 1 ] # This line is optional; if you specify it, make sure to use a valid queue number
```

You can find the number of available queues with `ethtool -n INTERFACE_NAME`.

Try to download the file using the configuration, e.g.:

```
> geth datasets/sls/X02DA/Data20/e11218/disk4/ElseMarieFossil/PP43780a.tar /tmp/test /path/to/hercules-config.toml
```

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
