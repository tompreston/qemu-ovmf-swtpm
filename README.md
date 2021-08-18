# Setting up QEMU with OVMF (UEFI) and swtpm (software TPM emulation)
This document is a step by step guide to setting up TPM emulation in QEMU with a OVMF.

I use Fedora, but I think the packages are similar on Debian.

There are a number of moving parts, but from the top-down:
* User can read TPM measurements in Linux guest OS via securityfs, when booted with UEFI firmware.
* Guest OS needs to be installed in a UEFI compatible way (installer started in UEFI mode).
* Installing an OVMF package gives us the binaries required for OVMF firmware (UEFI) in QEMU.
* Installing a swtpm package gives us the programs required software TPM emulation.
* QEMU needs to be started with flags for both OVMF and swtpm.

## Step 1: Install QEMU, OVMF and swtmp packages

    dnf install @virtualization edk2-ovmf swtpm


## Step 2: Install Debian into a VM with OVMF
Copy the OVMF files to the local dir:

    cp /usr/share/OVMF/OVMF_CODE.fd . # OVMF_CODE.fd has restricive permissions
    cp /usr/share/OVMF/OVMF_VARS.fd . # OVMF_VARS.fd contains writable state

Create a virtual disk image:

    qemu-img create debian.img 5G

Run QEMU with the following script and install Debian:

    #!/bin/sh
    # run-qemu-ovmf.sh
    qemu-system-x86_64 \
            -m 1024 \
            -drive file=OVMF_CODE.fd,format=raw,if=pflash \
            -drive file=OVMF_VARS.fd,format=raw,if=pflash \
            -cdrom debian-testing-amd64-netinst.iso \
            -drive file=debian.img,format=raw

Subsequent boots do not require the `cdrom` line.

Notes:
* You need to mount both `OVMF_CODE.fd` and `OVMF_VARS.fd` to the pflash interface.
* I used debian/insecure as a user/pass.
* In the future I should use a [Debian pre-seed](https://wiki.debian.org/DebianInstaller/Preseed)
  to automate the install.


## Step 3: Start swtpm and pass it to QEMU
Start the software TPM UNIX domain socket:

    mkdir /tmp/mytpm1
    swtpm socket \
        --tpmstate dir=/tmp/mytpm1 \
        --ctrl type=unixio,path=/tmp/mytpm1/swtpm-sock \
        --log level=20

Run QEMU, passing in the emulated TPM device:

    #!/bin/sh
    # run-qemu-ovmf.sh
    qemu-system-x86_64 \
        -m 1024 \
        -drive file=OVMF_CODE.fd,format=raw,if=pflash \
        -drive file=OVMF_VARS.fd,format=raw,if=pflash \
        -drive file=debian.img,format=raw \
        -chardev socket,id=chrtpm,path=/tmp/mytpm1/swtpm-sock \
        -tpmdev emulator,id=tpm0,chardev=chrtpm \
        -device tpm-tis,tpmdev=tpm0

Notes:
* The [QEMU tpm doc](https://github.com/qemu/qemu/blob/master/docs/specs/tpm.rst#the-qemu-tpm-emulator-device)
  helped the most here.
* The `-cdrom` line has been removed.


## Step 4: Dump TPM measurements
Boot the VM and log in. Check for the TPM driver in dmesg and dump the measurements:

    sudo dmesg | grep -i tpm
    sudo cat /sys/kernel/security/tpm0/ascii_bios_measurements
    sudo xxd /sys/kernel/security/tpm0/binary_bios_measurements

Notes:
* I've attached a screenshot.


## Further Reading
Here are a number of documents I found useful:
* https://en.opensuse.org/Software_TPM_Emulator_For_QEMU
* https://libvirt.org/formatdomain.html#elementsTpm
* https://github.com/qemu/qemu/blob/master/docs/specs/tpm.rst#the-qemu-tpm-emulator-device
* https://github.com/tianocore/tianocore.github.io/wiki/How-to-run-OVMF
* https://wiki.debian.org/UEFI
* https://wiki.debian.org/QEMU#Setting_up_a_stable_system
* https://wiki.ubuntu.com/UEFI/OVMF
* https://fedoraproject.org/wiki/Using_UEFI_with_QEMU
* https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface#UEFI_Shell
