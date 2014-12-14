mktplinkfw3
===========

Firmware tool for TP-Link firmwares with the version 3 header (0x03000000)


Format description
===========

Compared to the v2 file format, the v3 file format:
* Seems to use a simliar header format (see [mktplink2.c])
* There are two headers in the file, one starting at 0x0 and the other one starting at 0x20200
* The offsets given in the header starting at 0x0 do **not** point to the kernel anymore
* When cutting off everything before the second header (0x20200) then you can correct extract the kernel image (LZMA) from the file: ```
dd if=TD-W8970v1_0.6.0_2.8_up_boot\(130828\)_2013-08-28_10.41.41.bin bs=1 skip=131584 of=removed-datablock1-from-TD-W8970v1_0.6.0_2.8_up_boot.bin
```



References
===========

The following files were used for analyzing the file format:
* [TD-W8970_2013-04-15.zip] - The last firmware for the TD-W8970 with a v2 image file
* [TD-W8970_2013-08-28.zip] - The first firmware for the TD-W8970 with a v3 image file
* [ArcherD7b-2014-07-01.zip] - The firmware of another device with a v3 image file


[TD-W8970_2013-04-15.zip]: http://www.tp-link.com/resources/software/TD-W8970_V1_130415.zip
[TD-W8970_2013-08-28.zip]: http://www.tp-link.com/resources/software/TD-W8970_V1_130828.zip
[ArcherD7b-2014-07-01.zip]: http://www.tp-link.com.de/resources/software/Archer_D7b_V1_140701.zip
[mktplink2.c]: https://dev.openwrt.org/browser/trunk/tools/firmware-utils/src/mktplinkfw2.c
