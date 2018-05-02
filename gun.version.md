section gun.version 分析
=======================
第一在elf中的位置
----------------
```
Section Headers:
.gnu.version              VERSYM           0000000000000410  00000410
       000000000000001a  0000000000000002   A       3     0     2
       
Dynamic section
 0x000000006ffffff0 (VERSYM)             0x410
Hex dump of section '.gnu.version':
  0x00000410 00000000 00000000 00000000 02000100 ................
  0x00000420 01000100 01000100 0100              ..........
 
 Version symbols section '.gnu.version' contains 13 entries:
 Addr: 0000000000000410  Offset: 0x000410  Link: 3 (.dynsym)
  000:   0 (*local*)       0 (*local*)       0 (*local*)       0 (*local*)    
  004:   0 (*local*)       0 (*local*)       2 (GLIBC_2.2.5)   1 (*global*)   
  008:   1 (*global*)      1 (*global*)      1 (*global*)      1 (*global*)   
  00c:   1 (*global*)   
```
