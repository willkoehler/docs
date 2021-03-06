# Amazon Web Services

## AWS Instance types

For the web applications I'm developing there are three instance types that make sense.

### t1.micro

* 613 MB memory
* Up to 2 EC2 Compute Units (for short periodic bursts)
* EBS storage only
* 32-bit or 64-bit platform
* I/O Performance: Low
* API name: t1.micro
* see details in AWS Micro (t1.micro) instances below

### m1.small

* 1.70 GB memory
* 1 vCPU / 1 ECU
* 160 GB instance storage
* 32-bit or 64-bit platform
* I/O Performance: Low
* API name: m1.small

### m3.medium

* 3.75 GB memory
* 1 vCPU / 3 ECU
* 1 x 4GB SSD instance storage
* 64-bit platform
* I/O Performance: Moderate
* API name: m3.medium

### c3.large

* 3.75 GB of memory
* 2 vCPU / 7 ECU
* 2 x 16GB SSD instance storage
* 64-bit platform
* I/O Performance: Moderate
* API name: c3.large

### On demand pricing:

* **t1.micro:**   $14.60/month  (as of 4/1/2014)
* **m1.small:**   $32.12/month  (as of 4/1/2014)
* **m3.medium:**  $51.10/month  (as of 4/1/2014)
* **c3.large:**   $76.65/month  (as of 4/1/2014)

### Pricing with a 3 year commitment (heavy utilization)

* **t1.micro:**    $6.42/month  (as of 4/1/2014)
* **m1.small:**   $11.79/month  (as of 4/1/2014)
* **m3.medium:**  $20.31/month  (as of 4/1/2014)
* **c3.large:**   $30.17/month  (as of 4/1/2014)

## AWS Micro (t1.micro) instances

On t1.micro instance CPU instances are allocated in bursts. When not bursting, the steady-state CPU
capacity of the instance is very low - about 2% of the burst capacity. Basically this means your instance
has access to a fair amount of CPU power for ~10 seconds at a time. If your app does anything CPU
intensive for more than ~10 seconds, your instance CPU is scaled **WAY** back - to about 2% of the
burst capacity. This effectively cripples your server until the CPU-intensive task finishes and the
load drops. But since your server is now crawling along, it may take a long time for the task to finish.
If new requests come in during this time, it just compounds the problem.

If several users visit your app simultaneously, or a bot scans your site, your CPU burst limit can be
reached. In addition, background tasks such as munin-node updates or mysqldump can exceed the CPU burst
limit causing significant slowdowns for app users.

Another effect of CPU bursting is that t1.micro instances take several hours to bootstrap with Rubber (vs
~30 minutes on a m1.small instance) because of all the CPU intensive work required. It's best to bootstrap
the instance as a c1.medium and then stop and restart the instance as t1.micro (see Changing instance types)

Greg Wilson provides a good illustration of CPU limiting in the real world in [this blog entry](http://gregsramblings.com/2011/02/07/amazon-ec2-micro-instance-cpu-steal/).
There is a good technical writeup of CPU limiting in [Amazon's official documentation](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/index.html?concepts_micro_instances.html)

## %st stat in "top"

The %st stat in "top" shows you much much time other VMs are stealing time from your CPU. "st" stands for
‘Steal Time’ and is the amount of real cpu that the Xen Hypervisor has allocated to tasks other than
running your Virtual Machine (such as somebody else’s VM...).

Amazon uses steal time to throttle your CPU back to the rate allocated for your instance, regardless
of what other VMs are running on the hardware. (i.e. Amazon doesn't let you have more CPU than you've
purchased). This is readily apparent on t1.micro instances after your CPU burst time is exceeded. In this
case steal time pegs at 98% for the remainder of the CPU intensive task, effectively crippling your instance.

    top - 21:26:59 up 4 min,  1 user,  load average: 0.82, 1.22, 0.61
    Tasks:  95 total,   1 running,  94 sleeping,   0 stopped,   0 zombie
    Cpu(s):  0.0%us,  0.3%sy,  0.0%ni, 98.3%id,  0.0%wa,  0.0%hi,  0.0%si,  1.3%st
    Mem:   1757264k total,   323224k used,  1434040k free,    13452k buffers
    Swap:   917496k total,        0k used,   917496k free,    97284k cached

## AWS mounts secondary instance storage to /mnt

All AWS instances are given secondary instance storage (160GB for m1.small instances). By default this
storage is mounted on `/mnt`. This storage is included in the price of the instance, but will go away
when the instances fails or is terminated. So it should not be used for essential assets. Because `/mnt`
is already taken, you cannot easily mount an EBS volume to `/mnt` using Rubber.

Exception: AWS does not provide secondary instance storage for t1.micro instances. In this case the `/mnt`
mount point is available for an EBS volume.

## EBS performance and MySQL data

There is mixed information on the performance of the instance (ephemeral) storage vs EBS. It appears that
EBS performance may be better on average but EBS has high variability in random seek times. If EBS
performance becomes a factor, storing the database in instance storage is an option to consider. But if
we place the database in instance storage, backups must be done with mysqldump and s3 buckets. This has
performance implications. mysqldump creates a significant load on MySQL. Plus it's not clear how we
can get a consistent snapshot of the database while it's still active. Backups generated by mysqldump
also can take a long time to restore.
