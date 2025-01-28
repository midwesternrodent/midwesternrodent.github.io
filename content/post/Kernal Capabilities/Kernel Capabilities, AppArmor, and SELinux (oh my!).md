---
title: Kernel Capabilities, AppArmor, and SELinux (oh my!)
description: Where I use Cunningham's Law to show why it's not possible to get GPU monitoring tools working within an LXC.
slug: GPU-Monitoring_Problems
date: 2025-01-27 00:26:00+0000
categories:
    - Self-Hosting
tags:
    - LXC
    - Virtualization
    - Docker
    - Proxmox
    - Docker-Compose
    - Linux Kernel Capabilities
    - PAM
draft: true
---
## Kernel Capabilities, AppArmor, and SELinux (oh my!)

You can skip this section if you'd like, there are no instructions to be found here. Instead, I want to share what I learned from my deep dive on GPU monitoring within an LXC, and why I've come to the definitive conclusion that [**it is 100% not possible to do safely today**](https://en.wikipedia.org/wiki/Ward_Cunningham#%22Cunningham's_Law%22). If you don't care, feel free to head to the next section where the instructions continue.

For some more helpful error messages If we run `intel_gpu_top` in the LXC as a normal user we get the following which is a bit more helpful than the one liner we get as `root`.

```
Failed to initialize PMU! (Permission denied)  
When running as a normal user CAP_PERFMON is required to access performance  
monitoring. See "man 7 capabilities", "man 8 setcap", or contact your  
distribution vendor for assistance.  
  
More information can be found at 'Perf events and tool security' document:  
https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html
```

Knowing the [background on uid and gid mapping](#Background-on-UID-and-GID-Mapping), namely that privileged users and groups are mapped to unprivileged IDs between the LXC and the host, you might think like I did that we're just missing a permission on our LXC's host-side `root` account with the UID `100000`. All we need to do is add that permission and things should work. However, if you dig into the [kernel documentation on perf access control](https://www.kernel.org/doc/html/v5.1/admin-guide/perf-security.html#perf-events-perf-access-control) you'll see that things aren't so simple. For instance, `CAP_PERFMON` that is called out by `intel_gpu_top` as required, is not a user level permission, but an executable permission we need to give to `/usr/bin/intel_gpu_top`. Obviously, if we do this on our host it won't have any affect on the container. The same is true in the reverse, if our LXC gives it's copy of `/usr/bin/intel_gpu_top` the `CAP_PERFMON` capability that is all well and good as far as the LXC is concerned, but as soon as that process is sent to the host and the ID of the user calling it changes to `100000`, per [the capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html) `CAP_PERFMON` is stripped off along with any other capabilities of `intel_gpu_top`'s process and it fails with the exact same error as before.

>[**Effect of user ID changes on capabilities**](https://man7.org/linux/man-pages/man7/capabilities.7.html)
>
>To preserve the traditional semantics for transitions between 0
>and nonzero user IDs, the kernel makes the following changes to a
>thread's capability sets on changes to the thread's real,
>effective, saved set, and filesystem user IDs (using setuid(2),
>setresuid(2), or similar):
>
>If one or more of the real, effective, or saved set user IDs
>**was previously 0**, and as a result of the UID changes all of
>**these IDs have a nonzero value**, then **all capabilities are
>cleared** from the permitted, effective, and ambient capability
>sets.
>If the **effective user ID is changed from 0 to nonzero**, then
>**all capabilities are cleared** from the effective set.


In [another section of the perf events and tool security documentation for the Linux Kernel](https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html#privileged-perf-users-groups) we learn that it's possible to create a privileged group for performance monitoring. So lets just create one of those groups, pass it through to our LXC, and add our `root` user on the LXC to that group. In theory, we should be able to use `intel_gpu_top` without issue after performing these steps. Except that's where things get even weirder. I performed these steps as a test, following the same rules for [mapping the GIDs](#Mapping-the-GIDs) as previously. 

On my proxmox host and the LXC, I created a user with the UID and GID of `5000`. For simplicity, I just called them `perfmon`. 
```bash
groupadd -g 5000 perfmon
useradd perfmon -u 5000 -g 5000 -m -s /bin/bash
```

I then changed the ownership of `/usr/bin/intel_gpu_top` on my proxmox host to make the `perfmon` group the owner, and set the permissions to restrict the file according to what is on the kernel.org documentation.

```bash
chown root:perfmon /usr/bin/intel_gpu_top
chmod 750 /usr/bin/intel_gpu_top
```

I then changed some lines in `/etc/pve/lxc/100.conf` to allow this user to be passed through to the LXC. Just for testing purposes, I also decided to pass through the whole `/usr/bin/` directory on the proxmox host to the perfmon's home folder inside the LXC (not recommend for what are hopefully obvious reasons). So now all of the lines I've added to my LXC config now look like below.

```
mp0: /usr/bin,mp=/home/perfmon/bin  
lxc.cgroup2.devices.allow: c 226:0 rwm  
lxc.cgroup2.devices.allow: c 226:128 rwm  
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0, 0  
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file 0, 0  
lxc.idmap: u 0 100000 5000  
lxc.idmap: u 5000 5000 1  
lxc.idmap: u 5001 105001 60535  
lxc.idmap: g 0 100000 44  
lxc.idmap: g 44 44 1  
lxc.idmap: g 45 100045 58  
lxc.idmap: g 104 104 1  
lxc.idmap: g 105 100105 4894  
lxc.idmap: g 5000 5000 1  
lxc.idmap: g 5001 105001 60535
```

Lastly, let's get ridiculous on the proxmox host with the kernel capabilities of `intel_gpu_top`. The below command goes way above what should be needed for this binary.

```
setcap 'cap_perfmon,cap_sys_ptrace,cap_syslog,cap_sys_admin=ep' /usr/bin/intel_gpu_top
```

And let's not forget the `/etc/subgid` and `/etc/subuid` changes we need to make. Adding the following to each file.

```
root:5000:1
```

After all this, surely `intel_gpu_top` will now work in the unprivileged LXC. Let's give it a shot!

```
perfmon@lxc-blog:~/$ /home/bin/intel_gpu_top
Failed to initialize PMU! (Permission denied)

When running as a normal user CAP_PERFMON is required to access performance  
monitoring. See "man 7 capabilities", "man 8 setcap", or contact your  
distribution vendor for assistance.  
  
More information can be found at 'Perf events and tool security' document:  
https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html

perfmon@lxc-blog:~/$ $#QRV#$TB@A%W$QY^BW%#$Q%#QV%QVYVTY%$$
-bash: 0QRV#@A%W^BW%#%#QV%QVYVTY%767697: command not found
perfmon@lxc-blog:~/$
```

At this point, some curse words may have been uttered. However, in retrospect this makes sense. Remember, UID and GID mapping is not the only way the host is protected from unprivileged LXCs. Linux containers also take advantage of AppArmor and SELinux. Again, for **testing purposes** we can disable AppArmor fairly easily, I have not yet found a way to do this with SELinux.

```
lxc.apparmor.profile: unconfined
```

But even when this is done, the permission denied error persists. The only way I have gotten this error to resolve, is by changing `perf_event_paranoid` on the host, or by setting the LXC to privileged which gets around the UID remapping problem we keep bumping into, but we lose that protection as well.

I'm not completely sure why this continues to fail, but I have a hypothesis. A privileged LXC is started with certain capabilities such as `CAP_SYS_ADMIN`. This is why in our LXC.conf file we could add entries like `lxc.cap.keep: perfmon` or `lxc.cap.drop: sys_admin`. However, I don't believe that unprivileged containers start with these kernel capabilities. So telling our LXC to keep `perfmon` if it never even had it, will have no effect. And since the process spinning up `intel_gpu_top` does not have the `cap_perfmon` capability, it cannot spin up a process that requires it.

>[**Capability Bounding Set**](https://man7.org/linux/man-pages/man7/capabilities.7.html)
>
>The capability bounding set acts as a
>limiting superset for the capabilities that a thread can add
>to its inheritable set using capset(2).  This means that **if a
>capability is not in the bounding set, then a thread can't add
>this capability** to its inheritable set, even if it was in its
>permitted capabilities

This is where my trail runs cold. I'm not sure how to verify my statement about privileged containers getting capabilities and unprivileged not getting those capabilities. All I know, is the `lxc.cap.keep` options seem to have an affect in privileged containers (i.e. I can get that error to appear by dropping `perfmon`), but I don't see any change when utilizing this with an unprivileged LXC. I tried to spin up a privileged container and then add in my UID mapping to the lxc.conf, but this just resulted in the container starting but never getting a shell. I suppose people reasonably assume you wouldn't do that, since by definition a privileged container does not perform user mapping. At least, not on the level of an unprivileged LXC.

There may also be some AppArmor or SELinux wizardry someone could perform to pass these kernel capabilities back to the LXC, but this seemed like the point where I should probably take a break and do something more productive with my life