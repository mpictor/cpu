# cpu

[![CircleCI](https://circleci.com/gh/u-root/cpu.svg?style=svg)](https://circleci.com/gh/u-root/cpu)
[![Go Report Card](https://goreportcard.com/badge/github.com/u-root/cpu)](https://goreportcard.com/report/github.com/u-root/cpu)
[![GoDoc](https://godoc.org/github.com/u-root/cpu?status.svg)](https://godoc.org/github.com/u-root/cpu)
[![License](https://img.shields.io/badge/License-BSD%203--Clause-blue.svg)](https://github.com/u-root/cpu/blob/master/LICENSE)

A cpu command that uses either an external 9p server process or hugelgupf 9p package internally.


### Example

Prerequisites: install u-root command, install qemu, and build a kernel suitable for running in qemu.

```bash
#!/bin/bash -e


work=$(realpath $(dirname $0))


cpud_key=$work/cpukeys/ssh_host_rsa_key
cpu_key=$work/cpukeys/cpu_rsa

echo "create keys, if necessary"
if [[ ! -d $work/cpukeys ]]; then
  mkdir -p $work/cpukeys
fi

if [[ ! -f $cpud_key ]]; then
  ssh-keygen -f $cpud_key -N ''
fi

if [[ ! -f $cpu_key ]]; then
  ssh-keygen -f $cpu_key -N ''
fi

echo "create initramfs, if necessary"
if [[ ! -f $work/initramfs-cpud.cpio ]]; then
  u-root                                            \
    -build=bb                                       \
    -files  $cpud_key:etc/ssh/ssh_host_rsa_key      \
    -initcmd=/bbin/cpud                             \
    -files  $cpu_key.pub:key.pub                    \
    -o $work/initramfs-cpud.cpio                    \
    all                                             \
    github.com/u-root/cpu/cmds/cpud                 \
    github.com/u-root/cpu/cmds/cpu
fi

k=$work/test_kernel
if [[ ! -f $k ]]; then
  echo "missing kernel $k"
  exit 1
fi


echo "$(tput setaf 3; tput bold)in another window:"
echo "./cpu -key $cpu_key -hk $cpud_key.pub -sp 2222 localhost /bin/date$(tput sgr0)"
echo
echo "$(tput setaf 1)starting qemu...$(tput sgr0)"

qemu-system-x86_64 -kernel $k -initrd $work/initramfs-cpud.cpio                           \
    -smp 4 -m 1024m -enable-kvm -cpu host                                                 \
    -object rng-random,filename=/dev/urandom,id=rng0                                      \
    -device virtio-rng-pci,rng=rng0                                                       \
    -device e1000,netdev=n1                                                               \
    -netdev user,id=n1,hostfwd=tcp:127.0.0.1:2222-:23,net=192.168.1.0/24,host=192.168.1.1 \
    -nographic -serial stdio                                                              \
    -append 'earlyprintk=ttyS0,115200 console=ttyS0 -hk /etc/ssh/ssh_host_rsa_key quiet'  \
    -monitor /dev/null
```