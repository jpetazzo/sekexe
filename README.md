# SEparate KErnel eXEcution

SEKEXE uses User Mode Linux to run a Linux process within a "sub kernel",
and retrieve its output and exit status (as if it was running as a normal
shell command). This is half-way between full-fleged virtual machines
and lightweight containers: VMs can run any O/S; containers run the same
O/S and kernel as their host; SEKEXE only runs Linux-inside-Linux, but the
guest can be a different version.

This is useful if you want to:

- isolate a process, but your kernel doesn't support containers or other
  isolation features;
- run or test specific kernel features that are not supported by your
  current kernel;
- run a process as root, but do not have root privileges.

(Of course, in the last scenario, you will not gain any privilege that
you didn't have initially; you will be able to create a kind of virtual
machine, and you will have root privileges inside that pseudo-VM, but
you will not gain privileges outside.)

SEKEXE was developed for one very specific use-case: run the test-suite for
[Docker](https://github.com/dotcloud/docker) within [Travis CI](
https://travis-ci.org/). Docker needs root privileges, and a kernel with
namespaces, control groups, and AUFS support.


## How To Use It

```bash
./run "env ; exit 42"
echo $?
```

This will start a separate kernel, run "env ; exit 42" in it, then display
the exit value -- which should be 42 in that case.

If you run `./run` by itself (without arguments) you will get a shell inside
the environment. This mode is convenient to experiment within the environment.


## Requirements

You need the `slirp` package. On Debian/Ubuntu, `apt-get install slirp`
is your friend. On other distros, you're on your own, but it shouldn't be
very different.

Slirp is used to provide network connectivity without requiring root
permissions; if you do not need network access, and cannot install slirp,
you can remove the `eth0=slirp...` parameter from `run`.


## Kernel

The SEKEXE repository contains a 3.8 Linux kernel with support for LXC
and AUFS. It has no support for block devices or real filesystems, though.
If you want to replace it with another version of the kernel and need help,
let me know.


## Internals

As mentioned above, SEKEXE uses Slirp to provide network connectivity
without root access. This means that TCP and UDP traffic will work, but
you won't be able to ping.

An "exchange directory" (`$LOGDIR`) is created, and the command to run
is placed in it. Then the User Mode Linux kernel is started, with a custom
`init`. The new environment shares the filesystem of its host, so you
do not have to setup a chroot or something like that. It uses UML's `hostfs`
filesystem for that. The mount is done in read-only; so the environment
cannot affect anything in the host system (except within the `$LOGDIR`
directory, which is explicitly mounted read-write).

The custom `init` takes care of setting up the network, setting up
appropriate `tmpfs` mountpoints, and starting the process. It collects
its output and exit status, stores them in `$LOGDIR`.

The main `run` process then retrieves those items and exits appropriately.

The normal output of the environment is diverted to `/dev/null` because
it contains spurious kernel messages.
