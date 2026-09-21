## doc/rest-api.md
### 9
Local: Unix socket, protected by file permissions (the OS checks who you are).
Remote: TCP, protected by TLS (certificates prove who you are).

### 16
When you type lxc list, Your shell finds the lxc executable on your PATH and runs it, like any other program.
lxc parses your arguments and turns them into an HTTP request (GET /1.0/instances).
It opens the Unix socket file and sends that request.
The LXD daemon listening on that socket, receives the request

### 52 
lxd returns 3 kinds of responses: standard(sync), background(async),error

### 127 
response will always have status(for uhmans to read) and status_code for machines to read

### 161
not recursion but link expansion
By default (recursion=0), asking for a collection gives you only a list of pointers
recusrsion here is how many levels of URLs the server should replace with the actual objects before responding.

### 174
jut reqgular rest filtering 0Data

### 208 
Any operation which may take more than a second to be done must be done
  
## just running lxd

### lxd/main.go

#### 91
hmm it'd be useful to be able to reproduce a simple cobra app from memory cobra.md

#### 91
fills apps runE field
#### 109
fills apps PersistentPreRunE field(supposed to run before every runE)
I think my confusion was why encapsulate it in a cmdGlobal but as u said, every command is made a struct. refer to cobra.md
#### 118
good ol socket activation pattern. 
When LXD is installed as a system service (the snap, or a systemd package), it usually doesn't start at boot. Instead, systemd creates and holds the Unix socket itself. The first time anything connects to it (you run lxc list, say), systemd starts the LXD daemon and hands the socket over. This saves memory and boot time on machines where LXD is installed but rarely used.

That creates a problem: some things need LXD running at boot even if nobody connects. For example, instances configured with boot.autostart=true should come up after a reboot, and if LXD is listening on the network (core.https_address), remote clients expect it to be reachable.

activateifneeded solves that. It's run once at boot by the service manager, and it checks whether anything requires LXD to be up right now. It does this by reading LXD's database and config directly, without the daemon running. If something does need it, it connects to the socket, which triggers the activation. If not, it exits and leaves LXD dormant until someone uses it.

#### 123-228
i get the pattern

"but why do they all need to be fed a pointer to globalcmd i mean the important things there are the prerun it provides and the preflags, which they alrady inherit from the use lxd command"

Cobra inheritance gives subcommands the parsing and the prerun execution, but not a way to read the results in Go code. That's what the pointer is for.

When you run lxd init --debug, cobra accepts --debug on init because it's a persistent flag, and it runs globalCmd.Run before init's RunE. But the parsed value lands in globalCmd.flagLogDebug, because that's the variable the flag was bound to. If init's run method wants to check "am I in debug mode?", it needs a way to reach that field, and c.global.flagLogDebug is that way.

The pointer also carries the shared helpers, not just flag values. The asker is the obvious one: lxd init uses c.global.asker to prompt you interactively, and without the pointer every command would need its own stdin reader.

#### 231 
cobra execute, which in this case just does the runE for use lxd.
i am running $lxd and when thats done i'll come and run $lxc lists
so it does lxd cuz thats what i asked. hence why it doesnt do lxd init or lxd fork cuz i didnt ask for that

### lxd/main_daemon.go

#### 50
ensured the daemon runs as root

#### 57 exec.lookPath guard clause
exec.LookPath(p) searches your PATH for an executable named p, the same way your shell finds ls when you type it. It returns the full path (like /usr/bin/rsync) or an error if the program isn't found. Here the result is thrown away (_), because the code only cares whether each program exists.

As for why: LXD shells out to these tools instead of reimplementing them. It uses ip for network setup, rsync for copying and migrating instance data, setfattr for setting extended file attributes, and tar, unsquashfs, and xz for unpacking images. Checking at startup means a missing tool fails immediately with a clear error, rather than halfway through your first lxc launch. It's the same fail-early idea as the root check.

#### 68 Separating construction from startup.
newDaemon() function is a constructor fucntion that returns a *Daemon struct. it is not a method of Daemon struct

#### 70
So `signal.Notify(sigCh, unix.SIGPWR)` means: "Go runtime, when SIGPWR arrives, don't kill me. Put it in `sigCh` and I'll deal with it." Calling `Notify` several times with the same channel just adds more signals to the list sigs to be listened for, which is why the LXD code does four separate calls.

Receiving is an ordinary channel read. Here's a tiny program you can run:

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, unix.SIGINT)

fmt.Println("press Ctrl+C")
sig := <-sigCh              // blocks until the signal arrives
fmt.Println("got", sig, "- cleaning up")
```

#### 87
Why Stop runs in its own goroutine: shutdown can take a while (stopping instances on SIGPWR, for example). If the loop called d.Stop directly, it would be stuck there and couldn't read more signals. Running Stop separately keeps the loop draining the channel, so a second Ctrl+C gets logged and ignored instead of piling up.

and different signals tell stop function to do different things


### lxd/daemon.go
#### 1178 init daemon
Each log line marks the start of a chapter, and the code between two log lines is that chapter.

#### 1209
LXD has an internal event bus, d.events. Things like "instance started," "operation finished," and log messages get published there. External clients subscribe through GET /1.0/events, and you can watch it live:
lxd monitor - to tap into it

#### 1550
actaully start up the daemon


## build a container

## build a vm

Make sure your client points at your debug daemon first:

```bash
export LXD_DIR=/var/lib/lxd
```

**Container:**

```bash
~/go/bin/lxc launch ubuntu:24.04 c1
```

**VM:**

```bash
~/go/bin/lxc launch ubuntu:24.04 v1 --vm
```

The VM will likely need extra setup because you're running from source. The snap bundles QEMU and its UEFI firmware, but your build uses whatever is on the host, so install them first:

```bash
sudo apt install qemu-system-x86 qemu-utils ovmf
```

Your user also needs access to `/dev/kvm`, and since LXD injects `lxd-agent` into the VM, the `lxd-agent` binary from your build (in `~/go/bin` after `make`) needs to be findable in the daemon's PATH. If the VM fails to start, the error message will usually name what's missing.

Set a breakpoint in `instancesPost` in `lxd/instances_post.go` before running either command. From there you can follow the request into the async operation, then down to `driver_lxc.go` for the container or `driver_qemu.go` for the VM. The first launch will pause for a while at "Retrieving image" while the image downloads, which is normal.

### lxd/instances_post.go

#### 1522
all create instance and create container happens here