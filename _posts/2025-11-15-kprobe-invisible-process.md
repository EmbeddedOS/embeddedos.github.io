---
title: "Hiding process from OS using Kprobes"
description: >-
  Using Kprobes to make processes completely invisible, unkillable from the OS.

author: Cong
date: 2025-11-15 00:01:00 +0800
categories: [kernel, probe]
tags: [linux, kernel, invisible, unkillable, process, proc, kprobe, kretprobe]
image:
  path: assets/img/invisible_process.png
  alt: invisible process
published: true
---

Kprobes is a powerful feature in the kernel. In this blog, we will practice it by hooking into kernel points, change the kernel execution path to hide our process completely.

## Target platform

Since the register set is architecture specific and different kernel versions, configs have different system maps, symbol addresses, etc. It is important to note about our target system:
- Kernel version: `6.18.0-rc3`
- Architecture: `arm64`
- Board: `qemu-aarch64-system` virtual board.
- kernel config: makes sure Kprobes is enabled.

```text
CONFIG_KALLSYMS=y
CONFIG_KPROBES=y
CONFIG_KRETPROBES=y
CONFIG_KPROBE_EVENTS=y
```

## An introduction to kprobes

Kprobes (Kernel probes) is a mechanism that allows you to insert probes (breakpoints) at almost any instructions in the kernel and executes custom handler routines when those probes are hit.

How does it work? when a kprobe is registered, Kprobes make a copy of the probed instruction and replace the first byte(s) with breakpoint instructions. When the CPU hits the breakpoint instruction, a trap occurs, CPU's registers are saved and the control passes to Kprobes. It executes the *pre_handler* associated with the kprobe, passing the handler the addresses of the `kprobe struct` and the saved registers. Next, Kprobes single-steps its copy of the probed instruction. After the instruction is single-stepped, Kprobes executes the *post_handler*, if any. Execution then continues with the instruction following the probepoint.

![Kprobe replace instruction](assets/img/kprobe_replace_inst.png)

Since Kprobes can probe into kernel running code, it can change the register set. This is what we'll do in this blog, we change the kernel execution path by edit the saved `pc` register to point to our target instruction, and then return non zero value from the *pre_handler*, so Kprobes will bypass the single step and just return to the given address. Will dig deeper in next sections.

## How a process's info is exposed to user space?

Process information is primarily exposed to user space via a virtual file system called procfs and mounted at `/proc` (by default). Procfs provides a dynamic interface to kernel data structure to get system information or change kernel parameters (you can change kernel parameters at runtime (or we called tuning) by using `sysctl` via this interface but this is not what we're gonna discuss in this blog).

The process information details are under the dynamic folder `/proc/[pid]/`, for example:

- `cmdline`: The command-line arguments used to start the process.
- `status`: Detailed status information about the process.
- `exe`: A symbolic link to the executable file of the process.
- `cwd`: A symbolic link to the current working directory of the process.

User space tools such as `ps`, `top`, `free` check for this virtual fs and get system + processes information. We call procfs as a virtual fs since it's not actually existing on the hard disk. Every time you use system call to access them kernel redirect it to different part with regular directories or files.

![access_proc](assets/img/cat_proc_fs.png)

## Kernel symbol address lookup

In order to hook into kernel points you must know the target addresses or the symbol names. Linux provides several ways to archive those information:

- At compile time, kernel build system generates a system map file (mapping symbol with address) and a vmlinux (a kernel image in ELF format).
- At runtime, kernel also exposes a virtual file `/proc/kallsyms` to archive runtime symbol addresses (that includes dynamic kernel module symbol, and sometimes more accurate than the static system map file).

```bash
cat /proc/kallsyms | head -n 10
0000000000000000 A fixed_percpu_data
0000000000000000 A __per_cpu_start
...
```

For some Linux distribution, the system map and vmlinux files are installed into rootfs under `/boot` folder by default when buiding the rootfs image. Often found in:
- `/boot/System.map-$(uname -r)`.
- `vmlinuz`.
- `vmlinuz-$(uname -r)`.

Our target would be changing the kernel execution path, so it's very important to get not only the symbol table but also the `vmlinux` (ELF kernel image, often used for debugging), so we can do reverse engineer things :D.

## Hook into kernel proc vfs symbol

```c
static int proc_root_readdir(struct file *file, struct dir_context *ctx)
{
	if (ctx->pos < FIRST_PROCESS_ENTRY) {
		int error = proc_readdir(file, ctx);
		if (unlikely(error <= 0))
			return error;
		ctx->pos = FIRST_PROCESS_ENTRY;
	}

	return proc_pid_readdir(file, ctx);
}

/*
 * The root /proc directory is special, as it has the
 * <pid> directories. Thus we don't use the generic
 * directory handling functions for that..
 */
static const struct file_operations proc_root_operations = {
	.read		 = generic_read_dir,
	.iterate_shared	 = proc_root_readdir,
	.llseek		= generic_file_llseek,
};

/*
 * proc root can do almost nothing..
 */
static const struct inode_operations proc_root_inode_operations = {
	.lookup		= proc_root_lookup,
	.getattr	= proc_root_getattr,
};

/*
 * This is the root "inode" in the /proc tree..
 */
struct proc_dir_entry proc_root = {
	.low_ino	= PROCFS_ROOT_INO,
	.namelen	= 5,
	.mode		= S_IFDIR | S_IRUGO | S_IXUGO,
	.nlink		= 2,
	.refcnt		= REFCOUNT_INIT(1),
	.proc_iops	= &proc_root_inode_operations,
	.proc_dir_ops	= &proc_root_operations,
	.parent		= &proc_root,
	.subdir		= RB_ROOT,
	.name		= "/proc",
};
```

```bash
$ aarch64-none-linux-gnu-objdump vmlinux -d | grep -e "ffff8000805b0068 <proc_pid_readdir>:" -A 200

ffff8000805b0068 <proc_pid_readdir>:
...

```

## Make the process unkillable from user space

## Testing 

