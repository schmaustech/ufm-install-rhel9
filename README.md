# NVIDIA UFM on Red Hat Enterprise Linux 9

**Goal**: The goal of this document is to provide an easy to consume procedure for install UFM on Red Hat Enterprise Linux 9.

## Workflow Sections

- [Environment](#environment)
- [Configure Repos](#configure-repos)
- [Set Firewall Rules](#set-firewall-rules)
- [Disable SeLinux](#disable-selinux)
- [Install Software Dependencies](#install-software-dependencies)
- [Install UFM Software](#install-ufm-software)
- [Configure UFM](#configure-ufm)
- [Start UFM Services](#start-ufm-services)

## Environment

The environment consists of a R760xa server with RHEL running on it.

~~~bash
# cat /etc/redhat-release 
Red Hat Enterprise Linux release 9.7 (Plow)

# uname -a
Linux nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com 5.14.0-611.8.1.el9_7.x86_64 #1 SMP PREEMPT_DYNAMIC Thu Nov 13 05:30:00 EST 2025 x86_64 x86_64 x86_64 GNU/Linux
~~~

## Configure Repos

We need to configure a few repositories on the UFM host.  Those repositories include: CodeReady Builder, EPEL, NVIDIA Doca and Docker.  First we will enable the CodeReady Builder repository (assuming RHEL host is registered and has entitlement).

~~~bash
# subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
Repository 'codeready-builder-for-rhel-9-x86_64-rpms' is enabled for this system.
~~~

Next we can enable the EPEL respository.

~~~bash
# dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
Updating Subscription Management repositories.
Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)                                                                                                                            47 MB/s |  15 MB     00:00    
epel-release-latest-9.noarch.rpm                                                                                                                                                   1.1 MB/s |  19 kB     00:00    
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                              Architecture                                   Version                                            Repository                                            Size
===================================================================================================================================================================================================================
Installing:
 epel-release                                         noarch                                         9-10.el9                                           @commandline                                          19 k

Transaction Summary
===================================================================================================================================================================================================================
Install  1 Package

Total size: 19 k
Installed size: 26 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
  Running scriptlet: epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
Many EPEL packages require the CodeReady Builder (CRB) repository.
It is recommended that you run /usr/bin/crb enable to enable the CRB repository.

  Verifying        : epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
Installed products updated.

Installed:
  epel-release-9-10.el9.noarch                                                                                                                                                                                     
Complete!
~~~

Now we need to add the NVIDIA DOCA repository.

~~~bash
# cat <<EOF > /etc/yum.repos.d/doca.repo
[doca]
name=DOCA Online Repo
baseurl=https://linux.mellanox.com/public/repo/doca/3.1.0/rhel9.6/x86_64/
enabled=1
gpgcheck=0
EOF
~~~

Finally we will enable the Docker repository which is used for Docker as a requirement around UFM plugins which run as containers.

~~~bash
# dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
Updating Subscription Management repositories.
Adding repo from: https://download.docker.com/linux/rhel/docker-ce.repo
~~~

With all the repositories added our repolist should look like the following.

~~~bash
# yum repolist
Updating Subscription Management repositories.
repo id                                                                                     repo name
codeready-builder-for-rhel-9-x86_64-rpms                                                    Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
doca                                                                                        DOCA Online Repo
docker-ce-stable                                                                            Docker CE Stable - x86_64
epel                                                                                        Extra Packages for Enterprise Linux 9 - x86_64
epel-cisco-openh264                                                                         Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64
rhel-9-for-x86_64-appstream-rpms                                                            Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms                                                               Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
~~~

## Set Firewall Rules

There are some firewall rules we need to add in order to access the UFM web interface.  Below we basically need to open up http and https ports permanently.

~~~bash
# firewall-cmd --get-active-zones
public
  interfaces: eno12399 enp55s0np0

# firewall-cmd --zone=public --add-service=http
success

# firewall-cmd --zone=public --add-service=https
success

#  firewall-cmd --permanent --zone=public --add-service=http
success

# firewall-cmd --permanent --zone=public --add-service=https
success

# firewall-cmd --reload
success

# firewall-cmd --zone=public --list-services
cockpit dhcpv6-client http https ssh
~~~

## Disable SELinux

UFM requires SeLinux to be disabled as per their instructions so we will set it to disabled.

~~~bash
# sed -i "s/SELINUX=.*/SELINUX=disabled/" /etc/sysconfig/selinux
~~~

Reboot the host for the change to take effect.

## Install Software Dependencies 

There are a variety of software packages that need to be installed as dependencies before UFM can be installed.  We will capture those here for installation.

~~~bash
# dnf install -y wget bc sshpass lftp zip rsync telnet qperf dos2unix httpd php net-snmp net-snmp-libs net-snmp-utils mod_ssl libnsl libxslt sqlite mod_session cairo apr-util-openssl net-tools docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
~~~

Start, enable and check status of Docker

~~~bash
# systemctl start docker
# systemctl enable docker
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.

# systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
     Active: active (running) since Thu 2025-11-20 17:04:39 EST; 11s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 3625 (dockerd)
      Tasks: 21
     Memory: 107.7M (peak: 110.5M)
        CPU: 103ms
     CGroup: /system.slice/docker.service
             └─3625 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Nov 20 17:04:38 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:38.602574940-05:00" level=info msg="Deleting nftables IPv6 rules" error="exit status 1"
Nov 20 17:04:38 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:38.615498854-05:00" level=info msg="Firewalld: created docker-forwarding policy"
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.151067145-05:00" level=info msg="Loading containers: done."
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.156910506-05:00" level=info msg="Docker daemon" commit=e9ff10b containerd-snapshotter=true storage-driver=overla>
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.157296220-05:00" level=info msg="Initializing buildkit"
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.161871789-05:00" level=warning msg="git source cannot be enabled: failed to find git binary: exec: \"git\": exec>
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.163207822-05:00" level=info msg="Completed buildkit initialization"
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.165944984-05:00" level=info msg="Daemon has completed initialization"
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com dockerd[3625]: time="2025-11-20T17:04:39.165978374-05:00" level=info msg="API listen on /run/docker.sock"
Nov 20 17:04:39 nvd-srv-26.nvidia.eng.rdu2.dc.redhat.com systemd[1]: Started Docker Application Container Engine.
~~~

Install the DOCA drivers required to meet requirements for UFM.

~~~bash
# dnf install doca-ufm doca-kernel
Updating Subscription Management repositories.
Last metadata expiration check: 0:17:44 ago on Thu 20 Nov 2025 04:54:18 PM EST.
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                        Architecture                   Version                                                                 Repository                                             Size
===================================================================================================================================================================================================================
Installing:
 doca-kernel                                    x86_64                         3.1.0-091000                                                            doca                                                  7.3 k
 doca-ufm                                       x86_64                         3.1.0-091000                                                            doca                                                  6.9 k
Upgrading:
 rdma-core                                      x86_64                         2507mlnx58-1.2507097                                                    doca                                                   46 k
Installing dependencies:
 ibutils2                                       x86_64                         2.1.1-0.22300.MLNX20250720.g13bb9fedb.2507097                           doca                                                  3.9 M
 infiniband-diags                               x86_64                         2507mlnx58-1.2507097                                                    doca                                                  314 k
 kernel-core                                    x86_64                         5.14.0-570.62.1.el9_6                                                   rhel-9-for-x86_64-baseos-rpms                          18 M
 kernel-modules-core                            x86_64                         5.14.0-570.62.1.el9_6                                                   rhel-9-for-x86_64-baseos-rpms                          31 M
 kmod-iser                                      x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                   43 k
 kmod-isert                                     x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                   46 k
 kmod-kernel-mft-mlnx                           x86_64                         4.33.0-1.rhel9u6                                                        doca                                                   41 k
 kmod-knem                                      x86_64                         1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6                                doca                                                   37 k
 kmod-mlnx-ofa_kernel                           x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                  1.9 M
 kmod-srp                                       x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                   62 k
 kmod-xpmem                                     x86_64                         2.7.4-1.2507097.rhel9u6.rhel9u6                                         doca                                                  492 k
 libibumad                                      x86_64                         2507mlnx58-1.2507097                                                    doca                                                   27 k
 lsof                                           x86_64                         4.94.0-3.el9                                                            rhel-9-for-x86_64-baseos-rpms                         241 k
 mlnx-ofa_kernel                                x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                   38 k
 mlnx-ofa_kernel-devel                          x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                  2.3 M
 mlnx-ofa_kernel-source                         x86_64                         25.07-OFED.25.07.0.9.7.1.rhel9u6                                        doca                                                  2.8 M
 mlnx-tools                                     x86_64                         25.07-0.2507097                                                         doca                                                   78 k
 ofed-scripts                                   x86_64                         25.07-OFED.25.07.0.9.7                                                  doca                                                   65 k
 xpmem                                          x86_64                         2.7.4-1.2507097.rhel9u6                                                 doca                                                   20 k

Transaction Summary
===================================================================================================================================================================================================================
Install  21 Packages
Upgrade   1 Package

Total download size: 61 M
Is this ok [y/N]: y
Downloading Packages:
(1/22): doca-kernel-3.1.0-091000.x86_64.rpm                                                                                                                                         21 kB/s | 7.3 kB     00:00    
(2/22): doca-ufm-3.1.0-091000.x86_64.rpm                                                                                                                                            18 kB/s | 6.9 kB     00:00    
(3/22): kmod-iser-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                                       92 kB/s |  43 kB     00:00    
(4/22): infiniband-diags-2507mlnx58-1.2507097.x86_64.rpm                                                                                                                           496 kB/s | 314 kB     00:00    
(5/22): ibutils2-2.1.1-0.22300.MLNX20250720.g13bb9fedb.2507097.x86_64.rpm                                                                                                          3.9 MB/s | 3.9 MB     00:00    
(6/22): kmod-isert-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                                      98 kB/s |  46 kB     00:00    
(7/22): kmod-kernel-mft-mlnx-4.33.0-1.rhel9u6.x86_64.rpm                                                                                                                           102 kB/s |  41 kB     00:00    
(8/22): kmod-knem-1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                               94 kB/s |  37 kB     00:00    
(9/22): kmod-srp-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                                       131 kB/s |  62 kB     00:00    
(10/22): kmod-xpmem-2.7.4-1.2507097.rhel9u6.rhel9u6.x86_64.rpm                                                                                                                     706 kB/s | 492 kB     00:00    
(11/22): kmod-mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                          2.2 MB/s | 1.9 MB     00:00    
(12/22): libibumad-2507mlnx58-1.2507097.x86_64.rpm                                                                                                                                  67 kB/s |  27 kB     00:00    
(13/22): mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                                96 kB/s |  38 kB     00:00    
(14/22): mlnx-tools-25.07-0.2507097.x86_64.rpm                                                                                                                                     164 kB/s |  78 kB     00:00    
(15/22): mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                         2.6 MB/s | 2.3 MB     00:00    
(16/22): mlnx-ofa_kernel-source-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64.rpm                                                                                                        3.1 MB/s | 2.8 MB     00:00    
(17/22): lsof-4.94.0-3.el9.x86_64.rpm                                                                                                                                              1.1 MB/s | 241 kB     00:00    
(18/22): ofed-scripts-25.07-OFED.25.07.0.9.7.x86_64.rpm                                                                                                                            139 kB/s |  65 kB     00:00    
(19/22): xpmem-2.7.4-1.2507097.rhel9u6.x86_64.rpm                                                                                                                                   50 kB/s |  20 kB     00:00    
(20/22): kernel-core-5.14.0-570.62.1.el9_6.x86_64.rpm                                                                                                                               81 MB/s |  18 MB     00:00    
(21/22): kernel-modules-core-5.14.0-570.62.1.el9_6.x86_64.rpm                                                                                                                       75 MB/s |  31 MB     00:00    
(22/22): rdma-core-2507mlnx58-1.2507097.x86_64.rpm                                                                                                                                  97 kB/s |  46 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                               16 MB/s |  61 MB     00:03     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : kernel-modules-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                         1/23 
  Installing       : kernel-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                                 2/23 
  Running scriptlet: kernel-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                                 2/23 
  Installing       : kmod-mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                             3/23 
  Running scriptlet: kmod-mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                             3/23 
  Installing       : libibumad-2507mlnx58-1.2507097.x86_64                                                                                                                                                    4/23 
  Running scriptlet: libibumad-2507mlnx58-1.2507097.x86_64                                                                                                                                                    4/23 
  Installing       : ofed-scripts-25.07-OFED.25.07.0.9.7.x86_64                                                                                                                                               5/23 
  Running scriptlet: ofed-scripts-25.07-OFED.25.07.0.9.7.x86_64                                                                                                                                               5/23 
  Installing       : mlnx-tools-25.07-0.2507097.x86_64                                                                                                                                                        6/23 
  Installing       : ibutils2-2.1.1-0.22300.MLNX20250720.g13bb9fedb.2507097.x86_64                                                                                                                            7/23 
  Installing       : infiniband-diags-2507mlnx58-1.2507097.x86_64                                                                                                                                             8/23 
  Running scriptlet: infiniband-diags-2507mlnx58-1.2507097.x86_64                                                                                                                                             8/23 
  Installing       : kmod-iser-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        9/23 
  Running scriptlet: kmod-iser-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        9/23 
  Installing       : kmod-isert-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                      10/23 
  Running scriptlet: kmod-isert-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                      10/23 
  Installing       : kmod-srp-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        11/23 
  Running scriptlet: kmod-srp-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        11/23 
  Installing       : kmod-xpmem-2.7.4-1.2507097.rhel9u6.rhel9u6.x86_64                                                                                                                                       12/23 
  Running scriptlet: kmod-xpmem-2.7.4-1.2507097.rhel9u6.rhel9u6.x86_64                                                                                                                                       12/23 
  Upgrading        : rdma-core-2507mlnx58-1.2507097.x86_64                                                                                                                                                   13/23 
  Running scriptlet: rdma-core-2507mlnx58-1.2507097.x86_64                                                                                                                                                   13/23 
  Installing       : lsof-4.94.0-3.el9.x86_64                                                                                                                                                                14/23 
  Installing       : mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                 15/23 
  Running scriptlet: mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                 15/23 
Configured /etc/security/limits.conf

  Installing       : xpmem-2.7.4-1.2507097.rhel9u6.x86_64                                                                                                                                                    16/23 
  Installing       : mlnx-ofa_kernel-source-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                          17/23 
  Installing       : mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                           18/23 
  Running scriptlet: mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                           18/23 
  Installing       : kmod-knem-1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                               19/23 
  Running scriptlet: kmod-knem-1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                               19/23 
  Installing       : kmod-kernel-mft-mlnx-4.33.0-1.rhel9u6.x86_64                                                                                                                                            20/23 
  Running scriptlet: kmod-kernel-mft-mlnx-4.33.0-1.rhel9u6.x86_64                                                                                                                                            20/23 
  Installing       : doca-kernel-3.1.0-091000.x86_64                                                                                                                                                         21/23 
  Installing       : doca-ufm-3.1.0-091000.x86_64                                                                                                                                                            22/23 
  Running scriptlet: rdma-core-57.0-2.el9.x86_64                                                                                                                                                             23/23 
  Cleanup          : rdma-core-57.0-2.el9.x86_64                                                                                                                                                             23/23 
  Running scriptlet: rdma-core-57.0-2.el9.x86_64                                                                                                                                                             23/23 
  Running scriptlet: kernel-modules-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                        23/23 
  Running scriptlet: kernel-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                                23/23 
  Running scriptlet: mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                           23/23 
  Running scriptlet: rdma-core-57.0-2.el9.x86_64                                                                                                                                                             23/23 
Failed to start jobs: Failed to enqueue some jobs, see logs for details: No such file or directory

  Verifying        : doca-kernel-3.1.0-091000.x86_64                                                                                                                                                          1/23 
  Verifying        : doca-ufm-3.1.0-091000.x86_64                                                                                                                                                             2/23 
  Verifying        : ibutils2-2.1.1-0.22300.MLNX20250720.g13bb9fedb.2507097.x86_64                                                                                                                            3/23 
  Verifying        : infiniband-diags-2507mlnx58-1.2507097.x86_64                                                                                                                                             4/23 
  Verifying        : kmod-iser-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        5/23 
  Verifying        : kmod-isert-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                       6/23 
  Verifying        : kmod-kernel-mft-mlnx-4.33.0-1.rhel9u6.x86_64                                                                                                                                             7/23 
  Verifying        : kmod-knem-1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                8/23 
  Verifying        : kmod-mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                             9/23 
  Verifying        : kmod-srp-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                        10/23 
  Verifying        : kmod-xpmem-2.7.4-1.2507097.rhel9u6.rhel9u6.x86_64                                                                                                                                       11/23 
  Verifying        : libibumad-2507mlnx58-1.2507097.x86_64                                                                                                                                                   12/23 
  Verifying        : mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                                 13/23 
  Verifying        : mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                           14/23 
  Verifying        : mlnx-ofa_kernel-source-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                                                                                                                          15/23 
  Verifying        : mlnx-tools-25.07-0.2507097.x86_64                                                                                                                                                       16/23 
  Verifying        : ofed-scripts-25.07-OFED.25.07.0.9.7.x86_64                                                                                                                                              17/23 
  Verifying        : xpmem-2.7.4-1.2507097.rhel9u6.x86_64                                                                                                                                                    18/23 
  Verifying        : lsof-4.94.0-3.el9.x86_64                                                                                                                                                                19/23 
  Verifying        : kernel-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                                20/23 
  Verifying        : kernel-modules-core-5.14.0-570.62.1.el9_6.x86_64                                                                                                                                        21/23 
  Verifying        : rdma-core-2507mlnx58-1.2507097.x86_64                                                                                                                                                   22/23 
  Verifying        : rdma-core-57.0-2.el9.x86_64                                                                                                                                                             23/23 
Installed products updated.

Upgraded:
  rdma-core-2507mlnx58-1.2507097.x86_64                                                                                                                                                                            
Installed:
  doca-kernel-3.1.0-091000.x86_64                                    doca-ufm-3.1.0-091000.x86_64                                           ibutils2-2.1.1-0.22300.MLNX20250720.g13bb9fedb.2507097.x86_64          
  infiniband-diags-2507mlnx58-1.2507097.x86_64                       kernel-core-5.14.0-570.62.1.el9_6.x86_64                               kernel-modules-core-5.14.0-570.62.1.el9_6.x86_64                       
  kmod-iser-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                  kmod-isert-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                     kmod-kernel-mft-mlnx-4.33.0-1.rhel9u6.x86_64                           
  kmod-knem-1.1.4.90mlnx3-OFED.25.07.0.9.7.1.rhel9u6.x86_64          kmod-mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64           kmod-srp-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64                       
  kmod-xpmem-2.7.4-1.2507097.rhel9u6.rhel9u6.x86_64                  libibumad-2507mlnx58-1.2507097.x86_64                                  lsof-4.94.0-3.el9.x86_64                                               
  mlnx-ofa_kernel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64            mlnx-ofa_kernel-devel-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64          mlnx-ofa_kernel-source-25.07-OFED.25.07.0.9.7.1.rhel9u6.x86_64         
  mlnx-tools-25.07-0.2507097.x86_64                                  ofed-scripts-25.07-OFED.25.07.0.9.7.x86_64                             xpmem-2.7.4-1.2507097.rhel9u6.x86_64                                   

Complete!
~~~

## Install UFM Software

## Configure UFM

~~~bash
# usermod -aG docker $USER
~~~

## Start UFM Services


