# ReLVHDoISCSISR

This Xenserver supplimental pack allows the functionality to resignature a
given LVMoISCSI SR which allows the SR to be reattached to the pool. This pack
is targeted towards Xenserver 6.5


## Why? 

The primary objective of this project was to enable the ability to do QoS per
VM.  This leveraged the QoS capabilities provided by storages like
SolidFire/Equalogic etc.  This QoS capability can be achieved by assigning a
single LUN to a VDI. This LUN would be introduced to Xenserver as a single SR.
So essentially, we are doing a single VDI per LUN.

The main drawback of this approach is that users cannot leverage the
fast-snapshot/clone functionality offered by the backend storages as any
snapshot taken on the backend storage is unaware of the underlying VDI(s) and
metadata, and would copy the LUN block-by-block. Any attemt to attach this
cloned LUN onto Xenserver will result in an error as there will be metadata
conflicts.

## Overview of the resignature approach

The plugin provided here will resignature the UUIDs during a create/attach of
the cloned SR so there are no UUID conflicts. The resignature process is
invoked when an `sr-create` is called on a LUN with a single VDI with the
`device-config:resign=true` flag . The following steps are involved:

1. **LVM Resign:** The first step is to resignatre the LVM volumes, We generate
   unique ids for the LVM physcial volume, volume group and logical volumes
   present in the LVM volume group.

1. **SR Metadata Resign:** Once the LVM resignature is complete, we activate
   the new volume group and resignature the SR metadata stored in the `MGT`
   logical volume of the volume group.

1. **VDI Resign:** Finally, we resignature the VDIs which are present in a
   logical volume. If the VDI has any parent locators, we make sure that they
   point to the correct ones since they will change during the resign process.  We
   also delete any snapshots if they are present on the SR as we want the clone
   to only contain the active VDI. 

1. **SR Creation:** Once all entities have been resignatured, we proceed to
   attach this LUN as an `LVMoISCSI` SR.


## Configuration 

We have added a `resign=(true|false)` in the `device-config` when `sr-create`
is invoked. If no `resign` option is provided, the default is assumed to be
`false` in which case Xenserver will reformat and create an empty SR. We have
also added a global `resign=(true|false)` in `host-config:other-config`which
when set will try to do a resignature by default on an LVMoISCSI `sr-create`. 

## Building 

Setup a DDK VM and clone this repo there ([Instructions for setting up a DDK VM.](http://support.citrix.com/servlet/KbServlet/download/38324-102-714674/XenServer-6.5.0_Supplemental%20Packs%20and%20the%20DDK%20Guide.pdf))

``` bash
# git clone https://github.com/cloudops/ReLVHDoISCSISR.git
# cd ReLVHDoISCSISR
# make
```

This will generate `ReLVHDoISCSISR.iso` which contains the supplimental pack.
Install it by attaching the ISO to Xenserver DOM0 and running the install
script present in it. 

## Testing

We have tested this on Xenserver 6.5 with Soldfire and Equalogic as the storage
backends. 

## Example 

```bash
xe sr-create name-label=syed-single-clone type=lvmoiscsi \
                device-config:target=172.31.255.200 \
                device-config:targetIQN=$IQN  \
                device-config:SCSIid=$SCSIid \
                device-config:resign=true \
                shared=true 
```