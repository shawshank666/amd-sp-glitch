## Compiling the payloads 
### Install `arm-none-eabi-` toolchain in Ubuntu 21.04
You can download it from its [official webset](https://launchpad.net/gcc-arm-embedded/+download) and unzip it under file `~/Library`(or any other new folder). Then add its path to the environment variable. Open the file `.bashrc`:
```
gedit ~/.bashrc
```
Add this statement at the end of file `.bashrc`:
```
export PATH=$PATH:/home/shawshank/Library/gcc_arm_eabi_54/bin
```
Update the path:
```
source ~/.bashrc
```
Check out whether the path has already updated:
```
echo $PATH
```
Because `arm-none-eabi-` toolchain is for 32-bit system, so binary files like `arm-none-eabi-gcc` are not executable on 64-bit Ubuntu. That's the reason why it reports `make: arm-none-eabi-gcc: No such file or directory` after `make` although its path is correctly added:
>arm-none-eabi-gcc -Os -I../include -I../Lib/include -std=gnu99 -fomit-frame-pointer -nostartfiles -ffreestanding -Wextra -Werror -c -o _start.o _start.S
make: arm-none-eabi-gcc: No such file or directory
make: *** [Makefile:10: _start.o] Error 127

To solve this problem, essential dependency needs to be installed：
```
sudo apt install libc6-i386
```
Another package also works:
```
sudo apt install lib32z1
```
Note that installing `lib32z1` will also needs `libc6-i386`.
### Make the file
Finally locate to your target payload folder and run `make`:
```
cd payloads/hello-world && make
```
>arm-none-eabi-gcc -Os -I../include -I../Lib/include -std=gnu99 -fomit-frame-pointer -nostartfiles -ffreestanding -Wextra -Werror -c -o _start.o _start.S
arm-none-eabi-ld -T linker.ld _start.o -o hello-world.elf
arm-none-eabi-objcopy -O binary hello-world.elf hello-world.raw

`hello-world.raw` is the binary file to replace **PSPOS** in UEFI image.

---
## Analyzing PSP firmware with PSPTool
### Extract PSP firmware
To replace our payloads in the UEFI flash, we need [PSPTool](https://github.com/PSPReverse/PSPTool) to extract the PSP firmware:
``` 
pip install psptool
```
I use **MSI A320M-A PRO MAX** motherboard with AMD **Ryzen 3600** CPU. The motherboard has a JSPI connector so I can read and program the UEFI SPI flash with a CH341A programmer. The chip model is MX25U25673G_1.8V. The flash memory can be read by Asprogrammer:
![](.\image/2022-04-07-20-22-18.png)
Then I save it under `payloads` named `bios_origin.bin`.
Use PSPTool to extract all PSP firmware:
```
psptool bios_origin.bin
```
```
+-----------+---------+------+-------+---------------------+
| Directory |   Addr  | Type | Magic | Secondary Directory |
+-----------+---------+------+-------+---------------------+
|     0     | 0xa8000 | PSP  |  $PSP |          --         |
+-----------+---------+------+-------+---------------------+
+---+-------+----------+---------+---------------------------------+----------+----------+-----------------------------------------+
|   | Entry |  Address |    Size |                            Type | Magic/ID |  Version |                                    Info |
+---+-------+----------+---------+---------------------------------+----------+----------+-----------------------------------------+
|   |     0 |  0xa9000 |   0x240 |              AMD_PUBLIC_KEY~0x0 |     C25D |          |                                         |
|   |     1 | 0x2c0000 |  0x798c |          PSP_FW_BOOT_LOADER~0x1 |          | 0.5.0.47 |             signed(C25D), legacy header |
|   |     2 | 0x2c8000 | 0x1320f |              SMU_OFFCHIP_FW~0x8 |          |  0.0.0.0 |               compressed, legacy header |
|   |     3 |  0xaa000 |  0x5a24 | PSP_FW_RECOVERY_BOOT_LOADER~0x3 |          | 0.5.0.47 |             signed(C25D), legacy header |
|   |     4 | 0xfff000 |  0x1000 |           BIOS_RTM_FIRMWARE~0x6 |          |          |                                         |
|   |     5 | 0x2dc000 | 0x1e04e |           PSP_FW_TRUSTED_OS~0x2 |          | 0.5.0.47 | compressed, signed(C25D), legacy header |
|   |     6 |  0x87000 | 0x10000 |                 PSP_NV_DATA~0x4 |          |          |                                         |
|   |     7 | 0x300000 | 0x13122 |       PSP_SMU_FN_FIRMWARE~0x108 |          |  0.0.0.0 |               compressed, legacy header |
|   |     8 |  0xb0000 |   0x340 |          SEC_DBG_PUBLIC_KEY~0x9 |     15F1 |          |                                         |
|   |     9 |      0x1 |     0x0 |          SOFT_FUSE_CHAIN_01~0xb |          |          |                                         |
|   |    10 |  0xb1000 |   0x340 | PSP_BOOT_TIME_TRUSTLETS_KEY~0xd |     0483 |          |                                         |
|   |    11 | 0x314000 |  0x68dc |        PSP_AGESA_RESUME_FW~0x10 |          | 0.5.0.3E | compressed, signed(C25D), legacy header |
|   |    12 | 0x31c000 |  0x4fbd |          SMU_OFF_CHIP_FW_2~0x12 |          |  0.0.0.0 |               compressed, legacy header |
|   |    13 | 0x327000 | 0x14a9d |        !PSP_MCLF_TRUSTLETS~0x14 |          |  0.0.0.0 |               compressed, legacy header |
|   |    14 | 0x347000 |  0x4e66 |        !SMU_OFF_CHIP_FW_3~0x112 |          |  0.0.0.0 |               compressed, legacy header |
|   |    15 | 0x352000 |  0x1000 |              FW_PSP_SMUSCS~0x5f |          |          |                                         |
|   |    16 | 0x353000 |  0x1000 |          !FW_PSP_SMUSCS_2~0x15f |          |          |                                         |
|   |    17 | 0x354000 |  0x3000 |             PSP_S3_NV_DATA~0x1a |          |          |                                         |
+---+-------+----------+---------+---------------------------------+----------+----------+-----------------------------------------+


+-----------+----------+------+-------+---------------------+
| Directory |   Addr   | Type | Magic | Secondary Directory |
+-----------+----------+------+-------+---------------------+
|     1     | 0x188000 | PSP  |  $PSP |       0x4f0000      |
+-----------+----------+------+-------+---------------------+
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x188400 |   0x240 |              AMD_PUBLIC_KEY~0x0 |     60BB |           |                                    |
|   |     1 | 0x4f0400 |  0xd3c0 |          PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     2 | 0x188700 |  0xd3c0 | PSP_FW_RECOVERY_BOOT_LOADER~0x3 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     3 | 0x195b00 | 0x204d0 |              SMU_OFFCHIP_FW~0x8 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     4 | 0x1b6000 |  0x4a40 |          SMU_OFF_CHIP_FW_2~0x12 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     5 | 0x1bab00 | 0x22830 |       PSP_SMU_FN_FIRMWARE~0x108 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     6 | 0x1dd400 |  0x6f80 |        !SMU_OFF_CHIP_FW_3~0x112 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     7 | 0x1e4400 |    0x10 |               WRAPPED_IKEK~0x21 |          |           |                                    |
|   |     8 | 0x1e5000 |  0x1000 |               TOKEN_UNLOCK~0x22 |          |           |                                    |
|   |     9 | 0x1e6000 |  0x1860 |                 SEC_GASKET~0x24 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    10 | 0x1e7900 |  0x1840 |                           0x124 |     $PS1 |  A.2.4.26 |  signed(60BB), verified, encrypted |
|   |    11 | 0x1e9200 |  0x1860 |                           0x224 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    12 | 0x1eab00 |   0xdd0 |                       ABL0~0x30 |     AW0B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    13 | 0x1eb900 |  0xcce0 |                       ABL1~0x31 |     AW1B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    14 | 0x1f8600 |  0x9290 |                       ABL2~0x32 |     AW2B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    15 | 0x201900 |  0xbc80 |                       ABL3~0x33 |     AW3B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    16 | 0x20d600 |  0xd410 |                       ABL4~0x34 |     AW4B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    17 | 0x21ab00 |  0xcc80 |                       ABL5~0x35 |     AW5B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    18 | 0x227800 |  0x9ff0 |                       ABL6~0x36 |     AW6B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    19 | 0x231800 |  0xcec0 |                       ABL7~0x37 |     AW7B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    20 | 0x4f0000 |   0x400 |   !PL2_SECONDARY_DIRECTORY~0x40 |          |           |                                    |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+


+-----------+----------+-----------+-------+---------------------+
| Directory |   Addr   |    Type   | Magic | Secondary Directory |
+-----------+----------+-----------+-------+---------------------+
|     2     | 0x4f0000 | secondary |  $PL2 |          --         |
+-----------+----------+-----------+-------+---------------------+
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                        Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x4f0400 |  0xd3c0 |      PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     1 | 0x4fd800 |   0x240 |          AMD_PUBLIC_KEY~0x0 |     60BB |           |                                    |
|   |     2 | 0x4fdb00 |  0xf310 |       PSP_FW_TRUSTED_OS~0x2 |     $PS1 |  0.8.0.79 |             signed(60BB), verified |
|   |     3 |  0x87000 | 0x20000 |             PSP_NV_DATA~0x4 |          |           |                                    |
|   |     4 | 0x50cf00 | 0x204d0 |          SMU_OFFCHIP_FW~0x8 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     5 | 0x52d400 |   0x340 |      SEC_DBG_PUBLIC_KEY~0x9 |     ED22 |           |                                    |
|   |     6 |      0x1 |     0x0 |      SOFT_FUSE_CHAIN_01~0xb |          |           |                                    |
|   |     7 | 0x52d800 | 0x13460 | PSP_BOOT_TIME_TRUSTLETS~0xc |     $PS1 |   0.7.0.1 | compressed, signed(60BB), verified |
|   |     8 | 0x540d00 |  0x4a40 |      SMU_OFF_CHIP_FW_2~0x12 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     9 | 0x545800 |  0x1950 |           DEBUG_UNLOCK~0x13 |     $PS1 |  0.8.0.79 | compressed, signed(60BB), verified |
|   |    10 | 0x547200 |    0x10 |           WRAPPED_IKEK~0x21 |          |           |                                    |
|   |    11 | 0x548000 |  0x1000 |           TOKEN_UNLOCK~0x22 |          |           |                                    |
|   |    12 | 0x549000 |  0x1860 |             SEC_GASKET~0x24 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    13 | 0x54a900 |  0x1840 |                       0x124 |     $PS1 |  A.2.4.26 |  signed(60BB), verified, encrypted |
|   |    14 | 0x54c200 |  0x1860 |                       0x224 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    15 | 0x54db00 |  0x22f1 |                 MP2_FW~0x25 |          |  3.18.0.1 |                       signed(76E9) |
|   |    16 | 0x54fe00 |  0x3aa4 |                       0x125 |          |   5.2.2.1 |                       signed(76E9) |
|   |    17 | 0x553900 |  0x23e4 |                       0x225 |          |   4.2.1.1 |                       signed(76E9) |
|   |    18 | 0x555d00 | 0x1b790 |         DRIVER_ENTRIES~0x28 |     $PS1 |  0.8.0.79 |             signed(60BB), verified |
|   |    19 | 0x571500 |   0xdd0 |                   ABL0~0x30 |     AW0B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    20 | 0x572300 |  0xcce0 |                   ABL1~0x31 |     AW1B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    21 | 0x57f000 |  0x9290 |                   ABL2~0x32 |     AW2B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    22 | 0x588300 |  0xbc80 |                   ABL3~0x33 |     AW3B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    23 | 0x594000 |  0xd410 |                   ABL4~0x34 |     AW4B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    24 | 0x5a1500 |  0xcc80 |                   ABL5~0x35 |     AW5B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    25 | 0x5ae200 |  0x9ff0 |                   ABL6~0x36 |     AW6B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    26 | 0x5b8200 |  0xcec0 |                   ABL7~0x37 |     AW7B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    27 | 0x5c5100 | 0x22830 |   PSP_SMU_FN_FIRMWARE~0x108 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    28 | 0x5e7a00 |  0x6f80 |    !SMU_OFF_CHIP_FW_3~0x112 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    29 | 0x5eea00 | 0x22c20 |                       0x208 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    30 | 0x611700 |  0x75b0 |                       0x212 |          |   0.0.0.0 | compressed, signed(60BB), verified |
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+


+-----------+----------+------+-------+---------------------+
| Directory |   Addr   | Type | Magic | Secondary Directory |
+-----------+----------+------+-------+---------------------+
|     3     | 0x188000 | PSP  |  $PSP |       0x4f0000      |
+-----------+----------+------+-------+---------------------+
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x188400 |   0x240 |              AMD_PUBLIC_KEY~0x0 |     60BB |           |                                    |
|   |     1 | 0x4f0400 |  0xd3c0 |          PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     2 | 0x188700 |  0xd3c0 | PSP_FW_RECOVERY_BOOT_LOADER~0x3 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     3 | 0x195b00 | 0x204d0 |              SMU_OFFCHIP_FW~0x8 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     4 | 0x1b6000 |  0x4a40 |          SMU_OFF_CHIP_FW_2~0x12 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     5 | 0x1bab00 | 0x22830 |       PSP_SMU_FN_FIRMWARE~0x108 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     6 | 0x1dd400 |  0x6f80 |        !SMU_OFF_CHIP_FW_3~0x112 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     7 | 0x1e4400 |    0x10 |               WRAPPED_IKEK~0x21 |          |           |                                    |
|   |     8 | 0x1e5000 |  0x1000 |               TOKEN_UNLOCK~0x22 |          |           |                                    |
|   |     9 | 0x1e6000 |  0x1860 |                 SEC_GASKET~0x24 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    10 | 0x1e7900 |  0x1840 |                           0x124 |     $PS1 |  A.2.4.26 |  signed(60BB), verified, encrypted |
|   |    11 | 0x1e9200 |  0x1860 |                           0x224 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    12 | 0x1eab00 |   0xdd0 |                       ABL0~0x30 |     AW0B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    13 | 0x1eb900 |  0xcce0 |                       ABL1~0x31 |     AW1B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    14 | 0x1f8600 |  0x9290 |                       ABL2~0x32 |     AW2B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    15 | 0x201900 |  0xbc80 |                       ABL3~0x33 |     AW3B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    16 | 0x20d600 |  0xd410 |                       ABL4~0x34 |     AW4B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    17 | 0x21ab00 |  0xcc80 |                       ABL5~0x35 |     AW5B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    18 | 0x227800 |  0x9ff0 |                       ABL6~0x36 |     AW6B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    19 | 0x231800 |  0xcec0 |                       ABL7~0x37 |     AW7B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    20 | 0x4f0000 |   0x400 |   !PL2_SECONDARY_DIRECTORY~0x40 |          |           |                                    |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+


+-----------+----------+-----------+-------+---------------------+
| Directory |   Addr   |    Type   | Magic | Secondary Directory |
+-----------+----------+-----------+-------+---------------------+
|     4     | 0x4f0000 | secondary |  $PL2 |          --         |
+-----------+----------+-----------+-------+---------------------+
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                        Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x4f0400 |  0xd3c0 |      PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.8.0.79 |  signed(60BB), verified, encrypted |
|   |     1 | 0x4fd800 |   0x240 |          AMD_PUBLIC_KEY~0x0 |     60BB |           |                                    |
|   |     2 | 0x4fdb00 |  0xf310 |       PSP_FW_TRUSTED_OS~0x2 |     $PS1 |  0.8.0.79 |             signed(60BB), verified |
|   |     3 |  0x87000 | 0x20000 |             PSP_NV_DATA~0x4 |          |           |                                    |
|   |     4 | 0x50cf00 | 0x204d0 |          SMU_OFFCHIP_FW~0x8 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     5 | 0x52d400 |   0x340 |      SEC_DBG_PUBLIC_KEY~0x9 |     ED22 |           |                                    |
|   |     6 |      0x1 |     0x0 |      SOFT_FUSE_CHAIN_01~0xb |          |           |                                    |
|   |     7 | 0x52d800 | 0x13460 | PSP_BOOT_TIME_TRUSTLETS~0xc |     $PS1 |   0.7.0.1 | compressed, signed(60BB), verified |
|   |     8 | 0x540d00 |  0x4a40 |      SMU_OFF_CHIP_FW_2~0x12 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |     9 | 0x545800 |  0x1950 |           DEBUG_UNLOCK~0x13 |     $PS1 |  0.8.0.79 | compressed, signed(60BB), verified |
|   |    10 | 0x547200 |    0x10 |           WRAPPED_IKEK~0x21 |          |           |                                    |
|   |    11 | 0x548000 |  0x1000 |           TOKEN_UNLOCK~0x22 |          |           |                                    |
|   |    12 | 0x549000 |  0x1860 |             SEC_GASKET~0x24 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    13 | 0x54a900 |  0x1840 |                       0x124 |     $PS1 |  A.2.4.26 |  signed(60BB), verified, encrypted |
|   |    14 | 0x54c200 |  0x1860 |                       0x224 |     $PS1 |  A.2.3.27 |  signed(60BB), verified, encrypted |
|   |    15 | 0x54db00 |  0x22f1 |                 MP2_FW~0x25 |          |  3.18.0.1 |                       signed(76E9) |
|   |    16 | 0x54fe00 |  0x3aa4 |                       0x125 |          |   5.2.2.1 |                       signed(76E9) |
|   |    17 | 0x553900 |  0x23e4 |                       0x225 |          |   4.2.1.1 |                       signed(76E9) |
|   |    18 | 0x555d00 | 0x1b790 |         DRIVER_ENTRIES~0x28 |     $PS1 |  0.8.0.79 |             signed(60BB), verified |
|   |    19 | 0x571500 |   0xdd0 |                   ABL0~0x30 |     AW0B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    20 | 0x572300 |  0xcce0 |                   ABL1~0x31 |     AW1B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    21 | 0x57f000 |  0x9290 |                   ABL2~0x32 |     AW2B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    22 | 0x588300 |  0xbc80 |                   ABL3~0x33 |     AW3B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    23 | 0x594000 |  0xd410 |                   ABL4~0x34 |     AW4B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    24 | 0x5a1500 |  0xcc80 |                   ABL5~0x35 |     AW5B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    25 | 0x5ae200 |  0x9ff0 |                   ABL6~0x36 |     AW6B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    26 | 0x5b8200 |  0xcec0 |                   ABL7~0x37 |     AW7B | 19.8.12.0 | compressed, signed(60BB), verified |
|   |    27 | 0x5c5100 | 0x22830 |   PSP_SMU_FN_FIRMWARE~0x108 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    28 | 0x5e7a00 |  0x6f80 |    !SMU_OFF_CHIP_FW_3~0x112 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    29 | 0x5eea00 | 0x22c20 |                       0x208 |          |   0.0.0.0 | compressed, signed(60BB), verified |
|   |    30 | 0x611700 |  0x75b0 |                       0x212 |          |   0.0.0.0 | compressed, signed(60BB), verified |
+---+-------+----------+---------+-----------------------------+----------+-----------+------------------------------------+


+-----------+---------+------+-------+---------------------+
| Directory |   Addr  | Type | Magic | Secondary Directory |
+-----------+---------+------+-------+---------------------+
|     5     | 0xb8000 | PSP  |  $PSP |       0x3a0000      |
+-----------+---------+------+-------+---------------------+
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   |     0 |  0xb8400 |   0x240 |              AMD_PUBLIC_KEY~0x0 |     1BB9 |           |                                    |
|   |     1 | 0x3a0400 |  0xc580 |          PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.9.0.77 |  signed(1BB9), verified, encrypted |
|   |     2 |  0xb8700 |  0xb580 | PSP_FW_RECOVERY_BOOT_LOADER~0x3 |     $PS1 | FF.9.0.77 |             signed(1BB9), verified |
|   |     3 |  0xc3d00 | 0x1d860 |              SMU_OFFCHIP_FW~0x8 |          | 0.19.56.0 | compressed, signed(1BB9), verified |
|   |     4 |  0xe1600 | 0x193f0 |       PSP_SMU_FN_FIRMWARE~0x108 |          | 0.2B.18.0 | compressed, signed(1BB9), verified |
|   |     5 |  0xfaa00 |   0x340 |       OEM_PSP_FW_PUBLIC_KEY~0xa |     2793 |           |                                    |
|   |     6 |  0xfae00 |    0x10 |               WRAPPED_IKEK~0x21 |          |           |                                    |
|   |     7 |  0xfaf00 |   0xdb0 |                       ABL0~0x30 |     0BAW | 19.7.8.30 | compressed, signed(2793), verified |
|   |     8 |  0xfbd00 |  0xd0a0 |                       ABL1~0x31 |     AW1B | 19.7.8.30 | compressed, signed(2793), verified |
|   |     9 | 0x108e00 |  0xa4a0 |                       ABL2~0x32 |     AW2B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    10 | 0x113300 |  0xb4e0 |                       ABL3~0x33 |     AW3B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    11 | 0x11e800 |  0x9e50 |                       ABL4~0x34 |     AW4B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    12 | 0x128700 |  0xd6d0 |                       ABL5~0x35 |     AW5B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    13 | 0x135e00 |  0xb2c0 |                       ABL6~0x36 |     AW6B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    14 | 0x141100 |   0xc00 |                 SEC_GASKET~0x24 |     $PS1 |  13.2.0.9 | compressed, signed(1BB9), verified |
|   |    15 | 0x3a0000 |   0x400 |   !PL2_SECONDARY_DIRECTORY~0x40 |          |           |                                    |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+


+-----------+----------+-----------+-------+---------------------+
| Directory |   Addr   |    Type   | Magic | Secondary Directory |
+-----------+----------+-----------+-------+---------------------+
|     6     | 0x3a0000 | secondary |  $PL2 |          --         |
+-----------+----------+-----------+-------+---------------------+
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |    Size |                            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x3a0400 |  0xc580 |          PSP_FW_BOOT_LOADER~0x1 |     $PS1 |  0.9.0.77 |  signed(1BB9), verified, encrypted |
|   |     1 | 0x3aca00 | 0x4224c |           PSP_FW_TRUSTED_OS~0x2 |          |  0.9.0.77 |             signed(1BB9), verified |
|   |     2 | 0x3eed00 | 0x2309c |     PSP_BOOT_TIME_TRUSTLETS~0xc |          |   0.0.0.0 |                      legacy header |
|   |     3 |  0x87000 | 0x20000 |                 PSP_NV_DATA~0x4 |          |           |                                    |
|   |     4 | 0x411f00 | 0x1d860 |              SMU_OFFCHIP_FW~0x8 |          | 0.19.56.0 | compressed, signed(1BB9), verified |
|   |     5 | 0x42f800 | 0x193f0 |       PSP_SMU_FN_FIRMWARE~0x108 |          | 0.2B.18.0 | compressed, signed(1BB9), verified |
|   |     6 | 0x448c00 |  0x24a0 |        !SMU_OFF_CHIP_FW_3~0x112 |          | 0.2B.18.0 | compressed, signed(1BB9), verified |
|   |     7 | 0x44b100 |   0x340 |          SEC_DBG_PUBLIC_KEY~0x9 |     2921 |           |                                    |
|   |     8 | 0x44b500 |   0x340 |       OEM_PSP_FW_PUBLIC_KEY~0xa |     2793 |           |                                    |
|   |     9 |      0x1 |     0x0 |          SOFT_FUSE_CHAIN_01~0xb |          |           |                                    |
|   |    10 | 0x44b900 |   0x340 | PSP_BOOT_TIME_TRUSTLETS_KEY~0xd |     1E38 |           |                                    |
|   |    11 | 0x44bd00 |  0x2d20 |          SMU_OFF_CHIP_FW_2~0x12 |          | 0.19.56.0 | compressed, signed(1BB9), verified |
|   |    12 | 0x44eb00 |  0x1920 |               DEBUG_UNLOCK~0x13 |     $PS1 |  0.9.0.77 | compressed, signed(1BB9), verified |
|   |    13 | 0x450500 |    0x10 |               WRAPPED_IKEK~0x21 |          |           |                                    |
|   |    14 | 0x451000 |  0x1000 |               TOKEN_UNLOCK~0x22 |          |           |                                    |
|   |    15 | 0x452000 |   0xc00 |                 SEC_GASKET~0x24 |     $PS1 |  13.2.0.9 | compressed, signed(1BB9), verified |
|   |    16 | 0x452c00 |   0xdb0 |                       ABL0~0x30 |     0BAW | 19.7.8.30 | compressed, signed(2793), verified |
|   |    17 | 0x453a00 |  0xd0a0 |                       ABL1~0x31 |     AW1B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    18 | 0x460b00 |  0xa4a0 |                       ABL2~0x32 |     AW2B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    19 | 0x46b000 |  0xb4e0 |                       ABL3~0x33 |     AW3B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    20 | 0x476500 |  0x9e50 |                       ABL4~0x34 |     AW4B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    21 | 0x480400 |  0xd6d0 |                       ABL5~0x35 |     AW5B | 19.7.8.30 | compressed, signed(2793), verified |
|   |    22 | 0x48db00 |  0xb2c0 |                       ABL6~0x36 |     AW6B | 19.7.8.30 | compressed, signed(2793), verified |
+---+-------+----------+---------+---------------------------------+----------+-----------+------------------------------------+


+-----------+----------+------+-------+---------------------+
| Directory |   Addr   | Type | Magic | Secondary Directory |
+-----------+----------+------+-------+---------------------+
|     7     | 0x168000 | BIOS |  $BHD |       0x4d0000      |
+-----------+----------+------+-------+---------------------+
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |     Size |                          Type | Magic/ID |   Version |                               Info |
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x169000 |   0x2000 |                      0x100060 |          |           |                                    |
|   |     1 | 0x16b000 |   0x2000 |                      0x100068 |          |           |                                    |
|   |     2 | 0x16d000 |   0x2000 |                   FW_IMC~0x60 |          |           |                                    |
|   |     3 | 0x16f000 |   0x2000 |                      0x200060 |          |           |                                    |
|   |     4 | 0x171000 |   0x2000 |                          0x68 |          |           |                                    |
|   |     5 | 0x173000 |   0x2000 |                      0x200068 |          |           |                                    |
|   |     6 |      0x0 |      0x0 |                   FW_GEC~0x61 |          |           |                                    |
|   |     7 | 0xd80000 | 0x280000 |                          BIOS |          |           |                                    |
|   |     8 | 0x175000 |   0x3c40 |                      0x100064 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |     9 | 0x178d00 |    0x330 |                      0x100065 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    10 | 0x179100 |   0x4610 |                      0x400064 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    11 | 0x17d800 |    0x320 |                      0x400065 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    12 | 0x4d0000 |    0x400 | !BL2_SECONDARY_DIRECTORY~0x70 |          |           |                                    |
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+


+-----------+----------+-----------+-------+---------------------+
| Directory |   Addr   |    Type   | Magic | Secondary Directory |
+-----------+----------+-----------+-------+---------------------+
|     8     | 0x4d0000 | secondary |  $BL2 |          --         |
+-----------+----------+-----------+-------+---------------------+
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+
|   | Entry |  Address |     Size |            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+
|   |     0 | 0x4d1000 |   0x2000 |        0x100060 |          |           |                                    |
|   |     1 | 0x4d3000 |   0x2000 |        0x100068 |          |           |                                    |
|   |     2 | 0x4d5000 |   0x2000 |     FW_IMC~0x60 |          |           |                                    |
|   |     3 | 0x4d7000 |   0x2000 |        0x200060 |          |           |                                    |
|   |     4 | 0x4d9000 |   0x2000 |            0x68 |          |           |                                    |
|   |     5 | 0x4db000 |   0x2000 |        0x200068 |          |           |                                    |
|   |     6 |      0x0 |      0x0 |     FW_GEC~0x61 |          |           |                                    |
|   |     7 | 0xd80000 | 0x280000 |            BIOS |          |           |                                    |
|   |     8 | 0x298000 |  0x28000 | FW_INVALID~0x63 |          |           |                                    |
|   |     9 | 0x4dd000 |    0xc80 |            0x66 |          |           |                                    |
|   |    10 | 0x4ddd00 |    0xc80 |        0x100066 |          |           |                                    |
|   |    11 | 0x4dea00 |    0xc80 |        0x200066 |          |           |                                    |
|   |    12 | 0x4df700 |    0xc80 |        0x300066 |          |           |                                    |
|   |    13 | 0x4e0400 |    0xc80 |        0x400066 |          |           |                                    |
|   |    14 | 0x4e1100 |   0x3c40 |        0x100064 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    15 | 0x4e4e00 |    0x330 |        0x100065 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    16 | 0x4e5200 |   0x4610 |        0x400064 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    17 | 0x4e9900 |    0x320 |        0x400065 |     0x05 | 0.0.A1.41 | compressed, signed(1BB9), verified |
|   |    18 | 0x4ea000 |   0x1000 |            0x67 |          |           |                                    |
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+


+-----------+----------+------+-------+---------------------+
| Directory |   Addr   | Type | Magic | Secondary Directory |
+-----------+----------+------+-------+---------------------+
|     9     | 0x268000 | BIOS |  $BHD |       0x63f000      |
+-----------+----------+------+-------+---------------------+
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+
|   | Entry |  Address |     Size |                          Type | Magic/ID |   Version |                               Info |
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+
|   |     0 | 0x269000 |   0x2000 |                      0x100060 |          |           |                                    |
|   |     1 | 0x26b000 |   0x2000 |                      0x100068 |          |           |                                    |
|   |     2 | 0x26d000 |   0x2000 |                   FW_IMC~0x60 |          |           |                                    |
|   |     3 | 0x26f000 |   0x2000 |                          0x68 |          |           |                                    |
|   |     4 |      0x0 |      0x0 |                   FW_GEC~0x61 |          |           |                                    |
|   |     5 | 0xd80000 | 0x280000 |                          BIOS |          |           |                                    |
|   |     6 | 0x271000 |   0x3c40 |                      0x100064 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |     7 | 0x274d00 |    0x330 |                      0x100065 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |     8 | 0x275100 |   0x4610 |                      0x400064 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |     9 | 0x279800 |    0x320 |                      0x400065 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |    10 | 0x279c00 |   0x4830 |                     0x1100064 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    11 | 0x27e500 |    0x370 |                     0x1100065 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    12 | 0x27e900 |   0x47a0 |                     0x1400064 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    13 | 0x283100 |    0x340 |                     0x1400065 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    14 | 0x63f000 |    0x400 | !BL2_SECONDARY_DIRECTORY~0x70 |          |           |                                    |
+---+-------+----------+----------+-------------------------------+----------+-----------+------------------------------------+


+-----------+----------+-----------+-------+---------------------+
| Directory |   Addr   |    Type   | Magic | Secondary Directory |
+-----------+----------+-----------+-------+---------------------+
|     10    | 0x63f000 | secondary |  $BL2 |          --         |
+-----------+----------+-----------+-------+---------------------+
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+
|   | Entry |  Address |     Size |            Type | Magic/ID |   Version |                               Info |
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+
|   |     0 | 0x640000 |   0x2000 |        0x100060 |          |           |                                    |
|   |     1 | 0x642000 |   0x2000 |        0x100068 |          |           |                                    |
|   |     2 | 0x644000 |   0x2000 |     FW_IMC~0x60 |          |           |                                    |
|   |     3 | 0x646000 |   0x2000 |            0x68 |          |           |                                    |
|   |     4 |      0x0 |      0x0 |     FW_GEC~0x61 |          |           |                                    |
|   |     5 | 0xd80000 | 0x280000 |            BIOS |          |           |                                    |
|   |     6 | 0x298000 |  0x28000 | FW_INVALID~0x63 |          |           |                                    |
|   |     7 | 0x648000 |   0x3c40 |        0x100064 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |     8 | 0x64bd00 |    0x330 |        0x100065 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |     9 | 0x64c100 |   0x4610 |        0x400064 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |    10 | 0x650800 |    0x320 |        0x400065 |     0x05 | 0.0.A1.41 | compressed, signed(60BB), verified |
|   |    11 | 0x650c00 |   0x4830 |       0x1100064 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    12 | 0x655500 |    0x370 |       0x1100065 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    13 | 0x655900 |   0x47a0 |       0x1400064 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    14 | 0x65a100 |    0x340 |       0x1400065 |     0x05 |  0.0.18.5 | compressed, signed(60BB), verified |
|   |    15 | 0x65a500 |    0xc80 |            0x66 |          |           |                                    |
|   |    16 | 0x65b200 |    0xc80 |        0x100066 |          |           |                                    |
|   |    17 | 0x65bf00 |    0xc80 |        0x200066 |          |           |                                    |
|   |    18 | 0x65cc00 |    0xc80 |        0x300066 |          |           |                                    |
|   |    19 | 0x65d900 |    0xc80 |        0x400066 |          |           |                                    |
|   |    20 | 0x65e600 |    0xc80 |        0x500066 |          |           |                                    |
|   |    21 | 0x65f300 |    0x560 |            0x6a |          |   0.0.0.0 |                       signed(76E9) |
+---+-------+----------+----------+-----------------+----------+-----------+------------------------------------+
```

### Analyze the firmware
