# One-to-One IP Connectivity Test with QEMU Socket Netdev

This README describes how to set up two Jetstream2 CXL-Emulator hosts and connect their nested QEMU guests over a point-to-point TCP socket network, assign them unique IPs, and verify L3 connectivity.

## 1. 
- Jetstream2 account with at least **2 running cxl_emu instances** (e.g. `cxl_emu_1-of-4` and `cxl_emu_2-of-4`).  
- SSH access to each instance as `exouser`:
'''
ssh exouser@IP_ADDRESS_OF_INSTANCE
'''

- `qemu_images/` directory on each host containing:
  - `emucxl.qcow2` (nested-VM disk)
  - `start_vm.sh` (VM launch script)

