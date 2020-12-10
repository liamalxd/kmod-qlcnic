---
kmod-qlcnic
---
I recently purchased a HP NC523SFP (593717-B21) 10Gb 2-Port NIC, these are extremely cheap on auction sites making them really useful for a homelab setup.

After some digging it looks like these are actually QLogic QLE3242 cards.

I'm currently running CentOS 8.3 on the server that NIC is going to call home, unfortunately it looks like the last driver that was produced for this OS was around CentOS 7.2. It also doesn't look like elrepo even has a kmod rpm or driver disk for this.

Fortunately, there's a maintained qlcnic driver in the Linux kernel repository (https://github.com/torvalds/linux/tree/master/drivers/net/ethernet/qlogic/qlcnic)

Using this I was able to compile a working version of the driver for the CentOS 8.3 kernel.

---
Modifications to the source driver
---
In order to get the driver to compile on CentOS 8.3 (with gcc-8.3.1-5.1), the following was carried out:

- Install the kernel devel/header packages (You may also need to install gcc)
```
dnf install kernel-devel kernel-headers
```

- Checkout the driver
```
git clone https://github.com/torvalds/linux.git
```

- Navigate to the driver 
```
cd linux/drivers/net/ethernet/qlogic/qlcnic
```

- Add the following to the Makefile:
```
obj-m += qlcnic-y qlcnic.o

all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

- Add the following to the top of qlcnic_main.c:
```
#include "udp_tunnel.h"
```

- Copy udp_tunnel.h from linux/include/net/udp_tunnel.h to the current directory

- Edit udp_tunnel.h so that the following conditional section looks like this (around line 294):
```
/*#ifdef CONFIG_INET
extern const struct udp_tunnel_nic_ops *udp_tunnel_nic_ops;
#else*/

#define udp_tunnel_nic_ops      ((struct udp_tunnel_nic_ops *)NULL)

/*#endif*/
```
(The driver will compile without this step, but it will throw an undefined symbol error for 'udp_tunnel_nic_ops' when you try to load the driver)

- Edit qlcnic_ethtool.c and change the two 'fallthrough;' lines to 'break;'

- Compile the driver
```
make
```

---
Installation
---

**Keep in mind, you will need to rebuild this driver if you update your specific kernel version**

Once the compile has finished, you should have 'qlcnic.ko' which should be placed on the box you want to load the driver on.

In my case I placed the driver (located in this repo as qlcnic_el8_3.ko) in /usr/lib/modules/4.18.0-240.el8.x86_64/extra/qlcnic/qlcnic.ko

Once this is done I added the following so that the driver would be loaded on boot:

/etc/modprobe.d/qlcnic.conf:
```
install qlcnic insmod /lib/modules/4.18.0-240.el8.x86_64/extra/qlcnic/qlcnic.ko
```

/etc/modules-load.d/qlcnic.conf:
```
qlcnic
```

You should be able to load the driver manually with:
```
modprobe qlcnic
```

Check dmesg to see everything is working:
```
[    8.514728] qlcnic 0000:03:00.0: 2048KB memory map
[    8.864471] qlcnic 0000:03:00.0: Default minidump capture mask 0x1f
[    8.864595] qlcnic 0000:03:00.0: FW dump enabled
[    8.864714] qlcnic 0000:03:00.0: Supports FW dump capability
[    8.864834] qlcnic 0000:03:00.0: Driver v5.3.66, firmware v4.20.1
[    8.895157] qlcnic: c4:34:6b:cc:91:18: NC523SFP 10Gb 2-port Server Adapter Board Chip rev 0x54
[    8.895569] qlcnic 0000:03:00.0: using msi-x interrupts
[    8.910954] qlcnic 0000:03:00.0: eth4: XGbE port initialized
[    8.911283] qlcnic 0000:03:00.1: 2048KB memory map
[    8.958914] qlcnic 0000:03:00.1: Default minidump capture mask 0x1f
[    8.959040] qlcnic 0000:03:00.1: FW dump enabled
[    8.959156] qlcnic 0000:03:00.1: Supports FW dump capability
[    8.959274] qlcnic 0000:03:00.1: Driver v5.3.66, firmware v4.20.1
[    8.989776] qlcnic 0000:03:00.1: using msi-x interrupts
[    8.990099] qlcnic 0000:03:00.1: eth5: XGbE port initialized
[   42.353296] qlcnic 0000:03:00.0 eth4: Rx Context[0] Created, state 0x2
[   42.814184] qlcnic 0000:03:00.0 eth4: Tx Context[0x8000] Created, state 0x2
[   42.828260] qlcnic 0000:03:00.0 eth4: Tx Context[0x8008] Created, state 0x2
[   42.843922] qlcnic 0000:03:00.0 eth4: Tx Context[0x800a] Created, state 0x2
[   42.859728] qlcnic 0000:03:00.0 eth4: Tx Context[0x800c] Created, state 0x2
[   43.445481] qlcnic 0000:03:00.1 eth5: Rx Context[1] Created, state 0x2
[   43.840882] qlcnic 0000:03:00.1 eth5: Tx Context[0x8001] Created, state 0x2
[   43.856661] qlcnic 0000:03:00.1 eth5: Tx Context[0x8009] Created, state 0x2
[   43.872545] qlcnic 0000:03:00.1 eth5: Tx Context[0x800b] Created, state 0x2
[   43.887414] qlcnic 0000:03:00.1 eth5: Tx Context[0x800d] Created, state 0x2
[   44.036945] qlcnic 0000:03:00.0 eth4: NIC Link is up
[   44.037663] qlcnic 0000:03:00.1 eth5: NIC Link is up
[   46.194569] qlcnic 0000:03:00.0 eth4: Rx Context[0] Created, state 0x2
[   46.204096] qlcnic 0000:03:00.0 eth4: Tx Context[0x8000] Created, state 0x2
[   46.217866] qlcnic 0000:03:00.0 eth4: Tx Context[0x8008] Created, state 0x2
[   46.233528] qlcnic 0000:03:00.0 eth4: Tx Context[0x800a] Created, state 0x2
[   46.249217] qlcnic 0000:03:00.0 eth4: Tx Context[0x800c] Created, state 0x2
[   46.913763] qlcnic 0000:03:00.1 eth5: Rx Context[1] Created, state 0x2
[   46.933273] qlcnic 0000:03:00.1 eth5: Tx Context[0x8001] Created, state 0x2
[   46.948954] qlcnic 0000:03:00.1 eth5: Tx Context[0x8009] Created, state 0x2
[   46.964590] qlcnic 0000:03:00.1 eth5: Tx Context[0x800b] Created, state 0x2
[   46.979249] qlcnic 0000:03:00.1 eth5: Tx Context[0x800d] Created, state 0x2
[   47.020991] qlcnic 0000:03:00.0 eth4: active nic func = 2, mac filter size=32
[   47.022036] qlcnic 0000:03:00.1 eth5: active nic func = 2, mac filter size=32
[   48.035180] qlcnic 0000:03:00.0 eth4: NIC Link is up
[   48.035301] qlcnic 0000:03:00.1 eth5: NIC Link is up
```

---
Future Notes
---

Ideally this should be packaged as part of an kmod RPM as per https://wiki.centos.org/HowTos/BuildingKernelModules rather than using modprobe.
