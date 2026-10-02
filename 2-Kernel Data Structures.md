# Part 2: Kernel Data Structures (`task_struct`, VFS), PID Mechanics, and `/proc` Introspection

---

### PILLAR A: Ground Zero Concept

To the Linux kernel, a process isn't just a program running—it is a complex, active record stored inside the kernel’s internal memory database.

When a program is launched, the kernel must manage several operational questions continuously:

* *Who owns this execution context?* (Security Credentials & Permissions)
* *Which regions of physical RAM can this process access?* (Virtual Memory Map)
* *Which files, pipes, or network connections are open?* (File Descriptor Table)
* *Is the process currently executing on a CPU core, waiting for disk I/O, or sleeping?* (Execution State)

To handle this, the kernel creates a centralized control structure for every running process and thread.

#### The Core Problem: Process Isolation vs. System Management

If processes had unrestricted access to hardware or each other's memory, a bug or malicious action in one application could corrupt the entire system, read sensitive data from other applications, or crash the OS. The kernel operates as an isolated referee.

It gives each process the **illusion** that it possesses exclusive control over a continuous, private virtual memory space and CPU. Behind this abstraction, the kernel tracks, allocates, and restricts hardware access.

---

### PILLAR B: Low-Level Hardware & Kernel Mechanics

#### 1. The Central Data Structure: `task_struct`

In the Linux kernel source code (defined in `<linux/sched.h>`), every process or thread is represented by a C structure called `task_struct` (the **Process Control Block** or **PCB**).

```
 +-------------------------------------------------------------------+
 |                         task_struct (Kernel Space)                |
 +-------------------------------------------------------------------+
 |  - pid_t pid;                 <-- Process Identifier              |
 |  - volatile long state;       <-- TASK_RUNNING, TASK_INTERRUPTIBLE|
 |  - struct mm_struct *mm;      <-- Virtual Memory Management Struct|
 |  - struct files_struct *files;<-- File Descriptor Table Pointer   |
 |  - struct cred *cred;         <-- User ID, Group ID, Capabilities |
 |  - struct task_struct *parent;<-- Parent Process Pointer          |
 +-------------------------------------------------------------------+

```

Key members within `task_struct`:

* **`pid` (Process ID):** A numeric identifier assigned to the thread/process context.
* **`mm` (`mm_struct`):** Points to the memory management structure. It defines every valid Virtual Memory Area (VMA) the process is permitted to access.
* **`files` (`files_struct`):** Points to the **File Descriptor (FD) Table**. Index `0` represents `stdin`, `1` is `stdout`, `2` is `stderr`. Any opened file, pipe, or network socket receives an entry in this array.
* **`cred` (`cred` structure):** Holds the process's effective credentials (Real/Effective User ID, Group ID, and POSIX Capabilities).

#### 2. Virtual Memory Switching via `CR3`

On x86-64 architectures, the CPU relies on a control register named **`CR3`** to handle virtual memory address translation.

* `CR3` stores the physical base address of the top-level **Page Table** (PML4/PML5) for the currently active process.
* During a **Context Switch**, the kernel updates `CR3` to point to the page table of the newly scheduled process.
* Once `CR3` is updated, the Memory Management Unit (MMU) automatically translates virtual addresses using the new process's page table mapping, enforcing memory isolation between processes.

#### 3. `/proc`: Kernel Data Structure Introspection

Linux provides a virtual filesystem mounted at `/proc`. Files inside `/proc` do not reside on physical storage; they are synthetic interfaces generated dynamically by the kernel upon request.

Reading paths under `/proc/[PID]/` exposes internal information from that process's `task_struct`:

* `/proc/[PID]/maps` $\rightarrow$ Exposes layout defined in `mm_struct` (Virtual memory regions and permissions).
* `/proc/[PID]/fd/` $\rightarrow$ Exposes entries defined in `files_struct` (Open file descriptors and sockets).
* `/proc/[PID]/status` $\rightarrow$ Exposes process metadata including UIDs, parent PIDs, and state flags.

---

### PILLAR C: System & Administrative Inspection

You can inspect these kernel data structures directly using native system commands and file utilities.

#### 1. Mapping Process Memory Layout

To view the virtual memory map of a running process:

```bash
# View virtual memory regions and permissions for PID 1234
cat /proc/1234/maps

```

*Example Output Breakdown:*

```text
55a1b2000000-55a1b2001000 r--p 00000000 08:01 123456  /usr/bin/target
55a1b2001000-55a1b2005000 r-xp 00001000 08:01 123456  /usr/bin/target  <-- Executable Code (.text)
7ffda1200000-7ffda1221000 rw-p 00000000 00:00 0        [stack]           <-- Process Stack

```

#### 2. Inspecting Open File Descriptors

To inspect active descriptors maintained by a process:

```bash
# List open file descriptors for PID 1234
ls -l /proc/1234/fd/

```

*Example Output:*

```text
lrwx------ 1 user group 64 Oct  2 20:00 0 -> /dev/pts/0
lrwx------ 1 user group 64 Oct  2 20:00 1 -> /dev/pts/0
lrwx------ 1 user group 64 Oct  2 20:00 3 -> socket:[98765]

```

---

### PILLAR D: Security Concepts & Analysis Perspective

Understanding how the kernel represents processes provides insight into process boundaries, auditing mechanisms, and security configurations.

```
       Security Context Analysis & Inspection Model
       
      +------------------------------------------------+
      |             Process Security Boundary          |
      +------------------------------------------------+
                               |
                               v
      +------------------------------------------------+
      |   Kernel Data Verification: `task_struct`      |
      |   Validates credentials, permissions, and VMAs |
      +------------------------------------------------+
                               |
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
   [Process Inspection]                 [Access Control]
   Read `/proc/[PID]/maps`              Verify `cred` UID/GID
   for layout diagnostics.              against target resources.

```

#### 1. Security Architecture Concepts

* **Process Credential Isolation:**
System security relies on the integrity of the `cred` structure associated with a `task_struct`. Access control decisions (such as opening a file or binding a restricted port) compare the process's effective UID/GID and POSIX capabilities against the requested resource's access control list (ACL).
* **Information Disclosure Analysis via `/proc`:**
The `/proc` filesystem provides visibility into process state. In environments where process boundaries must be strictly maintained, unprivileged access to `/proc` layout files (`/proc/[PID]/maps`) can expose address spaces or operational metadata.

#### 2. Defensive Hardening Mechanics

* **`yama_ptrace_scope` Controls:**
* *Purpose:* Restricts process tracing and memory inspection capabilities (such as `ptrace` or debugger attachments).
* *Implementation:* Managed via `/proc/sys/kernel/yama/ptrace_scope`. Setting this value to `1` or higher restricts processes from attaching to or inspecting other running processes unless an explicit parent-child relationship exists or appropriate administrative privileges are present.


* **Restricting `/proc` Access (`hidepid` Mount Option):**
* *Purpose:* Restricts visibility of process directories under `/proc`.
* *Implementation:* System administrators can mount `/proc` using the `hidepid` flag (e.g., `hidepid=2`). This prevents standard users from observing process identifiers, command lines, or memory mappings belonging to other users on the system.