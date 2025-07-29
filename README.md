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


# Connecting all 4 instances with QEMU Socket Netdev
This section will show how to connect all 4 instances to each other.

## 1. Overview of configuration:

| Link   | Port | Subnet       | VM A IP           | VM B IP           |
|:-------|:-----|:-------------|:------------------|:------------------|
| 1 ↔ 2  | 6001 | 10.0.1.0/30  | 10.0.1.1 (VM 1)   | 10.0.1.2 (VM 2)   |
| 1 ↔ 3  | 6002 | 10.0.2.0/30  | 10.0.2.1 (VM 1)   | 10.0.2.2 (VM 3)   |
| 1 ↔ 4  | 6003 | 10.0.3.0/30  | 10.0.3.1 (VM 1)   | 10.0.3.2 (VM 4)   |
| 2 ↔ 3  | 6004 | 10.0.4.0/30  | 10.0.4.1 (VM 2)   | 10.0.4.2 (VM 3)   |
| 2 ↔ 4  | 6005 | 10.0.5.0/30  | 10.0.5.1 (VM 2)   | 10.0.5.2 (VM 4)   |
| 3 ↔ 4  | 6006 | 10.0.6.0/30  | 10.0.6.1 (VM 3)   | 10.0.6.2 (VM 4)   |

## 2. Editing start_vm.sh for each cxl_emu
- Now to set up the configuration above, log in as exouser@IP_ADDRESS_OF_CXL_EMU_X.
- Then cd into qemu_images and change each of their start_vm.sh as shown below
- Change cxl_emu_1 start_vm.sh to this:
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
-netdev socket,id=link12,listen=0.0.0.0:6001 \
-device virtio-net-pci,netdev=link12,mac=52:54:00:CC:01:01 \
-netdev socket,id=link13,listen=0.0.0.0:6002 \
-device virtio-net-pci,netdev=link13,mac=52:54:00:CC:01:02 \
-netdev socket,id=link14,listen=0.0.0.0:6003 \
-device virtio-net-pci,netdev=link14,mac=52:54:00:CC:01:03 \
-nographic


```

- Change cxl_emu_2 start_vm.sh to this:
```
#! /bin/bash

TOTAL_MEMORY=16384


sudo qemu-system-x86_64 -name emucxlVM \
-machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
-cpu host -smp cpus=4 -m ${TOTAL_MEMORY}M \
-device virtio-scsi-pci,id=scsi0 \
-device scsi-hd,drive=hd0 \
-drive file=/home/exouser/qemu_images/emucxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
-netdev user,id=mng,hostfwd=tcp::8081-:22 \
-device virtio-net-pci,netdev=mng,mac=52:54:00:AA:BB:02 \
-netdev socket,id=link12,connect=149.165.175.123:6001 \
-device virtio-net-pci,netdev=link12,mac=52:54:00:CC:02:01 \
-netdev socket,id=link23,listen=0.0.0.0:6004 \
-device virtio-net-pci,netdev=link23,mac=52:54:00:CC:02:02 \
-netdev socket,id=link24,listen=0.0.0.0:6005 \
-device virtio-net-pci,netdev=link24,mac=52:54:00:CC:02:03 \
-nographic



```

- Change cxl_emu_3 start_vm.sh to this:
```
#! /bin/bash

TOTAL_MEMORY=16384


sudo qemu-system-x86_64 -name emucxlVM \
-machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
-cpu host -smp cpus=4 -m ${TOTAL_MEMORY}M \
-device virtio-scsi-pci,id=scsi0 \
-device scsi-hd,drive=hd0 \
-drive file=/home/exouser/qemu_images/emucxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
-netdev user,id=mng,hostfwd=tcp::8082-:22 \
-device virtio-net-pci,netdev=mng,mac=52:54:00:AA:BB:03 \
-netdev socket,id=link13,connect=149.165.175.123:6002 \
-device virtio-net-pci,netdev=link13,mac=52:54:00:CC:03:01 \
-netdev socket,id=link23,connect=149.165.173.250:6004 \
-device virtio-net-pci,netdev=link23,mac=52:54:00:CC:03:02 \
-netdev socket,id=link34,listen=0.0.0.0:6006 \
-device virtio-net-pci,netdev=link34,mac=52:54:00:CC:03:03 \
-nographic



```

- Change cxl_emu_4 start_vm.sh to this:
```
#! /bin/bash

TOTAL_MEMORY=16384


sudo qemu-system-x86_64 -name emucxlVM \
-machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
-cpu host -smp cpus=4 -m ${TOTAL_MEMORY}M \
-device virtio-scsi-pci,id=scsi0 \
-device scsi-hd,drive=hd0 \
-drive file=/home/exouser/qemu_images/emucxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
-netdev user,id=mng,hostfwd=tcp::8083-:22 \
-device virtio-net-pci,netdev=mng,mac=52:54:00:AA:BB:04 \
-netdev socket,id=link14,connect=149.165.175.123:6003 \
-device virtio-net-pci,netdev=link14,mac=52:54:00:CC:04:01 \
-netdev socket,id=link24,connect=149.165.173.250:6005 \
-device virtio-net-pci,netdev=link24,mac=52:54:00:CC:04:02 \
-netdev socket,id=link34,connect=149.165.174.164:6006 \
-device virtio-net-pci,netdev=link34,mac=52:54:00:CC:04:03 \
-nographic


```

## 3. IP setup
- Now we need to bash start_vm.sh for each cxl_emu_x in this order so listeners are set up first: 1->2->3->4.
- When logged into VM1 do this:
```
sudo ip link set ens5 up
sudo ip addr add 10.0.1.1/30 dev ens5

sudo ip link set ens6 up
sudo ip addr add 10.0.2.1/30 dev ens6

sudo ip link set ens7 up
sudo ip addr add 10.0.3.1/30 dev ens7
```
- When logged into VM2 do this:
```
sudo ip link set ens5 up
sudo ip addr add 10.0.1.2/30 dev ens5

sudo ip link set ens6 up
sudo ip addr add 10.0.4.1/30 dev ens6

sudo ip link set ens7 up
sudo ip addr add 10.0.5.1/30 dev ens7
```


- When logged into VM3 do this:
```
sudo ip link set ens5 up
sudo ip addr add 10.0.2.2/30 dev ens5

sudo ip link set ens6 up
sudo ip addr add 10.0.4.2/30 dev ens6

sudo ip link set ens7 up
sudo ip addr add 10.0.6.1/30 dev ens7
```
- When logged into VM4 do this:
```
sudo ip link set ens5 up
sudo ip addr add 10.0.3.2/30 dev ens5

sudo ip link set ens6 up
sudo ip addr add 10.0.5.2/30 dev ens6

sudo ip link set ens7 up
sudo ip addr add 10.0.6.2/30 dev ens7
```

## 3. Verification
- After configuration, test each point‑to‑point link:
```
# From VM1:
ping -I ens5 -c2 10.0.1.2   # → VM2
ping -I ens6 -c2 10.0.2.2   # → VM3
ping -I ens7 -c2 10.0.3.2   # → VM4

# From VM2:
ping -I ens5 -c2 10.0.1.1   # → VM1
ping -I ens6 -c2 10.0.4.2   # → VM3
ping -I ens7 -c2 10.0.5.2   # → VM4

# From VM3:
ping -I ens5 -c2 10.0.2.1   # → VM1
ping -I ens6 -c2 10.0.4.1   # → VM2
ping -I ens7 -c2 10.0.6.2   # → VM4

# From VM4:
ping -I ens5 -c2 10.0.3.1   # → VM1
ping -I ens6 -c2 10.0.5.1   # → VM2
ping -I ens7 -c2 10.0.6.1   # → VM3
```

