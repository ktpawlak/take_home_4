# Testing software emulated TPM

## Purpose

The goal of this tutorial it to show software emulated TPM working with Ubuntu Core.

## Configure Ubuntu Core account
Follow instructions at: https://ubuntu.com/core/docs/uc20/install#heading--requirements to configure your Ubuntu SSO account. 

## Get latest Ubuntu docker image
The test environment will be built inside docker container
```
docker pull ubuntu:rolling
```

## Run the container
```
xhost +
docker run -e "DISPLAY=${DISPLAY:-:0.0}" --privileged -it --net=host ubuntu:rolling /bin/bash
```

## Prepare the environment
```
apt update && apt install dh-autoreconf libssl-dev \
     libtasn1-6-dev pkg-config libtpms-dev \
     net-tools iproute2 libjson-glib-dev \
     libgnutls28-dev expect gawk socat \
     libseccomp-dev make git gawk qemu-kvm wget -y
```

## Compile swtmp
```
git clone https://github.com/stefanberger/swtpm.git
cd swtpm
./autogen.sh --with-openssl --prefix=/usr
make -j4 && make -j4 check && make install
cd ..
```
## Start SWTPM
<pre>
mkdir <b>./mytpm0</b>
swtpm socket --tpm2 --tpmstate dir=./mytpm0 --ctrl type=unixio,path=<b>./mytpm0/</b>swtpm-sock --log level=20 &
</pre>

## Download Ubuntu Core Amd64 image
```
wget https://cdimage.ubuntu.com/ubuntu-core/20/stable/current/ubuntu-core-20-amd64.img.xz
xz -d ubuntu-core-20-amd64.img.xz
```

## Start qemu image:
<pre>
qemu-system-x86_64 -smp 2 -m 2048 -net nic,model=virtio -net user,hostfwd=tcp::8022-:22,hostfwd=tcp::8090-:80 -vga qxl -drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,unit=0,readonly=on -drive file=<b>ubuntu-core-20-amd64.img</b>,cache=none,format=raw,id=disk1,if=none -device virtio-blk-pci,drive=disk1,bootindex=1 -machine accel=kvm -chardev socket,id=chrtpm,path=<b>./mytpm0/</b>swtpm-sock -tpmdev emulator,id=tpm0,chardev=chrtpm -device tpm-tis,tpmdev=tpm0
</pre>
You will be asked to configure the machine.

# Obtain bios measurements
<pre>
From outside docker container call:
ssh -p 8022 <b>&lt;SSO Username&gt;</b>@localhost "sudo cat /sys/kernel/security/tpm0/binary_bios_measurements" | strings | grep cmdline
</pre>
