QEMU_DISPLAY ?= none
ARCH ?= x86
ifeq ($(ARCH),x86)
b = b
endif

ZIMAGE = $(KDIR)/arch/$(ARCH)/boot/$(b)zImage
KCONFIG = $(KDIR)/.config
NTTCP = $(KDIR)/tools/labs/templates/assignments/6-e100/nttcp

YOCTO_URL = http://downloads.yoctoproject.org/releases/yocto/yocto-2.3/machines/qemu/qemu$(ARCH)/
YOCTO_IMAGE = core-image-minimal-qemu$(ARCH).ext4
#YOCTO_IMAGE = core-image-minimal-dev-qemu$(ARCH).ext4
#YOCTO_IMAGE = core-image-sato-dev-qemu$(ARCH).ext4
#YOCTO_IMAGE = core-image-sato-qemu$(ARCH).ext4
#YOCTO_IMAGE = core-image-sato-sdk-qemu$(ARCH).ext4

QEMU_OPTS = -kernel $(ZIMAGE) \
	-device virtio-serial \
	-chardev pty,id=virtiocon0 -device virtconsole,chardev=virtiocon0 \
	-serial pipe:pipe1 -serial pipe:pipe2 \
	-netdev tap,id=lkt-tap0,ifname=lkt-tap0,script=no,downscript=no -net nic,netdev=lkt-tap0,model=virtio \
	-netdev tap,id=lkt-tap1,ifname=lkt-tap1,script=no,downscript=no -net nic,netdev=lkt-tap1,model=i82559er \
	-drive file=$(YOCTO_IMAGE),if=virtio,format=raw \
	-drive file=disk1.img,if=virtio,format=raw \
	-drive file=disk2.img,if=virtio,format=raw \
	--append "root=/dev/vda loglevel=15 console=hvc0 pci=noacpi" \
	--display $(QEMU_DISPLAY) -s -m 256

ifdef ENABLE_KVM
QEMU_OPTS += -enable-kvm -cpu host
ENABLE_KVM_SUDO = sudo
endif
#boot流程如下:
#(1) 编译内核代码(在.modinst里面)
#(2) kconfig: 先把事先准备好的.config文件拷贝到内核目录下(这个应该本来是用make config, 或者make memuconfig 配置的)
#(3) make olddefconfig 使用之前复制的.config配置内核
#(4) make mod2yesconfig 应该是把所有内核模块都加入构建
#(5) make 构建内核
#(6) make modules 编译模块（这里貌似没有）
#(7) 下载yocto镜像，就是个文件系统(里面有些linux命令行之类的)，调用prepare-image脚本开始搞镜像
#(8) 裁剪镜像大小（这个暂时不明白），挂载，在镜像目录上make modules_install，安装模块
#(9) 创建一个空文件.modinst告诉makefile内核已经编译过了
#(10) 下面开始创建网络，
#(11) ip tuntap add 添加网卡 172.213.0, 172.30.0
#(12) dnsmasq 配置dhcp
#(13) 创建管道
#(14) 创建两个100M的文件系统
boot: .modinst lkt-tap0 lkt-tap1 pipe1.in pipe1.out pipe2.in pipe2.out disk1.img disk2.img nttcp-run
	$(ENABLE_KVM_SUDO) ARCH=$(ARCH) qemu/qemu.sh $(QEMU_OPTS)

TEMPDIR := $(shell mktemp -u)

$(KCONFIG): qemu/kernel_config.x86
	cp $^ $@
	$(MAKE) -C $(KDIR) olddefconfig
	$(MAKE) -C $(KDIR) mod2yesconfig

zImage: $(ZIMAGE)

$(ZIMAGE): $(KCONFIG)
	$(MAKE) -C $(KDIR)
	$(MAKE) -C $(KDIR) modules

.modinst: $(ZIMAGE) $(YOCTO_IMAGE)
	mkdir $(TEMPDIR)
	sudo mount -t ext4 -o loop $(YOCTO_IMAGE) $(TEMPDIR)
	sudo $(MAKE) -C $(KDIR) modules_install INSTALL_MOD_PATH=$(TEMPDIR)
	sudo umount $(TEMPDIR)
	rmdir $(TEMPDIR)
	sleep 1 && touch .modinst

gdb: $(ZIMAGE)
	gdb -ex "target remote localhost:1234" $(KDIR)/vmlinux

$(YOCTO_IMAGE):
	wget $(YOCTO_URL)/$(YOCTO_IMAGE)
	sudo ARCH=$(ARCH) qemu/prepare-image.sh $(YOCTO_IMAGE)

lkt-tap0:
	qemu/create_net.sh $@

lkt-tap1:
	qemu/create_net.sh $@

lkt-tap-smbd:
	qemu/create_net.sh $@


nttcp-run: $(NTTCP) lkt-tap1
	$(NTTCP) -v -i &

pipe1.in:
	mkfifo $@

pipe1.out:
	mkfifo $@

pipe2.in: pipe1.out
	ln $< $@

pipe2.out: pipe1.in
	ln $< $@

disk0.img:
	qemu-img create -f raw $@ 100M

disk1.img:
	qemu-img create -f raw $@ 100M

disk2.img:
	qemu-img create -f raw $@ 100M

clean::
	-rm -f .modinst
	-rm -f disk1.img disk2.img
	-rm -f pipe1.in pipe1.out pipe2.in pipe2.out

# Run with one of the following:
#     make -j$(nproc) console
#     make -j$(nproc) gui

# Attach debugger with
#     make gdb

# Stop with
#     sync # make sure all filesystem changes are propagated
#     CTRL-A X # kill qemu, faster and more reliable than poweroff etc.

# Compile in skel directories with something like:
#     alias kmake='make -C "$HOME/src/linux/" M="$(pwd)"'
#     kmake

YOCTO_ROOTFS = core-image-minimal-qemu$(ARCH).tar.bz2

console: $(ZIMAGE) rootfs lkt-tap0 lkt-tap1 lkt-tap-smbd disk0.img disk1.img disk2.img pipe1.in pipe1.out pipe2.in pipe2.out
	MODE=console qemu/run-qemu.sh

checker: $(ZIMAGE) rootfs lkt-tap0 lkt-tap1 lkt-tap-smbd disk0.img disk1.img disk2.img pipe1.in pipe1.out pipe2.in pipe2.out
	MODE=checker qemu/run-qemu.sh

gui: $(ZIMAGE) rootfs lkt-tap0 lkt-tap1 lkt-tap-smbd disk0.img disk1.img disk2.img pipe1.in pipe1.out pipe2.in pipe2.out
	MODE=gui qemu/run-qemu.sh

rootfs: $(YOCTO_ROOTFS)
	mkdir -p rootfs
	tar -xf core-image-minimal-qemux86.tar.bz2 -C rootfs
	sed -i 's@/sbin/getty@& -n -l "/sbin/rootlogin"@' rootfs/bin/start_getty
	printf '%s\n' '#!/bin/sh' '/bin/login -f root' > rootfs/sbin/rootlogin
	chmod +x rootfs/sbin/rootlogin
	mkdir -p rootfs/home/root/skels
	echo "//10.0.2.1/skels /home/root/skels cifs port=4450,guest,user=dummy 0 0" >> rootfs/etc/fstab
	echo "hvc0:12345:respawn:/sbin/getty 115200 hvc0" >> rootfs/etc/inittab

$(YOCTO_ROOTFS):
	wget $(YOCTO_URL)/$(YOCTO_ROOTFS)


.PHONY: console gui boot gdb clean lkt-tap0 lkt-tap1 lkt-tap-smbd
