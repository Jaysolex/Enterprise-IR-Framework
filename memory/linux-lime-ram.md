# Linux RAM Acquisition — LiME

## What LiME Is
LiME (Linux Memory Extractor) is a loadable kernel module that acquires
full RAM dumps from live Linux systems. Unlike Windows which has built-in
methods, Linux requires a kernel module compiled for the target kernel version.

## Installation and Build
```bash
# Install dependencies
sudo apt-get install build-essential linux-headers-$(uname -r) gcc-12 -y

# Download LiME
git clone https://github.com/504ensicsLabs/LiME

# Build for current kernel
cd LiME/src
make clean
make
```

Output: `lime-[kernel-version].ko`

## Acquire RAM
```bash
sudo insmod lime-6.8.0-124-generic.ko "path=/tmp/memdump.lime format=lime"
```

## Verify
```bash
ls -lh /tmp/memdump.lime
```

## Lab Results — SIEMServer
- Kernel: 6.8.0-124-generic
- Module: lime-6.8.0-124-generic.ko
- Output: /tmp/memdump.lime
- Size: 4.0GB

## Analyse with Volatility
```bash
vol -f /tmp/memdump.lime linux.pslist
vol -f /tmp/memdump.lime linux.netstat
vol -f /tmp/memdump.lime linux.bash
```

## Key Notes
- Must compile LiME for the exact kernel version of target machine
- Output file is read-only (root owned) — normal
- 4GB dump = machine had 4GB RAM — size matches physical RAM
- Hash the dump immediately for chain of custody
```bash
sha256sum /tmp/memdump.lime > /tmp/memdump.lime.sha256
```
