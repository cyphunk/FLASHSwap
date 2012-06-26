# Dumping Memory by Swapping CF FLASH
I'm exploring a method for dumping memory from an embedded device whereby I take the target FLASH chip and put it in place of the FLASH chip found in a Compact Flash card. I will give an overview of the resources and plan for this but at the moment I'd appreciate comment on an issue im having thus far.

## Status & Question
I found a Hama 4GB CF card that contained a 2GB (16Gb) Hynix 8bit FLASH ([H27UAG8T2ATR](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/hynix_hy27ua.pdf)). I have a target with another 256MB (2Gb) Hynix 8bit flash ([HY27UF082G2G](http://catalog.gaw.ru/project/download.php?id=11311)). I swapped them and now I'm having an issue when attaching under linux. The objective is to make a raw copy:

```
$ dmesg
[11647.924271] sd 19:0:0:0: Attached scsi generic sg2 type 0
[11658.096299] sd 19:0:0:0: [sdc] Attached SCSI removable disk
$ dd if=/dev/sdc of=/tmp/sdc.raw count=10
dd: opening `dev/sdc': No medium found
```

It appears the CF controller ([Phison PS3006-L](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/Phison PS3006.pdf)) was found but that either it or the linux driver do not see the CF details required to use it. Here is a comparison of the udev information for the tampered CF and an untampered CF (32MB):

```
// The tampered CF:
$ udevadm info -a -p $(udevadm info -q path -n /dev/sdc)
  looking at device '/devices/pci0000:00/0000:00:1d.7/usb1/1-2/1-2:1.0/host19/ta
rget19:0:0/19:0:0:0/block/sdc':
    KERNEL=="sdc"
    SUBSYSTEM=="block"
    DRIVER==""
    ATTR{range}=="16"
    ATTR{ext_range}=="256"
    ATTR{removable}=="1"
    ATTR{ro}=="0"
    ATTR{size}=="0"
    ATTR{alignment_offset}=="0"
    ATTR{capability}=="53"
    ATTR{stat}=="       0        0        0        0        0        0        0 
       0        0        0        0"
    ATTR{inflight}=="       0        0"
    
// The 32MB untampered CF:
$ udevadm info -a -p $(udevadm info -q path -n /dev/sdc)
  looking at device '/devices/pci0000:00/0000:00:1d.7/usb1/1-2/1-2:1.0/host18/ta
rget18:0:0/18:0:0:0/block/sdc':
    KERNEL=="sdc"
    SUBSYSTEM=="block"
    DRIVER==""
    ATTR{range}=="16"
    ATTR{ext_range}=="256"
    ATTR{removable}=="1"
    ATTR{ro}=="0"
    ATTR{size}=="62720"
    ATTR{alignment_offset}=="0"
    ATTR{capability}=="53"
    ATTR{stat}=="      21        0      168      144        0        0        0 
       0        0      144      144"
    ATTR{inflight}=="       0        0"
```

Notice the ATTR{stat} and {size} are undefined for the tampered CF but complete for the untampered CF. Apart from those lines the output is identicle. So is there a way to "tell" the driver and force the SIZE and STAT or should I start to dig into the CF controller itself and perhaps understand how it is configured on the hardware more?

For the record, here is a comparison of the messages sent to dmesg for the two:

```
// The tampered CF:
[11642.896267] scsi19 : SCSI emulation for USB Mass Storage devices
[11642.904632] usb-storage: device found at 24
[11642.904641] usb-storage: waiting for device to settle before scanning
[11647.904337] usb-storage: device scan complete
[11647.910828] scsi 19:0:0:0: Direct-Access     Generic- Multi-Card       1.00 PQ: 0 ANSI: 0 CCS
[11647.924271] sd 19:0:0:0: Attached scsi generic sg2 type 0
[11658.096299] sd 19:0:0:0: [sdc] Attached SCSI removable disk

// The 32MB untampered CF:
[10723.704979] scsi18 : SCSI emulation for USB Mass Storage devices
[10723.716213] usb-storage: device found at 23
[10723.716222] usb-storage: waiting for device to settle before scanning
[10728.716331] usb-storage: device scan complete
[10728.726672] scsi 18:0:0:0: Direct-Access     Generic- Multi-Card       1.00 PQ: 0 ANSI: 0 CCS
[10728.737444] sd 18:0:0:0: Attached scsi generic sg2 type 0
[10729.775013] sd 18:0:0:0: [sdc] 62720 512-byte logical blocks: (32.1 MB/30.6 MiB)
[10729.775850] sd 18:0:0:0: [sdc] Write Protect is off
[10729.775860] sd 18:0:0:0: [sdc] Mode Sense: 03 00 00 00
[10729.775869] sd 18:0:0:0: [sdc] Assuming drive cache: write through
[10729.785770] sd 18:0:0:0: [sdc] Assuming drive cache: write through
[10729.785789]  sdc:
[10729.800259] sd 18:0:0:0: [sdc] Assuming drive cache: write through
[10729.800278] sd 18:0:0:0: [sdc] Attached SCSI removable disk
```

The other differences to note between the 2GB and 256MB FLASH chips other that size are the Page and Block size. The 2GB has 512+16 spare page size with 16K+512 spare block size (8 bit bus width). The 265MB has 2K+64 spare page size and 12K+4K spare block size.

The [Phison PS3006-L](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/Phison PS3006.pdf) controller does not appear to have any configurable memory, if I read the datasheet correct. It appears its settings are configured by toggling pins. So this might be something to investigate to see if a simple change of state on some of the pins might allow the controller to read the swapped FLASH.

## Target Cards

### CF_cannon_32MB

- Controller: Sandisk\n S1422221\n SDTNF-256\n S228036N
- Memory: Sandisk\n 20-99-00033-1\n M230-M465072R

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/CF_cannon_32MB/DSC06409_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_cannon_32MB/DSC06409.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/CF_cannon_32MB/DSC06410_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_cannon_32MB/DSC06410.JPG)


### CF_hama_2GB_6MBs (current)

- Controller: [Phison PS3006-L](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/Phison PS3006.pdf)
- Memory: Hynix [H27UAG8T2ATR](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/hynix_hy27ua.pdf) (aka HY27U)
  - page size: 512+16 spare 8 bytes (bus width)
  - block size: 16K +512 spare bytes
  - voltage: 3.3v

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06398_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06398.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06402_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06402.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06403_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_2GB_6MBs/DSC06403.JPG)

### CF_hama_4GB_6MBs

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06394_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06394.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06396_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06396.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06397_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_hama_4GB_6MBs/DSC06397.JPG)

### CF_sandisk_4GB_30MBs

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06405_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06405.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06406_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06406.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06407_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06407.JPG)
  [![img4](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06408_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/CF_sandisk_4GB_30MBs/DSC06408.JPG)

### SD_easystore_4GB

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/SD_easystore_4GB/DSC06436_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_easystore_4GB/DSC06436.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/SD_easystore_4GB/DSC06437_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_easystore_4GB/DSC06437.JPG)

### SD_hama_8GB

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06411_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06411.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06412_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06412.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06413_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06413.JPG)
  [![img4](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06414_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06414.JPG)
  [![img5](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06415_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06415.JPG)
  [![img6](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06416_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_hama_8GB/DSC06416.JPG)


### SD_samsung_4GB

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06417_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06417.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06418_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06418.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06419_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06419.JPG)
  [![img4](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06420_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06420.JPG)
  [![img5](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06421_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06421.JPG)
  [![img6](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06422_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06422.JPG)
  [![img7](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06423_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06423.JPG)
  [![img8](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06424_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06424.JPG)
  [![img9](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06425_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06425.JPG)
  [![img10](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06426_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06426.JPG)
  [![img11](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06428_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06428.JPG)
  [![img12](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06429_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06429.JPG)
  [![img13](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06430_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06430.JPG)
  [![img14](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06431_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06431.JPG)
  [![img15](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06432_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_samsung_4GB/DSC06432.JPG)


### SD_sandisk_4GB

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06439_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06439.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06440_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06440.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06441_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sandisk_4GB/DSC06441.JPG)


### SD_sony_4GB

- Controller:
- Memory:

  [![img1](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06433_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06433.JPG)
  [![img2](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06434_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06434.JPG)
  [![img3](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06435_sm.JPG)](http://github.com/cyphunk/FLASHSwap/raw/master/SD_sony_4GB/DSC06435.JPG)
