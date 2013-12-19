SMI-S iSCSI Cinder Driver for Havana
====================================

Copyright (c) 2013 EMC Corporation
All Rights Reserved

Licensed under EMC Freeware Software License Agreement
You may not use this file except in compliance with the License.
You may obtain a copy of the License at

        https://github.com/emc-openstack/freeware-eula/
        blob/master/Freeware_EULA_20131217_modified.md
        
The EMCSMISISCSIDriver is based on the existing ISCSIDriver, with the ability to create/delete and attach/detach volumes and create/delete snapshots, etc.

The EMCSMISISCSIDriver executes the volume operations by communicating with the backend EMC storage. It uses a CIM client in python called PyWBEM to make CIM operations over HTTP.

The EMC CIM Object Manager (ECOM) is packaged with the EMC SMI-S Provider. It is a CIM server that allows CIM clients to make CIM operations over HTTP, using SMI-S in the backend for EMC storage operations.

The EMC SMI-S Provider supports the SNIA Storage Management Initiative (SMI), an ANSI standard for storage management. It supports VMAX and VNX storage systems.

Requirements
------------ 

EMC SMI-S Provider V4.5.1 and higher is required. SMI-S can be downloaded from EMC's support web site.  SMI-S can be installed on a non-OpenStack host.  Supported platforms include different flavors of Windows, RedHat, and SuSE Linux.  Refer to the EMC SMI-S Provider release notes for supported platforms and installation instructions.  Note that storage arrays have to be discovered on the SMI-S server before using Cinder Driver.  Follow instructions in the release notes to discover the arrays.

SMI-S is usually installed at /opt/emc/ECIM/ECOM/bin on Linux and C:\Program Files\EMC\ECIM\ECOM\bin on Windows.  After installing and configuring SMI-S, go to that directory and type “TestSmiProvider.exe”.  After entering the test program, type “dv” and examine the output.  Make sure that the arrays are recognized by the SMI-S server before using Cinder Driver.

EMC storage VMAX Family and VNX Series are supported.

Supported Operations
--------------------

The following operations will be supported on both VMAX and VNX arrays:
* Create volume
* Delete volume
* Attach volume
* Detach volume
* Create snapshot
* Delete snapshot
* Create cloned volume
* Copy image to volume
* Copy volume to image

The following operations will be supported on VNX only:
* Create volume from snapshot
* Extend volume

Preparation
-----------

* Install python-pywbem package. For example:
$sudo apt-get install python-pywbem
* Setup SMI-S. Download SMI-S from EMC Support website and install it following the instructions of SMI-S release notes. Add your VNX/VMAX arrays to SMI-S following the SMI-S release notes.
* Register with VNX.
* Create Masking View on VMAX.

Register with VNX
-----------------

For a VNX volume to be exported to a Compute node, the node needs to be registered with VNX first.
On the Compute node 1.1.1.1, do the following (assume 10.10.61.35 is the iscsi target):
$ sudo /etc/init.d/open-iscsi start
                        
$ sudo iscsiadm -m discovery -t st -p 10.10.61.35
                        
$ cd /etc/iscsi
                        
$ sudo more initiatorname.iscsi
                        
$ iscsiadm -m node
                        
Log in to VNX from the Compute node using the target corresponding to the SPA port:
$ sudo iscsiadm -m node -T iqn.1992-04.com.emc:cx.apm01234567890.a0 -p 10.10.61.35 -l
                        
Assume iqn.1993-08.org.debian:01:1a2b3c4d5f6g is the initiator name of the Compute node. Login to Unisphere, go to VNX00000->Hosts->Initiators, Refresh and wait until initiator iqn.1993-08.org.debian:01:1a2b3c4d5f6g with SP Port A-8v0 appears.

Click the "Register" button, select "CLARiiON/VNX" and enter the host name myhost1 and IP address myhost1. Click Register. Now host 1.1.1.1 will appear under Hosts->Host List as well.
Log out of VNX on the Compute node:
$ sudo iscsiadm -m node -u
                        
Log in to VNX from the Compute node using the target corresponding to the SPB port:
$ sudo iscsiadm -m node -T iqn.1992-04.com.emc:cx.apm01234567890.b8 -p 10.10.10.11 -l
                   
In Unisphere register the initiator with the SPB port.

Log out:
$ sudo iscsiadm -m node -u

Repeat the above steps to register all SPA and SPB ports configured on VNX.
                        
Create Masking View on VMAX
---------------------------

For VMAX, user needs to do initial setup on the Unisphere for VMAX server first. On the Unisphere for VMAX server, create initiator group, storage group, port group, and put them in a masking view. Initiator group contains the initiator names of the openstack hosts. Storage group should have at least 6 gatekeepers.

Config file cinder.conf
-----------------------

Make the following changes in /etc/cinder/cinder.conf.

For VMAX, we have the following entries where 10.10.61.45 is the IP address of the VMAX iscsi target.
iscsi_target_prefix = iqn.1992-04.com.emc
iscsi_ip_address = 10.10.61.45
volume_driver = cinder.volume.drivers.emc.emc_smis_iscsi.EMCISCSIDriver
cinder_emc_config_file = /etc/cinder/cinder_emc_config.xml
                        
For VNX, we have the following entries where 10.10.61.35 is the IP address of the VNX iscsi target.
iscsi_target_prefix = iqn.2001-07.com.vnx
iscsi_ip_address = 10.10.61.35
volume_driver = cinder.volume.emc.EMCISCSIDriver
cinder_emc_config_file = /etc/cinder/cinder_emc_config.xml
                         
Restart the cinder-volume service.

Config file cinder_emc_config.xml
---------------------------------

Create the file /etc/cinder/cinder_emc_config.xml. We don't need to restart service for this change.

For VMAX, we have the following in the xml file:
<?xml version='1.0' encoding='UTF-8'?>
<EMC>
<StorageType>xxxx</StorageType>
<MaskingView>xxxx</MaskingView>
<EcomServerIp>x.x.x.x</EcomServerIp>
<EcomServerPort>xxxx</EcomServerPort>
<EcomUserName>xxxxxxxx</EcomUserName>
<EcomPassword>xxxxxxxx</EcomPassword>
</EMC>
                        
For VNX, we have the following in the xml file:
<?xml version='1.0' encoding='UTF-8'?>
<EMC>
<StorageType>xxxx</StorageType>
<EcomServerIp>x.x.x.x</EcomServerIp>
<EcomServerPort>xxxx</EcomServerPort>
<EcomUserName>xxxxxxxx</EcomUserName>
<EcomPassword>xxxxxxxx</EcomPassword>
</EMC>
                        
MaskingView is required for attaching VMAX volumes to an OpenStack VM. A Masking View can be created using Unisphere for VMAX. The Masking View needs to have an Initiator Group that contains the initiator of the OpenStack compute node that hosts the VM.

StorageType is the thin pool where user wants to create volume from.  Thin pools can be created using Unisphere for VMAX and VNX.  Note that the StorageType tag is not required any more in this Havana release.  Refer to the following "Multiple Pools and Thick/Thin Provisioning" section on how to support thick/thin provisioning.

EcomServerIp and EcomServerPort are the IP address and port number of the ECOM server which is packaged with SMI-S. EcomUserName and EcomPassword are credentials for the ECOM server.

Multiple Pools and Thick/Thin Provisioning
------------------------------------------

There’s an enhancement to support multiple pools and thick provisioning in addition to thin provisioning.  

With this enhancement, the <StorageType> tag in cinder_emc_config.xml is not mandatory any more.  There are two ways of specifying a pool:

1.	Use the <StorageType> tag as before.  In this case, pool name is specified in the <Storagetype> tag.  Only thin provisioning is supported.

2.	Use Cinder Volume Type to define a pool name and provisioning type.  The pool name is the name of a pre-created pool as before.  The provisioning type could be either thin or thick. 

Here is an example of how to use method 2.   First create volume types.  Then define extra specs for each volume type.

cinder --os-username admin --os-tenant-name admin type-create "High Performance"
cinder --os-username admin --os-tenant-name admin type-create "Standard Performance"
cinder --os-username admin --os-tenant-name admin type-key "High Performance" set storagetype:pool=smi_pool
cinder --os-username admin --os-tenant-name admin type-key "High Performance" set storagetype:provisioning=thick
cinder --os-username admin --os-tenant-name admin type-key "Standard Performance" set storagetype:pool=smi_pool
cinder --os-username admin --os-tenant-name admin type-key "Standard Performance" set storagetype:provisioning=thin

In the above example, two volume types are created.  They are “High Performance” and “Standard Performance”.   For High Performance, “storagetype:pool” is set to “smi_pool” and “storagetype:provisioning” is set to “thick”.  Similarly for Standard Performance, “storagetype:pool” is set to “smi_pool” and “storagetype:provisioning” is set to “thin”.  If “storagetype:provisioning” is not specified, it will be default to thin.

Note: Volume Type names “High Performance” and “Standard Performance” are user-defined and can be any names.  Extra spec keys “storagetype:pool” and “storagetype:provisioning” have to be the exact names listed here.  Extra spec value “smi_pool” is just your pool name.  Extra spec value for “storagetype:provisioning” has to be either “thick” or “thin”.
The driver will look for volume type first.  If volume type is specified when creating a volume, the driver will look for volume type definition and find the matching pool and provisioning type.  If volume type is not specified, it will fall back to use the <StorageType> in cinder_emc_config.xml.
