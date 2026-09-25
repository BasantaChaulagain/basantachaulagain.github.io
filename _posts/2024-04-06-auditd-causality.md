---
title: "Tracing an Attack Through Linux Audit Logs"
date: 2024-04-06
layout: single
classes: 
  - wide
  - small-font
redirect_from: 
  - /blog/auditd-causality
author_profile: false
published: true
header:
  image: assets/images/auditd-causality-header.png
  teaser: assets/images/auditd-causality-header.png
excerpt: >
  An alert gives you one file or one process ID. Causality analysis over auditd logs turns that single clue into the full story of how an attack got in and what it touched.
---

An alert gives you almost nothing. One suspicious path, one PID, one inode number.

The questions that matter come next: **how did this get here**, and **what else did it touch?** The first decides whether you have an incident, the second decides how bad it is. Linux records enough to answer both (**auditd** captures process creation, file access, and network activity at the syscall level), but not in a form you can query.

---

## Why Raw Audit Logs Don't Answer Questions

A busy host emits well over a million records a day, and the format is built for completeness rather than analysis:

- A single syscall spans **multiple records** that must be joined on event ID.
- File operations reference **numeric descriptors**, not paths. Knowing a process wrote to fd 7 is meaningless until you resolve what fd 7 pointed to at that event; the same descriptor is reused after close().
- Paths are recorded relative to a cwd that lives in a different record.

So normalization comes first: one row per event, descriptors already resolved to (inode, absolute path) or (ip, port) for sockets. This is the unglamorous half of audit forensics and most of the work.

---

## Forward and Backward Tracking

Once normalized, treat processes, files, and sockets as nodes and syscalls as directed edges. Both investigative questions become **taint propagation** over that graph, differing only in scan direction:

- **Backward tracking** starts from a PID and consumes the log in reverse to find origin: which binary was execve'd, which parent fork'd it, which read introduced the payload. Root cause.

- **Forward tracking** starts from an inode and scans forward to find consequences: who read the file, what those processes wrote, which socket the data left through. Blast radius.

The propagation rules are small. execve of a tainted inode taints the process; read/recv from a tainted inode taints the reading unit; write/send from a tainted unit taints the destination inode or socket; fork/clone from a tainted unit taints the child. Iterate until the tainted set stops growing.

Details bite here: clone() creates both processes and threads, distinguished only by whether the child-stack argument is non-null; taint them identically and the process tree is wrong. And accept() returns the new descriptor rather than receiving it as an argument, so the fd must be read from the return value.

---

## The Part That Breaks Naively

Propagate at process granularity and you get **dependency explosion**; the problem that made audit log provenance impractical for years.

A long-running server reads hundreds of files over an hour. If any input is tainted, the whole process is tainted, and every subsequent output looks causally downstream of the attack. Trace forward from a real compromise and you implicate half the system.

**Execution partitioning**, introduced by Lee, Zhang, and Xu in <a href="https://kyuhlee.github.io/publications/ndss13.pdf">BEEP (NDSS 2013)</a>, fixes this by splitting a process into units, typically iterations of its request-handling loop, identified by (tid, thread creation time, loop id, iteration). Taint propagates per unit, so a poisoned request taints only the unit that handled it. Concurrent requests stay independent unless they genuinely share state.

Two filters matter more than they sound:

- **Library files are excluded.** Nearly every process maps libc; left in, shared objects become hubs connecting everything to everything.
- **Propagation is time-bounded.** Inode numbers are recycled on delete and recreate, so an inode alone is not an identity. Keying on (inode, created_eid) and constraining propagation to a causally valid window prevents edges between unrelated files that reused a number.

---

## The Pipeline

1. **Normalize** raw audit logs into a 34-field CSV: event ID, timestamp, syscall and return value, args, tid/pid/ppid, resolved descriptors with inodes and absolute paths, socket addresses, uid/euid, cwd.

2. **Index** process, descriptor, and inode tables in a first pass and cache them to disk. Investigations are rarely one query, and rebuilding this state per query is wasted work.

3. **Track** forward or backward from an inode or PID, then emit Graphviz. In the graph, processes are represented as ovals, files as rectangles,and sockets as diamonds.

---

## Where This Leads

This all assumes you can read the logs freely, which stops being true once audit data moves to cloud storage and has to be encrypted first. Decrypting an entire archive to investigate one incident exposes far more than the investigation needs.

That is the problem I took up in <a href="https://ieeexplore.ieee.org/document/10917745">FA-SEAL</a>, which runs this same recursive causal tracking over encrypted logs while decrypting only the segments an investigation actually touches. The tracking logic underneath is what this post describes.

---

## Learn More

- <a href="https://github.com/BasantaChaulagain/log_analyzer">**Code:**</a> audit log normalization plus forward and backward tracking.

Credit where it is due. The execution partitioning that makes any of this tractable is not my contribution — it comes from <a href="https://kyuhlee.github.io/publications/ndss13.pdf">BEEP (Lee, Zhang, and Xu, NDSS 2013)</a>. My contributions to this repository are the structured CSV conversion pipeline, the time-bounded propagation logic, and substantial changes to the forward and backward tracking implementations.
