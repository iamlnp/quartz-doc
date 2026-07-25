---
写作年份: 2024
tags:
  - Linux
  - 符号调试
---

eu-unstrip 是一款工具，通常用于调试符号的处理。它是 **elfutils** 套件的一部分，主要应用于管理和分析 **ELF（Executable and Linkable Format）** 文件。以下是它的核心功能和用法：

**功能**

eu-unstrip 的主要目的是将调试符号从一个独立的调试信息文件（通常是 .debug 文件）加载回原始的可执行文件或共享库中。它的操作可以被视为“还原”或“重新合并”调试信息的过程。

通常情况下，开发者会通过 strip 命令从二进制文件中移除调试符号，生成一个体积较小的可执行文件，并将调试符号存储在单独的 .debug 文件中。eu-unstrip 则用于在需要的时候重新将这些符号整合到二进制文件中。

**举例**
```bash
[root@node204 coredump]# eu-unstrip -n --core core.md_recov_open.167.1970788.1733902770
0x555a4e612000+0x119d000 5cd48eac41a8d3890cae503d4d86f8ce5a696109@0x555a4e61230c . - /usr/bin/tfs-mds
0x7ffc519f0000+0x2000 ce389f24534af4dcb9a496aec8aadae2908bdb57@0x7ffc519f07d8 . - linux-vdso.so.1
0x7f4a55762000+0xc358 e94ffb2ea2a456d563044f319764d437861c5b8a@0x7f4a55762248 /lib64/libnss_sss.so.2 - libnss_sss.so.2
0x7f4a49ba2000+0x2d2a58 711ad5c27f4346ae33c99ca1134427c9d5a874f1@0x7f4a49ba2248 /lib64/libprotoc.so.23 - libprotoc.so.23
0x7f4a49e75000+0x65060 3d366074b4b3a297153c09fc0a128f2f74504023@0x7f4a49e75248 /lib64/libleveldb.so.1 - libleveldb.so.1
0x7f4a49edb000+0x19988 3bbdbb31b210b89af0e7d77551290f22f4fe58ba@0x7f4a49edb280 /lib64/libunwind.so.8 - libunwind.so.8
0x7f4a49ef5000+0x9028 676af3c2eea87a73880d2412735f3d8043e63763@0x7f4a49ef52d8 /lib64/libatomic.so.1 - libatomic.so.1
0x7f4a55771000+0x28150 6a88a97ed9153f173663bcfc9734f830515b681a@0x7f4a55771248 /lib64/ld-linux-x86-64.so.2 - ld-linux-x86-64.so.2
0x7f4a49f01000+0x1b74d8 cee276b8f1ba886af26c3efb9699369b3e1ab5b4@0x7f4a49f012f0 /lib64/libc.so.6 - libc.so.6
0x7f4a4a0b9000+0x1a2d0 3012e3abb9d0a588610dd373683c41933b3ccd27@0x7f4a4a0b92d8 /lib64/libgcc_s.so.1 - libgcc_s.so.1
0x7f4a4a0d4000+0x182018 5da130340ede0bc0ac6e4345c40b422bf5a9b58a@0x7f4a4a0d4248 /lib64/libm.so.6 - libm.so.6
0x7f4a4a257000+0x197620 36c04f26c5fb95d5386e7a4e8097d23e4d48c66d@0x7f4a4a257310 /lib64/libstdc++.so.6 - libstdc++.so.6
0x7f4a4a3ef000+0x28948 8ecccdb61ea8ffc472e0b92ed146efc167eaf678@0x7f4a4a3ef280 /lib64/libudev.so.1 - libudev.so.1
0x7f4a4a418000+0x2f18c0 9f6cc1cf225cef0ebebd8701c0e819d56c7b35d9@0x7f4a4a418248 /lib64/libcrypto.so.1.1 - libcrypto.so.1.1
0x7f4a4a70c000+0x50728 d59eba3759c735c273a0e872cb9455777b4dbce6@0x7f4a4a70c280 /lib64/libblkid.so.1 - libblkid.so.1
0x7f4a4a75d000+0x188a0 c5133023d652cd80ff4831a7085a55d0c9978ecc@0x7f4a4a75d248 /lib64/libresolv.so.2 - libresolv.so.2
0x7f4a4a776000+0x10b7e28 da1bde2fd2d3a6d549cf7eb6a61f2478e3ab2f63@0x7f4a4a776280 /lib64/libbrpc.so - libbrpc.so
0x7f4a4b82e000+0x8cdbeb0 920b22302d04756603b5ad0015cb62e2396be3ff@0x7f4a4b82e280 /usr/lib64/ceph/libceph-common.so.2 - libceph-common.so.2
0x7f4a5450a000+0x2f97c0 f8b2ee2e3bbc77eb66c7e6f52e2ff2acb4c2685a@0x7f4a5450a280 /lib64/libprotobuf.so.23 - libprotobuf.so.23
0x7f4a54804000+0x931c0 3b21289f65bc082b8a12fcfb75247119c2f54633@0x7f4a54804248 /lib64/libssl.so.1.1 - libssl.so.1.1
0x7f4a5489a000+0x201e0 48e4322e4b575dfb6ca284b92a8995ce22e50ca1@0x7f4a5489a2b8 /lib64/libpthread.so.0 - libpthread.so.0
0x7f4a548bb000+0x27528 b2187642c29be7a4ed8c78eeb47431e4a481ff72@0x7f4a548bb248 /lib64/libgflags.so.2.2 - libgflags.so.2.2
0x7f4a548e3000+0x206580 5049b554a71c0d54390f09149d74259a78f56733@0x7f4a548e3280 /lib64/libtcmalloc.so.4 - libtcmalloc.so.4
0x7f4a54aea000+0x141cc8 9782a848acd28f2030dbdc305324a558ed796156@0x7f4a54aea280 /lib64/librados.so.2 - librados.so.2
0x7f4a54c2c000+0x58c408 a58fae7541bb6c4cd11a026ba0eb162fb9820978@0x7f4a54c2c280 /lib64/librbd.so.1 - librbd.so.1
0x7f4a551b9000+0xaa00 16f8fdd1cf3977ccc2c5ae8d4241a8caa6df6d65@0x7f4a551b9248 /lib64/librt.so.1 - librt.so.1
0x7f4a551c6000+0x19010 3777f9530165107f131ca26b81bdd5f2372d610b@0x7f4a551c6248 /lib64/libz.so.1 - libz.so.1
0x7f4a551e0000+0x39160 d7d8fc893223528ead1ec3fd4639e32e72d55863@0x7f4a551e0248 /lib64/liblz4.so.1 - liblz4.so.1
0x7f4a5521a000+0xb018 cb331116d43d9f7f2a7a51e40ff48f6cdd3f8f64@0x7f4a5521a248 /lib64/libsnappy.so.1 - libsnappy.so.1
0x7f4a55226000+0x62e50 213ea38a68d7191105d1d80e76f2df62f7242c6a@0x7f4a55226248 /lib64/libperftrace.so - libperftrace.so
0x7f4a55289000+0x4ce6a8 82eaed4de728dcedfe3abaf7e6169d353686abbc@0x7f4a55289280 /usr/lib64/ceph/libtfs_msg.so - libtfs_msg.so
0x7f4a55758000+0x4090 663ae415234288e334e25ae539a3cc72f2fccd12@0x7f4a55758248 /lib64/libdl.so.2 - libdl.so.2
```

**用法**

```bash
With -n no files are written, but one line to standard output for each module:
	START+SIZE BUILDID FILE DEBUGFILE MODULENAME
START and SIZE are hexadecimal giving the address bounds of the module.
BUILDID is hexadecimal for the build ID bits, or - if no ID is known; the
hexadecimal may be followed by @0xADDR giving the address where the ID resides
if that is known.  FILE is the file name found for the module, or - if none was
found, or . if an ELF image is available but not from any named file.
DEBUGFILE is the separate debuginfo file name, or - if no debuginfo was found,
or . if FILE contains the debug information.

 Input selection options:
      --core=COREFILE        Find addresses from signatures found in COREFILE
      --debuginfo-path=PATH  Search path for separate debuginfo files
  -e, --executable=FILE      Find addresses in FILE
  -k, --kernel               Find addresses in the running kernel
  -K, --offline-kernel[=RELEASE]   Kernel with all modules
  -M, --linux-process-map=FILE   Find addresses in files mapped as read from
                             FILE in Linux /proc/PID/maps format
  -p, --pid=PID              Find addresses in files mapped into process PID

  -f, --match-file-names     Match MODULE against file names, not module names
  -i, --ignore-missing       Silently skip unfindable files

 Output options:
  -a, --all                  Create output for modules that have no separate
                             debug information
  -d, --output-directory=DIRECTORY
                             Create multiple output files under DIRECTORY
  -F, --force                Force combining files even if some ELF headers
                             don't seem to match
  -m, --module-names         Use module rather than file names
  -n, --list-only            Only list module and file names, build IDs
  -o, --output=FILE          Place output into FILE
  -R, --relocate             Apply relocations to section contents in ET_REL
                             files

  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version
```