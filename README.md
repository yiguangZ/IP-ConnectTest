# One-to-One IP Connectivity Test with QEMU Socket Netdev

This README describes how to set up two Jetstream2 CXL-Emulator hosts and connect their nested QEMU guests over a point-to-point TCP socket network, assign them unique IPs, and verify L3 connectivity.

## 1. Configuring listerner and connector
- Jetstream2 account with at least **2 running cxl_emu instances** (e.g. `cxl_emu_1-of-4` and `cxl_emu_2-of-4`).  
- SSH access to each instance as `exouser` on your terminal:
```
ssh exouser@IP_ADDRESS_OF_INSTANCE
//cxl_emu_1 is 149.165.xxx.123
```
- enter the password for exouser when prompted.
- Now you should be in the terminal for exouser.
- Next we need to make some changes to start_vm.sh, enter the following commands to exouser terminal:
```
cd ~/qemu_images
vim start_vm.sh
```
- Now to configure cxl_emu_1 as the listener, change it's start_vm.sh to this:
```
#! /bin/bash

TOTAL_MEMORY=16384

sudo qemu-system-x86_64 -name emucxlVM \
-machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
-cpu host -smp cpus=4 -m ${TOTAL_MEMORY}M \
-device virtio-scsi-pci,id=scsi0 \
-device scsi-hd,drive=hd0 \
-drive file=/home/exouser/qemu_images/emucxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
-netdev user,id=mng,hostfwd=tcp::8080-:22 \
-device virtio-net-pci,netdev=mng,mac=52:54:00:AA:BB:01 \
-netdev socket,id=cross,listen=0.0.0.0:5555 \
-device virtio-net-pci,netdev=cross,mac=52:54:00:AA:BB:02 \
-nographic
```
- This will Forward host port 8080→VM SSH
- Listen on TCP/5555 for peer cxl_emu_2 which will be our conector
- Expose two separate virtio‑net devices (one for management, one for cross‑host).

- Now for cxl_emu_2 we need to change its start_vm.sh so it can be a connector:
```
//cxl_emu_2 start_vm.sh
#! /bin/bash

TOTAL_MEMORY=16384

sudo qemu-system-x86_64 -name emucxlVM \
  -machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
  -cpu host -smp cpus=4 -m ${TOTAL_MEMORY}M \
  -device virtio-scsi-pci,id=scsi0 \
  -device scsi-hd,drive=hd0 \
  -drive file=/home/exouser/qemu_images/emucxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
  -netdev user,id=mng,hostfwd=tcp::8081-:22 \
  -device virtio-net-pci,netdev=mng,mac=52:54:00:AA:BB:03 \
  -netdev socket,id=cross,connect=149.165.175.123:5555 \
  -device virtio-net-pci,netdev=cross,mac=52:54:00:AA:BB:04 \
  -nographic
```
- Once both start_vm.sh are configured, we are ready to run the start_vm.
## 2. Link set up
- Run start_vm.sh in both cxl_emu_1 and cxl_emu_2 by typing the following command into their terminal:
```
bash start_vm.shh
```
- Now this should prompt you to a login command line, enter your username and password. Ex(username jack, password ...)
- Once logged in, in nested VM-A(or cxl_emu_1) enter the following command:
```
sudo ip link set dev ens5 up
sudo ip addr add 10.1.1.1/24 dev ens5
```
- Now in nested VM-B(or cxl_emu_2) enter the following command:
```
sudo ip link set dev ens5 up
sudo ip addr add 10.1.1.2/24 dev ens5
```
- Now the link set up is done.

## 3. Test connectivity
- Now to test the connectivity simply do this:
```
ping -c4 10.1.1.2   # from A
ping -c4 10.1.1.1   # from B
```
- then you should see sucessfull packet transmission on both ends.



