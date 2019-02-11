# SystemDon't: Killing It Softly With Cgroups

## It just suddenly happened

All bugs seem to happen on Fridays. This one was no different. On Friday, Nov 30, my team got a bug report titled "Garden fails to restart after upgrade". Great! It had happened in the pipeline of Cloud Foundry's persistence team and one of their Cloud Foundry deployment's VMs wouldn't come up after redeploy because the Rep could not talk to Garden - the container engine of Cloud Foundry. Luckily the persistence team did some good initial debugging and found out the following: There was a container process that had survived the restart of the Garden daemon - something that is not expected to happen. Also the Garden logs contained multiple log lines like this one:

```
{"timestamp":"2018-11-29T19:57:17.935639633Z","level":"error","source":"guardian","message":"guardian.api.garden-server.destroy.failed","data":{"error":"timed out waiting for container kill"}}
```

It seemed that there was a container that somehow entered some weird unkillable state, so when Garden restarted and tried to clean up all containers as it usually does, it timed out waiting for this one to die. At some point `monit` (the service manager responsible for monitoring all the jobs on the VM) would restart the Garden job and the whole thing would repeat over and over again.

Having found out that this stuck container was responsible for all the trouble, the persistence people killed it with SIGKILL, thus unblocking Garden, and the problem went away. Unfortunately so did all evidence. Given that there were no steps to reproduce, our only option was to just wait for it to happen again. At least we knew how to recover from it.

## Hold on! What is Rep? What is Garden?

Let me take a step back and try to explain some terms an concepts so that you know what I am talking about. Skip this paragraph if everything sounds clear so far. I am an engineer on Cloud Foundry's Garden team, and we build Cloud Foundry's container engine. Cloud Foundry itself is a PaaS that focuses on your app code and makes it easy to push and scale your app in the cloud. Cloud Foundry apps run in containers and managing those containers is what the Garden team does. Cloud Foundry is a distributed system consisting of may different types of VMs. In this article we are going to focus on the cell - the type of VM where app processes are run. The two most important jobs that run on the cell are the Garden daemon and the Rep. The Garden daemon is a server that exposes a low level API for container management. It does not know about apps, routes, services, etc. It only knows about container processes. The Rep is Garden's connection to the rest of the system. It knows how to start and stop apps, run healthchecks and other higher level concerns. One of its responsibilities is to report the cell ready after an upgrade. It does so by creating a simple container just to make sure that Garden is ready to serve requests. So in the above described situation the Rep was unhappy, because it was unable to talk to Garden, since garden was busy waiting for the unkillable container to die before handling new API events.

Garden is a constantly evolving project. We are trying to keep up with container tech out there, so we have been an early adopter of `runc` and are currently transitioning to `containerd` because we have a lot of overlap with those projects. Currently Garden can be deployed in legacy mode in which case it will be directly using `runc` and in containerd mode in which case it will be using containerd, which in turn is using `runc`. Cloud Foundry is already using containerd by default since cf-deployment v6.3.0 which came out on Nov 27, 2018, shortly before unkillable containers got reported.

## And it happened again

Luckily, not long after, the same problem manifested itself on another system (again running in containerd mode) and this time we had a chance to spend some time on the cell before anyone killed the errant container. 

Containerd is using runc underneath to manage its containers. When Garden wants to destroy a container it instructs containerd to kill it using the `containerd.WithKillAll` option, which results in a call  to the `runc kill --all` command. When given the `--all` option runc will not kill the init process of the container, but would instead signal all processes that it find in the `devices` cgroup associated with the container. So we decided to look more closely at the broken process' cgroups and they surely looked terribly wrong.

In case you don't know, cgroups are a kernel isolation concept used to impose certain limits on processes in the linux OS. They are represented by a virtual filesystem of type `cgroupfs` usually mounted on `/sys/fs/cgroup`. Each process is a member of one cgroup, that is represented by a directory. For example `/sys/fs/cgroup/memory/my-cgroup`: 
```
$ tree /sys/fs/cgroup/memory/my-cgroup/

/sys/fs/cgroup/memory/my-cgroup/
├── cgroup.procs
├── ...
├── memory.limit_in_bytes
├── memory.usage_in_bytes
├── memory.use_hierarchy
└── memory.use_hierarchy
```
In the example above, `memory` is one of the many available cgroup subsystems and `my-cgroup` is the name of the cgroup. If we list the cgroup directory we will find many useful virtual files that give us information about current memory usage (memory.usage_in_bytes), allows us to set limits (memory.limit_in_bytes) and so on. The `cgroup.procs` file contains the pids of all processes that are running in this cgroup. 

When Garden creates containers it makes sure that container processes are running in properly configured cgroups in the format `/sys/fs/cgroup/<subsystem>/garden/<container-id>/`. But when we listed the cgroup paths for the errant container here is what we got:

```
# cat /proc/<garden-init-pid>/cgroup

12:blkio:/                                                         # <- wrong
11:hugetlb:/garden/<container-id>
9:freezer:/garden/<container-id>
8:perf_event:/garden/<container-id>
7:net_cls,net_prio:/garden/<container-id>
6:memory:/                                                         # <- wrong
5:cpuset:/garden/<container-id>
4:pids:/                                                           # <- wrong
3:cpu,cpuacct:/                                                    # <- wrong
2:devices:/system.slice/runit.service                              # <- wrong
1:name=systemd:/system.slice/runit.service/garden/<container-id>
```

Half of the cgroup paths look all right since they are properly nested under the garden cgroup, but the other half look wrong. Most notably the devices cgroup seems to be messed up. A quick check of the container's `state.json` revealed that runc does not suspect that the container's cgroups got altered:

```
cat /run/containerd/runc/garden/<container-id>/state.json

...
  "cgroups": {
    "path":"garden/cake",
    ...
  },
...  

```

What's more - we observed that `/sys/fs/cgroup/devices/system.slice/runit.service/garden/<container-id>/` was an existing directory and its `cgroup.procs` file was empty. Instead, the pid of the container init process was found in `/sys/fs/cgroup/devices/system.slice/runit.service/cgroup.procs` along with many other pids such as the pid of the Garden daemon. It seemed as if someone moved the container init process upwards in the cgroup hierarchy. At least for some of the subsystems. This observation explained why the container could not be killed. When runc tried to kill it with `runc kill --all` it loaded all pids from the associated devices cgroup, which according to its state file was `/sys/fs/cgroup/devices/system.slice/runit.service/garden/<container-id>/`, found no pids there so the kill was a noop. Containerd, not suspecting what had happened, just hung waiting for the process to exit. When we killed the errant container ungracefully with SIGKILL everything unblocked. But who is it that corrupts container cgroups? And what can we do about it?

## Great, lets reproduce!

We now had basic understanding of what was bugging the system and how to work around the problem, but we could not fix it since we had no clue of what was causing it. So we started trying to make it happen. We needed to have a list of reproduction steps in order to experiment and eventually narrow it down until we found the actual root cause. Unfortunately we had no idea where to start. Here are excerpts from the battle diary:

### 11.12.2018

We started off by trying the most direct thing - we deployed a Cloud Foundry, pushed a `nfs-broker` - a service that is pushed as a regular app and registered as a platform service, then triggered a redeploy. We did that because it was exactly what the persistence team were doing when they first saw this happen. Fortunately after several attempts the exact same symptoms occurred. But it was inconsistent so we still did not know what was causing it.

### 14.12.2018

We observed the same symptoms by just stopping the garden job, rather than triggering a full redeploy. This made it possible to write a simple script that restarts the garden job and checks the state of the app's cgroups for us. This script would sometimes hit the problem, but very unreliably.

### 19.12.2018

After running our script for a while with no results, we recreated the cell and tried again and the problem occurred shortly after. We were able to repeat this several times. All our efforts to reproduce the issue without recreating the cell were unfruitful. It looked like the recreate was the essential ingredient, though. That's why we decided to recreate, re-push the app and just leave the system be idle for some time instead of constantly restarting Garden. Guess what: after about 20 min the nfs-broker app cgroups were messed up! So we didn't need to do anything and the problem would just happen by itself. Interesting.

### 20.12.2018

Even though we were able to reproduce the problem more reliably we still had way too many moving parts. We were unsure which of the following had anything to do with it:
- the nfs-broker app: we hadn't yet seen this without this app deployed
- containerd mode: This problem happened on two systems, both in containerd mode
- systemd: we have observed before that systemd is slightly changing what the cgroups filesystem looked like
- the stemcell: maybe the latest stemcell brought in a kernel bug, that was causing this behaviour

Now that we had a relatively consistent way to reproduce, we started eliminating some of these variables. The only problem was that reproduction was terribly slow: Cell recreation typically takes around 10 minutes, add 20 minutes of waiting and you got a 30 minute turnaround. Definitely not great.

The first thing we tried was to replace the nfs-broker with a simple test app. So we recreated the cell, pushed the test app and waited. After ~20 minutes it payed off - the app's cgroups were messed up. So we just eliminated the `nfs-broker`. Next thing we created a script that would print out the date once it starts and once again later when it finds the app's cgroups  in a messed up state. We ran the same experiment a couple more times, this time recording the timestamps and determined that the app breaks *exactly* 15 minutes after the cell starts up.

One of our experiments reported that the cgroups were altered at `13:01:24`. A quick inspection of the systemd logs revealed something quite interesting:

```
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 systemd[1]: Starting Cleanup of Temporary Directories...
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 systemd[1]: Started Cleanup of Temporary Directories.
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 audit[1]: SERVICE_START pid=1 uid=0 auid=4294967295 ses=4294967295 msg='unit=systemd-tmpfiles-clean comm="systemd" exe="/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 audit[1]: SERVICE_STOP pid=1 uid=0 auid=4294967295 ses=4294967295 msg='unit=systemd-tmpfiles-clean comm="systemd" exe="/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 audispd[1288]: node=f2758868-a54e-41fe-bdb1-7acfdcf02fe7 type=SERVICE_START msg=audit(1545310877.968:733): pid=1 uid=0 auid=4294967295 ses=4294967295 msg='unit=systemd-tmpfiles-clean comm="systemd" exe="/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
Dec 20 13:01:17 f2758868-a54e-41fe-bdb1-7acfdcf02fe7 audispd[1288]: node=f2758868-a54e-41fe-bdb1-7acfdcf02fe7 type=SERVICE_STOP msg=audit(1545310877.968:734): pid=1 uid=0 auid=4294967295 ses=4294967295 msg='unit=systemd-tmpfiles-clean comm="systemd" exe="/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
```

Shortly before cgroups got messed up systemd triggered a `tmpfiles-clean` service. Of course this might have been a mere coincidence so we performed the experiment a couple of times more. The pattern persisted! There was definitely some relation between this service being run and the cgroups being messed up. And everything was pointing to systemd. Given that this was happening a fixed time after cell recreation it definitely sounded like a scheduled systemd service. Indeed, a quick filename search brought this file to our attention:

```
$ cat /lib/systemd/system/systemd-tmpfiles-clean.timer

[Unit]
Description=Daily Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
```

Bingo! It all fitted together! Exactly 15 minutes after machine boot systemd would trigger this service and this would somehow lead to our container's cgroups being weird. So we tried to make it happen again by killing the errant process, pushing a new healthy app and manually triggering the service by running:

```
$ systemctl start systemd-tmpfiles-clean.service
```

Unfortunately nothing happened and the app was still healthy. Next thing, we re-ran the same experiment after a recreate and the app was broken after we did that. Maybe the service was doing some idempotent operation and that's why we could only reproduce once per recreate? We had no idea. On the plus side we had cut 15 minutes off of the turnaround, which was something.

We took a look at the service definition:
```
$ cat /lib/systemd/system/systemd-tmpfiles-clean.service

[Unit]
Description=Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)
DefaultDependencies=no
Conflicts=shutdown.target
After=local-fs.target time-sync.target
Before=shutdown.target

[Service]
Type=oneshot
ExecStart=/bin/systemd-tmpfiles --clean
IOSchedulingClass=idle
```

It was pretty straightforward - just running a simple command. Maybe we don't need systemd then? Maybe we can reproduce by just running the same command that systemd ends up running. However, execing `/bin/systemd-tmpfiles --clean` didn't do anything. It looked like the systemctl part was somehow significant. We wrote to the systemd developers mailing list, hoping to get some insights. Unfortunately, we never heard back from them.

Some googling around systemd and cgroups quickly revealed that other container runtimes such as LXC have had [a suspiciously similar problem](https://github.com/systemd/systemd/issues/4079). They had fixed it by registering a systemd unit and setting a `Delegate=yes` property on in. This way they were telling systemd that their service is managing its own cgroups and that systemd should not mess with them. Before they did that systemd would move processes between cgroups according to its own rules. This was exactly what we were observing. The garden/<container-id> cgroup was created by Garden and systemd did not know about it, so at some point it just moved processes running in that cgroup to `/system.slice/runit.service` - that direct parent that systemd had created. What a sneaky default behaviour! Everything was making sense now... Well actually not quite everything. We had moved to systemd almost a year ago when we migrated to `ubuntu xenial`. Why had we not seen this problem before? What made it manifest itself now?

We still hadn't ruled containerd out of the equation. We had only seen this problem on systems running in containerd mode and the problem surfaced shortly after we made containerd mode the default. So containerd should have something to do with it. In order to prove that we created a deployment with containerd mode switched off and performed our reproduction steps. After running the temp files cleanup we checked the app's cgroups and they looked like this:

```
# cat /proc/<garden-init-pid>/cgroup

12:blkio:/                                                         # <- wrong
11:hugetlb:/garden/<container-id>
9:freezer:/garden/<container-id>
8:perf_event:/garden/<container-id>
7:net_cls,net_prio:/garden/<container-id>
6:memory:/                                                         # <- wrong
5:cpuset:/garden/<container-id>
4:pids:/                                                           # <- wrong
3:cpu,cpuacct:/                                                    # <- wrong
2:devices:/system.slice/runit.service                              # <- wrong
1:name=systemd:/system.slice/runit.service/garden/<container-id>
```

What!? How? So containerd had nothing to do with it! But if that was true it seemed like incredible luck that this went unnoticed for a year. We were interested to see if the Garden server would hang in the same way when we restart it. Not at all! The daemon quickly came back up and the container was gone. Garden somehow managed to kill it on startup. Immensely confused, we started carefully comparing containerd to non-containerd code. We noticed that when not using containerd, Garden was trying to kill the broken process using `runc kill`, but instead of waiting indefinitely for the process to exit it was timing out and eventually killing the process with SIGKILL. If you remember this was exactly what the persistence team did to recover their system. So the legacy non-containerd mode was able to accidentally auto-recover from this situation. It's great that we are making Garden increasingly robust, isn't it?

## To wrap it up

Finally everything was making sense. When we moved to xenial we effectively started running our cells with systemd as the service manager. Systemd expected that it was the only process on the host that deals with cgroups, because it was not told otherwise. Garden was silently managing the cgroups for the containers it was creating. All these things would lead to sytemd moving containers out of their respective cgroups. This was a problem, because this could mean that containers might sometimes get their limits lifted. However, because of sheer chance, this went unnoticed until we made containerd mode the default. After we did that the broken containers became effectively unkillable and got in our radar range.

## All right, let's fix it!

Now that we knew what was going on we were ready to start working on a fix. We had one more small problem though - reproduction still involved a recreate of a virtual machine, which was slow and hard to automate. So we had to do it manually over and over, which was tedious and error prone. What's more we still did not completely understand why the bug would manifest itself just once, 15 mins after machine creation. We hadn't found any documentation describing when systemd would reorganize cgroups. We hadn't heard back from the mailing list yet. So we decided to read some systemd code. We found something that was looking promising in the [cgroups.c](https://github.com/systemd/systemd/blob/f5855697aa19fb92637e72ab02e4623abe77f288/src/core/cgroup.c#L1655) file:

```c
  /* First, create our own group */
  created = cg_create_everywhere(u->manager->cgroup_supported, target_mask, u->cgroup_path);
  ...
  /* Preserve enabled controllers in delegated units, adjust others. */
  if (created || !u->cgroup_realized || !unit_cgroup_delegate(u)) {
    CGroupMask result_mask = 0;

    /* Enable all controllers we need */
    r = cg_enable_everywhere(u->manager->cgroup_supported, enable_mask, u->cgroup_path, &result_mask);
    ...
  }
```

This definitely looked like the condition under which systemd would reorder cgroups. And indeed one condition to skip doing this is when the unit is delegated (has `Delegate=yes`). But it is not the only condition. It seems that when running a unit systemd first tries to create a new cgroup for it and returns whether id did so or not. Presumably if the cgroup already existed it would return `false` and it would not execute the cgroup adjustment code. This could explain why `systemctl start systemd-tmpfiles-clean.service` would only reproduce the problem on first execution after machine creation.The second time around `cg_create_everywhere` would return false and cgroups will not be adjusted. 

We immediately tested our hypothesis. We defined and installed a dummy systemd service:

```
# write a service definition
$ cat >/lib/systemd/system/custom.service <<EOF
[Unit]
Description=Custom
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/bin/true
EOF

# install the new service
$ systemctl daemon-reload
```

Then we pushed a healthy app and without recreating the machine ran `systemctl start custom.service`. The fact that this was a brand new service would hopefully make `cg_create_everywhere` return true and make systemd adjust the cgroups. We checked the app's cgroups and they were altered exactly as we have seen so many times! It was now trivial to automate the bug reproduction steps. We have won! Hooray!

## Where is the fix, though?

Yeah, we hadn't theoretically won just yet, but the hard part seemed to be over. We were already quite sure that setting `Delegate=yes` would solve all our problems. What's more it looked like `runc` had a flag `systemd-cgroups` that would do it for us. It was a no-brainer, right? Well, not exactly. 

If you remember we are using runc both directly and indirectly - directly in legacy mode and indirectly in containerd mode. So we would have to fix this in both modes. What's more, looking at the runc code it seemed like this flag is completely switching the cgroup manager in runc and we were worried if all our features (like rootless) would continue to work properly after the change. It seemed like a big and dangerous move that could make this whole saga run for another month. Instead, we decided to try something simpler - create a simple service file for the Garden daemon. Only problem was that we are already using `monit` as a serivce manager as it is a kind of a requirement in Cloud Foundry. So we were worried that the two service managers might step on each other's toes. After some reading and experimentation we found a systemd service configuration that looked safe:

```
[Unit]
Description=garden-runc container runtime

[Service]
ExecStart=/var/vcap/jobs/garden/bin/garden_start
ExecStop=/var/vcap/jobs/garden/bin/garden_stop
Delegate=yes
KillMode=none
```

It is quite minimal: `ExecStart` and `ExecStop` tell systemd what commands it should use to start and stop our service. The `garden_start` and `garden_stop` are the exact same binaries that monit would use to start our service. The `Delegate=yes` tells systemd to not ever touch our beloved cgroups and `KillMode=none` makes sure that systemd would not kill Garden, so all control remains in the hands of monit. Note that while monit knows about and monitors the pid file of Garden, systemd does not have this information, so it is just a dumb proxy we use merely to declare a delegate systemd service. From monit's point of view the only change is that it now needs to call `systemctl start garden.service` and `systemctl stop garden.service` instead of `garden_start` and `garden_stop`.

Needless to say, after we deployed the modified version of our release and ran the reproduction script cgroups stayed correct and everything was happy! Finally!

# The End

---

The Garden team intends to share more about other investigations which have thrilled, baffled and haunted us, so stay tuned…
 

Big thanks to all Garden team members for all their help and support: Claudia Beresford, Julia Nedialkova, Danail Branekov, Tom Godkin, Giuseppe Capizzi. Product Manager: Julz Friedman.
