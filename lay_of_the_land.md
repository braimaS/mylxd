## The naming trap first

"LXC" refers to two unrelated things.

**The `lxc` command** is LXD's client. It's written in Go, lives in the `lxc/` folder of the LXD repo, and only sends REST requests to the daemon. It never touches the kernel.

**The LXC project** is a separate, older project (from around 2008), written in C. Its core is **liblxc**, the library that actually talks to kernel primitives: namespaces, cgroups, seccomp, AppArmor. The project also ships standalone tools with dashed names (`lxc-create`, `lxc-start`, `lxc-attach`) for using liblxc without LXD. These have nothing to do with the `lxc` command.

The overlap is historical: LXD was built by the LXC team as the next layer on top of LXC, and they reused the familiar `lxc` name for the new client.

## How the pieces fit

**liblxc** creates **system containers**, which run a full Linux userspace with its own init system, users, and services, so they feel like lightweight VMs. It's low-level: it knows how to start one container, but nothing about image servers, storage pools, clusters, or remote APIs.

**LXD** is the management layer on top. It's a Go daemon that adds the REST API, images, storage pools and volumes, managed networks, profiles, projects, snapshots, migration, clustering, and authentication. An analogy: liblxc is to LXD roughly what runc is to containerd/Docker.

LXD calls liblxc through **go-lxc**, a thin cgo binding (Go code calling C functions), which is why LXD needs cgo. Your launch.json shows this directly: the `CGO_CFLAGS` and `LD_LIBRARY_PATH` entries pointing at `go/deps/liblxc` exist so the Go compiler can find the liblxc that `make deps` built.

**QEMU** is how LXD runs **virtual machines** (supported since LXD 4.0). QEMU is a large, separate C program, not Go, and combined with the kernel's KVM it provides hardware-accelerated VMs. LXD doesn't link QEMU as a library. It launches `qemu-system-x86_64` as its own process and controls it over a socket using **QMP** (the QEMU Machine Protocol, a JSON command channel) for actions like pause, hot-plug a disk, or shut down.

The two paths:

```
container:  lxc (CLI) ──REST──> lxd ──(cgo calls)──────────> liblxc ──> kernel
VM:         lxc (CLI) ──REST──> lxd ──(spawn process, QMP)──> qemu ───> KVM
```

Both sit behind a common instance interface, which is the `driver_lxc.go` vs `driver_qemu.go` split in `lxd/instance/drivers/`. It's also why `lxd-agent` exists: inside a VM there's no shared kernel to reach into, so an agent running in the guest handles `lxc exec` and file transfers.

The concept worth stating in an interview is that LXD gives one API and one CLI for both containers and VMs. A user types `lxc launch ubuntu:24.04 c1` or adds `--vm`, and storage, networking, snapshots, and migration all work the same way.

**LXCFS** is a small related piece: a FUSE filesystem that makes files like `/proc/meminfo` and `/proc/cpuinfo` show a container's *own* limits, so `free` and `top` report sensible numbers inside a container.

## liblxc and OCI

liblxc does **not** follow the OCI spec. It predates OCI (2015) and has its own configuration format. The differences:

**Config format.** An OCI runtime like runc takes a bundle: a `config.json` following the OCI spec plus a root filesystem. liblxc takes its own key-value configuration:

```
lxc.rootfs.path = dir:/var/lib/lxd/containers/c1/rootfs
lxc.idmap = u 0 1000000 1000000000
lxc.cgroup2.memory.max = 2G
lxc.net.0.type = veth
lxc.net.0.link = lxdbr0
```

**What runs inside.** OCI containers are designed around one application process. LXC containers boot a full init and run a whole OS.

**Lifecycle.** OCI containers are often treated as disposable and rebuilt from images. LXC containers are long-lived: you upgrade packages inside them, snapshot them, and keep them for years.

In LXD's code, `driver_lxc.go` translates an instance's LXD config (limits, devices, idmaps) into these `lxc.*` keys and hands them to liblxc through go-lxc before starting the container. If you step through `Start()` in the debugger, you'll see the keys being set, which is the direct equivalent of a higher-level tool writing an OCI `config.json` for runc.

There's some crossover. The LXC project has long had an `oci` template that unpacks an OCI image into an LXC container, and both Incus and LXD have been adding ways to run application containers from OCI registries. In every case the images are converted into LXC's model. liblxc itself doesn't become an OCI runtime and can't replace runc under containerd. Check the current LXD docs for how far OCI support goes, since it has been moving quickly.

## MicroCloud

MicroCloud is Canonical's way to stand up a small private cloud quickly, typically three or more machines. It combines **LXD** in cluster mode for compute, **MicroCeph** for distributed storage (a simplified Ceph deployment), and **MicroOVN** for software-defined networking (a simplified OVN deployment). Setup discovers the other machines on the network and configures all three together, so instances can move between machines while storage and networking span the cluster. You saw a trace of it in `init`: the user agent gets a `microcloud` feature tag when the cluster is part of one.

## Snap packaging

The snap is LXD's primary, officially supported distribution method.

- **Bundled dependencies.** It ships its own liblxc, QEMU, dqlite, storage tools, and more, so LXD behaves the same on every distro. That's also why building from source was more work: you had to supply all of that yourself.
- **Channels and tracks.** Users pick an LTS track (for example, `5.21/stable`) or `latest/stable` for feature releases, and snapd refreshes automatically within the track. LTS releases are supported for years and line up roughly with Ubuntu LTS.
- **Code traces.** The `shared.SnapSetHealth(...)` calls in `init` report daemon health to snapd, and the rolling-upgrade retry loop around the cluster database exists partly because snap refreshes can upgrade cluster members at slightly different times.
- **Different paths.** The snap uses `/var/snap/lxd/common/lxd` rather than `/var/lib/lxd`, which is why your source build and the snap don't collide on data.

## Incus and the 2023 split

LXD originally lived under the **Linux Containers** project (linuxcontainers.org) alongside LXC and LXCFS. In July 2023, Canonical moved LXD in-house to `github.com/canonical/lxd`. Shortly afterward, members of the Linux Containers community, including former LXD lead Stéphane Graber, forked it as **Incus**, now developed under the Linux Containers project. Later in 2023, Canonical relicensed new LXD contributions under **AGPLv3** with a contributor license agreement, while Incus remains Apache 2.0. The codebases have been diverging since, and Incus can't simply pull in new LXD code.

For the interview, know the history accurately and focus on LXD's direction. Canonical's side centers on closer integration with its products (MicroCloud, Ubuntu Pro, the snap ecosystem) and long-term enterprise support. You can mention that you know Incus exists without taking sides.

## One-paragraph summary

"LXC is a C project whose library, liblxc, creates system containers using kernel features, with its own config format rather than OCI. LXD is Canonical's Go daemon on top of it. It adds a REST API, images, storage, networking, and clustering, and it also manages QEMU virtual machines behind the same interface. Its `lxc` command is just a REST client. It's distributed mainly as a snap with bundled dependencies and LTS tracks. MicroCloud combines an LXD cluster with MicroCeph and MicroOVN to build small private clouds. In 2023, LXD moved from the Linux Containers project to Canonical, and the community forked it as Incus."

My knowledge runs to around mid-2026, so check the LXD release notes before the interview for the current LTS track, OCI support, and any MicroCloud changes.

I can also turn this into a doc you can keep and revise as your interview notes.