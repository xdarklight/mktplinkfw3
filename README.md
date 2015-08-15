Firmware tool for TP-Link firmwares with the version 3 header.

## Format description
The v3 header is an extension of the v2 header (see [mktplink2.c]). Images with header version 3 have a bootloader bundled, similar to v1 header images. Images with header version 3 are RSA signed. The signature is validated by the firmware update routine. The signature **is not** validated by the bootloader on boot.

It's not unlikely that devices which expect a v2 header, accept the v3 header as well, since TP-Link switched from v2 to v3 on already deployed devices like the TD-W8980v1.

```
struct fw_header {
    uint32_t     version;        /* 0x00: header version */
    char         fw_version[48]; /* 0x04: fw version string */
    uint32_t     hw_id;          /* 0x34: product id */
    uint32_t     hw_ver;         /* 0x38: product version */
    uint32_t     add_hw_ver;     /* 0x3c: additional hardware version */
    uint8_t      md5sum1[MD5SUM_LEN]; /* 0x40 */
    uint32_t     unk1;           /* 0x50: 0x00000000 */
    uint8_t      md5sum2[MD5SUM_LEN]; /* 0x54 */
    uint32_t     unk2;           /* 0x64: 0xffffffff */
    uint32_t     kernel_la;      /* 0x68: kernel load address */
    uint32_t     kernel_ep;      /* 0x6c: kernel entry point */
    uint32_t     fw_length;      /* 0x70: total length of the image */
    uint32_t     kernel_ofs;     /* 0x74: kernel data offset */
    uint32_t     kernel_len;     /* 0x78: kernel data length */
    uint32_t     rootfs_ofs;     /* 0x7c: rootfs data offset */
    uint32_t     rootfs_len;     /* 0x80: rootfs data length */
    uint32_t     boot_ofs;       /* 0x84: bootloader offset */
    uint32_t     boot_len;       /* 0x88: bootloader length */
    uint16_t     magic1;         /* 0x8c: swRevision[0] */
    uint8_t      sver_hi;        /* 0x8e: swRevision[1] */
    uint8_t      sver_lo;        /* 0x8f: swRevision[2] */
    uint8_t      magic2;         /* 0x90: platformVer[0] */
    uint8_t      ver_hi;         /* 0x91: platformVer[1] */
    uint8_t      ver_mid;        /* 0x92: platformVer[2] */
    uint8_t      ver_lo;         /* 0x93: platformVer[3] */
    uint32_t     unk3;           /* 0x94: FIXME: changing value */
    uint8_t      sig1[128];      /* 0xD0: signature (boot)+kernel+rootfs */
    uint8_t      sig2[128];      /* 0x150: unused: 0x00 */
    uint8_t      pad[48];
}
```

The value of the ```version``` field is ```0x03 0x00 0x00 0x00``` and the value of the ```fw_version``` field starts with ```ver. 2.0```.

The firmware **must** have two headers, one starting at ```0x00``` and the other one starting at ```0x020200```. All v3 firmware images have a bootloader update bundled. The bootloader is ```0xFF``` padded and starts at ```0x200```, right after the first header. The maximum size of the bootloader is 0x20000 byte (128kb). The kernel+rootfs starts at ```0x20400```, right after the second header.

Every field that isn't mentioned afterwards has the same value for both headers:

Only the ```md5sum1``` field of the first header is validated by the firmware update routine. The second header has ```0x00``` as ```md5sum1```.

The ```fw_length``` field value is the file size starting from and including the header which contains the value.

The fields ```boot_ofs``` and ```boot_len``` of the first head are evaluated to determine the flash offset to which the firmware should be written. If ```boot_ofs``` and ```boot_len``` are ```0x00 0x00 0x00 0x00```, the **entire image** - including the first header, bootloader and the second header - is written to the flash starting at ```0x20000``` (after bootloader). Otherwise the image is written to the start of the flash and overwrites the bootloader. In case the image is written to the start of the flash, the first header is stripped by the firmware update routine. That means, the ```boot_ofs``` value should normally be ```0x00 0x00 0x00 0x00```.

Maybe not directly related to the v3 header version: The kernel+roots part of the image needs to be ```0xFF``` padded till the size of header+kernel+rootfs+padding matches the size of the firmware mtd partition. The stock firmware update routine doesn't use the value of the ```fw_length``` field (any longer?) to determine the end position of the flash erase. The real filesize is used instead. Without erasing the whole to-be firmware mtd partition, the sanity checks of the rootfs splitter probably preventing the creation of the jffs2 rootfs_data partition because of existing data.

## MD5 Salt
The ```md5salt_boot``` has been changed in contrast to [mktplink2.c] \(or is wrong in [mktplink2.c]\). The actual value is

```
char md5salt_boot[MD5SUM_LEN] = {
        0x8c, 0xef, 0x33, 0x5f, 0xd5, 0xc5, 0xce, 0xfa,
        0xac, 0x9c, 0x28, 0xda, 0xb2, 0xe9, 0x0f, 0x42,
};
```

### RSA Signature
The v3 images are signed, but the content of the first header **is not** covered by the signature. That means, crossflasher can alter the ```hw_id```, ```hw_ver``` and ```add_hw_ver``` fields, recalculate the ```md5sum1``` and use the firmware to crossflash a device. The mentioned fields of the second header are not validated by the the firmware update routine.

The file ```lib/libcmm.so``` contains the base64 encoded public key of the pair. The format of the public key is "MS PUBLICKEYBLOB":

```
BgIAAACkAABSU0ExAAQAAAEAAQD54+t3X+bMvuKUfm03w6prR+S+BRjefof9XuPFVew1mftBLi4IPmBc8fb5XJXSusmDXHa/SmSaH4dvNWE5xUuvzc9p2sWxczWEvGqAi4rNk82WtKn4JUgJoalOBOwLavO2ilq4MIcBNi4bYJ6s0vU243zlgFW7p29IsA64d3LY6Q==
```

The signature is - **unconditionally/hard-coded** - read from the ```sig1``` field of the second header (```0x202D0```). That's why the second header is mandatory. Even if the signature is only read from the second header, both header of stock firmware images have the 128 bit RSA signature stored at their relative position ```0xD0```.

The signature is stored in reverse order (```0xA1 0xB2 0xC3``` => ```0xC3 0xB2 0xA1```).

Prior to calculating the to be signed hash, the value of the```sig1``` field of the second header is ```0x00```.

The contents of the signature is composed of the static hex values (salt?) ```0x30 0x21 0x30 0x09 0x06 0x05 0x2b 0x0e 0x03 0x02 0x1a 0x05 0x00 0x04 0x14``` directly followed by sha1(md5(image excluding the first header) used as hex values.

### "validating" a TP-Link signature
The following shell script extracts the signature input from official TP-Link images.

```shell
#!/bin/sh

for i in ./images/*boot* ; do
  echo "--- $i ---"

  if [ -f "$i" ]; then
    # get signature as hex string from image file (one hex
    # value per line) and reverse all lines
    sig=$(xxd -s +208 -l 128 -c 1 -plain "$i" | tac)

    # convert hex string to binary and let openssl extract
    # the signed content
    echo -n $sig | xxd -revert -plain | openssl rsautl \
              -hexdump -verify -pubin \
              -inkey tp-link_pubkey.bin.ms_publickeyblob \
              -keyform MS\ PUBLICKEYBLOB
  fi
done
```

### creating a signature
The following shell script creates a signature, signed with **your private key**.

```shell
#!/bin/bash
dd if=./openwrt-lantiq-xrx200-TDW8970-sysupgrade.image skip=512 bs=1 \
    | md5sum | awk '{ printf $1 }' | xxd -revert -plain \
    | sha1sum | awk '{ printf "3021300906052b0e03021a05000414"$1 }' | xxd -revert -plain \
    | openssl rsautl -sign -inkey ./my.ms_privatekeyblob.bin -keyform MS\ PRIVATEKEYBLOB \
    | xxd -c 1 -plain | tac | xxd -revert -plain > sig_reversed
```

# Firmware validation order

The validation of the firmware images consists of the following steps in the outlined order:

1. check ```md5sum1```
2. check ```sig1```
3. compare ```hw_id```
4. compare ```hw_ver```
5. compare ```add_hw_ver```

Due to lack of interest, it's not tracked down where the values of ```*hw*``` are stored on the device.

# What has been achieved so far
Using a TD-W8980v1 with stock firmware [TD-W8980v1_0.6.0_1.3_up_boot(131012)_2013-10-12.bin]:

- crossflash a TD-W8980v1 to [TD-W9980v1_0.6.0_1.7_up_boot(140819)_2014-08-19_14.15.39.bin] and back to [TD-W8980v1_0.6.0_1.3_up_boot(131012)_2013-10-12.bin] via stock firmware update routine

Using a TD-W8980v1 with firmware [TD-W8980v1_0.6.0_1.3_up_boot(131012)_2013-10-12.bin] + ```lib/libcmm.so``` patched with my public key:

- resign and install stock firmware via stock firmware update routine
- sign, install and boot an OpenWrt image, bundled with the stock tp-link bootloader, via stock firmware update routine (patches applied; rootfs is ```0xFF``` padded to the size of the firmware mtd partition)

# Missing parts
- the purpose of the value starting at position ```0x94``` of the header(s) is unknown
- the TP-Link private key is still required to create images which are accepted by the stock firmware update routine

There is no obvious way to get an OpenWrt image booted which doesn't bundle a bootloader, has ```boot_ofs``` and ```boot_len``` values of ```0x00```, the required second header at ```0x20200``` followed by kernel+rootfs. Changing the ```kernel_ofs``` value to ```0x00 0x02 0x04 0x00``` for both headers didn't worked as expected. I guess the ```kernel_ofs``` should be read by the bootloader to identify the kernel start address within the flash. Either the bootloader has a hard-coded kernel start address or my value is wrong.

## OpenWrt
- sysupgrade doesn't like the new header version

# References
The following files were used for analysing/verifying the file format:

* [TD-W8970v1_0.6.0_2.1_up(130415).bin]
* [TD-W8970v1_0.6.0_2.8_up_boot(130828)_2013-08-28_10.41.41.bin]
* [TD-W8970v1_0.6.0_2.12_up_boot(140613)_2014-06-13_09.17.23.bin]
* [TD-W8970B(DE)v1_0.6.0_2.9_up_boot(140722)_2014-07-22_11.04.53.bin]
* [TD-W8970B(DE)v1_0.6.0_2.10_up_boot(141008)_2014-10-08_15.49.52.bin]
* [TD-W8970B(DE)v1_0.6.0_2.11_up_boot(150526)_2015-05-26_15.16.02.bin]
* [TD-W8980v1_0.6.0_1.3_up_boot(131012)_2013-10-12.bin]
* [TD-W8980v1_0.6.0_1.7_up_boot(140619)_2014-06-19_14.30.19.bin]
* [TD-W8980v1_0.6.0_1.7_up_boot(140919)_2014-09-19_15.09.20.bin]
* [TD-W8980v1_0.6.0_1.8_up_boot(150514)_2015-05-14_11.16.43.bin]
* [TD-W8980B(DE)v1_0.6.0_2.5_up_boot(131108)_2013-11-08_10.26.11.bin]
* [TD-W8980B(DE)v1_0.6.0_2.5_up_boot(140423)_2014-04-23_14.21.47.bin]
* [TD-W9980v1_0.6.0_1.7_up_boot(140819)_2014-08-19_14.15.39.bin]
* [TD-W9980v1_0.6.0_1.8_up_boot(140925)_2014-09-25_09.12.45.bin]
* [TD-W9980v1_0.6.0_1.10_up_boot(141215)_2014-12-15_11.50.34.bin]
* [TD-W9980v1_0.6.0_1.12_up_boot(150507)_2015-05-07_11.12.40.bin]
* [TD-W9980B(DE)v1_0.6.0_2.8_up_boot(140924)_2014-09-24_10.03.20.bin]
* [TD-W9980B(DE)v1_0.6.0_2.9_up_boot(141114)_2014-11-14_14.14.32.bin]
* [TD-W9980B(DE)v1_0.6.0_2.10_up_boot(150519)_2015-05-19_16.01.15.bin]
* [Archer_D7v1_0.9.1_0.9_up_boot(141014)_2014-10-14_14.49.54.bin]
* [Archer_D7v1_0.9.1_0.11_up_boot(150324)_2015-03-24_14.05.43.bin]
* [Archer_D7v1_0.9.1_0.12_up_boot(150514)_2015-05-14_09.22.09.bin]

[mktplink2.c]: https://dev.openwrt.org/browser/trunk/tools/firmware-utils/src/mktplinkfw2.c
[TD-W8970v1_0.6.0_2.1_up(130415).bin]: http://www.tp-link.com/resources/software/TD-W8970_V1_130415.zip
[TD-W8970v1_0.6.0_2.8_up_boot(130828)_2013-08-28_10.41.41.bin]: http://www.tp-link.com/resources/software/TD-W8970_V1_130828.zip
[TD-W8970v1_0.6.0_2.12_up_boot(140613)_2014-06-13_09.17.23.bin]: http://www.tp-link.com/resources/software/TD-W8970_V1_140613.zip
[TD-W8970B(DE)v1_0.6.0_2.9_up_boot(140722)_2014-07-22_11.04.53.bin]: http://www.tp-link.de/resources/software/TD-W8970B\(DE_1.0_140722.zip
[TD-W8970B(DE)v1_0.6.0_2.10_up_boot(141008)_2014-10-08_15.49.52.bin]: http://www.tp-link.com/resources/software/TD-W8970B_V1_141008_DE.zip
[TD-W8970B(DE)v1_0.6.0_2.11_up_boot(150526)_2015-05-26_15.16.02.bin]: http://www.tp-link.de/resources/software/TD-W8970B\(DE_V1_150526.zip
[TD-W8980v1_0.6.0_1.3_up_boot(131012)_2013-10-12.bin]: http://www.tp-link.com/resources/software/TD-W8980_V1_140619.zip
[TD-W8980v1_0.6.0_1.7_up_boot(140619)_2014-06-19_14.30.19.bin]: http://www.tp-link.com/resources/software/TD-W8980_V1_140619.zip
[TD-W8980v1_0.6.0_1.7_up_boot(140919)_2014-09-19_15.09.20.bin]: http://www.tp-link.com/resources/software/TD-W8980_V1_140919.zip
[TD-W8980v1_0.6.0_1.8_up_boot(150514)_2015-05-14_11.16.43.bin]: http://www.tp-link.com/resources/software/TD-W8980_V1_150514.zip
[TD-W8980B(DE)v1_0.6.0_2.5_up_boot(131108)_2013-11-08_10.26.11.bin]: http://www.tp-link.com/resources/software/TD-W8980B_V1_131212_DE.zip
[TD-W8980B(DE)v1_0.6.0_2.5_up_boot(140423)_2014-04-23_14.21.47.bin]: http://www.tp-link.com/resources/software/TD-W8980B_V1_140423_DE.zip
[TD-W9980v1_0.6.0_1.7_up_boot(140819)_2014-08-19_14.15.39.bin]: http://www.tp-link.com/resources/software/TD-W9980_V1_140819.rar
[TD-W9980v1_0.6.0_1.8_up_boot(140925)_2014-09-25_09.12.45.bin]: http://www.tp-link.com/resources/software/TD-W9980_V1_140925.zip
[TD-W9980v1_0.6.0_1.10_up_boot(141215)_2014-12-15_11.50.34.bin]: http://www.tp-link.com/resources/software/TD-W9980_V1_141215.zip
[TD-W9980v1_0.6.0_1.12_up_boot(150507)_2015-05-07_11.12.40.bin]: http://www.tp-link.com/resources/software/TD-W9980_V1_150507.zip
[TD-W9980B(DE)v1_0.6.0_2.8_up_boot(140924)_2014-09-24_10.03.20.bin]: http://www.tp-link.com/resources/software/TD-W9980B_V1_140924_DE.zip
[TD-W9980B(DE)v1_0.6.0_2.9_up_boot(141114)_2014-11-14_14.14.32.bin]: http://www.tp-link.com/resources/software/TD-W9980B_V1_141114_DE.zip
[TD-W9980B(DE)v1_0.6.0_2.10_up_boot(150519)_2015-05-19_16.01.15.bin]: http://www.tp-link.de/resources/software/TD-W9980B\(DE_V1_150519.zip
[Archer_D7v1_0.9.1_0.9_up_boot(141014)_2014-10-14_14.49.54.bin]: http://www.tp-link.com/resources/software/Archer_D7_V1_141014.zip
[Archer_D7v1_0.9.1_0.11_up_boot(150324)_2015-03-24_14.05.43.bin]: http://www.tp-link.com/resources/software/Archer_D7_V1_150324.zip
[Archer_D7v1_0.9.1_0.12_up_boot(150514)_2015-05-14_09.22.09.bin]: http://www.tp-link.com/resources/software/Archer_D7_V1_150514.zip
