Volatility Framework: bitlocker
===============================

This plugin finds and extracts Full Volume Encryption Key (FVEK) from memory dumps and/or hibernation files. This allows rapid unlocking of systems that had BitLocker encrypted volumes mounted at the time of acquisition.

Supported memory images:
- Windows 10 (*work in progress*)
- Windows 8.1
- Windows Server 2012 R2
- Windows 8
- Windows Server 2012
- Windows 7
- Windows Server 2008 R2
- Windows Server 2008
- Windows Vista


Example case - Windows 7 SP1 x64
--------------------------------

*Evidence: Raw HDD image*

**1) Determine partition layout and identify BitLocker volume**

```console
elceef@cerebellum:~$ fdisk -l john_win7_x64.dd
Disk john_win7_x64.dd: 298.1 GiB, 320072933376 bytes, 625142448 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x51c47769

Device                    Boot     Start       End   Sectors   Size Id Type
john_win7_x64.dd1 *         2048   1050623   1048576   512M  7 HPFS/NTFS/exFAT
john_win7_x64.dd2        1050624 316475391 315424768 150.4G  7 HPFS/NTFS/exFAT
john_win7_x64.dd3      316475392 625137663 308662272 147.2G  7 HPFS/NTFS/exFAT
```

The last one starting from sector 316475392 is BitLocker protected. It can be verified by lookig at the filesystem header. Volumes encrypted with BitLocker will have a different signature than the standard NTFS header. A BitLocker encrypted volume starts with the "-FVE-FS-" signature.

```console
elceef@cerebellum:~$ hexdump -C -s $((512*316475392)) -n 16 john_win7_x64.dd
25ba100000  eb 58 90 2d 46 56 45 2d  46 53 2d 00 02 08 00 00  |.X.-FVE-FS-.....|
```

**2) Locate and convert hibernation file**

Mount the system volume starting from sector 1050624 in read-only mode.

```console
elceef@cerebellum:~$ sudo mount -o loop,ro,offset=$((512*1050624)) john_win7_x64.dd /mnt/1
```

Convert hibernation file *hiberfil.sys* for further forensic analysis.

```console
elceef@cerebellum:~$ vol -f /mnt/1/hiberfil.sys --profile Win7SP1x64 imagecopy -O hiberfil.raw
```

**3) Use the bitlocker plugin to extract FVEK**

The plugin scans the memory image for BitLocker cryptographic allocations (memory pools) and extracts AES keys (FVEK).

```console
elceef@cerebellum:~$ vol -f hiberfil.raw --profile Win7SP1x64 bitlocker
Volatility Foundation Volatility Framework 2.5

Address : 0xfa8009958c10
Cipher  : AES-256
FVEK    : d5b6e71adb0c2e2d38dafdcedade8fc11e8be631b9fed5b2ba5b51ba32a57cd1
TWEAK   : 49f9ecd5ddffcae44cde7f7a578b9a3ca5e79087826779e147de89423ebdf3f3

```

**4) Decrypt and access the volume**

Decrypt the volume on-the-fly using previously extracted FVEK.

```console
elceef@cerebellum:~$ sudo bdemount -k d5b6e71adb0c2e2d38dafdcedade8fc11e8be631b9fed5b2ba5b51ba32a57cd1:49f9ecd5ddffcae44cde7f7a578b9a3ca5e79087826779e147de89423ebdf3f3 -o $((512*316475392)) john_win7_x64.dd /crypt/1
```

Finally mount and access the filesystem.

```console
elceef@cerebellum:~$ sudo mount -o loop,ro /crypt/1/bde1 /mnt/2
elceef@cerebellum:~$ ls /mnt/2
CONFIDENTIAL
```


Example case - Windows 8.1 x86
------------------------------

*Evidence: Raw memory image*

Windows 8 and newer versions use Cryptography API: Next Generation (CNG) which creates a lot of dynamically allocated memory pools. For this reason, the keys are often located in several places in the memory.

```console
elceef@cerebellum:~$ vol -f john_win81_x86.raw --profile Win81U1x86 bitlocker
Volatility Foundation Volatility Framework 2.5

Address : 0x872db068
Cipher  : AES-128
FVEK    : 48286dcd34d3ff215d705d68c5df4f08

Address : 0x9ef55b08
Cipher  : AES-128
FVEK    : 48286dcd34d3ff215d705d68c5df4f08

Address : 0xa4748b08
Cipher  : AES-128
FVEK    : 48286dcd34d3ff215d705d68c5df4f08

```


Contact
-------

To send questions, comments or a chocolate, just drop an e-mail at
[marcin@ulikowski.pl](mailto:marcin@ulikowski.pl)

You can also reach me via:

- Twitter: [@elceef](https://twitter.com/elceef)
- LinkedIn: [Marcin Ulikowski](https://pl.linkedin.com/in/elceef)

