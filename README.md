# Fuzzing
Fuzzing: Quick and dirty
author: d3k4

## Virtual machine information

username: `root`/`redteam`
password: `redteam`

```bash
root@fuzz:~# uname -r
4.18.0-16-generic
root@fuzz:~# cat /etc/issue
Ubuntu 18.10 \n \l
```

## Environment

```bash
# AFL does a lot of writes, and breaks SSD (SANS)
mkdir mkdir /mnt/ramdisk
mount -t tmpfs -o size=512m tmpfs /mnt/ramdisk
# Set the SOFT and HARD core dumpts settings
ulimit -c unlimited
# Tell the kernel what to do with the with the core files
echo core >/proc/sys/kernel/core_pattern
```

## Scenarios


```
+-------------+          +---------+           +--------------------+
| Source Code +---yes---->  input? +---stdin--->   compile & fuzz   |
+-----+-------+          +----+----+           +------+----------+--+
      |                       |                       ^          ^
      |                      else                     |          |
      |                       |                       |          |
      |                       v                       |          |
      |                  +----+---------+      +------+------+   |
      |                  | C/Cpp skilz? +--y---> Edit source |   |
      |                  +----+---------+      +-------------+   |
      |                       |                                  |
      no                      n                                  |
      |                       |                                  |
      |                  +----v------------+                     |
      |                  | Instumentation  +----------works------+
      |                  | Libraries?      |
      |                  +----+------------+
      |                       |
      |                       |
      |                       |
+-----v-------+               |                +--------------------+
|  input?     +------stdin--------------------->  afl qemu          |
+-----+-------+               |                +--------------------+
      |                       |
      |                       |
      |                       |
      |                       |
      |                       v                +--------------------+
      +---------------else----+--------------->+  afl unicorn / emu |
                                               +--------------------+
```

## AFL

```bash
root@fuzz:~# afl-
afl-analyze  afl-clang++  afl-fuzz     afl-gcc      afl-plot     afl-tmin     
afl-clang    afl-cmin     afl-g++      afl-gotcpu   afl-showmap  afl-whatsup 
```


### Basic fuzzing workflow

1. Compile the binary with AFL
2. Find test cases
3. Run the fuzzer
4. Triage
5. 0dayz


##### Compilation

```bash
CC=/usr/local/bin/afl-gcc ./configure
CXX=/usr/local/bin/afl-g++ ./configure
AFL_USE_ASAN=1 ./configure CC=afl-gcc CXX=afl-g++ LD=afl-gcc--disable-shared
```

For statically compiled binaries add `--disable-shared`

`CXX=/usr/local/bin/afl-g++ ./configure`

* This will avoid the `LD_PRELOAD` calls. 

Additional environment configurations:

- `AFL_USE_ASAN=1` - Include address sanitizer
- `AFL_INST_RATIO=100` - how much instrumentation do we want to add [1-100]
- `AFL_HARDEN=1` - provoking quicker crashes

### Prepare test cases

If we have many test cases, we can use `afl-cmin` and `afl-tmin`:

```
afl-cmin -i <test case folder> -o <chosen tests folder> -- [path_to_binary] @@
afl-tmin -i <test case folder> -o <chosen tests folder> -- [path_to_binary] @@
```
* `afl-cmin` - minimizes the list of test cases;
* `afl-tmin` - minimizes the size of the test files;

### Fuzzing

The most important options are `-i <dir>` (input) and `-o <dir>` (output)
- the test cases directory should be specified in `-i`;
- all crashesh and current state will be saved in `-o` directory;

Master/Slave concept is available, respectively `-m <Mname>` and `-s <Sname>`

```bash
afl-fuzz -i in/ -o out/ -- [path_to_binary] @@
afl-fuzz -M master1 -i in/ -o out/ [path_to_binary] @@
afl-fuzz -S slave1 -i in/ -o out/ [path_to_binary] @@
afl-fuzz -S slave2 -i in/ -o out/ [path_to_binary] @@
```

* `@@` marks where AFL should include the test case.

#### Dotnet fuzzing

In order to fuzz dotnet(core) binaries one should use [sharpfuzz](https://github.com/Metalnem/sharpfuzz). Example of the technique can be found [here](https://github.com/Metalnem/sharpfuzz-samples)

In order to skip paching and rebuilding the AFL binaries you need to set the following env variable:
`AFL_SKIP_BIN_CHECK`

#### Stats

It is possible to check the statuses of the cluster:

`afl-whatsup <output folder>`


### Source code NOT available (ref. afl/README.md)

When source code is *NOT* available, the fuzzer offers experimental support for fast, on-the-fly instrumentation of black-box binaries. This is accomplished with a version of QEMU running in the lesser-known "user space emulation" mode.

For additional instructions and caveats, see `qemu_mode/README.qemu`.

The mode is approximately 2-5x slower than compile-time instrumentation, is
less conductive to parallelization, and may have some other quirks.

`afl-fuzz -Q -i in/ -o out/ -- [path_to_binary] @@`

* The `-Q` enables the QEMU emulation;


## ANGR Fuzzing 

preps:

```
echo 1 | sudo tee /proc/sys/kernel/sched_child_runs_first
echo core >/proc/sys/kernel/core_pattern
```

