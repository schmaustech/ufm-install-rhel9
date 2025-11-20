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

## Disable SeLinux

UFM requires SeLinux to be disabled as per their instructions so we will set it to disabled.

~~~bash
# sed -i "s/SELINUX=.*/SELINUX=disabled/" /etc/sysconfig/selinux
~~~

## Install Software Dependencies 

## Install UFM Software

## Configure UFM

## Start UFM Services


