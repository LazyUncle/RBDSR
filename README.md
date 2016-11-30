# RBDSR - XenServer Storage Manager plugin for CEPH
This plugin adds support of Ceph block devices into XenServer.
It supports creation of VDI as RBD device in Ceph pool. 
It uses Ceph snapshots and clones to handle VDI snapshots. It also supports Xapi Storage Migration (XSM) and XenServer High Availability (HA).

You can change the following device configs using device-config args when creating PBDs on each hosts:
- cephx-id: the cephx user id to be used. Default is admin for the client.admin user.
- rbd-mode: can be kernel, fuse or nbd. Default is nbd.

## Installation 

This plugin uses **rbd**, **rbd-nbd** add **rbd-fuse** utilities for manipulating RBD devices, so you need to install ceph-common, rbd-nbd and rbd-fuse packages from ceph repository on your XenServer hosts.

1. Make backup copy of ```CentOS-Base.repo``` and ```Citrix.repo```
 
		# cd /etc/yum.repos.d/
		# cp CentOS-Base.repo CentOS-Base.repo.orig
		# cp Citrix.repo Citrix.repo.orig

2. Make these changes in ```CentOS-Base.repo```

	In ```[base]``` section

	comment out the line
	
		baseurl=http://www.uk.xensource.com/linux/distros/CentOS/7.0.1406/os/x86_64/

	uncomment the line
	
		#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/

	In ```[updates]``` section

	comment out the line

		baseurl=http://www.uk.xensource.com/linux/distros/CentOS/7.0.1406/updates-20140901/x86_64/

	uncomment the line

		#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/

	In ```[extras]``` section

	comment out the line

		baseurl=http://www.uk.xensource.com/linux/distros/CentOS/7.0.1406/extras-20140902/x86_64/

	uncomment the line

		#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/

	replace ```$releasever``` in uncommented lines with 7.2.1511

3. Disable Citrix.repo

		[citrix]
		enabled=0

4. Create ```ceph.repo``` file and put in it next lines:

		[ceph]
		name=Ceph packages for $basearch
		baseurl=http://download.ceph.com/rpm-infernalis/el7/$basearch
		enabled=1
		priority=2
		gpgcheck=1
		type=rpm-md
		gpgkey=https://download.ceph.com/keys/release.asc
		
		[ceph-noarch]
		name=Ceph noarch packages
		baseurl=http://download.ceph.com/rpm-infernalis/el7/noarch
		enabled=1
		priority=2
		gpgcheck=1
		type=rpm-md
		gpgkey=https://download.ceph.com/keys/release.asc
		
		[ceph-source]
		name=Ceph source packages
		baseurl=http://download.ceph.com/rpm-infernalis/el7/SRPMS
		enabled=0
		priority=2
		gpgcheck=1
		type=rpm-md
		gpgkey=https://download.ceph.com/keys/release.asc

	I use Infernalis release but you can use other.
	If so then you should change the ```baseurl``` lines in ceph.repo accordingly your release

5. Install ```the release.asc``` key:

		# rpm --import 'https://download.ceph.com/keys/release.asc'

6. Install required packages:

		# yum install epel-release	
		# yum install yum-plugin-priorities.noarch
		# yum install snappy leveldb gdisk python-argparse gperftools-libs
		# yum install fuse fuse-libs
		# yum install ceph-common rbd-fuse rbd-nbd

7. Restore backup copy of ```CentOS-Base.repo``` and ```Citrix.repo```

		# cp CentOS-Base.repo.orig CentOS-Base.repo
		# cp Citrix.repo.orig Citrix.repo

8. Disable ```epel.repo```

		[epel]
		enabled=0
	
		[epel-debuginfo]
		enabled=0
	
		[epel-source]
		enabled=0

9. Disable ceph.repo

		[ceph]
		enabled=0
	
		[ceph-noarch]
		enabled=0
	
		[ceph-source]
		enabled=0

10. Put the ```waitdmmerging.sh``` into ```/usr/bin/``` and change permission: 

		# cp waitdmmerging.sh /usr/bin
		# chmod 755 /usr/bin/waitdmmerging.sh

11. Copy ```ceph_plugin.py``` into ```/etc/xapi.d/plugins``` and rename it to ```ceph_plugin``` 

		# cp ceph_plugin.py /etc/xapi.d/plugins/ceph_plugin

12. Copy the ```RBDSR.py``` and ```cephutils.py``` into  ```/opt/xensource/sm``` and compile them:

		# cp cephutils.py RBDSR.py /opt/xensource/sm
		# cd /opt/xensource/sm
		# python -m compileall RBDSR.py
		# python -O -m compileall RBDSR.py
		# python -m compileall cephutils.py
		# python -O -m compileall cephutils.py

12. Set execute permissions on all python scripts

		# chmod +x /etc/xapi.d/plugins/ceph_plugin
		# chmod +x /opt/xensource/sm/RBDSR.py
		# chmod +x /opt/xensource/sm/cephutils.py

14. Make softlink to ```RBDSR.py``` in ```/opt/xensource/sm```
	
		# ln -s RBDSR.py RBDSR 

14. Add RBDSR plugin to whitelist of SM plugins in ```/etc/xapi.conf```

		# Whitelist of SM plugins
		sm-plugins= rbd cifs ext nfs iscsi lvmoiscsi dummy file hba rawhba udev iso lvm lvmohba lvmofcoe

15. Create ```/etc/ceph/ceph.conf``` accordingly you Ceph cluster. The easiest way is just copy it from your Ceph cluster node

16. Copy ```/etc/ceph/ceph.client.admin.keyring``` to XenServer hosts from your Ceph cluster node. 

17. Restart XAPI tool-stack on XenServer hosts

		# xe-toolstack-restart 


## Usage

1. Create a pool on your Ceph cluster to store VDI images (should be executed on Ceph cluster node):

		# uuidgen
		4ceb0f8a-1539-40a4-bee2-450a025b04e1

		# ceph osd pool create RBD_XenStorage-4ceb0f8a-1539-40a4-bee2-450a025b04e1 128 128 replicated

2. Introduce the pool created in previous step as Storage Repository on XenServer hosts:

		  xe sr-introduce name-label="CEPH RBD Storage" type=rbd uuid=4ceb0f8a-1539-40a4-bee2-450a025b04e1 shared=true content-type=user
		
3. Run the ```xe host-list``` command to find out the host UUID for Xenserer host:

		# xe host-list
		uuid ( RO) : 83f2c775-57fc-457b-9f98-2b9b0a7dbcb5
		name-label ( RW): xenserver1
		name-description ( RO): Default install of XenServer

4. Create the PBD using the device SCSI ID, host UUID and SR UUID detected above:

		# xe pbd-create sr-uuid=4ceb0f8a-1539-40a4-bee2-450a025b04e1 host-uuid=83f2c775-57fc-457b-9f98-2b9b0a7dbcb5
		aec2c6fc-e1fb-0a27-2437-9862cffe213e

	If you would like to use a different cephx user or rbd mode, use the follwing device-config:
		
		# xe pbd-create sr-uuid=4ceb0f8a-1539-40a4-bee2-450a025b04e1 host-uuid=83f2c775-57fc-457b-9f98-2b9b0a7dbcb5 device-config:cephx-id=xenserver device-config:rbd-mode=kernel
		

5. Attach the PBD created with xe pbd-plug command:

		# xe pbd-plug uuid=aec2c6fc-e1fb-0a27-2437-9862cffe213e
		
	The SR should be connected to the XenServer hosts and be visible in XenCenter.
