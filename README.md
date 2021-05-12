# MemJail

Got a program which sucks up all the RAM and doesn't leave room for anything
else?  I'm looking at you, Chrome.

Put that misbehaving program in a memory jail.  Limit the amount of RAM it can
use, so it won't mess with the rest of the system.


## Requirements

  * Linux 4 or newer
  * SystemD (sorry)
  * zsh


## Usage

### memjail

Creates memory-restricted containers.

Takes three parameters:

  * RAM limit in GiB.
  * Program to run (optional; defaults to a shell).
  * Name of this container (optional; defaults to a random number).

Example:

```
memjail 8 zsh chrome
```

Creates a new jail with a 8 GiB limit.  The command to run is `zsh`, and the
name of the jail is `chrome`.  The idea here would be to then run Chrome from
that shell, and all processes in the container would be limited to a maximum
combined usage of 8 GiB RAM and 8 GiB swap.

To stop the jail, simply close the first process created within it.

### memjail-monitor

Shows a summary of resource usage within memjail containers.  Useful either
directly in a terminal, or in a status widget like Conky.

Takes one parameter:

  * Jail name (optional, shows all by default).

In general, just run `memjail-monitor` by itself, to get a real-time summary of
resource use in all jails.  The output looks something like this:

```
--- chrome ---
  59% @ 46 PIDs
4.761 / 8 GiB
0.000 / 8 swap
```

This info also gets written to `~/ram/memjail.status` so it can be easily
included in status widgets like Conky.  The "ram" dir should typically be a
symlink to a tmpfs which stores files only in RAM.  On a typical desktop
system, this will be something like...

```
> ls -l ~/ram
lrwxrwxrwx 1 myuser myuser 14 Jan  1  2020 /home/myuser/ram -> /run/user/1000/
> df /run/user/1000
Filesystem     1K-blocks  Used Available Use% Mounted on
tmpfs            1601044    28   1601016   1% /run/user/1000
```

## Why does it need SystemD?

Two reasons:

  * Most Linux systems now use systemd... and systemd takes offense to
    processes existing outside of its cgroup hierarchy.  So if we *don't* use
    systemd, it has a bad habit of stealing processes from our containers and
    migrating them to its own...  which frees them from their resource
    constraints and completely defeats the point of isolating them in the first
    place.  I've sworn rather a lot at Lennart Poettering for this, and wrote
    scripts to monitor when it happened, and to fight back by re-asserting
    control of those processes.  It's a huge pain.  So instead of doing things
    that way, it's a lot easier to register a container within systemd.  Then
    it doesn't break when systemd gets bored.

  * Because systemd has a "systemd-run" utility which does most of the work.
    All we have to do is call it, then set a couple resource limits, and
    systemd does all the setup and tear-down.


## Installation

Download the source.  Put the scripts somewhere in your `$PATH`.

For example:

```
cd ~/src
git clone http://github.com/ToyKeeper/memjail
cd ~/bin/
ln -s ~/src/memjail/memjail .
ln -s ~/src/memjail/memjail-monitor .
```

On relatively old Linux systems with cgroups v1, it may also be necessary to
enable memory cgroups.  Add this in `/etc/default/grub.conf` (or similar):

```
GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1"
```

... then run "update-grub" and reboot.  But newer systems should have memory
cgroups enabled already.

