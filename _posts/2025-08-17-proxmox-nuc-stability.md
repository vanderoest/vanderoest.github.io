---
layout: post
title: Proxmox unstable on Intel NUC8i7BEB
subtitle: Classic bugsafari
cover-img: /assets/img/NUC6inside.jpg
thumbnail-img: /assets/img/datacenter1.jpg
share-img: /assets/img/NUC6inside.jpg
tags: [proxmox, nuc]
author: Arjan
---

I bought a used Intel NUC8i7BEB to replace my borked NUC7i7BNH (that story will be a separate post). The plan was to use it as a second Proxmox node in my home setup. Without much research I happily started installing Proxmox, only to find the machine showed erratic behaviour and crashed at random moments.  

First thing I noticed: it ran hot. During some `stress-ng` tests it immediately shot up to 100 degrees, with the fan spinning in full panic. Not great. Time to open it up, clean it, and apply fresh thermal paste. After tweaking the cooling profile in the UEFI settings the thermal situation improved a lot. Under load the CPU would rise slowly to 95°C at low fan speeds, then quickly drop back to a steady 75°C, even after an hour of stress testing. Idle temperatures sat comfortably between 35 and 40°C. But… Proxmox still crashed.  

During these “crashes” the console froze in place, the cursor disappeared, and the network would either become unresponsive (no ping) or still respond to ping but refuse SSH logins.  

After digging around it turned out the NUC8i7BEB is one of the worst choices for Proxmox, thanks to a messy combination of chipset quirks, energy management, and the kernel used in Proxmox 8.4. With the help of ChatGPT 5 and a lot of reading, I ended up with these tweaks:  

**Disable EEE on the `eno1` NIC:**  

```
auto eno1
iface eno1 inet manual
    post-up /sbin/ethtool --set-eee eno1 eee off || true
```

This prevents the weird network behaviour.  

**Mitigate GPU i915 driver issues** by adding this to `/etc/default/grub`:  


```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on i915.force_probe=* i915.enable_dc=0 i915.enable_psr=0"
```

Don’t forget:  
```
update-grub
```


