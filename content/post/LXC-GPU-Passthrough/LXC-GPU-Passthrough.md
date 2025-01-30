---
title: Sharing a GPU with multiple LXCs
description: Get the full functionality from your GPU within an LXC, and keep it available for other LXCs and host processes.
slug: LXC-GPU-Passthrough
date: 2025-01-27 00:26:00+0000
categories:
    - Self-Hosting
tags:
    - LXC
    - Virtualization
    - Hardware Passthrough
draft: false
---

## Background
Every now and again on some of the Discord server's where I'm a member, people are posting questions about sharing GPUs with multiple unprivileged LXCs in proxmox. I think I've seen questions pop up every one or two weeks from someone. It's to the point I've got a copy-pasta in my markdown notes on the research that I've done to get my setup with Frigate and Jellyfin working. So I figured I'd finally sit down, clean up my notes, and publish this generic guide for anyone wanting to do the same thing. I'll cover nested GPU passthrough (i.e. for Docker in an LXC) in another blog post.

### Reference Material

[Dougiebabe's discussion on the frigate github page](https://github.com/blakeblackshear/frigate/discussions/5773) was of immense help to me when I set all this up the first time. I wanted to avoid running `chmod 666` on my graphics card though, so I spent some time figuring out the proper users to pass through to my LXC. The comments on this [post on the proxmox subreddit](https://www.reddit.com/r/Proxmox/comments/mttwtf/gpu_passthrough_for_unpriviliged_containers/) were a big help in wrapping my mind around what I was trying to do. As was the [official proxmox documentation](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers).

## Passing through the GPU to the LXC

I am assuming you've already created an unprivileged container in proxmox's web UI. I am using a Debian 12 container on my machine, and I'll be passing through a 12th gen Intel i5 to the LXC.

### Major and Minor numbers from the Host

Run `lspci -nnv | grep VGA` on the proxmox host to confirm your GPU is detected. When I run this I see the following for my iGPU and Nvidia 1660Ti.

```
00:02.0 VGA compatible controller [0300]: Intel Corporation Alder Lake-S GT1 [UHD Grap  
hics 770] [8086:4690] (rev 0c) (prog-if 00 [VGA controller])  
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 T  
i] [10de:2182] (rev a1) (prog-if 00 [VGA controller])  
root@lovelace:~# lspci -nnv | grep VGA  
00:02.0 VGA compatible controller [0300]: Intel Corporation Alder Lake-S GT1 [UHD Grap  
hics 770] [8086:4690] (rev 0c) (prog-if 00 [VGA controller])  
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 T  
i] [10de:2182] (rev a1) (prog-if 00 [VGA controller])
```

If you have two GPUs like me you need to be able to tell which is which. To do this, run `ls -l /sys/class/drm/renderD*/device/driver` to find the drivers used by these GPUs. This gives me the following output:

```
lrwxrwxrwx 1 root root 0 Jan 27 12:04 /sys/class/drm/renderD128/device/driver -> ../..  
/../bus/pci/drivers/i915  
lrwxrwxrwx 1 root root 0 Jan 27 12:04 /sys/class/drm/renderD129/device/driver -> ../..  
/../../bus/pci/drivers/nouveau
```

 I'll also run  `ls -l /sys/class/drm/card*/device/driver` which gives me the following:

```
lrwxrwxrwx 1 root root 0 Jan 27 12:04 /sys/class/drm/card0/device/driver -> ../../../.  
./bus/pci/drivers/nouveau  
lrwxrwxrwx 1 root root 0 Jan 27 12:04 /sys/class/drm/card1/device/driver -> ../../../b  
us/pci/drivers/i915
```
`i915` is the driver for Intel, and `nouveau` is the driver for Nvidia cards. So `card1` and `render128`is my intel iGPU, and `card0` and `render129` is my Nvidia discrete GPU.

 Now that you know which is which, lets run `ls -l /dev/dri` on the proxmox host to get our major and minor numbers. These numbers appear after the User and Group in the `ls -l` output.

```
drwxr-xr-x 2 root root        120 Jan 20 08:26 by-path  
crw-rw---- 1 root video  226,   0 Jan 17 14:16 card0  
crw-rw---- 1 root video  226,   1 Jan 20 08:26 card1  
crw-rw---- 1 root render 226, 128 Jan 17 14:16 renderD128  
crw-rw---- 1 root render 226, 129 Jan 17 14:16 renderD129
```

So we can see for `card1` my major number is `226` and my minor number is `0`. For `render128` my major number is `226` and my minor number is `128`. Keep these handy, we'll need them in the next step.

### Passing through the device to the LXC

If your LXC is running, shut it off first and then in your proxmox host's console navigate to `/etc/pve/lxc/`. You should find a file there named with your CT ID and `.conf`, for me this is `100.conf`, yours will differ.

At the bottom of the lxc.conf add the following. Replace my major and minor numbers (`226:0` and `226:128`) / device path (`/dev/dri/renderD128` and `/dev/dri/card1`) with the ones on your proxmox host.

```
lxc.cgroup2.devices.allow: c 226:0 rwm  
lxc.cgroup2.devices.allow: c 226:128 rwm  
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0, 0  
lxc.mount.entry: /dev/dri/card1 dev/dri/card1 none bind,optional,create=file 0, 0
```

### Background on UID and GID Mapping

You can skip to [Mapping the GIDs](#mapping-the-gids) if you just want the next steps. This section helps you understand why you need to do all of this mapping.

If you were to start your LXC now and run `ls -l /dev/dri` you'll see something like the following. You'll note only my iGPU is now visible. The 1660Ti on the other hand is still safely walled off from this LXC.

```
total 0
crw-rw---- 1 nobody nogroup 226,   0 Jan 17 20:16 card1
crw-rw---- 1 nobody nogroup 226, 128 Jan 17 20:16 renderD128
```

The `nobody` `nogroup` you see above is due to the way the host maps users to the unprivileged LXC to prevent overlap with the host system. On an unprivileged LXC the UID for root *appears to be* `0` when you query the value from within the LXC. However, if you were to track the processes started by the LXC on the proxmox host, you would see the UID for `root` on that LXC is actually `100000`. This is to provide separation between the host's privileged accounts and the LXC. This is also why a privileged LXC can be so easy to set up, but also more dangerous. Because this mapping does not happen, the privileged LXC's `root` user maps directly to the host's `root` user, making malware escape far easier. If an unprivileged LXC's `root` account is compromised in this way, that `root` user is mapped to an unprivileged user on the proxmox host making malware escape far more difficult. Privileged LXCs are still protected through other means like [apparmor](https://apparmor.net/), but they lack the "safe by design" philosophy of an unprivileged LXC and in the words of the project they [*"aren't and cannot be root-safe."*](https://linuxcontainers.org/lxc/security/#privileged-containers)

[Linuxcontainers.org's security page](https://linuxcontainers.org/lxc/security/) has an approachable breakdown on the steps they've taken to secure both privileged and unprivileged LXCs if you'd like to learn more.

This user mapping is a great security feature, but it does mean we're not done yet. We need to map the `video` and `render` groups on the host to the `video` and `render` groups on the LXC so that our LXC has permission to use the GPU we just passed through. This will mean that if malware on an unprivileged LXC breaks through the other protections like it would in a privileged LXC, it would be able to control processes on the GPU. So we have the same problem as a privileged LXC in that sense.

However, we are locking down the rest of the system. So while an infected LXC could conceivably see, kill, and modify transcoding processes in progress started by another LXC using the same GPU passthrough method, it is blocked from accessing files and other processes not running on the GPU. So by mapping the users and not using privileged containers, we are limiting the possible scope of any malware that might find its way onto our LXCs.

### Mapping the GIDs

First, we need to find the GID for the `video` and `render` groups on the proxmox host. Simply run `getent group video` and `getent group render` which will give you something like below:

``` 
video:x:44:    
render:x:104:
```

Your output may be different so do not blindly copy my config unless you're certain you have the same GIDs.

For my config, I need to pass through GID `44` and `104` to the LXC. Once again find your `.conf` file at `/etc/pve/lxc/`. The re-mapping we're about to do can get a little confusing as we're adding way more than just two statements to re-map only two groups. The reason for this is we need to completely override the LXC's default mapping to prevent duplicates.

For example, if we only map our video group ID `44` on the host to `44` in the container. The LXC will still map all other groups to 100000+. So, we'll have a video group with GID `44`, but we'll also have a container video group with GID `100044`. The LXC software will thankfully not put up with this duplicate nonsense and will refuse to start.

To account for this, we need to map all GIDs up to `44` to those `100000` GIDs, put an exception to this mapping for group `44`, and then continue mapping to those other `100000` positions, making the same exception for the `render` GID of `104`. This means you have a little bit of arithmetic to do, but I promise it's not as complicated as it sounds at first glance.

First, in your lxc.conf add the following. This will be the same for your system as it is mine, unless you're also mapping a specific user to the container in which case you'll need the arithmetic we're going to do in the next section for your user mappings as well.

```
lxc.idmap: u 0 100000 65536
```

Lets walk through this statement. `lxc.idmap:` is the command for mapping UIDs and GIDs, `u` specifically calls out that this is a UID mapping. `0` tells us we're starting with the UID `0` on the LXC (which is `root`). `100000` tells us that the `0` user will be mapped to `100000` on the host. `65536` tells us we will continue in this manner for 65,536 UIDs ending at UID `65535` on the LXC and `165535` on the host (`65535` being the maximum UID).

This gets a little more complicated when we try to add the GIDs. But let me walk you through it. 

First, we need to add a group mapping from 0 up to just before the video group's GID. So in my case, I'll add the below to my config.

```
lxc.idmap: g 0 100000 44
```

This will remap all *GIDs* starting a `0` on the LXC and `100000` on the host, and continue for `44` GIDs (including the 0th mapping, watch for off by one errors in your calculations). This ends us at GID `43` just before the `video` GID on my host. You will want to replace `44` with the GID of the `video` group on your host system in the above line.

Next, I'll add to this:

```
lxc.idmap: g 44 44 1
```

This is much simpler to understand, we're mapping GID `44` on the LXC to GID `44` on the host and continuing for `1` GID. So, this only remapping GID `44`. Replace `44` with the GID for the `video` group on your host.

Now, we need to remap every ID between `video` and `render`. In my case `render` is GID `104`, so I want to remap everything up to and including GID `103` with this next statement. To do this, I'll add the following to my config.

```
lxc.idmap: g 45 100045 58
```

We start at `45` and `100045` because we've already mapped all GIDs up to that point. We do this `58` times since we want to end up at `103` and `103 - 45 = 58` (watch for off by one errors in your calculations). To make it easier to find what should go where on your system you can use the below formula.

```
lxc.idmap: g ($VIDEO_GID + 1) ($VIDEO_GID + 100,001) (($RENDER_GID - 1) - ($VIDEO_GID + 1)) 
```

Now we can finally map the `render` GID. Replace `104` below with the GID of the `render` group on your host.

```
lxc.idmap: g 104 104 1
```

Once again, we're starting at `104` on the LXC and mapping to `104` on the host and repeating this `1` time. So we're only mapping GID `104`.

We just have one more section to add, we need to map all other groups. In my case, I add the following to my config.

```
lxc.idmap g 105 100105 65431
```

We have already mapped up to `105` on the LXC and `100105` on the host so that is self explanatory. We do this `65431` times because we want to map everything up to *and including* the maximum GID of `65535`.  You can use the below formula to help you find the values on your system.

```
lxc.idmap g ($RENDER_GID + 1) ($RENDER_GID + 100001) (65535 - $RENDER_GID)
```

For reference, here are all of the lines I added to my config in this section. Again, do not blindly copy this config unless your sure your GIDs are the same.

```
lxc.idmap: u 0 100000 65536  
lxc.idmap: g 0 100000 44  
lxc.idmap: g 44 44 1  
lxc.idmap: g 45 100045 58  
lxc.idmap: g 104 104 1  
lxc.idmap: g 105 100105 65431
```

### Allowing the Mapping

Before the LXC will launch, we need to allow the user that starts the LXC (`root` on proxmox), to actually perform the mappings we've configured for our LXCs. This is done by editing `/etc/subGID` and adding the following:

```
root:104:1  
root:44:1
```

Working left to right, the above allows the `root` user to map GIDs starting at `104` or `44` and allows this mapping for the next `1` items. So, it allows `root` to map `44` and `104`.

Note: keep any lines already in this file. They allow the LXC to perform the mapping to unprivileged users I mentioned earlier.

## GPU Monitoring

### Disclaimer

At this point you will be able to launch the LXC, and you will have full access to the GPU you passed through. The only thing you will not have access to is monitoring tools like `intel_gpu_top`. Attempting to run it in your LXC will give you the following error.

```
Failed to initialize PMU! (Permission denied)
```

You can get this working as well, but if you don't need monitoring (and *really* ask yourself if you *actually* need monitoring *within* the LXC) you should stop here. Your container is able to utilize the GPU just fine as it is now, it just wont be able to show you any statistics as those are locked away on the host. Enabling performance monitoring through the workaround I'm about to show you, will give **all** of your LXCs and other users on your hypervisor permission to do the same. How dangerous this actually ends up being, I'm not exactly sure. However, I would certainly not perform this on a server that housed or processed any sensitive data. The LXC and other users on the machine will have permission to see any processes on the GPU, but unless they also have access to those `video` and `render` groups I don't believe they will be able to access the GPU directly and modify those processes. Regardless of if that is the case, if you don't have a serious need for monitoring inside the LXC, do not perform this workaround - I don't even use it on my server.

As an alternative, you can install your monitoring tools like `intel-gpu-tools` on the proxmox host directly, and that is what I recommend. I know this is anathema in some circles since you're modifying the host, but if you need monitoring and don't want to cause a potential security problem, it's really your best bet at the time of writing.

### Workaround for Monitoring Within the LXC (not recommended)

If you **need** monitoring *inside* the LXC. The only way I have been able to get this to work is by changing the [perf_event_paranoid level in the kernel](https://www.kernel.org/doc/html/v5.1/admin-guide/perf-security.html#perf-events-perf-unprivileged-users) of the proxmox host. Hopefully, that sentence makes you realize why I really don't recommend you do this. Here be dragons. By default on proxmox this is set to `4`, if you set the value to `0` then suddenly inside your LXC you stop receiving the permission denied error when running `intel_gpu_top`. You can safely do this as a quick test with the following command.

```
sysctl kernel.perf_event_paranoid=0
```

You can now run `intel_gpu_top` within your LXC and it will work as expected, showing you all processes on the LXC that are running on the GPU, This value will reset to `4` after you reboot your proxmox server, but if you want to revert it without a reboot simply change it back manually and it will take effect immediately.

If you want this to persist after a reboot of the proxmox host, you can create the file `/etc/sysctl/local.conf` and paste the same command you used above into that file.

## Recap and next steps
You now have your GPU available for use inside your LXC, and you can use these same steps to share that GPU between multiple LXCs as well as the host. If you'd like to learn more about why GPU monitoring within an LXC is such a difficult topic, check out my [next blog post](https://midwesternrodent.com/p/gpu-monitoring_problems/) whenever I publish it in the next day or so. This one was getting a little long to add potentially irrelevant information. Until next time!