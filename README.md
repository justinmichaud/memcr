# memory checkpoint and restore

[![Build status](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-x86_64.yml/badge.svg)](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-x86_64.yml)
[![Build status](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-arm.yml/badge.svg)](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-arm.yml)
[![Build status](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-arm64.yml/badge.svg)](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-arm64.yml)
[![Build status](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-riscv64.yml/badge.svg)](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-riscv64.yml)
[![Build status](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-clang.yml/badge.svg)](https://github.com/LibertyGlobal/memcr/actions/workflows/ci-clang.yml)

memcr was written as a PoC to demonstrate that it is possible to temporarily reduce RSS of a target process without killing it. This is achieved by freezing the process, checkpointing its memory to a file and restoring it later when needed.

The idea is based on concepts seen in ptrace-parasite and early [CRIU](https://github.com/checkpoint-restore/criu) versions. The key difference is that the target process is kept alive and memcr manipulates its memory with `madvise()` `MADV_DONTNEED` syscall to reduce RSS. VM mappings are not changed.

#### building

```
make
```
##### compilation options
You can enable support for compression and checksumming of memory dump file:
 - `COMPRESS_LZ4=1` - requires liblz4
 - `COMPRESS_ZSTD=1` - requires libzstd
 - `CHECKSUM_MD5=1` - requires libcrypto and openssl headers

 There is also `ENCRYPT` option for building `libencrypt.so` that provides sample implementation of encryption layer based on libcrypto API. memcr is not linked with libencrypt.so, but it can be preloaded with `LD_PRELOAD`.
 - `ENCRYPT=1` - requires libcrypto and openssl headers

For debugging the parasite that runs inside the target process:
 - `PARASITE_DEBUG=1` - parasite spins on a flag at `service()` entry and memcr skips `PTRACE_SEIZE` on the parasite so gdb can attach. See [debugging the parasite](#debugging-the-parasite) below.

##### compilation on Ubuntu 24.04:
```
sudo apt-get install liblz4-dev liblz4-1
sudo apt-get install libzstd-dev libzstd1
sudo apt-get install libssl-dev libssl3
```

```
make COMPRESS_LZ4=1 COMPRESS_ZSTD=1 CHECKSUM_MD5=1 ENCRYPT=1
```

##### cross compilation
Currently, supported architectures are x86_64, arm, arm64 and riscv64. You can cross compile memcr by providing `CROSS_COMPILE` prefix. i.e.:
```
make CROSS_COMPILE=arm-linux-gnueabihf-
make CROSS_COMPILE=aarch64-linux-gnu-
```
##### yocto
There is a generic `memcr.bb` recipe provided that you can copy into your yocto layer and build memcr as any other packet with bitbake.
```
bitbake memcr
```

#### debugging the parasite
The parasite runs as a `clone()`-created task inside the target process and is normally `PTRACE_SEIZE`d by memcr's watch thread, which prevents gdb from attaching. Building with `PARASITE_DEBUG=1` turns on two things:
 - the parasite spins on a `volatile` flag (`parasite_wait_for_dbg`) at the very top of `service()`, before any syscall — so the target won't crash before you attach;
 - memcr's `parasite_watch_thread` returns without seizing the parasite, so gdb can attach to the parasite tid.

A normal checkpoint/restore cycle will not complete in this mode (the watch thread is the channel that signals parasite exit). It is strictly a crash-hunting build.

Build:
```
make clean && make COMPRESS_LZ4=1 COMPRESS_ZSTD=1 CHECKSUM_MD5=1 PARASITE_DEBUG=1
```

Run memcr against your target. It will log the parasite tid and the gdb command to use, e.g.:
```
memcr-client -l 9000 -p $(pidof WebKitWebProcess) --checkpoint

[+] PARASITE_DEBUG: skipping PTRACE_SEIZE on parasite 12345 so gdb can attach
[+] PARASITE_DEBUG: gdb -p 12345 ; (gdb) set var parasite_wait_for_dbg=0
```

Attach gdb. The parasite blob is mmaped as an anonymous `rwxp` page in the target; locate it and load symbols at that address (the linker script puts every parasite section into a single `.blob` starting at offset 0, so a single `add-symbol-file` is enough):
```
sudo gdb -p <parasite_tid>
(gdb) shell grep rwxp /proc/<parasite_tid>/maps
(gdb) add-symbol-file parasite.bin.o <load_addr>
(gdb) break service
(gdb) set var parasite_wait_for_dbg = 0
(gdb) continue
```

You can now step through `sys_socket`, `sys_bind`, `sys_listen`, the pagemap open, and the accept loop to find where the parasite faults.

If gdb refuses to attach with "Operation not permitted", either run gdb as root or set `/proc/sys/kernel/yama/ptrace_scope` to `0`.

#### how to use memcr
Basic usage to tinker with memcr is:
```
memcr -p <target pid>
```
For the list of available options, check memcr help:
```
memcr [-h] [-p PID] [-d DIR] [-S DIR] [-G gid] [-N] [-l PORT|PATH] [-g gid] [-n] [-m] [-f] [-z lz4|zstd] [-c] [-e] [-t] [-V]
options:
  -h --help             help
  -p --pid              target process pid
  -d --dir              dir where memory dump is stored (defaults to /tmp)
  -S --parasite-socket-dir      dir for memcr-internal restore socket (parasite uses TCP loopback, see -N)
  -G --parasite-socket-gid      deprecated: parasite now uses TCP loopback, this flag is ignored
  -N --parasite-socket-netns    enter network namespace of parasite when connecting to its TCP loopback port
        (required if parasite is running in a container with netns)
  -l --listen           work as a service waiting for requests on a socket
        -l PORT: TCP port number to listen for requests on
        -l PATH: filesystem path for UNIX domain socket file (will be created)
  -g --listen-gid       group ID for listen UNIX domain socket file, valid only in service mode for UNIX domain socket
  -n --no-wait          no wait for key press
  -m --proc-mem         get pages from /proc/pid/mem
  -f --rss-file         include file mapped memory
  -z --compress         compress memory dump with lz4 (default) or zstd
  -c --checksum         enable md5 checksum for memory dump
  -e --encrypt          enable encryption of memory dump
  -t --timeout          timeout in seconds for checkpoint/restore execution in service mode
  -V --version          print version and exit
```
memcr also supports client / server scenario where memcr runs as a daemon and listens for commands from a client process. The main reason for supporting this is that memcr needs rather high privileges to hijack target process and it's a good idea to keep it separate from memcr-client that can run in a container with low privileges.

memcr daemon:
```
sudo memcr -l 9000 -zc
```
memcr client:
```
memcr-client -l 9000 -p 1234567 --checkpoint
memcr-client -l 9000 -p 1234567 --restore
```
Due to high priviledges of the memcr daemon it is recommended to run memcr daemon process as non-root user with elevated Linux capabilities and permissions, the details are described in: [doc/security_considerations.md](doc/security_considerations.md)
