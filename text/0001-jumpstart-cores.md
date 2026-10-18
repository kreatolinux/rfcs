# Jumpstart cores

| | |
| --- | --- |
| Status | Draft, revision 8 |
| Target | Jumpstart 2.x (first implementation after v1.2.0) |
| Owner | `jumpctl` and the Jumpstart daemon |
| Depends on | Kongue structured header values and manifest mode (§10.2, §10.3); native targets in `jumpstart_dec(5)` (§22) |
| Updated | 2026-10-18 |

## Abstract

This document specifies **cores**, the mechanism Jumpstart uses to host
service managers. The long-term goal is for any service manager (systemd,
OpenRC, dinit, runit, s6, SysV init, and others) to run under Jumpstart as a
core.

A core declares, in a machine-checkable manifest, every operation it
implements, the backend that performs each operation, the options that govern
how that backend runs, and the Kongue functions or executables involved.
Jumpstart routes requests to cores using only these declarations. It never
infers behavior, falls back to another core implicitly, or applies an
undocumented default.

`serviceHandler` is the only core compiled into Jumpstart. Every other core,
such as a systemd compatibility core, is an extension discovered from the
filesystem. Each core names its own units, owns its own processes inside its
own cgroup, and decides how its units come up at boot. Cores can be loaded,
unloaded, and reloaded in a running daemon one at a time. The system daemon
and per-user daemons each have their own cores, which are kept strictly apart
so that user cores can never affect the system daemon.

## Table of contents

1. Conventions
2. Terminology
3. Motivation
4. Goals and non-goals
5. Architecture
6. Core identity and naming
7. Operating modes
8. Discovery
9. Enabled-state storage
10. The `init.kg` manifest
11. Operation model
12. Operation registry
13. Backends
14. Unit naming and request routing
15. Process ownership
16. Kongue runtime
17. Exec protocol
18. Error model
19. Filesystem state, logging, and resources
20. Daemon lifecycle
21. Command-line interface
22. Native targets
23. Versioning and compatibility
24. Security considerations
25. Packaging responsibilities
26. Implementation structure
27. Tests
28. Acceptance criteria
29. Open questions

Appendices: A (`serviceHandler` resolved manifest), B (example systemd
manifest), C (manifest grammar), D (mapping other service managers),
E (revision history).

---

## 1. Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be
interpreted as described in RFC 2119 and RFC 8174 when, and only when, they
appear in capitals.

Paths are given for a conventional `/usr` prefix. `~` means the home directory
of the daemon's user as defined in §7.1. All paths are injectable for testing
(§26).

Code blocks labelled `kongue` are Kongue documents. Code blocks labelled
`text` are layouts or grammar. Code blocks labelled `console` show terminal
sessions; `$` marks a user prompt and `#` a root prompt.

Byte order means comparing names as unsigned byte strings. It is not
locale-aware.

### 1.1 The "no hidden defaults" rule

The central design rule of this specification is:

> **Jumpstart MUST NOT decide how an operation is performed from anything other
> than the core's manifest and the defaults written in this document.**

In practice:

1. Every manifest key is either REQUIRED, or OPTIONAL with a default stated in
   this document. No default is "implementation-defined".
2. Options that affect security or lifecycle (`sandbox` where it applies,
   `timeout`) are REQUIRED and have no default.
3. An operation that a core does not declare is unsupported. Jumpstart MUST NOT
   satisfy it with another core unless the manifest declares that delegation
   explicitly (§13.4).
4. `jumpctl cores show --resolved` (§21.11) MUST print the fully resolved
   operation table, with every default filled in, so administrators can see
   exactly what Jumpstart will do.

Where a behavior belongs to the service manager a core implements (how its
units are named, how it orders them, what "enabled" means to it), the
specification says so explicitly and leaves it to the core. That is a
documented delegation of responsibility, not a hidden default.

## 2. Terminology

Core
: A module that implements service-manager operations for Jumpstart.

Native core
: `serviceHandler`, which is compiled into Jumpstart.

Extension core
: A core installed as a directory beneath a core root (§8).

Discovered core
: The native core, or an extension that passes discovery validation (§8).

Enabled core
: A discovered core that is selected in persistent state (§9). The daemon tries
  to load it at startup.

Loaded core
: A core that is running in a daemon and receiving requests (§20.1).

Subsystem
: A family of related operations, such as `service`, `mount`, or `log`. Each
  subsystem's operations are listed in the registry (§12).

Verb
: The action part of an operation within a subsystem, such as `start`.

Operation
: A fully qualified `subsystem.verb` pair, such as `service.start`. Operations
  are the unit of capability, routing, and dispatch.

Operation entry
: One item of a manifest's `supported_ops` list. It binds one operation to a
  backend.

Backend
: The mechanism that performs an operation: `custom`, `exec`, `delegate`,
  `unsupported`, or (for the native core only) `native`.

Role
: A named slot in a `custom` operation that is filled by a Kongue function. In
  `manifest_version: 1` the only role is `handler`.

Managed payload
: The upstream software a core drives, such as the systemd binaries driven by a
  systemd core. It is distinct from the extension directory itself.

Unit
: A thing a core manages, such as a service, mount, timer, or target.

Unit name
: The name of a unit, as defined by its owning core (§14.1). Jumpstart treats
  unit names as opaque strings.

Owner
: The single core responsible for a given unit.

Supervisor
: An optional long-lived process belonging to a core that owns the core's
  service processes (§15.2).

Adapter
: A process that receives operation requests over the exec protocol (§17).

Target
: A unit that groups other units and can be reached as a whole. Runlevels are
  targets (§12.7).

Activation
: Bringing a core's default set of units up after it loads, and taking them
  down before it unloads (§11.5, §20.6).

Installation and enablement are separate. Installing an extension makes it
discoverable but never executes or enables it. Enabling it selects it for
loading. Neither extensions nor their managed payloads are installed by
Jumpstart; that is kpkg's job (§25).

## 3. Motivation

Revision 1 described each core's capabilities as a flat list of strings:

```kongue
supported_ops:
  - "start"
  - "stop"
```

That is not enough for a core such as systemd, which has many independent
subsystems a compatibility layer may reproduce in part: services, mounts,
timers, sockets, targets, and the journal. It is even less enough for the goal
of running *every* service manager under Jumpstart, because those managers
disagree on almost everything: how units are named and stored, who owns
service processes, how boot ordering works, and what "enabled" means. A flat
list cannot express:

- which subsystem an operation belongs to (`start` a service versus `start` a
  mount);
- how the operation is carried out (Kongue code, an adapter binary, or another
  core);
- the constraints it runs under (sandboxing, timeouts, privilege);
- which functions implement the work; or
- whether an unlisted operation is intentionally unsupported or merely
  forgotten.

Without those, Jumpstart would have to guess, which is exactly the behavior
this specification forbids (§1.1). This specification replaces the list with
structured operation entries:

```kongue
supported_ops:
  - name: "unit.validate"
    backend: "custom"
    options:
      sandbox: true
      timeout: "10s"
    functions:
      - handler: "validate_unit"
  - name: "service.start"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "90s"
```

It also leaves the parts that genuinely differ between managers (unit names,
process supervision, boot ordering) to each core, behind fixed interfaces.

## 4. Goals and non-goals

### 4.1 Goals

- Make it possible for any service manager to run under Jumpstart as a core,
  without Jumpstart special-casing any of them.
- Keep Jumpstart fully functional with only `serviceHandler`.
- Discover independently installed extension cores without a compiled catalog.
- Let administrators enable and disable cores persistently, and load, unload,
  or reload a single core in a running daemon without restarting the others.
- Support per-user cores that are wholly isolated from the system daemon
  (PID 1), and start per-user daemons for logged-in users.
- Describe every operation a core performs, and how it performs it,
  declaratively and completely.
- Route each request to exactly one core deterministically, or fail with a
  specific error.
- Give every core's processes a defined owner and a defined end, so that no
  core can leak, orphan, or strand processes.
- Define how the whole system shuts down, including every loaded core.
- Keep discovery and listing free of code execution.
- Keep the generic implementation free of any reference to systemd, dinit,
  OpenRC, or any package manager.

### 4.2 Non-goals

- Package management. Jumpstart does not build, download, install, upgrade,
  or remove extensions or their managed payloads, and has no package-manager
  backend or integration. It only discovers what is already on disk.
- Upgrading, downgrading, and rolling back core files. Cores are packaged and
  managed by kpkg, which owns their file-level lifecycle. Jumpstart's only
  obligation is API compatibility (§23): it rejects cores it cannot run, and it
  reloads a core when asked (§20.5).
- Cross-core ordering. A unit owned by one core cannot depend on, or be ordered
  after, a unit owned by another core. Each core orders only its own units.
- Compatibility commands such as `systemctl`, `rc-service`, `rc-update`, `sv`,
  `service`, `update-rc.d`, `shutdown`, and `telinit`. Cores MAY ship such
  shims as part of their payload. If they do, the shims SHOULD drive Jumpstart
  through `jumpctl` or the control socket (§14.5) rather than a separate
  daemon. A shim MAY instead talk to the core's own socket, for example when
  the manager's interface can express things `jumpctl` cannot (job modes,
  `--no-block`, property queries, `is-active` exit codes). Commands that do so
  bypass Jumpstart: they take none of its locks (§11.6), are outside its
  admission limits (§19.3), its timeouts and its cancellation (§11.6, §13.7),
  and do not appear in its operation log (§19.2). The core is responsible for
  equivalent behavior: it serializes such requests with the ones Jumpstart
  routes to it in its own job engine, limits them, and records what it ran.
  Jumpstart neither ships nor specifies them.
- A remote or compiled catalog of possible extensions.
- Implementing the systemd, dinit, OpenRC, or other cores. This document only
  defines the contract they implement. Appendices B and D are illustrative.

## 5. Architecture

```text
             ┌──────────────────────────── jumpctl ─────────────────────────────┐
             │ cores list/enable/disable/show/check, cores run   (local files)  │
             │ unit verbs, units, logs, cores load/unload/...,                  │
             │ poweroff/reboot/halt, users, env                  (via socket)   │
             └───────────────┬──────────────────────────────────┬───────────────┘
                             │ reads/writes                     │ JSON over the
                             ▼                                  ▼ mode's socket (§7)
 core roots  ──────────►  Discovery (§8)         ┌──────────────────────────────┐
 enabled state ───────►  Enabled state (§9)      │       Jumpstart daemon       │
                                                 │  Core registry & lifecycle   │
                                                 │  Router (§14)                │
                                                 │  Dispatcher (§13)            │
                                                 │  Reaper, cgroups (§15)       │
                                                 └──┬──────┬──────┬──────┬──────┘
                                                    │      │      │      │
                                          native ───┘      │      │      └── delegate
                              custom (Kongue runner, §16) ─┘      └── exec (adapter, §17)

          per core:  /sys/fs/cgroup/jumpstart/cores/CORE/
                       control/   adapters, Kongue runners, supervisor
                       units/…    the core's unit processes (core-managed)
```

The pipeline for every unit request is fixed:

1. **Resolve** the unit reference to an owner core (§14.3).
2. **Map** the request to an operation name (§14.4).
3. **Look up** the operation entry in the owner's resolved table.
4. **Dispatch** to the declared backend with its resolved options (§13).
5. **Return** a result or a typed error (§18).

There is no step at which Jumpstart chooses a different core or backend than
the manifest declares.

## 6. Core identity and naming

A core name:

- MUST be 1 to 64 bytes long;
- MUST begin with an ASCII letter;
- MUST contain only ASCII letters, digits, `_`, or `-`; and
- is case-sensitive.

The following names are reserved and MUST NOT be used by extensions:
`serviceHandler`, `jumpstart`, `native`, `all`, `none`.

## 7. Operating modes

Jumpstart runs in one of two modes: **system mode**, normally as PID 1, and
**user mode**, as an unprivileged per-user daemon. Cores follow the same split.
Each mode has its own core roots, enabled state, per-core directories,
cgroups, and socket.

### 7.1 Mode selection

- The daemon runs in user mode if and only if it is started with `--user`.
  `--user` requires a non-zero real UID, and system mode requires real UID 0.
  Any other combination exits with status 2 before doing anything else. (Today
  `jumpstart --user` as root runs user mode on system paths. This
  specification forbids that.)
- `jumpctl` operates in system mode when its real UID is 0, and in user mode
  otherwise. There is no option to choose the mode.
- In user mode, `~` is the home directory from the password database entry of
  the real UID (`getpwuid(getuid())`). Jumpstart MUST NOT take it from `HOME`
  or any other environment variable.

### 7.2 Paths

| Purpose | System mode | User mode |
| --- | --- | --- |
| System core root | `/usr/lib/jumpstart/cores` | `/usr/lib/jumpstart/cores` (read-only use, §7.3) |
| User core root | — | `~/.local/lib/jumpstart/cores` |
| Unit config directory (native) | `/etc/jumpstart` | `~/.config/jumpstart` |
| Enabled state | `/etc/jumpstart/cores/enabled` | `~/.config/jumpstart/cores/enabled` |
| State lock | `/etc/jumpstart/cores/.lock` | `~/.config/jumpstart/cores/.lock` |
| Persistent per-core state | `/var/lib/jumpstart/cores/CORE` | `~/.local/state/jumpstart/cores/CORE` |
| Runtime per-core state | `/run/jumpstart/cores/CORE` | `RUNTIME/jumpstart/cores/CORE` |
| Core cgroup | `/sys/fs/cgroup/jumpstart/cores/CORE` | `/sys/fs/cgroup/jumpstart/users/UID/cores/CORE` |
| Daemon log | `/var/log/jumpstart.log` | `~/.local/state/jumpstart/jumpstart.log` |
| Daemon socket | `/run/jumpstart.sock` | `~/.local/share/jumpstart/jumpstart.sock` |
| Linger markers | `/etc/jumpstart/users/linger/` | — |

In user mode, `RUNTIME` is `/run/user/UID` when that directory exists, is
owned by the user, and has mode `0700` (it is created by the system daemon,
§7.6). Otherwise `RUNTIME` is `~/.local/share/jumpstart/runtime`, which the
user daemon empties when it starts because it lives on persistent storage.

The user socket lives in its own directory, `~/.local/share/jumpstart/`, which
the user daemon creates with mode `0700`. The previous location
(`~/.local/share/jumpstart.sock`) is no longer used.

Paths are fixed. Apart from `~` as defined in §7.1, Jumpstart MUST NOT consult
`XDG_*` or other environment variables to relocate them. The system core root
SHOULD derive from the compile-time prefix. Tests MUST be able to inject every
root.

### 7.3 Which cores each mode sees

A manifest declares the modes it supports in its `modes` field (§10.4).

- **System mode** discovers `serviceHandler` and the system core root. It sees
  only extensions whose `modes` include `"system"`.
- **User mode** discovers `serviceHandler`, then the system core root
  (extensions whose `modes` include `"user"`), then the user core root. Every
  user-root extension MUST include `"user"` in `modes` (`E_MANIFEST`
  otherwise).

In user mode, a user-root extension whose name matches a system extension is
rejected with `E_NAME`. User cores never shadow system cores. System cores
loaded by a user daemon run entirely with that user's credentials.

Listing order in user mode is `serviceHandler`, then system extensions in byte
order, then user extensions in byte order.

A system extension that does not list a mode is not visible in that mode. This
is not an error.

### 7.4 Isolation

User cores MUST be wholly separate from the system daemon, so that installing
or running a user core can never escalate privilege:

1. The system daemon MUST NOT discover, read, parse, load, or execute anything
   from a user core root, user enabled state, user per-core directories, or
   user socket. Its only interactions with a user daemon are the ones in §7.6:
   creating its runtime directory and cgroup, spawning it with dropped
   credentials, signalling it, and observing its exit.
2. A user daemon runs with only its user's credentials. It MUST NOT be setuid,
   MUST NOT hold capabilities, and has no privileged channel to the system
   daemon.
3. In user mode, every operation runs as the daemon's user. The `user` option
   and the `account` field (§10.4, §13.1) are ignored there, and
   `cores show --resolved` shows the effective user.
4. Delegation (§13.4) never crosses daemons. Targets MUST be loaded in the same
   daemon.
5. The system socket MUST be created with mode `0600`, owned by root. The
   daemon MUST also check each peer's credentials (`SO_PEERCRED`) and reject
   peers whose UID is not 0 with `E_PERMISSION`.
6. The user socket MUST be mode `0600` inside a `0700` directory. The daemon
   MUST reject peers whose UID differs from its own.
7. Both daemons MUST handle the socket robustly (§14.5): malformed input,
   oversize lines, and idle connections produce errors or are closed, and MUST
   NOT stop or stall the daemon.
8. In system mode, cores MUST NOT read unit definitions from files or
   directories that are not owned by root or that are writable by group or
   other. Such definitions MUST be ignored and reported in `unit.load` and
   `unit.validate` diagnostics. Jumpstart cannot enforce this, because unit
   storage is core-defined (§14.1). It is a requirement on every system core.
9. In user mode, extension directories and manifests in the user root MUST be
   owned by the user and not writable by group or other. System extensions
   seen in user mode keep the system-mode ownership rules (§8.3).
10. In user mode, `sandbox: true` uses unprivileged user namespaces. If they
    are unavailable, the operation fails with `E_SANDBOX`.
11. User mode never performs PID 1 duties.

### 7.5 Operation environment

Every process Jumpstart starts for a core (Kongue runners, adapters,
supervisors, §15) receives an environment built in this order, where later
layers override earlier ones:

1. **Base environment.**
   - System mode: `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`
     and `LANG=C.UTF-8`.
   - User mode: `HOME`, `USER`, `LOGNAME`, and `SHELL` from the password
     database; `PATH=/usr/local/bin:/usr/bin:/bin`; `LANG=C.UTF-8`; and
     `XDG_RUNTIME_DIR=/run/user/UID` when `RUNTIME` is `/run/user/UID`
     (§7.2).
2. **User environment store** (user mode only). Variables set with
   `jumpctl env` (§21.9), such as `DISPLAY`, `WAYLAND_DISPLAY`,
   `DBUS_SESSION_BUS_ADDRESS`, or a different `PATH`. The store is held in
   memory by the user daemon and starts empty. It cannot set `HOME`, `USER`,
   `LOGNAME`, `SHELL`, or any `JUMPSTART_*` variable.
3. **Manifest `environment`** of the operation (§13.1), or of the supervisor
   (§15.2).
4. **`JUMPSTART_*` variables** (§16.2). These can never be overridden.

Nothing else is inherited. In particular, the daemon's own environment is not
passed through. Long-lived processes (supervisors and persistent adapters)
receive the environment as it is when they start. A running supervisor or
persistent adapter therefore sees later `jumpctl env` changes only after the
core is reloaded (§20.5), while every process Jumpstart starts afterwards, such
as a oneshot adapter or a Kongue runner, sees them at once.

A core MAY offer its own command that pushes environment variables into its
running supervisor or adapter without a reload, for example a compatibility
front-end's `import-environment` (§4.2). Such a command bypasses the daemon's
environment store: Jumpstart does not see, validate, persist, or log what it
passes, and the store's contents do not change. Keeping it consistent with
this section (in particular the protected variables `HOME`, `USER`, `LOGNAME`,
`SHELL`, and `JUMPSTART_*`) is the core's responsibility.

### 7.6 User daemon sessions

The system daemon starts and stops user daemons for logged-in users:

1. Jumpstart ships a PAM session module, `pam_jumpstart.so`. On session open
   and close it sends a session request (§14.5) to the system socket as root,
   with the user's UID and an opaque session identifier.
2. Session requests for UID 0 are answered with success and otherwise
   ignored, because root has no user daemon (§7.1).
3. The system daemon tracks open sessions per UID. Each session is tied to
   the process that sent the open request: the daemon takes a pidfd for the
   peer, and the session closes either
   on a matching close request or when that process exits. A login process
   that dies without closing its session therefore cannot leak it.

   The pidfd comes from `SO_PEERPIDFD` where the kernel supports it. Otherwise
   the daemon reads the PID from `SO_PEERCRED`, calls `pidfd_open`, and then
   checks that the process's start time (field 22 of `/proc/PID/stat`) is
   earlier than the connection's acceptance and that its UID is still 0. If
   the check fails, the PID was reused and the request fails with
   `E_PERMISSION`.
4. On the first open session for a UID that has no user daemon, the system
   daemon:
   1. creates `/run/user` if needed, owned by root with mode `0755`;
   2. mounts a dedicated tmpfs on `/run/user/UID` with
      `size=10%,mode=0700,uid=UID,gid=GID,nosuid,nodev`. If `/run/user/UID`
      already exists and is not such a mount, the session start fails with
      `E_PERMISSION`. The size cap keeps one user from filling `/run`;
   3. creates the cgroup `/sys/fs/cgroup/jumpstart/users/UID/` and its leaf
      `daemon/`, and delegates `users/UID/` to the user exactly as in §15.1
      rule 4 (the directory and its `cgroup.procs`, `cgroup.threads`, and
      `cgroup.subtree_control`, nothing else);
   4. spawns `jumpstart --user` as the user directly into `users/UID/daemon/`.
      It sets supplementary groups (`initgroups`), the GID, and the UID, drops
      all capabilities, sets `PR_SET_NO_NEW_PRIVS`, and uses the user-mode
      base environment (§7.5) and the home directory as working directory;
      and
   5. waits up to 10 seconds for the user socket to accept connections, then
      answers the session request. Failure is reported in the response but
      does not block the login.
5. On the last session close for a UID, unless the user lingers, the system
   daemon sends `SIGTERM` to the user daemon, waits up to 30 seconds, sends
   `SIGKILL`, kills anything left in the user's cgroup (§15.1), and removes
   the cgroup. It then unmounts `/run/user/UID` (lazily if busy) and removes
   the now-empty mount point. It never walks or deletes files inside the
   user's runtime directory.
6. **Lingering.** A user lingers when the root-owned empty file
   `/etc/jumpstart/users/linger/USERNAME` exists. Markers are keyed by user
   name and resolved to a UID through the password database when they are
   read, so renaming a user drops their linger setting. Lingering users'
   daemons are started at system boot, after every system core has loaded
   and finished activation (§20.6), and are not stopped when their last
   session closes. `jumpctl users linger` (§21.8) manages the markers.
7. A user daemon exit is **unexpected** unless the system daemon caused it
   (§7.6 step 5, §20.8) or the daemon exited with status 0 (it was asked to
   stop, for example by its own user). After an unexpected exit while the user
   has open sessions or lingers, the system daemon restarts it, at most 5
   times in 60 seconds. After that, or after an exit with status 0, it logs
   the event and does not restart the daemon until the user's next session
   opens.
8. A user may also run `jumpstart --user` by hand. Such a daemon gets no
   delegated cgroup, so it can load only `serviceHandler` (extension cores
   fail with `E_CGROUP`, §15.1), and it uses the fallback `RUNTIME` (§7.2).
   If a hand-started daemon already holds the user socket when the system
   daemon tries to start one, the new daemon finds the socket in use and
   exits, logging why. The system daemon reports this in the session
   response, leaves the
   hand-started daemon alone, and does not count the exit toward the restart
   budget in step 7. It tries again on the user's next session open.

Without cgroup v2, steps 4.3 and 5's cgroup handling are skipped. User
daemons still start and stop with sessions, but they can load only
`serviceHandler` (§15.1).

The system daemon performs only the steps above. It never connects to a user
socket, never reads files the user controls, and never acts on anything a
user daemon says.

## 8. Discovery

### 8.1 Core roots

Each mode discovers the roots listed in §7.3, at the paths in §7.2. The rest
of this section describes one root. In user mode the procedure runs over the
system root and then the user root.

### 8.2 Discovery procedure

Discovery produces an ordered list of cores:

1. The virtual native entry `serviceHandler`, always first.
2. Every direct child of each core root, validated as described in §8.3, in
   byte order of name within each root.

There is no compiled catalog. Discovery MUST NOT execute any Kongue function,
command, interpolation, or executable. It MUST NOT query any package-manager
database.

If a core root does not exist, it contributes nothing. If it exists but cannot
be read, discovery fails.

### 8.3 Extension layout and validation

```text
/usr/lib/jumpstart/cores/
  systemd/
    init.kg            REQUIRED  manifest (§10)
    bin/               OPTIONAL  executables referenced by exec backends and supervisors
    lib/               OPTIONAL  data and Kongue helpers, never executed directly
    share/             OPTIONAL  read-only data
```

Every direct child of a core root is a claim to be an installed extension.

**Ownership.** The *trusted owner* is root for the system core root, and the
user for the user core root. Every one of the following MUST be owned by the
trusted owner (or by root) and MUST NOT be writable by group or other:

- the core root and each of its parent directories;
- every directory and file in the extension tree, recursively; and
- in system mode, `/etc/jumpstart/cores`, its enabled-state directory and
  lock, `/var/lib/jumpstart/cores`, and `/run/jumpstart/cores`.

**No symlinks.** The core root, and every entry in an extension tree at any
depth, MUST NOT be a symbolic link.

Discovery MUST reject the core, with the reason, when any of these holds:

| Condition | Error code (§18) |
| --- | --- |
| Entry is not a directory | `E_LAYOUT` |
| Name invalid or reserved (§6) | `E_NAME` |
| User-root name matches a system extension (§7.3) | `E_NAME` |
| Any symlink in the extension tree | `E_LAYOUT` |
| Top-level entry other than `init.kg`, `bin`, `lib`, `share` | `E_LAYOUT` |
| `init.kg` missing or not a regular file | `E_MANIFEST` |
| Manifest fails parsing or validation (§10) | `E_MANIFEST` |
| Manifest `name` differs from the directory name | `E_MANIFEST` |
| User-root manifest whose `modes` lacks `"user"` | `E_MANIFEST` |
| Ownership or mode rule above violated | `E_PERMISSION` |
| Any path referenced by the manifest escapes the core directory | `E_LAYOUT` |
| Manifest requires an incompatible Jumpstart version (§23.2) | `E_VERSION` |

A system extension whose `modes` lacks the current mode is not listed, and
this is not an error.

**Executing core files.** Ownership can change between discovery and use.
Jumpstart MUST therefore open each executable and manifest with
`O_NOFOLLOW`, re-check owner, mode, and file type with `fstat` on the open
descriptor, and execute executables through that descriptor (`execveat` with
`AT_EMPTY_PATH`, or `fexecve`). A failed re-check fails the operation, or the
load, with `E_PERMISSION`.

Discovery of one invalid extension does not prevent discovery of others.
Invalid extensions are reported by `jumpctl cores check` (§21.12) and are never
listed as discovered. An invalid extension that is enabled fails to load
(§20.2).

### 8.4 Determinism

Two discovery runs over identical filesystem contents MUST produce the same
cores in the same order, and the same set of rejected entries with the same
error codes.

## 9. Enabled-state storage

### 9.1 Layout

```text
/etc/jumpstart/cores/enabled/
  serviceHandler
  systemd
```

Each enabled core has an empty regular marker file named exactly after the
core. Markers are used rather than links, so enabled state cannot redirect
discovery and does not depend on installation paths.

The state directory MUST contain only valid markers. A file with an invalid
name, a non-empty file, a directory, or a symlink makes the state invalid
(`E_STATE`).

System-mode paths are shown. User mode uses the user paths from §7.2 with
identical rules. The two enabled sets are independent: enabling a core in one
mode never affects the other.

### 9.2 Uninitialized state

If the enabled-state directory does not exist, the state is
**uninitialized** and the effective enabled set is exactly `{serviceHandler}`.
This preserves existing installations.

The first successful `enable` or `disable` materializes the effective set as
markers before applying the change. So the first `enable systemd` produces
markers for both `serviceHandler` and `systemd`.

An existing but empty enabled directory is invalid (`E_STATE`). It is never
treated as a request to restore defaults. How the daemon starts with invalid
state is defined in §20.2.

### 9.3 Atomicity

All state changes hold an exclusive lock on the mode's state lock (§7.2), so
concurrent `jumpctl` invocations are serialized.

- **Materialization** builds the complete marker set in a temporary sibling
  directory on the same filesystem, `fsync`s each marker and the directory,
  then `rename`s the directory into place. The target path does not exist at
  that point, so the rename is atomic.
- **Enabling** an already-materialized state creates the marker as a
  temporary file in the state directory, `fsync`s it, renames it to its final
  name, and `fsync`s the directory.
- **Disabling** unlinks the marker and `fsync`s the directory.

The last-enabled-core rule (§9.4) is checked under the lock, so an unlink can
never leave the directory empty. A crash at any point MUST leave either the old
or the new complete state.

### 9.4 Invariants

- The enabled set MUST NOT be empty once initialized.
- Delegation targets of enabled cores MUST be enabled (§13.4).

`jumpctl` enforces both when it writes state. An enabled core that is missing
or invalid is not a state error. It fails to load (§20.2).

The enabled set says what loads at daemon start. The loaded set (§20.1) says
what is running now. They differ after `cores load` or `cores unload`
(§21.10), or when a core fails to load.

## 10. The `init.kg` manifest

### 10.1 Overview

The manifest is a Kongue document. Its header declares metadata, subsystems,
activation, an optional supervisor, and operations. Its function blocks hold
initialization, shutdown, and `custom` backend code.

```kongue
manifest_version: 1
name: "example"
description: "Example core"
core_version: "0.1.0"
version: "1.0"
jumpstart: ">=1.3.0, <2.0.0"
modes:
  - "system"

subsystems:
  - "unit"
  - "service"
activation: "core"

supervisor:
  executable: "bin/example-supervisor"

supported_ops:
  - name: "unit.resolve"
    backend: "exec"
    options:
      executable: "bin/example-adapter"
      mode: "persistent"
      timeout: "5s"
  - name: "service.start"
    backend: "exec"
    options:
      executable: "bin/example-adapter"
      mode: "persistent"
      timeout: "90s"
  # ...every other unit.* and service.* verb, see §11.3

init {
  print "example core initialized"
}
```

### 10.2 Kongue structured header values (dependency)

Kongue headers currently support string scalars, lists of strings, and flat
objects of string values. The manifest needs:

- lists whose items are mappings (`- name: …` followed by further keys);
- mappings nested inside mappings (`options:`);
- lists inside mappings (`writable_paths:`); and
- typed scalars: booleans (`true`, `false`) and non-negative integers.

This specification therefore depends on a Kongue grammar extension, which
MUST be specified in `kongue(5)` before this one is implemented. The extension
MUST:

1. use indentation-based nesting consistent with existing Kongue headers. A
   mapping's keys share one indentation column, and a list item's continuation
   keys align with the first key after `- `;
2. reject tabs in header indentation and inconsistent indentation;
3. reject duplicate keys within one mapping;
4. remain purely declarative, so header values are never interpolated or
   evaluated; and
5. leave existing documents parsing exactly as they do now.

Typed scalars apply only in manifest mode (§10.3). In ordinary Kongue
documents, such as kpkg run files and decfiles, `true` and `42` keep their
current meaning as strings.

The grammar is summarized in Appendix C.

### 10.3 Kongue manifest mode (dependency)

Ordinary Kongue is deliberately forgiving: variable names are case- and
style-insensitive (`core_version`, `coreVersion`, and `core-version` are the
same variable), the legacy form `name_qualifier` is equivalent to the
qualified form `name qualifier`, and headers accept `+:` and `-:` to modify
values. Those rules would make manifest validation ambiguous. Jumpstart
therefore parses `init.kg` in a **manifest mode**, which `kongue(5)` MUST
define, with these rules:

1. Header keys at every level are compared exactly: case-sensitive, with no
   style folding. Every key MUST be written exactly as this specification
   spells it.
2. The underscore form is never treated as a qualifier. `core_version` is one
   key, not `core` qualified by `version`.
3. `+:` and `-:` are rejected.
4. Unquoted scalars other than `true`, `false`, and non-negative decimal
   integers are rejected. Strings MUST be quoted.
5. Function names are compared exactly. A function named `init` or
   `shutdown` MUST NOT be qualified, and no other function name may begin
   with `init_` or `shutdown_`. Because Kongue stores a qualified function
   `name qualifier` as `name_qualifier`, a manifest that defines both forms
   for the same pair (such as `start_unit mount { … }` and
   `func start_unit_mount { … }`) is rejected.
6. Variable references in function bodies (`$name`, `${name}`, `${name.key}`,
   and bare names in conditions) are compared exactly too. `${Op.unit}` does
   not refer to `op`.
7. Comments (`#` to end of line) are allowed on their own lines and after any
   header value.
8. The `spawn` expression form (§16.3) is parsed as a call with a
   comma-separated list of quoted string arguments. Each argument is a
   separate token in the parse tree, so commas and quotes that appear in
   interpolated values cannot add arguments. This requires a Kongue parser
   change: today's expression parser does not support multi-argument calls.
   The grammar belongs in `kongue(5)` and is a dependency of this spec.
9. Function and method names in expressions MUST be literal. A call whose
   name is computed from a variable or interpolation (such as
   `${${name}(…)}`) is rejected, so static validation (§10.6) sees every
   function that can run.

### 10.4 Top-level fields

| Field | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `manifest_version` | integer | yes | — | Manifest format version. MUST be `1` for this specification. |
| `name` | string | yes | — | Core name (§6). MUST equal the directory name. |
| `description` | string | no | `""` | One-line human description, ≤ 200 bytes, no newlines. |
| `core_version` | string | yes | — | Version of the extension, `MAJOR.MINOR.PATCH`, each a non-negative decimal integer without leading zeros. |
| `version` | string | yes | — | Upstream manager version targeted. Opaque, non-empty, ≤ 64 bytes. |
| `jumpstart` | string | yes | — | Supported Jumpstart version range (§23.2). |
| `modes` | list of strings | yes | — | Modes the core supports: one or both of `"system"` and `"user"`, without duplicates (§7.3). |
| `account` | string | no | `"root"` | System mode only: the account that owns the core's state and runtime directories and cgroup (§15.1, §19.1), and the only non-root account its operations and supervisor may run as (§13.1). A non-root `account` caps what the core can do: it cannot, for example, run units as other users or bind privileged ports unless its manager gains those abilities some other way. Ignored in user mode. |
| `subsystems` | list of strings | yes | — | Subsystems this core claims (§11.3). May be empty only if `supported_ops` is empty. |
| `activation` | string | depends | — | `"core"` or `"per-unit"` (§11.5). REQUIRED if any unit-addressed subsystem is claimed, rejected otherwise. |
| `supervisor` | mapping | no | absent | The core's supervisor process (§15.2). |
| `supported_ops` | list of operation entries | yes | — | Operation entries (§11). |

Unknown top-level fields MUST be rejected (`E_MANIFEST`). Forward
compatibility is handled by `manifest_version`, not by ignoring fields.

### 10.5 Functions

| Function | Req. | Meaning |
| --- | --- | --- |
| `init` | yes | Run each time the daemon loads the core (§20.3). |
| `shutdown` | no | Run each time the daemon unloads the core (§20.4). If absent, this step does nothing. |
| any other | no | Usable only as the handler of a `custom` operation (§13.2), or called from another function. |

Function blocks use Kongue's standard syntax (`init { … }`, optionally
qualified, as in `start_unit mount { … }`). Custom helpers use `func name { … }`.
`func init` is not valid Kongue and MUST be rejected.

Exactly one `init` MUST exist. At most one `shutdown` MAY exist. A function
that is defined but not referenced by `init`, `shutdown`, a handler binding,
or another function SHOULD produce a warning from `jumpctl cores check`, but
is not an error.

### 10.6 Static validation

Validation MUST be performed without execution and MUST check:

- the Kongue document parses in manifest mode (§10.3);
- all top-level fields are known, unique, and of the right type;
- every operation entry is valid (§11.2), and the completeness and activation
  rules hold (§11.3, §11.5);
- every handler binding names an existing function (§13.2);
- no `custom` handler, or function reachable from one, uses `exec` or
  `${exec(…)}` (§16.4);
- `sandbox` appears exactly where §13.1 requires it;
- every referenced path is inside the core directory, exists, and, for
  executables, is a regular file under `bin/` with the owner execute bit set;
- delegation entries are well-formed (§13.4; cross-core checks happen at load
  time); and
- `jumpstart` parses as a version range.

Validation errors MUST name the manifest path, the 1-based line where
available, the field path (for example `supported_ops[3].options.timeout`),
and a concise reason.

## 11. Operation model

### 11.1 Operation names

An operation name has the form:

```text
operation = subsystem "." verb
subsystem = lower *( lower / digit / "-" )
verb      = lower *( lower / digit / "-" )
```

Only operations listed in the registry (§12) are valid in
`manifest_version: 1`. Unknown subsystems, unknown verbs, and reserved
subsystems (§12.13) are rejected.

### 11.2 Operation entries

Each item of `supported_ops` is a mapping:

| Key | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `name` | string | yes | — | Operation name (§11.1). |
| `backend` | string | yes | — | One of `custom`, `exec`, `delegate`, `unsupported`. `native` is reserved for `serviceHandler`. |
| `options` | mapping | depends on backend | `{}` where allowed | Backend options (§13). Unknown keys are rejected. |
| `functions` | list of single-key mappings | `custom` only | — | Handler binding (§13.2). Rejected for any other backend. |
| `description` | string | no | `""` | Human note shown by `jumpctl cores show`. |

Every operation name MUST appear at most once per manifest. The order of
entries has no semantic meaning. `jumpctl cores show` prints entries in
registry order.

### 11.3 Completeness rule

For each subsystem listed in `subsystems`, the manifest MUST contain an entry
for **every** verb that the registry defines for that subsystem. A verb the
core does not implement MUST be listed with `backend: "unsupported"`.

A manifest MUST NOT contain entries for subsystems it does not list in
`subsystems`.

**Unit-addressed subsystems** are `service`, `mount`, `timer`, `socket`,
`target`, `swap`, and `path`. A core that claims any of them MUST also claim
`unit`, and none of the following may be `unsupported`:

- `unit.list` and `unit.resolve`, which Jumpstart needs to route requests
  (§14.3) and to find a core's units; and
- the `status` verb of each claimed unit-addressed subsystem, which Jumpstart
  needs to find active units before unloading (§20.4).

This keeps every core's capability surface complete and explicit. When a
future manifest version adds a verb, existing manifests stay valid because
they are validated against their own `manifest_version`'s registry (§23.3).

### 11.4 Unsupported operations

An operation is unsupported for a core when its subsystem is not claimed, when
its entry uses `backend: "unsupported"`, or when the registry does not define
that verb for the unit's subsystem (for example `restart` on a mount). In each
case the request fails with `E_UNSUPPORTED`. Jumpstart MUST NOT retry it on
any other core, and MUST NOT substitute a different operation.

### 11.5 Activation strategies

Service managers bring units up at boot in very different ways: systemd
reaches `default.target`, OpenRC runs the `sysinit`, `boot`, and `default`
runlevels, dinit starts its `boot` service, and SysV init runs the `rcN.d`
scripts in order. Jumpstart cannot know any of these orders. A core that
claims unit-addressed subsystems therefore declares how its units are
activated:

- **`activation: "core"`.** The core activates and deactivates its own units.
  `unit.activate` and `unit.deactivate` MUST NOT be `unsupported`. After the
  core loads, Jumpstart calls `unit.activate` once. Before unloading with units
  stopped, it calls `unit.deactivate` once (§20.4, §20.6). The core performs
  every step in its own order.
- **`activation: "per-unit"`.** Jumpstart activates the core's units itself.
  `unit.activate` and `unit.deactivate` MUST be `unsupported`. After the core
  loads, Jumpstart starts every unit that `unit.list` reports with
  `enablement: "enabled"`, in list order. Before unloading with units stopped,
  it stops every unit that makes the core busy (§20.4 precondition 4) in
  reverse list order; like the busy check, that leaves `mount` and `swap` units
  that are plainly `active` alone. This suits simple cores whose units have no
  ordering between them.

`serviceHandler` uses `"core"` (Appendix A). Cores for managers with their own
dependency graph, targets, or runlevels SHOULD use `"core"`.

### 11.6 Common semantics

Operations fall into three classes:

- **Read-only:** `core.verify`, `unit.list`, `unit.resolve`, `unit.validate`,
  `unit.translate`, every `<sub>.status`, `target.get-default`, and
  `log.read`. They MUST NOT change anything.
- **Streaming:** `log.follow`. It is read-only and runs until cancelled.
- **Mutating:** every other operation.

Unless the registry says otherwise for a specific operation:

- **Idempotency.** Operations marked idempotent MUST succeed without side
  effects when the target is already in the requested state. `start` of an
  active unit and `stop` of an inactive unit are successes.
- **Serialization.** Jumpstart takes these locks, and a request that must wait
  waits until the lock is free or its own timeout expires, whichever is
  first:
  - **Unit lock.** Mutating operations on the same unit never run
    concurrently.
  - **Core lock.** `unit.load` and `target.isolate` change many units at once,
    so each takes the core lock. Nothing else that mutates the same core runs
    alongside them.
  - **Log lock.** `log.write`, `log.rotate`, and `log.vacuum` of one core are
    serialized with each other only.
  - **No lock.** `unit.activate` and `unit.deactivate` take no lock. They can
    run for minutes, and an administrator must still be able to start or stop
    a single unit meanwhile. The core's own job engine MUST handle unit
    requests that arrive while activation is in progress. Read-only and
    streaming operations never take or wait for any lock.
- **Timeout.** The `timeout` option bounds the whole operation, including all
  adapter round trips. On expiry Jumpstart cancels the operation (§13.7) and
  returns `E_TIMEOUT`.
- **Atomicity.** Cores SHOULD leave a unit in either its previous or its
  requested state. If they cannot, they MUST report `E_PARTIAL` with details.
- **Readiness.** `start`, `restart`, and `isolate` MUST NOT report `active`
  before the unit is ready by its manager's own definition (for example
  `sd_notify` `READY=1`, the s6 readiness file descriptor, or dinit
  ready-notification). Until then, the unit's state is `activating`. A core
  MAY return from `start` as soon as the unit is `activating`. The result
  `state` tells the caller which happened.
- **Reloading.** A unit that is applying a reload (`service.reload`, or one the
  unit started itself) MAY report `reloading` (§12.11) while it does. `reload`
  MAY return as soon as the unit is `reloading`, as `start` MAY return at
  `activating`, and the result `state` tells the caller which happened.
  `reloading` is in progress, never a failure: no Jumpstart operation treats it
  as `failed` or `inactive`.

## 12. Operation registry

This section is the complete operation vocabulary for `manifest_version: 1`.

Column legend:

- **Idem.**: idempotent (§11.6). `—` means not applicable.
- **Arguments**: keys of the request `args` object (§16, §17). All are strings
  unless stated.
- **Result**: keys of the response `result` object. `—` means empty.
- **Routing**: `unit` operations need a resolved unit (§14.3). `core`
  operations target the core itself. `global` operations target the whole core.

Every `enable` and `disable` verb in this section takes the same arguments:

- `now` (bool): also `start` (for `enable`) or `stop` (for `disable`) the unit.
- `target` (string, optional): a unit name owned by the same core whose
  subsystem is `target`. The unit is enabled into, or disabled from, that
  target or runlevel only. Without `target`, the core applies its own default
  enablement, as its manager defines it (systemd's `[Install]` section,
  OpenRC's `default` runlevel, `serviceHandler`'s `default` target, §22).

Every `status` verb in this section also accepts the optional result fields
`since`, `substate`, and `result` (§12.11) in addition to the fields in its
row.

### 12.1 `core`: managed payload

This acts on the core's managed payload, not on the extension directory. The
`core` operations are local only: the daemon never routes a request to them
and no control-socket command maps to them (§14.4). `jumpctl cores run`
(§21.15) is the only way to invoke them. It runs them without the daemon,
like every other operation it accepts.

There are deliberately no `build`, `install`, `upgrade`, or `remove` verbs.
Building the payload and getting it onto the system is package management,
which kpkg does and which is outside the scope of a service manager (§4.2).

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `verify` | yes | core | — | `installed` (bool), `version` | Report payload presence and version. |

### 12.2 `unit`: unit definitions and activation

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `list` | yes | global | `subsystem` (optional) | `units` (list of `{name, subsystem, enablement}`) | List the units this core currently knows, with their enablement state (§12.12). Template units and other names that exist only on demand MAY be omitted. |
| `resolve` | yes | global | `name` | `owned` (bool), `name`, `subsystem` | Say whether this core owns the unit name, and if so its canonical name and subsystem (§14.3). |
| `load` | yes | global | — | `loaded` (int), `errors` (list) | Reread the core's unit definitions, like a daemon reload. MUST NOT start or stop units. |
| `validate` | yes | unit | — | `valid` (bool), `diagnostics` (list) | Statically check one unit definition. |
| `translate` | yes | unit | — | `decfile` (string) | Render an equivalent native `.kg` decfile (`jumpstart_dec(5)`), or fail with `E_UNSUPPORTED` if there is no faithful equivalent. MUST NOT write files. |
| `activate` | no | global | — | `started` (int), `failed` (list) | Bring up the core's default set of units, in the core's own order (§11.5). |
| `deactivate` | no | global | — | `stopped` (int), `failed` (list) | Stop every active unit of this core, in the core's own order. On success no unit of the core is `active`, `activating`, `reloading`, or `deactivating`, except that a core MAY leave units of the subsystems `mount` and `swap` `active` (below). |

A mount or swap unit represents kernel state that outlives the core: stopping
it on `cores unload --stop-units` would unmount `/usr` or deactivate swap
under a running system, and Jumpstart's own shutdown does that in its fixed
order (§20.8 step 5). After a successful `unit.deactivate`, a core therefore
MAY leave units whose subsystem is `mount` or `swap` in the state `active`. It
MUST NOT leave them `activating`, `reloading`, or `deactivating`, and it MUST
leave no unit of any other subsystem `active`, `activating`, `reloading`, or
`deactivating`. Jumpstart's unload busy check ignores such active units
(§20.4), so a core whose only active units are mounts and swap is not busy.
This changes only what `unit.deactivate` has to achieve: `mount.stop` and
`swap.stop` on a single unit are unchanged. `serviceHandler` does not use the
exemption (§22.5).

`unit.resolve` MUST be read-only, and its answer MUST depend only on the
core's currently loaded unit definitions, not on which units are running.
`name` in the result is the core's canonical form of the requested name, such
as an alias resolved to its real unit.

### 12.3 `service`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Start the service. |
| `stop` | yes | unit | — | `state` | Stop the service and every process belonging to it. |
| `restart` | no | unit | — | `state` | Stop, then start. Starts the service if it is inactive. |
| `reload` | no | unit | — | `state` | Ask the running service to reload its configuration. `E_UNIT_STATE` if inactive. |
| `kill` | no | unit | `signal` (name such as `SIGHUP`/`HUP`, or decimal number) | — | Send a signal to the service's processes without changing its state or enablement. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the service. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the service. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `main_pid` (int, optional), `pids` (list of int, optional), `exit_code` (int, optional) | Report state. |

### 12.4 `mount`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Mount. |
| `stop` | yes | unit | `lazy` (bool) | `state` | Unmount. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the mount. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the mount. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `source`, `target`, `fstype` | Report state. |

### 12.5 `timer`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Arm the timer. |
| `stop` | yes | unit | — | `state` | Disarm the timer. Does not stop a unit it already triggered. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the timer. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the timer. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `next_run` (RFC 3339, optional), `last_run` (optional) | Report state. |

### 12.6 `socket`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Begin listening. |
| `stop` | yes | unit | — | `state` | Stop listening. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the socket. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the socket. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `listen` (list) | Report state. |

Passing the listening socket to the service it activates is the core's job.
Jumpstart is not involved.

### 12.7 `target`

Targets group units. Runlevels are targets: an OpenRC runlevel, a SysV
runlevel, a systemd `.target`, and a dinit boot service are all modelled as
target units of their core, and switching runlevel is `target.isolate`.

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Reach the target, starting its members. |
| `stop` | yes | unit | — | `state` | Stop the target's members. |
| `isolate` | no | unit | — | `state` | Reach the target and stop every unit this core owns that the target does not require. MUST NOT affect units owned by other cores. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the target into another target. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable it. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `members` (list) | Report state. |
| `set-default` | yes | unit | — | — | Make this target the one `unit.activate` reaches. |
| `get-default` | yes | global | — | `name` | Report the target `unit.activate` reaches. |

### 12.8 `swap`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Activate the swap device or file. |
| `stop` | yes | unit | — | `state` | Deactivate it. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the swap unit. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the swap unit. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `device`, `priority` (int, optional) | Report state. |

### 12.9 `path`

A path unit watches filesystem paths and starts another unit, owned by the
same core, when they change.

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `start` | yes | unit | — | `state` | Begin watching. |
| `stop` | yes | unit | — | `state` | Stop watching. Does not stop a unit it already triggered. |
| `enable` | yes | unit | `now`, `target` (§12) | — | Enable the path unit. |
| `disable` | yes | unit | `now`, `target` (§12) | — | Disable the path unit. |
| `status` | yes | unit | — | `state`, `detail`, `enablement`, `watches` (list), `triggers` | Report state. |

### 12.10 `log`

| Verb | Idem. | Routing | Arguments | Result | Semantics |
| --- | --- | --- | --- | --- | --- |
| `read` | yes | unit or global | `since`, `until` (RFC 3339, optional), `lines` (int, optional, 1–10000, default 1000) | `entries` (list) | Return at most `lines` entries, the most recent ones in the window. |
| `follow` | — | unit or global | `since` (optional) | streamed `entry` events (§17.4) | Stream entries until cancelled. |
| `write` | no | unit or global | `level`, `message` | — | Append an entry. |
| `rotate` | no | global | — | — | Rotate storage. |
| `vacuum` | no | global | `max_size` (bytes, int, optional), `max_age` (duration, optional) | `freed` (bytes, int) | Delete old entries. At least one argument is required. |

Log entries have the shape `{time, level, unit, message}`. `level` is one of
`debug`, `info`, `notice`, `warn`, `error`, or `fatal`.

### 12.11 Unit states

`state` values are drawn from a closed set:

| State | Meaning |
| --- | --- |
| `active` | Running and ready, mounted, armed, listening, or reached. |
| `inactive` | Not active, no error recorded. |
| `activating` | Transitioning to active, or started but not yet ready (§11.6). |
| `deactivating` | Transitioning to inactive. |
| `reloading` | Active and applying a configuration reload; the unit keeps serving (§11.6). |
| `failed` | Last activation or exit failed. |
| `unknown` | The core cannot determine the state. `detail` MUST explain why. |

`detail` is free-form human text, ≤ 512 bytes.

A core whose manager has no distinct reloading state keeps reporting `active`;
that conforms. Where a core does report `reloading`:

- **Busy check (§20.4).** A `reloading` unit makes its core busy, whatever its
  subsystem. In particular a `mount` or `swap` unit that is `reloading` is not
  covered by the exemption of §12.2, which applies to a plainly `active` one.
- **Deactivation.** `unit.deactivate` treats a `reloading` unit like an active
  one: it stops it, or reports it in `failed`.
- **Activation (§11.6, §20.6).** A `<sub>.start`, `service.reload`, or
  `service.restart` whose `state` is `reloading` is a success. Per-unit
  activation does not wait for the unit to leave `reloading` and counts it as
  started.
- **Notifications (§17.5).** `reloading` is an allowed `state`.

**Optional status fields.** Every `<subsystem>.status` result (§12.3 to
§12.9) MAY carry the following fields in addition to the fields of its row.
Each is optional and independent of the others:

| Field | Type | Meaning |
| --- | --- | --- |
| `since` | string, RFC 3339 timestamp | When the unit entered its current state. |
| `substate` | string, 1 to 64 bytes of lowercase ASCII letters, digits, and `-` | The manager's own finer state, such as `running`, `exited`, `start-pre`, or `auto-restart`. |
| `result` | string, same restrictions as `substate` | How the last run ended in the manager's vocabulary, such as `success`, `exit-code`, or `timeout`. |

Jumpstart validates them like every other result field (§16.3) and never
interprets them: they play no part in routing, locking, activation, or the
busy check, and `jumpctl status` prints them (§21.2). `serviceHandler` never
sets them. `unit.list`, `jumpctl units`, and `jumpctl cores list` are
unaffected by this subsection.

### 12.12 Enablement states

`enablement` values are drawn from a closed set, so that every manager's notion
of "enabled" maps onto it:

| Value | Meaning |
| --- | --- |
| `enabled` | Explicitly enabled, so `unit.activate` (or per-unit activation) will start it. |
| `disabled` | Can be enabled but is not. |
| `static` | Cannot be enabled by itself; it starts only when something else pulls it in, such as a target's fixed member list. |
| `indirect` | Not enabled itself, but pulled in by another enabled unit or alias. |
| `masked` | Prevented from starting at all. |
| `generated` | Created dynamically by the core (for example from `/etc/fstab` or `/etc/inittab`); its enablement follows the source it was generated from. |

### 12.13 Reserved subsystems

Service managers such as systemd cover more ground than the subsystems above.
The names below are set aside for areas that a later manifest version may
define. A core cannot claim them in `manifest_version: 1` (`E_MANIFEST`). This
stops each core from inventing its own incompatible `network.*` verbs before
Jumpstart defines them once for every core.

Reserving a name is not a promise to add it. A core can still run the daemons
behind these areas, such as `systemd-networkd`, as ordinary `service` units.

`slice`, `scope`, and `device` in particular remain reserved for operation
sets that a later manifest version may define. A core MAY model a slice (a
resource-control group) as a `target` unit that groups its member units
without conflict: such a unit belongs to the `target` subsystem and claims
nothing reserved. Scopes and devices are simply not addressable through
Jumpstart: no operation exists for them, a core cannot claim those names, and
it does not report units of those subsystems.

| Name | Area |
| --- | --- |
| `boot` | Boot loader entries and boot-time checks |
| `device` | Device units and hotplug (udev) |
| `home` | Managed home directories |
| `hostname` | Hostname and machine metadata |
| `login` | Seats, sessions, and user logins (logind) |
| `network` | Network configuration (networkd) |
| `power` | Suspend and hibernate. Poweroff, reboot, and halt are native (§20.8). |
| `resolve` | DNS resolution (resolved) |
| `scope` | Groups of externally started processes |
| `slice` | Resource-control groups. Each core already has its own cgroup subtree (§15.1). |
| `sysusers` | Declarative system users and groups |
| `time` | Clock and time synchronization |
| `tmpfiles` | Creation and cleanup of temporary files and directories |

## 13. Backends

### 13.1 Common options

These keys are valid in `options` for the `custom` and `exec` backends, except
where a table below says otherwise.

| Key | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `timeout` | duration string | yes | — | Upper bound on the operation (§11.6). |
| `sandbox` | bool | see below | — | Run inside the sandbox (§13.8). |
| `user` | string | no | the manifest's `account` | System mode: account to run as. MUST be `"root"` or the manifest's `account`, and MUST exist at dispatch time. Ignored in user mode (§7.4). |
| `writable_paths` | list of strings | no | `[]` | Extra absolute paths writable inside the sandbox. Each MUST exist at dispatch time. Valid only with `sandbox: true`. |
| `network` | bool | no | `false` | Give the sandboxed operation the host's network instead of loopback only. Valid only with `sandbox: true`. |
| `environment` | mapping of string to string | no | `{}` | Extra environment variables (§7.5). Keys MUST match `[A-Z_][A-Z0-9_]*` and MUST NOT start with `JUMPSTART_`. |
| `working_directory` | string | no | `"/"` | Absolute path. Under `sandbox: true` it MUST be within a writable path. |

**Where `sandbox` applies.** The sandbox exists to contain code that only
needs to look at things. It is therefore available only for read-only and
streaming operations (§11.6):

- For read-only and streaming operations with the `custom` backend, or with
  `exec` in `oneshot` mode, `sandbox` is REQUIRED.
- For every other operation, and for every `exec` entry in `persistent` mode,
  `sandbox` MUST be absent (`E_MANIFEST`). Those operations always run
  unsandboxed, in the core's control cgroup (§15.1), as the effective user.

Mutating operations cannot be sandboxed, because the processes they start
(services, mounts, swap) must live in the real system and outlive the
operation. A persistent adapter therefore also serves its read-only operations
(`log.read`, for example) unsandboxed, and SHOULD treat any file it reads for
them as untrusted input; a core that wants such an operation sandboxed
declares it `custom` or oneshot.

`writable_paths` and `network` loosen the sandbox, so setting them without
`sandbox: true` is rejected (`E_MANIFEST`) rather than ignored. In system
mode, `writable_paths` MUST NOT name a path under a home directory or under
another core's per-core directories.

Duration strings are a positive integer followed by `ms`, `s`, `m`, or `h`
(such as `90s`). The maximum is `24h`. `0` and unitless numbers are invalid.

### 13.2 `custom`: Kongue functions

The operation runs a function defined in the same `init.kg`, in a Kongue
runner process (§16.1).

`functions` is a list of single-key mappings that bind roles to function names:

```kongue
functions:
  - handler: "start_service"
```

Rules:

- In `manifest_version: 1`, every operation has exactly one role, `handler`,
  and it MUST be bound. Unknown roles are rejected. The list form leaves room
  for a later manifest version to define operations with several roles.
- Function references are `"name"` or `"name qualifier"`, using Kongue's
  qualified-name syntax. The function MUST exist and MUST NOT be `init` or
  `shutdown`.
- One function MAY be the handler of several operations. It can tell them
  apart with `op.name` (§16.2).
- Each operation runs in a fresh runner process, with a context initialized
  from the manifest header.

The handler succeeds if it runs to completion. It fails if a command it runs
exits non-zero, if it calls `fail` (§16.3), or if it raises a Kongue runtime
error.

`custom` accepts only the common options (§13.1).

### 13.3 `exec`: adapter executable

The operation is sent to an adapter process using the protocol in §17.

| Key | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `executable` | string | yes | — | Path relative to the core directory, under `bin/`. |
| `arguments` | list of strings | no | `[]` | Extra argv entries, passed literally and never shell-expanded. |
| `mode` | string | no | `"oneshot"` | `oneshot`: one process per request. `persistent`: one long-lived process per core, started at load and reused (§17.6). |
| common | | | | As in §13.1. |

All `exec` entries that reference the same executable with `mode:
"persistent"` share one process. Their `user`, `environment`,
`working_directory`, and `arguments` MUST be identical, or validation fails.

`core.*` operations run without a daemon (§21.15), so they MUST NOT use
`mode: "persistent"`. Other operations may use it. `jumpctl cores run` starts
a persistent adapter itself for a single request and closes it afterwards
(§21.15), so it never uses the daemon's shared adapter.

### 13.4 `delegate`: another core

The operation is forwarded to another core.

| Key | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `core` | string | yes | — | Target core name. |
| `operation` | string | no | the entry's `name` | Operation to invoke on the target. MUST be in the same subsystem. |

Common options are rejected; the target's options apply.

Rules:

- When the delegating core is enabled, the target MUST be discovered, valid,
  and enabled. When it is loaded, the target MUST be loaded in the same daemon
  (§20.3). In both cases the target MUST support the operation with a backend
  other than `unsupported`.
- Delegation chains MUST be acyclic. A chain longer than four hops is
  rejected with `E_DELEGATION`.
- Delegation forwards the request unchanged, including the unit name. The
  target MUST be able to address that unit name. Typically the operation is
  `global`, or the target resolves the same name.
- Disabling a core that is a delegation target of an enabled core, or
  unloading one that is a target of a loaded core, fails with `E_DELEGATION`.
  The error names the dependent core.

### 13.5 `unsupported`

The operation is explicitly not implemented. `options` and `functions` MUST be
absent. Requests fail with `E_UNSUPPORTED`.

### 13.6 `native` (serviceHandler only)

`native` binds an operation directly to compiled code. It accepts no options.
It is valid only in the virtual manifest of `serviceHandler` (Appendix A).

### 13.7 Cancellation

Every Kongue runner and every `oneshot` adapter runs in its own process group
(§15.4). On timeout or client cancellation, Jumpstart:

1. sends `SIGTERM` to that process group (runners and oneshot adapters), or a
   `cancel` message (persistent adapters, §17.3);
2. waits up to 5 seconds; then
3. sends `SIGKILL` to that process group. For a persistent adapter, the
   adapter process is killed and restarted lazily on the next request.

Jumpstart does not signal the process group of an operation that completes
normally. A process that must outlive a cancelled operation, such as a daemon
started by an init script, MUST leave the operation's process group (for
example with `setsid`), or be started by the core's supervisor (§15.2).

### 13.8 Sandbox

With `sandbox: true`, the operation runs:

- in new mount, PID, IPC, UTS, and network namespaces, with only a loopback
  interface unless `network: true`;
- with the root filesystem read-only except the core's state and runtime
  directories (§19.1), a private `/tmp`, and each entry of `writable_paths`
  (a state directory or `writable_paths` entry that does not exist yet is
  skipped, not created);
- with every capability removed from the bounding, permitted, effective,
  inheritable, and ambient sets;
- with `PR_SET_NO_NEW_PRIVS` set; and
- under a seccomp filter that denies at least `mount`, `umount2`,
  `pivot_root`, `swapon`, `swapoff`, `reboot`, `kexec_load`,
  `kexec_file_load`, `init_module`, `finit_module`, and `delete_module`.

If the platform cannot provide every one of these properties, the operation
MUST fail with `E_SANDBOX`. It MUST NOT run unsandboxed.

## 14. Unit naming and request routing

### 14.1 Unit names

Unit names are defined by the core that owns them. Jumpstart treats a unit
name as an opaque string. It never parses it, strips suffixes from it, or
derives it from a filename. How a core stores its units (one file per unit,
one directory per unit, many units in one file, a compiled database, or units
that exist only on demand, such as systemd template instances) is entirely
the core's concern.

Jumpstart imposes only these constraints, so that names are safe to route and
log:

- 1 to 255 bytes of valid UTF-8; and
- no `/`, no ASCII control characters, and no whitespace.

A name MAY begin with `-`: systemd's root mount and root slice are `-.mount`
and `-.slice`. On a command line such a name cannot be told from an option, so
`jumpctl` accepts it only after `--` (§21.1). Everywhere else (the control
protocol, the exec protocol, routing, name indexes, and logs) it is an
ordinary name. A program that a handler starts with a unit name as an argument
has to protect itself against a leading `-` (§16.4).

Everything else, such as case sensitivity, suffixes like `.service`,
templates like `getty@tty1.service`, and aliases, is defined by the core and
exposed through `unit.resolve` and `unit.list` (§12.2).

`serviceHandler`'s naming rules are its own. A native unit name is a decfile
name, optionally with a `::` sub-unit, and `serviceHandler` also accepts the
legacy suffixes `.kg`, `.service`, `.mount`, and `.timer`, as Jumpstart does
today. This normalization now happens inside `serviceHandler`'s
`unit.resolve`. `jumpctl` MUST stop normalizing names on the client side,
because doing so would corrupt extension unit names.

### 14.2 Unit references

```text
unit-ref = [ core-name "/" ] unit-name
```

A qualified reference such as `systemd/sshd.service` names the owner
explicitly. Core names and unit names cannot contain `/`, so the first `/`
always separates the core.

### 14.3 Resolution

Cores in the loaded set (§20.1) take part. Cores in the effective enabled set
(§9) that are not loaded, because they are `failed` or `unloaded`, also take
part, but cannot answer (below). Other cores do not take part.

**Name index.** For each core, Jumpstart keeps an index of the unit names the
core is known to own: every name from its most recent `unit.list` result,
plus every canonical name and requested name for which its `unit.resolve`
answered `owned: true`. Jumpstart seeds the index with one `unit.list` call
when the core is published (§20.3) and refreshes it after each `unit.list`
and `unit.load`. Each update is also written atomically to
`/run/jumpstart/name-index/CORE` (user mode:
`RUNTIME/jumpstart/name-index/CORE`, §7.2), one name per line. The file is
kept when the core unloads or fails, so the index outlives the core until
the next boot or session. Jumpstart ignores the files of cores that are not
enabled. The index is never used to choose an owner. It is used only to
decide whether a core that cannot answer might own a name.

1. **Qualified** (`CORE/UNIT`): if `CORE` is `loading`, wait for its load to
   finish, within the request's timeout. If `CORE` is then not loaded, return
   the error for its state (§20.1). Otherwise Jumpstart calls `CORE`'s
   `unit.resolve` with `UNIT`. If `owned` is false, the request fails with
   `E_NOT_FOUND`.
2. **Unqualified** (`UNIT`): first wait until no core is `loading`, within the
   request's timeout (`E_TIMEOUT` otherwise). This keeps answers during boot
   the same as answers after boot. Then Jumpstart calls `unit.resolve` on every
   loaded core that claims `unit`, including `serviceHandler`. These calls are
   read-only and MAY run concurrently.

   A core **cannot answer** if it is enabled but `failed` or `unloaded`, if
   it is `degraded`, or if its `unit.resolve` fails or times out. For each
   such core:
   - if its name index contains `UNIT`, the request fails with that core's
     error (`E_CORE_FAILED`, `E_CORE_UNLOADED`, `E_DEGRADED`, or the
     `unit.resolve` error), naming the core and asking for a qualified
     reference;
   - if it has no index at all (for example, it failed during boot before
     publishing), and `serviceHandler`'s answer of `owned: true` relied on
     stripping a legacy suffix (§14.1), the request fails with that core's
     error, because the suffixed name more likely belongs to it;
   - otherwise it is excluded, and the response carries a warning naming it.

   Because `serviceHandler`'s `unit.resolve` is native, it tells the router
   whether a match relied on a legacy suffix directly. This is internal to
   Jumpstart and not a registry field (Appendix A).

   A request from a process in a `loading` core's `control/` cgroup (§15.1),
   such as `jumpctl` called from that core's `init`, does not wait for that
   core. For that request the core counts as not loaded, so a qualified
   reference to it returns `E_CORE_STATE` at once. This prevents a core's
   load from waiting on itself.

   Among the cores that answered:
   - exactly one answers `owned: true`: it is the owner;
   - none does: `E_NOT_FOUND`;
   - several do: `E_AMBIGUOUS`, and the error lists the qualified
     alternatives (`serviceHandler/sshd`, `systemd/sshd.service`).

A core that cannot answer therefore never causes a request to be routed to a
different core for a name it was known to own. A slow or broken core only
affects names it is known to own.

In both cases the operation is dispatched with the canonical `name` and the
`subsystem` from the owner's `unit.resolve` result.

Unqualified names that only `serviceHandler` claims behave exactly as they do
today. A name claimed both natively and by an extension becomes ambiguous and
must be qualified.

### 14.4 Request-to-operation mapping

Unit commands (§21.2) map to `SUBSYSTEM.VERB`, where `SUBSYSTEM` is the
subsystem from the owner's `unit.resolve` result:

| `jumpctl` | Operation | Arguments |
| --- | --- | --- |
| `start UNIT` | `<sub>.start` | — |
| `stop UNIT` | `<sub>.stop` | `lazy: false` for mounts |
| `restart UNIT` | `service.restart` | — |
| `reload UNIT` | `service.reload` | — |
| `kill UNIT [--signal S]` | `service.kill` | `signal` (default `SIGTERM`) |
| `enable UNIT [--now] [--target T]` | `<sub>.enable` | `now`, `target` |
| `disable UNIT [--now] [--target T]` | `<sub>.disable` | `now`, `target` |
| `status UNIT` | `<sub>.status` | — |
| `isolate UNIT` | `target.isolate` | — |
| `set-default UNIT` | `target.set-default` | — |
| `get-default [--core CORE]` | `target.get-default` (global) | — |
| `units validate UNIT` | `unit.validate` | — |
| `units translate UNIT` | `unit.translate` | — |
| `logs read/follow/write [UNIT]` | `log.read`/`log.follow`/`log.write` | from options |

A verb that the registry does not define for the unit's subsystem fails with
`E_UNSUPPORTED` (§11.4). Jumpstart does not substitute another operation.

The `target` argument of `enable` and `disable` is passed to the owner as
given. It names a unit of the same core, so `--target` takes a bare unit name,
not a unit reference.

### 14.5 Control socket protocol

`jumpctl` and `pam_jumpstart.so` talk to the daemon over the mode's socket
(§7.2) using newline-delimited JSON. One connection carries one request.

**Requests.** Every request has a `client` object (`name`, `version`) and
exactly one of the following:

```json
{"client":{"name":"Jumpstart CLI","version":"1.3.0"},
 "service":{"name":"systemd/sshd.service","action":"restart","now":"false","args":{}}}
```

`service` is a unit request. `name` carries the unit reference, `action` the
verb (`start`, `stop`, `restart`, `reload`, `kill`, `enable`, `disable`,
`status`, `isolate`, `set-default`, `validate`, `translate`, `log-read`,
`log-follow`, `log-write`), and `args` the operation arguments. Clients
written for the four actions the daemon handles today (`start`, `stop`,
`enable`, `disable`), without `args`, keep working.

```json
{"client":{…},"op":{"core":"systemd","name":"log.rotate","args":{}}}
```

`op` is a global operation on one core (used by `units`, `logs`, and
`get-default` without a unit).

```json
{"client":{…},"core":{"name":"systemd","action":"reload","stop_units":false}}
```

`core` is a lifecycle request. `action` is `load`, `unload`, `reload`,
`status`, or `list-loaded`.

```json
{"client":{…},"session":{"action":"open","uid":1000,"id":"c2"}}
{"client":{…},"power":{"action":"poweroff"}}
{"client":{…},"users":{"action":"list"}}
{"client":{…},"env":{"action":"set","vars":{"DISPLAY":":0"}}}
```

`session` (§7.6), `power` (§20.8), and `users` (§21.8) are accepted only by
the system daemon. `env` (§21.9) is accepted only by user daemons. Requests
sent to the wrong mode fail with `E_USAGE`.

**Responses.** The daemon MUST answer every request with exactly one response
line:

```json
{"type":"response","status":"ok","result":{"state":"active"},"warnings":[]}
{"type":"response","status":"error","error":{"code":"E_UNSUPPORTED","message":"…"}}
```

Streaming operations send `event` lines (§17.4) before the response. List
fields of a result MAY be delivered as `item` events (§17.4) to stay within
the line limit. If the client closes the connection before the response, the
request is cancelled (§13.7). Clients written before this specification
ignore the response, which keeps them compatible.

**Robustness.** The daemon MUST:

- reject request lines longer than 1 MiB;
- close connections that send no complete request line within 10 seconds;
- answer malformed JSON, unknown fields, and wrongly typed values with
  `E_PROTOCOL` and close the connection, and never let a parse error stop the
  daemon;
- serve connections concurrently, up to the limits in §19.3; and
- keep signal handling and reaping (§15.3) responsive while requests are in
  progress. As PID 1, Jumpstart must never stall on a client.

## 15. Process ownership

Every process Jumpstart starts for a core, and every process those processes
start, has a defined owner and a defined end. This is what lets supervision
trees (runit, s6, dinit), daemonizing init scripts (SysV, OpenRC), and
socket-activating managers (systemd) all run under Jumpstart without leaking
or stranding processes.

### 15.1 Cgroups

Jumpstart requires cgroup v2. In system mode, if no cgroup2 filesystem is
mounted at `/sys/fs/cgroup`, Jumpstart mounts one there during early boot,
alongside the other filesystems it mounts (§20.2). If the kernel does not
support cgroup v2, extension cores fail to load with `E_CGROUP`, and
`serviceHandler` runs without cgroups, as it does today.

Each loaded core has its own subtree:

```text
/sys/fs/cgroup/jumpstart/
  cores/CORE/
    control/        Kongue runners, adapters, and the supervisor (Jumpstart-managed)
    units/          the core's unit processes (core-managed)
  users/UID/        delegated to that user's daemon (§7.6)
    daemon/         the user daemon itself
    cores/CORE/     same layout as above, managed by the user daemon
```

cgroup v2 allows processes only in leaf cgroups of a subtree that distributes
controllers to its children. Every process therefore lives in a leaf:
`control/`, a cgroup the core creates below `units/`, or `users/UID/daemon/`.
`jumpstart/`, `jumpstart/cores/`, `jumpstart/users/`, `cores/CORE/`,
`users/UID/`, and `users/UID/cores/` never contain processes. PID 1 itself
stays in the root cgroup.

Rules:

1. Jumpstart creates `cores/CORE/`, `control/`, and `units/` when the core
   loads (§20.3). In `jumpstart/`, `jumpstart/cores/`, `jumpstart/users/`,
   `cores/CORE/`, and `units/`, it enables in `cgroup.subtree_control` every
   controller available from the parent.
2. Jumpstart starts every process it runs for the core in `control/`.
3. The core MUST place each unit's processes in a cgroup it creates under
   `units/`, and MUST NOT move any process outside `cores/CORE/`. Jumpstart
   cannot prevent a root core from writing elsewhere in the hierarchy. This is
   a requirement on every core.
4. When `account` is not root, Jumpstart delegates `cores/CORE/` to the
   account. It changes the owner of exactly that directory and its
   `cgroup.procs`, `cgroup.threads`, and `cgroup.subtree_control` files, and
   then of `units/` and the same three files in it. `control/` and its files
   stay owned by root. Moving a process requires write access to
   `cgroup.procs` of the common ancestor (`cores/CORE/`), so the account's
   supervisor and adapters can then move processes from `control/` into
   cgroups below `units/`, but cannot move anything into `control/` or out of
   the core's subtree. The account cannot change its own limits (such as
   `memory.max` of `cores/CORE/`), because those files stay owned by root.
5. When a core unloads (§20.4), Jumpstart writes `1` to `cgroup.kill` of
   `cores/CORE/`, waits up to 5 seconds for `cgroup.events` to report
   `populated 0`, and removes the subtree.
6. `serviceHandler`'s native units run in
   `cores/serviceHandler/units/UNIT.unit/`. Native `stop` kills the unit's
   cgroup after the stop sequence, which is how it reaches every process of
   the service.
7. Every cgroup directory a core creates below `units/` MUST end with the
   suffix `.unit`, which no cgroup interface file uses. A directory name is
   valid when it is 6 to 255 bytes long including `.unit` (at least one byte
   before the suffix), contains no `/`, NUL, or ASCII control character, and
   is neither `.` nor `..`. Nothing else is restricted: backslash and `%` are
   valid, so a core can keep systemd's `\x2d` escapes or percent-encode them,
   as it prefers. No valid name collides with a cgroup interface file
   (`cgroup.procs`, `memory.max`, ...), because every valid name ends in
   `.unit`. Jumpstart rejects other names when a core asks it to create the
   directory, and cores escape unit names accordingly. The name before `.unit`
   is core-defined, but the mapping from unit names MUST be injective, so two
   units never share a directory. `serviceHandler`'s native directory is
   `UNIT.unit` with the unit name unchanged (rule 6), and a native unit whose
   name would not form a valid name, or that contains a backslash, fails at
   that point exactly as before.
8. Jumpstart starts each process directly in its cgroup
   (`clone3` with `CLONE_INTO_CGROUP`, or by joining the cgroup in the child
   before `execve`), so no process ever runs outside its cgroup.
9. A core MAY nest directories below `units/`, for example one directory per
   slice with each unit's directory inside its slice's. Every directory at
   every depth follows rule 7. A directory with child directories holds no
   processes (the leaf rule above), so a core makes only leaves hold
   processes and enables controllers in the `cgroup.subtree_control` of every
   directory that has children (`ensureNestedUnitCgroup` does this, §26.1).
   Jumpstart changes no ownership below `units/`: the account already owns
   `units/` and its delegation files (rule 4), which is what lets it create
   the nested directories. The unload kill (rule 5) and the busy check (§20.4)
   cover the whole subtree. A core removes a nested directory with `rmdir`
   only, bottom-up (`removeNestedUnitCgroup`), and finds the unit that owns a
   process from `/proc/PID/cgroup` by taking the chain of `.unit` components
   below `units/`, ending at the first component that does not end in `.unit`
   (the sub-cgroups of a delegated service belong to the service that owns
   the chain, whatever they are named) (`unitOfPid`).

The system daemon mounts cgroup2 with the `nsdelegate` option. If
`/sys/fs/cgroup` already holds a cgroup v1 or hybrid hierarchy (mounted by an
initramfs, for example), Jumpstart leaves it alone and treats cgroup v2 as
unavailable.

In user mode, the daemon uses the subtree delegated to it by the system daemon
(§7.6). If it has none, extension cores fail to load with `E_CGROUP`.

### 15.2 Supervisors

A core MAY declare a supervisor: a long-lived process that owns the core's
service processes. Managers that have their own supervising daemon (dinit,
`s6-svscan`, `runsvdir`, a systemd compatibility manager) use it, or a small
wrapper, as the supervisor. Their services are then children of the
supervisor, not of the adapter, so adapter restarts never affect them.

```kongue
supervisor:
  executable: "bin/dinit-wrapper"
  arguments:
    - "--system"
  ready_timeout: "30s"
```

| Key | Type | Req. | Default | Meaning |
| --- | --- | --- | --- | --- |
| `executable` | string | yes | — | Path relative to the core directory, under `bin/`. |
| `arguments` | list of strings | no | `[]` | Extra argv entries, passed literally. |
| `ready_timeout` | duration | yes | — | How long the supervisor has to complete its handshake. |
| `user` | string | no | the manifest's `account` | As in §13.1. |
| `environment` | mapping | no | `{}` | As in §13.1. |
| `working_directory` | string | no | `"/"` | As in §13.1. |

Lifecycle:

1. Jumpstart starts the supervisor during load (§20.3), after `init`, in the
   core's `control/` cgroup, with the descriptors in §15.4. The supervisor
   speaks the `supervisor` role of the exec protocol (§17) on descriptors 3 and
   4: the parent side of the handshake is Jumpstart's, and the core
   implements the child side (in Nim, on top of `protocol.nim`, §26.1).
2. Jumpstart makes the supervisor a child subreaper
   (`PR_SET_CHILD_SUBREAPER`) in the child process before `execve`. The
   attribute survives `execve`, so orphaned descendants reparent to the
   supervisor rather than to Jumpstart. The supervisor MUST NOT clear it.
3. The supervisor receives no operation requests. It MAY send notifications
   (§17.5). Adapters and runners coordinate with it however the core chooses,
   for example through a socket in the core's runtime directory.
4. If the supervisor exits while the core is loaded, the core becomes
   `degraded` (§20.7). Jumpstart does not restart it, because a restarted
   supervisor would not own the existing service processes. Reloading the
   core recovers it.
5. On unload, after `shutdown`, Jumpstart closes the supervisor's descriptor
   3, sends `SIGTERM`, waits up to 10 seconds, and sends `SIGKILL`. The cgroup
   kill (§15.1) then removes anything left.

A core without a supervisor is valid. Processes its operations leave running
(daemons started by init scripts, for example) are reparented to Jumpstart and
reaped by it (§15.3). They stay in the core's cgroup, so they still end when
the core unloads.

### 15.3 Reaping

The system daemon is PID 1. Every user daemon makes itself a child subreaper
at startup. In both cases, orphaned processes become Jumpstart's children, and
Jumpstart MUST reap all of them:

- One SIGCHLD-driven reaper calls `waitpid(-1, …, WNOHANG)` until no exited
  child remains.
- Every process Jumpstart starts is registered with the reaper before it can
  exit. Its exit status is delivered to its owner (the operation, the
  adapter's or supervisor's lifecycle, or a native service).
- Exited children that nobody registered are reaped and logged at `debug`.
- No other code calls `wait`, `waitpid`, or `waitForExit`. `serviceHandler`'s
  per-service `waitForExit` threads MUST move onto the reaper, so there is no
  race between them.

### 15.4 Descriptors and process attributes

Every process Jumpstart starts for a core gets:

- standard input from `/dev/null`;
- standard output and standard error each connected to a pipe that Jumpstart
  reads and logs (§19.2);
- for adapters, supervisors, and Kongue runners: descriptor 3 (Jumpstart to
  process) and descriptor 4 (process to Jumpstart) for the exec protocol
  (§17);
- no other open descriptors (the exceptions are the interpreter descriptor of
  scripts, which the spawn layer documents, and what a core's helper process
  asks for below);
- a new session and process group (`setsid`);
- membership in the core's `control/` cgroup (§15.1); and
- the environment in §7.5.

Processes MUST set `FD_CLOEXEC` on descriptors 3 and 4, or otherwise keep
their children from inheriting them. A daemon that inherited a protocol
descriptor would hold it open forever and corrupt the protocol. Because
standard output is only a log, a forked daemon that keeps it open can no
longer interfere with the protocol.

**Helper processes.** A core's own helper process, such as the per-unit
executor a systemd-style supervisor starts, is not a protocol child. The
core starts it with the library's spawn layer (§26.1), which has two options
beyond the defaults above:

- `explicitStdio`: three descriptors the core already holds become 0, 1, and 2
  (a socket, a file, a terminal, a journal stream). Jumpstart then creates no
  pipes for the process and logs nothing from it. The descriptors are the
  caller's and are not consumed.
- `passFds`: further descriptors of the core (socket-activation sockets, a
  context descriptor) appear in the child at fixed numbers, each with
  `FD_CLOEXEC` cleared.

Rules: descriptors 3 and 4 are the exec protocol descriptors and exist only
when the process is started as a protocol child (the spawn option `protocol`);
without it they are closed, and a helper MAY receive its own descriptors at 3
and 4 through `passFds`, which is how socket activation starts at 3. When
`protocol` is set, 3 and 4 cannot be targets. Descriptors 5 (the interpreter
descriptor) and 6 (the spawn layer's status descriptor) are reserved, targets
are at most 1023, and targets are distinct. Everything else is closed before
the program runs: only 0 to 2, the protocol descriptors when requested,
descriptor 5 for scripts, and the passed targets stay open. A `source` MAY be
equal to another entry's target or to one of the caller's 0 to 2, because the
spawn layer copies every source before it changes any descriptor. An invalid
descriptor or target is refused before anything is started.

## 16. Kongue runtime

### 16.1 Execution

Every Kongue execution for a core (`init`, `shutdown`, `custom` handlers, and
`jumpctl cores run`) runs in a separate runner process. Jumpstart starts
`/usr/lib/jumpstart/kongue-runner`, which it executes through a verified file
descriptor (§8.3), with the attributes in §15.4. It then sends the core
directory, the function to run, and the request over the exec protocol with
role `runner` (§17).

Kongue's `cd`, `env`, `local`, and `global` therefore affect only the runner,
never the daemon. `print` writes to the runner's standard output, which is
logged. The runner reports `respond`, `fail`, and `log` (§16.3) back to
Jumpstart over descriptor 4.

### 16.2 Objects and environment variables

`core` (read-only):

| Key | Value |
| --- | --- |
| `name` | Core name |
| `core_version` | From manifest |
| `version` | From manifest |
| `root` | Absolute core directory |
| `state_dir` | §19.1 |
| `runtime_dir` | §19.1 |
| `mode` | `system` or `user` |
| `jumpstart_version` | Running Jumpstart version, such as `1.3.0` |

`op` (read-only; absent in `init` and `shutdown`):

| Key | Value |
| --- | --- |
| `name` | Operation name, such as `service.start` |
| `role` | `handler` |
| `unit` | Canonical unit name, or `""` for core and global operations |
| `subsystem` | Subsystem |
| `request_id` | Unique request identifier |
| `deadline` | RFC 3339 deadline |
| `args` | Object of request arguments (§12). Booleans and integers are rendered as strings. |

Environment variables:

- Every process Jumpstart starts for a core gets `JUMPSTART_CORE_NAME`,
  `JUMPSTART_CORE_ROOT`, `JUMPSTART_CORE_STATE_DIR`,
  `JUMPSTART_CORE_RUNTIME_DIR`, and `JUMPSTART_MODE`.
- Kongue runners and `oneshot` adapters handling a request also get
  `JUMPSTART_OP`, `JUMPSTART_UNIT`, `JUMPSTART_REQUEST_ID`, and
  `JUMPSTART_ARG_<KEY>` (with the key upper-cased).
- Persistent adapters and supervisors serve many requests, so they receive
  request data only through the protocol.

### 16.3 Commands

The runtime adds these commands to Kongue's built-ins:

| Command | Usage | Meaning |
| --- | --- | --- |
| `respond` | `respond KEY VALUE` | Set a scalar result field, or append one value to a list-of-scalars field. |
| `respond-item` | `respond-item FIELD KEY=VALUE…` | Append one object to a list-of-objects field, such as `unit.list`'s `units`. |
| `spawn` | `spawn PROGRAM ARG…` | Run a program with exactly these arguments, without a shell. A non-zero exit fails the handler. |
| `fail` | `fail CODE MESSAGE` | Abort the operation with an error code from §18 (`E_BACKEND` if `CODE` is not a defined code). |
| `log` | `log LEVEL MESSAGE` | Write to the core's log (§19.2). |
| `filesystems-ready` | `filesystems-ready` | Send the `filesystems-ready` message (§17.5). Allowed in handlers and in `init`. |

`spawn` also has an expression form, `${spawn("PROGRAM", "ARG", …)}`, with
two methods:

- `${spawn(…).exit()}` runs the program and yields its exit status as a
  decimal string; and
- `${spawn(…).output()}` runs the program and yields its standard output, with
  one trailing newline removed. At most 1 MiB is captured; longer output fails
  the handler with `E_BACKEND`. Standard error is logged.

The expression form never fails the handler because of a non-zero exit, so
handlers can branch on status commands:

```kongue
func service_status {
  if "${spawn(\"/sbin/rc-service\", \"--\", \"${op.unit}\", \"status\").exit()}" == "0" {
    respond state "active"
  } else {
    respond state "inactive"
  }
  respond detail ""
  respond enablement "enabled"
}
```

Values are converted to the type the registry gives the field: `true` and
`false` for booleans, an optional `-` followed by decimal digits for integers,
and the text as given for strings. Jumpstart validates the finished result
against the registry. A missing required field, an unknown field, or a value
that cannot be converted fails the operation with `E_PROTOCOL`.

### 16.4 Shell safety

Kongue's `exec` runs its command through `/bin/sh -c`. Unit names and request
arguments come from clients and from unit files, so letting them reach a shell
would allow command injection. Checking command strings for request data is
not enough, because the data can be copied into a local variable, passed as a
function argument, or exported with `env` first.

The rule is therefore structural:

- **`custom` handlers MUST NOT use the shell.** Static validation (§10.6)
  rejects any use of `exec` or `${exec(…)}` in a handler function, or in any
  function reachable from a handler through calls. A function called both
  from `init`/`shutdown` and from a handler follows the handler rule.
- **`init` and `shutdown` MAY use `exec`.** They receive no request data
  (`op` is absent, §16.2). They SHOULD NOT pass data that handlers or the
  core's processes wrote (files in the state or runtime directory, for
  example) to `exec`, because that data may originate from a request. Use
  `spawn` for such values.

Handlers reach programs only through `spawn`, where each argument becomes
exactly one argv entry after interpolation. In both forms, the runtime first
parses the program and arguments into a list, using the source text only.
It then interpolates each element on its own, and the result is an opaque
value that is never split, re-parsed, or re-tokenized. A unit name such as
`a", "-x` therefore stays one argument, quotes and commas included:

```kongue
func start_service {
  spawn "/usr/bin/sv" "start" "--" "${op.unit}"
  respond state "active"
}
```

A unit name MAY begin with `-` (§14.1), so a program could take it for an
option; `spawn` keeps it one argument but cannot stop the program from
interpreting it. Handlers MUST therefore pass `--` before a unit name where the
program supports it.

## 17. Exec protocol

### 17.1 Transport

Adapters, supervisors, and Kongue runners talk to Jumpstart over descriptor 3
(Jumpstart to process) and descriptor 4 (process to Jumpstart). Messages are
newline-delimited JSON: one UTF-8 JSON object per line, at most 1 MiB per
line. Standard output and standard error are logged, at `info` and `warn`
respectively (§19.2), and never parsed.

The protocol identifier is `jumpstart-core/1`.

### 17.2 Handshake

On start, Jumpstart sends:

```json
{"type":"hello","protocols":["jumpstart-core/1"],"role":"adapter",
 "core":"systemd","mode":"system","jumpstart":"1.3.0"}
```

`role` is `adapter`, `supervisor`, or `runner`. The process MUST reply with
one protocol from the offered list:

```json
{"type":"hello","protocol":"jumpstart-core/1","role":"adapter",
 "operations":["service.start","service.stop"]}
```

For adapters, `operations` MUST equal the set of operations the manifest
routes to this executable, whether it runs as `oneshot` or `persistent`. A
mismatch fails with `E_PROTOCOL`, which catches manifests that have drifted
from their adapter. Supervisors and runners omit `operations`.

The reply MUST arrive within the operation's timeout (oneshot adapters and
runners), 10 seconds (persistent adapters), or `ready_timeout`
(supervisors, §15.2).

### 17.3 Requests and responses

Request (Jumpstart to adapter or runner):

```json
{"type":"request","id":"7f3c…","op":"service.start","unit":"sshd.service",
 "args":{},"deadline":"2026-10-16T12:00:30Z"}
```

A runner's request also carries `function`, the Kongue function to run.

Success:

```json
{"type":"response","id":"7f3c…","status":"ok","result":{"state":"active"}}
```

Failure:

```json
{"type":"response","id":"7f3c…","status":"error",
 "error":{"code":"E_BACKEND","message":"ExecStart= failed with status 203"}}
```

Cancellation (Jumpstart to adapter):

```json
{"type":"cancel","id":"7f3c…"}
```

A process MUST send exactly one `response` per `request` id. A cancelled
request MUST still receive a response, normally `E_CANCELLED`.

### 17.4 Events

Events belong to a request and precede its response.

- **`entry`**: one log entry of a streaming operation (`log.follow`):

  ```json
  {"type":"event","id":"7f3c…","event":"entry","data":{"time":"…","level":"info","unit":"sshd.service","message":"…"}}
  ```

- **`item`**: one element of a list field of the result, used when a result
  would not fit in one line. Jumpstart appends items to the named field in the
  order received, before any elements that the response itself contains:

  ```json
  {"type":"event","id":"7f3c…","event":"item","field":"units","data":{"name":"sshd.service","subsystem":"service","enablement":"enabled"}}
  ```

### 17.5 Notifications

Adapters and supervisors MAY report unit state changes that no request asked
for, such as a service that crashed or became ready:

```json
{"type":"notify","unit":"sshd.service","state":"failed","detail":"exited with status 1"}
```

Notifications have no `id`. `state` MUST be a §12.11 state. Jumpstart logs
each notification and records it as the unit's last-known state, which it uses
when the core is degraded (§20.4). It does not act on notifications otherwise.
More than 100 notifications per second from one process are dropped, and the
drop is logged.

Adapters, supervisors, and runners of system-mode cores MAY also send:

```json
{"type":"filesystems-ready"}
```

It has no `id` and no reply. It reports that the core has finished mounting
the system's filesystems (§20.2). Jumpstart ignores it in user mode and after
filesystems are already ready.

### 17.6 Process modes

- **oneshot**: Jumpstart starts the adapter, completes the handshake, sends one
  request, reads one response, closes descriptor 3, and waits up to 5 seconds
  for exit. After that it applies §13.7 cancellation to the process group. A
  non-zero exit, or a forced end, after a successful response is logged but
  does not change the result. Exit before a response returns `E_BACKEND`.
- **persistent**: Jumpstart starts the adapter during load (§20.3), keeps it
  running, and multiplexes requests by `id`. An unexpected exit fails in-flight
  requests with `E_BACKEND`. The adapter is restarted lazily on the next
  request, at most 5 times in 60 seconds. After that, the core becomes
  `degraded` (§20.7). On unload, Jumpstart closes descriptor 3, waits 5
  seconds, then applies §13.7 cancellation.

Runners follow the oneshot pattern, with one request per process.

`jumpctl cores run` (§21.15) drives a persistent adapter through the same
life cycle for one request: it starts the process, completes the handshake,
sends the request, and then closes descriptor 3 and applies the unload rules
above. The process is private to that command and is never shared.

Malformed JSON, unknown `type` values, duplicate or unknown ids, and oversize
lines fail the affected request with `E_PROTOCOL`, or all of the process's
in-flight requests if the id cannot be determined.

## 18. Error model

Every failure has exactly one code:

| Code | Meaning | `jumpctl` exit |
| --- | --- | --- |
| `E_USAGE` | Invalid command line, or a request sent to the wrong mode | 2 |
| `E_NAME` | Invalid, reserved, or colliding core name | 1 |
| `E_LAYOUT` | Invalid extension layout or path | 1 |
| `E_MANIFEST` | Manifest parse or validation error | 1 |
| `E_VERSION` | Incompatible Jumpstart or manifest version | 1 |
| `E_PERMISSION` | Ownership, mode, credential, or privilege error | 1 |
| `E_STATE` | Invalid enabled-state directory, or a change that would leave the enabled set empty | 1 |
| `E_CORE_STATE` | Core lifecycle refusal: the core is loading or unloading, or it is the only loaded core | 1 |
| `E_UNIT_STATE` | Operation invalid in the unit's current state | 1 |
| `E_CORE_UNLOADED` | Core not loaded in this daemon | 1 |
| `E_CORE_FAILED` | Core failed to load; `cores status` shows why | 1 |
| `E_NOT_FOUND` | Core or unit not found | 1 |
| `E_AMBIGUOUS` | Unit name claimed by more than one core | 1 |
| `E_UNSUPPORTED` | Operation not supported by the owning core | 3 |
| `E_DELEGATION` | Invalid or broken delegation | 1 |
| `E_SANDBOX` | Sandbox unavailable | 1 |
| `E_CGROUP` | cgroup v2 or a delegated subtree is unavailable | 1 |
| `E_TIMEOUT` | Operation timed out | 4 |
| `E_CANCELLED` | Operation cancelled | 4 |
| `E_BUSY` | Core has active units and stopping them was not requested | 1 |
| `E_LIMIT` | A connection or concurrency limit was reached (§19.3) | 1 |
| `E_SHUTDOWN` | The system or user daemon is shutting down | 1 |
| `E_BACKEND` | Backend reported failure | 1 |
| `E_PARTIAL` | Unit or core left in an intermediate state | 1 |
| `E_PROTOCOL` | Protocol or result-schema violation | 1 |
| `E_DEGRADED` | Core degraded at runtime (§20.7) | 1 |
| `E_INIT` | Core `init`, supervisor, or adapter startup failed | 1 |
| `E_IO` | Filesystem or socket I/O error, including "daemon not running" | 1 |

`jumpctl` diagnostics go to standard error in one of two forms:

```text
jumpctl: CORE: E_CODE: message
jumpctl: E_CODE: message
```

The first form is used when a specific core is involved, the second
otherwise. Diagnostics MUST NOT contain stack traces. The daemon logs the same
code and message, plus the request id.

## 19. Filesystem state, logging, and resources

### 19.1 Per-core directories

| Purpose | Path (§7.2) | Created | Lifetime |
| --- | --- | --- | --- |
| Persistent state | `/var/lib/jumpstart/cores/CORE/` | On load once filesystems are ready, otherwise later (§20.3 step 5); mode `0700`, owned by `account` (user mode: the user) | Survives unload, reboot, and disable. Removed only by an administrator. |
| Runtime state | `/run/jumpstart/cores/CORE/` | On load, mode `0700`, owned by `account` (user mode: the user) | Removed on unload. |

Operations running as root can always reach these directories. Operations
running as `account` own them.

The state directory may not exist yet when an operation runs, notably during
early boot with `mount_fstab = false`. `JUMPSTART_CORE_STATE_DIR` (§16.2) is still
set. Cores MUST tolerate its absence and MUST NOT create it themselves.

### 19.2 Logging

Each daemon writes its log to the path in §7.2. (`serviceHandler` currently
initializes logging with the system paths even in user mode. That MUST
change.) Until filesystems are ready (§20.2), and whenever the log file
cannot be written, the system daemon keeps log lines in a ring buffer of
1 MiB in memory. When the buffer is full, the oldest lines are dropped and
one line recording how many were dropped is kept in their place. The buffer
is flushed when filesystems become ready, and writing is retried every 60
seconds after a write failure. Each core logs with the source tag `core:CORE`. Every operation logs
one start line and one completion line containing the request id, operation,
unit, backend, duration, and result code.

Output of core processes (§15.4) is logged with the request id, or with
`core:CORE/supervisor` or `core:CORE/adapter:NAME` for long-lived processes.
Standard output is logged at `info` and standard error at `warn`.

The `log.*` operations are a core feature for its own units. They do not
replace Jumpstart's own log.

### 19.3 Resource limits

These limits are constants of this specification, not tunables, in
`manifest_version: 1`:

| Limit | Value | Beyond the limit |
| --- | --- | --- |
| In-flight mutating operations per core | 16 | Queued in arrival order |
| In-flight read-only and streaming operations per core, excluding `unit.resolve` | 64 | `E_LIMIT` |
| In-flight `unit.resolve` calls per core | 256 | Queued in arrival order |
| Concurrent socket connections per daemon | 256 | `E_LIMIT`, then close |
| Notifications per process per second | 100 | Dropped and logged |

An admitted mutating operation holds one of its core's 16 slots until it
returns, is cancelled, or reaches its `timeout` (§11.6). A core whose mutating
operations block for long (a start that waits for readiness, §11.6) can
therefore occupy every slot for up to that `timeout`, and further mutating
operations wait in the queue.

## 20. Daemon lifecycle

### 20.1 Core states

Each daemon tracks a runtime state for every discovered core:

| State | Meaning | Requests to the core |
| --- | --- | --- |
| `unloaded` | Not loaded in this daemon. | `E_CORE_UNLOADED` |
| `loading` | Load procedure in progress (§20.3). | Wait for the result, up to their own timeout |
| `loaded` | Routed and serving requests. | Dispatched |
| `unloading` | Unload procedure in progress (§20.4). | `E_CORE_STATE` once the core is closed |
| `failed` | The last load attempt failed. The error is kept for `cores status`. | `E_CORE_FAILED` |
| `degraded` | Loaded, but its supervisor exited or its persistent adapter exceeded its restart budget (§20.7). | `E_DEGRADED` |

The **loaded set** is the set of cores in `loaded` or `degraded`. Runtime
states are never persisted. The enabled set (§9) only decides what the daemon
tries to load when it starts.

A loaded core is also marked **stale** when its `init.kg` on disk differs
from the one it was loaded from (compared by size, modification time, and
SHA-256). Staleness is informational. Jumpstart keeps running the loaded
version until the core is reloaded (§20.5).

### 20.2 Startup

System mode:

1. Perform the native PID 1 duties:
   - mount `/proc`, `/dev` (devtmpfs), `/dev/pts`, `/sys`, `/run`, and, if
     needed, cgroup2 at `/sys/fs/cgroup` (§15.1);
   - if `mount_fstab` is enabled (below), remount `/` read-write and mount
     `/etc/fstab` entries (`mount -a`);
   - set the hostname;
   - disable the kernel's Ctrl-Alt-Del reboot (`reboot(RB_DISABLE_CAD)`), so
     that it arrives as `SIGINT` (§20.8); and
   - install the reaper (§15.3) and shutdown signal handlers.

   `mount_fstab` is a boolean in the `[System]` section of
   `/etc/jumpstart/main.conf`. It defaults to `true`, which is today's
   behavior. Set it to `false` when a core does its own filesystem checking
   and mounting during activation, as OpenRC (`localmount`, `fsck`), SysV
   `checkroot`, and runit or s6-linux-init stage 1 do. Jumpstart then leaves
   `/` as the kernel mounted it and mounts nothing from `/etc/fstab`. The API
   filesystems above are always mounted.

   **Filesystems ready.** Jumpstart writes persistent files (core state
   directories, §19.1, and the daemon log, §19.2) only after the system's
   filesystems are ready. With `mount_fstab = true`, they are ready as soon
   as `mount -a` returns. With `mount_fstab = false`, they become ready at the
   first of:
   - any loaded core reporting it, with the `filesystems-ready` message
     (§17.5) or the Kongue `filesystems-ready` command (§16.3), typically at
     the end of its own mount stage; or
   - every core loaded at startup having finished activation (§20.6).

   The point is reached once per boot, and the report that reached it is
   logged with its core. Later reports are ignored. User mode is always
   ready, because its paths (§7.2) live in the user's home directory and in
   runtime directories that exist before the daemon starts.
2. Open the control socket and begin accepting requests. Requests for cores
   that are still loading wait, as in §20.1.
3. Discover all cores (§8).
4. Read the effective enabled set (§9). If the state directory is invalid,
   log `E_STATE`, report it in `cores status`, and use `{serviceHandler}` as
   the set to load. This is a fixed rule, not a fallback: a broken state file
   must never leave PID 1 without its native core.
5. Order the cores to load so that every delegation target comes before the
   cores that delegate to it. Ties keep listing order.
6. Run the load procedure (§20.3) for each core in that order.
7. Once every loaded core has finished activation (§20.6), whether it
   succeeded or failed, start the daemons of lingering users (§7.6).

User mode is the same, except that step 1 is replaced by making the daemon a
child subreaper, installing the reaper and signal handlers, and emptying the
fallback runtime directory if it is in use (§7.2), and step 7 is skipped.

Cores load independently. A core that is missing, invalid, or fails to load
enters `failed`, and the error is logged with its code. A core that delegates
to a `failed` core also fails (`E_DELEGATION`). The remaining cores still
load. Jumpstart MUST NOT replace a failed core with another, MUST NOT route
its units elsewhere, and MUST NOT change persistent state.

### 20.3 Load procedure

The same procedure is used at startup, by `cores load`, by
`cores enable --now`, and by `cores reload`. Loading one core never restarts
or pauses any other core.

1. If the core is `loaded` or `degraded`, succeed without doing anything. If
   it is `loading` or `unloading`, fail with `E_CORE_STATE`.
2. Discover and validate this core afresh from disk (§8). Nothing from an
   earlier load is reused.
3. Check that every delegation target is loaded (`E_DELEGATION`).
4. Resolve its operation table. For extensions, check that cgroup v2 (and, in
   user mode, a delegated subtree) is available (`E_CGROUP`).
5. Create its cgroup subtree (§15.1) and its runtime directory (§19.1). If
   filesystems are ready (§20.2), also create its state directory. Failure to
   create the state directory is logged and does not fail the load, because
   `/var` may still be read-only or not yet mounted. Jumpstart retries when
   filesystems become ready and before each dispatch to the core, and
   creates nothing below a mount point that is not yet mounted (it never
   writes into a directory that a later `/etc/fstab` mount would shadow).
6. Run `init` in a Kongue runner (§16.1), with a fixed timeout of 60 seconds,
   unsandboxed, as `account` (system mode) or the user (user mode).
7. If a supervisor is declared, start it and complete its handshake (§15.2).
8. Start persistent adapters and complete their handshakes (§17.6).
9. Publish the core to the router. Only from this point are requests routed to
   it, and its state becomes `loaded`. If it claims `unit`, call `unit.list`
   once to seed its name index (§14.3). A failure here is logged and leaves
   the index empty, but does not fail the load.
10. Schedule its activation (§20.6), which runs asynchronously.

If any of steps 2 to 8 fails, Jumpstart rolls the load back in reverse:

- it stops any persistent adapters and the supervisor;
- it runs `shutdown` (60 seconds) if `init` completed;
- it kills and removes the cgroup subtree (§15.1); and
- it removes the runtime directory.

The core's state becomes `failed`, and the error is returned (`E_INIT` for
steps 6 to 8). The persistent state directory is left in place.

`serviceHandler` loads by running the existing service-handler initialization
directly, with no manifest file, no Kongue, and no runner.

### 20.4 Unload procedure

The same procedure is used by `cores unload`, `cores disable --now`,
`cores reload`, and daemon shutdown. Unloading one core never restarts or
pauses any other core.

Preconditions, checked before anything changes:

1. The core is `loaded` or `degraded`. If it is `unloaded` or `failed`,
   succeed without doing anything. If it is `loading` or `unloading`, fail with
   `E_CORE_STATE`.
2. No other loaded core delegates to it (`E_DELEGATION`, naming the dependent
   cores).
3. It is not the only loaded core (`E_CORE_STATE`), except during daemon
   shutdown.
4. It has no active units, unless the caller asked to stop them. The core is
   **busy** if either of these holds:
   - its `units/` cgroup reports `populated 1` in `cgroup.events` (§15.1),
     meaning some unit process still exists. Jumpstart owns this cgroup, so
     the check works even when the core cannot answer; or
   - for a `loaded` core, any unit from `unit.list` reports `active`,
     `activating`, `deactivating`, or `reloading` through `<sub>.status`,
     except a unit of the subsystem `mount` or `swap` that reports `active`.
     Such a unit represents kernel state that a core MAY leave active after
     `unit.deactivate` (§12.2), and Jumpstart's own shutdown unmounts and
     deactivates it (§20.8 step 5). A `mount` or `swap` unit that is
     `activating`, `deactivating`, or `reloading` still counts. The exception
     does not apply to `serviceHandler`, whose native units all count (§22.5).

   For a `degraded` core, the cgroup check is the only one used, and
   last-known states (§17.5) only name units in the error message. Without
   cgroup v2 (only `serviceHandler` can then be loaded), the status check
   alone is used. If the core is busy and stopping was not requested, the
   unload fails with `E_BUSY` and lists the active units, or says that unit
   processes remain.

Steps:

1. **Close** the core to clients. New unit and global requests get
   `E_CORE_STATE`. Streaming operations are cancelled immediately. From now
   on, only Jumpstart's own requests reach the core.
2. **Stop units**, if requested:
   - For a `loaded` core with `activation: "core"`, call `unit.deactivate`.
   - For a `loaded` core with `activation: "per-unit"`, stop each unit that
     counts for the busy check above (so not an `active` `mount` or `swap`
     unit) in the reverse of the order `unit.list` returned them.
   - For a `degraded` core, skip this step and log that its units will be
     ended by the cgroup kill in step 6.

   If stopping fails, Jumpstart reopens the core to clients and aborts the
   unload with `E_PARTIAL`. The core stays loaded with its remaining units.
3. **Drain.** Wait for in-flight operations, each up to its own deadline but
   at most 90 seconds in total, then cancel the rest (§13.7).
4. Run `shutdown`, if defined, with a 60-second timeout. A failure is logged
   and the unload continues.
5. Stop persistent adapters (§17.6) and the supervisor (§15.2).
6. Kill and remove the core's cgroup subtree (§15.1).
7. Remove the runtime directory. The state becomes `unloaded`.

Jumpstart never stops a core's units unless asked. Unloading a core with
running units is always an explicit choice (`--stop-units`).

### 20.5 Reload and upgrades

`reload` is an unload (§20.4) followed by a load (§20.3), using the same
stop-units choice. It is allowed for `loaded`, `degraded`, and `failed` cores
(for a `failed` core it is just a load). If the load fails, the core is left
`failed`. Jumpstart does not keep or restore the previous version.

Core files are installed, upgraded, and rolled back by kpkg (§4.2). After kpkg
changes a loaded core's files, the core is stale until it is reloaded. kpkg
packages SHOULD run `jumpctl cores reload CORE` in a post-install step when
the core is loaded. To roll back, the administrator downgrades the package
with kpkg and reloads. Jumpstart's only concern during upgrades is
compatibility: a reloaded manifest is validated like any other, and an
incompatible `manifest_version` or `jumpstart` range fails the load with
`E_VERSION`.

While a loaded core's files are being replaced, a oneshot adapter or runner
may run either version of an executable. The handshake (§17.2) detects an
adapter that no longer matches the loaded manifest, and the request fails with
`E_PROTOCOL` instead of misbehaving.

### 20.6 Activation

When a core finishes loading, Jumpstart activates it asynchronously, as
declared by its `activation` field (§11.5):

- `"core"`: call `unit.activate` once.
- `"per-unit"`: call `unit.list`, then `<sub>.start` for each unit with
  `enablement: "enabled"`, in list order.

`serviceHandler` activates natively by reaching its default target (§22.5).
Activation results and failures are logged per unit. They do not change the
core's state and never unload it. Cores without unit-addressed subsystems are
not activated.

Each core orders only its own units. Activations of different cores may run
concurrently. Ordering across cores is out of scope (§4.2). For `"per-unit"`
activation, a `<sub>.start` whose `state` is `activating` or `reloading` counts
as started; Jumpstart does not wait for the unit to become `active` (§11.6,
§12.11).

### 20.7 Degraded cores

A core becomes `degraded` when its supervisor exits (§15.2) or a persistent
adapter exceeds its restart budget (§17.6). Its requests fail with
`E_DEGRADED`, it is skipped in unqualified resolution (§14.3), and other cores
are unaffected. A degraded core is recovered with `cores reload`, which needs
no answer from the broken component (§20.4). Degradation is runtime-only and
does not change persistent state.

### 20.8 System shutdown

System shutdown is native to Jumpstart. Cores take part only through their
unload procedure.

**Triggers.** Exactly these start a shutdown:

| Trigger | Action |
| --- | --- |
| `jumpctl poweroff` or `SIGPWR` or `SIGTERM` to PID 1 | Power off |
| `jumpctl reboot` or `SIGINT` to PID 1 (Ctrl-Alt-Del) | Reboot |
| `jumpctl halt` | Halt |

The first trigger wins. Later triggers are logged and ignored.

**Sequence.** The steps and timeouts are fixed:

1. Enter shutdown. New requests other than `status` and `cores status` fail
   with `E_SHUTDOWN`.
2. Stop every user daemon: `SIGTERM`, wait up to 30 seconds, `SIGKILL`, then
   kill its cgroup (§7.6).
3. Unload every loaded core with units stopped, in the reverse of load order,
   so that delegating cores unload before their targets. `serviceHandler` is
   included. Each core's unload is limited to 90 seconds. When the limit
   expires, Jumpstart skips to that core's cgroup kill and marks it
   `unloaded`.
4. Send `SIGTERM` to every remaining process except PID 1, wait up to 5
   seconds, then send `SIGKILL`.
5. Deactivate all swap, then unmount every filesystem in the reverse of
   `/proc/self/mountinfo` order, except `/`, `/proc`, `/sys`, `/dev`, and
   `/run`. A mount that cannot be unmounted is detached lazily (`MNT_DETACH`)
   and logged.
6. Remount `/` read-only and `sync`.
7. Call `reboot(2)` with `RB_POWER_OFF`, `RB_AUTOBOOT`, or `RB_HALT_SYSTEM`.

A failing step is logged, and the sequence continues. It always reaches step
7.

### 20.9 User daemon shutdown

A user daemon shuts down on `SIGTERM`, whether sent by the system daemon
(§7.6) or by anyone else. It enters shutdown (new requests other than `status`
fail with `E_SHUTDOWN`), unloads every loaded core with units stopped in the
reverse of load order, and exits with status 0.

Each core's unload is limited to 20 seconds, and the whole shutdown to 25
seconds. When either limit expires, the daemon skips to the cgroup kill of
the remaining cores. This keeps a user daemon inside the 30 seconds the system
daemon allows it (§7.6, §20.8), so user units are stopped in order rather than
killed.

## 21. Command-line interface

### 21.1 Overview

```text
jumpctl start UNIT
jumpctl stop UNIT
jumpctl restart UNIT
jumpctl reload UNIT
jumpctl kill UNIT [--signal SIGNAL]
jumpctl enable UNIT [--now] [--target TARGET]
jumpctl disable UNIT [--now] [--target TARGET]
jumpctl status UNIT
jumpctl isolate UNIT
jumpctl set-default UNIT
jumpctl get-default [--core CORE]

jumpctl units list [--core CORE] [--subsystem SUBSYSTEM]
jumpctl units load [--core CORE]
jumpctl units validate UNIT
jumpctl units translate UNIT

jumpctl logs read [UNIT] [--core CORE] [--since TIME] [--until TIME] [--lines N]
jumpctl logs follow [UNIT] [--core CORE] [--since TIME]
jumpctl logs write [UNIT] [--core CORE] --level LEVEL MESSAGE
jumpctl logs rotate --core CORE
jumpctl logs vacuum --core CORE [--max-size BYTES] [--max-age DURATION]

jumpctl poweroff
jumpctl reboot
jumpctl halt

jumpctl users list
jumpctl users linger list
jumpctl users linger enable USER
jumpctl users linger disable USER

jumpctl env list
jumpctl env set KEY=VALUE…
jumpctl env unset KEY…
jumpctl env import KEY…

jumpctl cores list [--enabled | --loaded]
jumpctl cores status [CORE]
jumpctl cores enable CORE [--now]
jumpctl cores disable CORE [--now] [--stop-units]
jumpctl cores load CORE
jumpctl cores unload CORE [--stop-units]
jumpctl cores reload CORE [--stop-units]
jumpctl cores show CORE [--resolved]
jumpctl cores check [CORE]
jumpctl cores run CORE OPERATION [UNIT] [KEY=VALUE…]
```

`units`, `logs`, `users`, `env`, and `cores` MUST be real nested cligen
command groups, with `--help` at every level.

Commands marked **local** below work on files only. Every other command uses
the mode's socket (§14.5), and fails with `E_IO` and changes nothing if the
daemon is not running.

`jumpctl` operates in the mode given by its real UID (§7.1). `poweroff`,
`reboot`, `halt`, and `users` exist only in system mode. `env` exists only in
user mode. Using them in the other mode fails with `E_USAGE`.

`UNIT` is a unit reference (§14.2). `jumpctl` passes it to the daemon exactly
as typed, without normalizing it (§14.1).
(`cores run` is the exception: it takes a core-local unit name, §21.15.)

A unit name MAY begin with `-` (§14.1), which `jumpctl` cannot tell from an
option. `jumpctl` therefore accepts such a name only after `--`, in every
command that takes a `UNIT` (`status`, `start`, ..., `units validate`,
`units translate`, `logs read`, `logs follow`, `logs write`, and `cores run`):

```console
# jumpctl status -- -.mount
# jumpctl kill --signal HUP -- -.mount
# jumpctl logs read -- systemd/-.mount
# jumpctl cores run -- systemd mount.status -.mount
```

`--` ends option parsing for the rest of the command line, so every later word
is positional; it changes no other rule (`status -- -.a -.b` still has two
units). Before `--`, a word that begins with `-` is an option, and an unknown
one fails with `E_USAGE` (exit status 2) before any I/O, so `jumpctl status
-.mount` is a usage error. A qualified reference (`systemd/-.mount`) begins
with the core name and needs no `--`.

### 21.2 Unit commands

Unit commands map to operations as described in §14.4. The existing commands
(`start`, `stop`, `enable`, `disable`, `status`) keep their output text:

```console
# jumpctl start sshd
jumpstart: started unit sshd
```

Today `jumpctl` prints these lines without waiting for the daemon. From now on
it prints them only after a successful response. Failures print a diagnostic
(§18) and exit with the matching status.

The new verbs print `jumpstart: restarted unit NAME`, `reloaded`, `killed`, or
`isolated`, using the name as typed.

- `kill` sends `SIGTERM` unless `--signal` is given. `--signal` accepts a
  signal name with or without the `SIG` prefix, such as `HUP` or `SIGHUP`, or
  a decimal signal number.
- `status` goes through the daemon for every unit, native ones included
  (today it reads native status files directly). It prints
  `jumpstart: unit is reporting status as 'STATE'`. `STATE` is the native
  status text for native units (Appendix A) and the §12.11 state for
  extension units. For extension units, when the result carries `since`,
  `substate`, or `result` (§12.11), `jumpctl` then prints one `KEY: VALUE` line
  for each of them that is present, in the order `since`, `substate`, `result`,
  after the first line, which does not change. Native output is therefore
  byte-identical: `serviceHandler` never sets those fields.
- `--target TARGET` on `enable` and `disable` names a target unit of the same
  core (§14.4).

### 21.3 Targets and defaults

- `isolate UNIT` switches to the target `UNIT`. For runlevel-based managers
  this is a runlevel change (§12.7).
- `set-default UNIT` makes the target `UNIT` its core's default and prints
  `jumpstart: default target of CORE is now NAME`.
- `get-default` prints one line, `CORE/NAME`, for each loaded core whose
  `target.get-default` is supported, in load order. `--core` limits it to one
  core.

### 21.4 `units`

- `units list` calls `unit.list` on each loaded core, or on `--core` only. It
  prints one unit per line as `CORE/UNIT SUBSYSTEM ENABLEMENT`. Cores are
  listed in load order, and units in the order each core returned them. Cores
  that do not claim `unit` are skipped.
- `units load` calls `unit.load` on each loaded core that supports it, or on
  `--core` only. It prints `CORE: loaded N` per core, and the errors it
  reports.
- `units validate UNIT` calls `unit.validate`. It prints the diagnostics, and
  exits 1 when the unit is not valid.
- `units translate UNIT` calls `unit.translate` and prints the decfile to
  standard output. It never writes files.

### 21.5 `logs`

`logs` commands map to `log.*`. With `UNIT`, the request is routed to the
unit's owner. Without `UNIT`, `--core` is REQUIRED (`E_USAGE`) and the
operation is global for that core.

- `read` prints one entry per line as `TIME LEVEL UNIT MESSAGE`. `--lines`
  defaults to 1000 and may be at most 10000 (§12.10).
- `follow` prints the same format until interrupted. An interrupt closes the
  connection, which cancels the request (§14.5).
- `TIME` arguments are RFC 3339 timestamps. `vacuum` requires at least one of
  its options.

### 21.6 Power

`poweroff`, `reboot`, and `halt` start the system shutdown (§20.8). They print
`jumpctl: powering off`, `jumpctl: rebooting`, or `jumpctl: halting` once the
daemon confirms that shutdown has begun. The connection may then close at any
time.

### 21.7 Output conventions

- Unit commands (§21.2, §21.3) print success lines with the `jumpstart:`
  prefix, for compatibility with existing scripts.
- All other commands print success lines with the `jumpctl:` prefix, or plain
  data lines where this section says so.
- Errors always use the diagnostic format in §18, on standard error.

### 21.8 `users` (system mode)

- `users list` prints one line per user daemon the system daemon manages:
  `UID USER SESSIONS STATE LINGER`. `STATE` is `running`, `starting`,
  `stopping`, or `failed`, and `LINGER` is `linger` or `-`.
- `users linger enable USER` and `users linger disable USER` are **local**.
  `USER` MUST be an existing user name in the password database, other than
  root (`E_NOT_FOUND` or `E_USAGE` otherwise). The commands create or remove
  `/etc/jumpstart/users/linger/USER` atomically. A
  lingering user's daemon starts at the next boot, or at their next session,
  whichever comes first. Disabling linger does not stop a running user daemon
  that still has sessions.
- `users linger list` is **local** and prints one user name per line.

### 21.9 `env` (user mode)

These commands manage the user environment store (§7.5):

- `env list` prints `KEY=VALUE` lines in byte order of key.
- `env set KEY=VALUE…` sets variables. `env unset KEY…` removes them.
- `env import KEY…` copies the named variables from `jumpctl`'s own
  environment, which is how a login script passes `DISPLAY`,
  `WAYLAND_DISPLAY`, or `DBUS_SESSION_BUS_ADDRESS` to the user daemon.
  Variables that are not set in `jumpctl`'s environment are reported and
  skipped.

Keys MUST match `[A-Z_][A-Z0-9_]*`. `HOME`, `USER`, `LOGNAME`, `SHELL`, and
`JUMPSTART_*` are rejected (`E_USAGE`). Changes apply to processes started
afterwards (§7.5).

### 21.10 Live core commands

These change the running daemon without touching persistent state.

- `cores load CORE` runs the load procedure (§20.3) and prints
  `jumpctl: loaded core CORE`. The core MUST be discovered and valid, but need
  not be enabled. The result lasts until the daemon restarts.
- `cores unload CORE [--stop-units]` runs the unload procedure (§20.4) and
  prints `jumpctl: unloaded core CORE`.
- `cores reload CORE [--stop-units]` runs §20.5 and prints
  `jumpctl: reloaded core CORE`.
- `cores status [CORE]` prints one line per discovered core, in listing order:
  `CORE STATE`, followed by ` stale` when it applies, and by ` E_CODE: message`
  for `failed` cores. For example, `systemd loaded stale`. If the enabled
  state is invalid (§20.2), it first prints `E_STATE: message`.

### 21.11 `show` (local)

`show` prints the manifest summary and the operation table, one operation per
line, in registry order:

```console
$ jumpctl cores show systemd
name: systemd
core_version: 0.1.0
version: 255
jumpstart: >=1.3.0, <2.0.0
modes: system user
account: root
activation: core
supervisor: bin/systemd-compat-manager
enabled: yes
subsystems: core unit service mount timer target log
operations:
  core.verify      custom       handler=verify_payload
  unit.list        exec         bin/systemd-compat (persistent)
  ...
  service.reload   exec         bin/systemd-compat (persistent)
```

With `--resolved`, every option is printed with its effective value in the
current mode, including defaults from this specification and the effective
user. Each defaulted value is marked `(default)`. This is the authoritative
view of how Jumpstart will run each operation.

For `serviceHandler`, `show` prints the virtual manifest from Appendix A.

### 21.12 `check` (local)

`check` validates all extensions, or one named core. It reports each error as
in §18, plus warnings (§10.5), and exits 1 if any error was found.

### 21.13 `list` (local)

```console
$ jumpctl cores list
serviceHandler
systemd
```

The output is one discovered core per line, in listing order (§7.3). There are
no headings, decorations, versions, or colors. `--enabled` filters by the
enabled set and is local. `--loaded` filters by the loaded set and uses the
socket. Both keep the same order. An empty result prints nothing. Invalid
extensions are not listed; `check` reports them. Without `--loaded`, the
command exits 0 even when some extensions are invalid, and 1 only if discovery
itself fails.

### 21.14 `enable` and `disable`

```console
# jumpctl cores enable systemd
jumpctl: enabled core systemd
```

Without `--now`, these are **local**. They change only persistent state, which
takes effect the next time the daemon starts.

`enable` validates `CORE`, requires it to be discovered and valid, checks that
its delegation targets are enabled, materializes state if it is uninitialized
(§9.2), and writes the marker atomically. Enabling an already-enabled core
succeeds without changes. Enabling a core never disables another.

`disable` fails without changes if it would empty the enabled set (`E_STATE`)
or break a delegation between enabled cores (`E_DELEGATION`). Disabling an
already-disabled core succeeds without changes. A disabled extension still
appears in `list`.

With `--now`, the live change happens first, and the persistent change happens
only if it succeeds:

- `enable --now` loads the core (§20.3), then writes the marker. If the core
  is already loaded, the load step succeeds without doing anything and the
  marker is still written. If the load fails, nothing is persisted.
- `disable --now [--stop-units]` unloads the core (§20.4), then removes the
  marker. If the unload fails, nothing is persisted and the core stays loaded.
  `--stop-units` is valid only with `--now`.

If the persistent write fails after a successful live change, the command
reports `E_IO` and says that the live and persistent states now differ.
`cores status` and `cores list --enabled` show the difference.

### 21.15 `run` (local)

```text
jumpctl cores run CORE OPERATION [UNIT] [KEY=VALUE…]
```

`run` executes **one operation of one core** without the daemon. It is a
debugging tool: it shows exactly what a core's backend, adapter, or handler
receives and answers, with none of the daemon's routing, scheduling, or
state in between. Any operation of the core's resolved manifest (§21.11) is
accepted: `core.verify`, and also operations such as `unit.list`,
`service.status`, `service.start`, `target.get-default`, `log.read`, and
`unit.resolve` (§12). The core MUST be discovered and valid, but need not be
enabled or loaded. `run` never connects to a daemon socket. In system mode it
runs as root, and in user mode as the user, with the same discovery and
ownership checks as every other command (§8.3).

```console
# jumpctl cores run systemd core.verify
installed: true
version: 255
```

#### Arguments

`OPERATION` is a registry name (§12). Its routing column decides whether
`UNIT` is given:

| Routing (§12) | Operations | `UNIT` |
| --- | --- | --- |
| `core`, `global` | `core.verify`, `unit.list`, `unit.resolve`, `unit.load`, `unit.activate`, `unit.deactivate`, `target.get-default`, `log.rotate`, `log.vacuum` | MUST be absent (`E_USAGE` otherwise) |
| `unit` | `unit.validate`, `unit.translate`, and every `<sub>.<verb>` of a unit-addressed subsystem | REQUIRED (`E_USAGE` when missing) |
| `unit` or `global` | `log.read`, `log.follow`, `log.write` | Optional. Without it the operation is global, as with `--core` and no unit (§21.5) |

`UNIT` is the first argument after `OPERATION` that contains no `=`. An
argument that contains `=` is always a `KEY=VALUE`, so a unit whose name
contains `=` cannot be addressed by `run`. `UNIT` is passed to the backend
**exactly as typed**, as the core's own name for the unit. `run` does not
call `unit.resolve`, does not normalize or qualify the name, and does not
accept the qualified form `CORE/UNIT` (§14.2): a `/` in `UNIT` fails with
`E_USAGE`, as does any other invalid unit name (§14.1). The operator sees
precisely what the adapter receives.

`KEY=VALUE` arguments are converted to the type the registry gives the
argument (§12, §16.3): booleans are `true` or `false`, integers are decimal,
and strings are taken as given, including any further `=`. Unknown keys,
duplicate keys, bad values, and arguments that are not scalars (none exist in
`manifest_version: 1`) are `E_USAGE`. All usage errors are reported before
any process starts. An operation name that is syntactically valid but absent
from the registry, or from the core's manifest, is `E_UNSUPPORTED`.

#### Execution

`run` resolves the entry for `OPERATION` in the core's manifest for the
current mode, with the rules of §11.4, and then runs it through the same
backend layer as the daemon. Only where the process runs differs:

| Backend | What `run` does |
| --- | --- |
| `custom` | Starts a Kongue runner (§16.1) for the handler. |
| `exec`, `oneshot` | Starts the adapter, completes the handshake (§17.2), sends the request, reads the response, closes descriptor 3, and waits for the process (§17.6). |
| `exec`, `persistent` | Starts the adapter itself, as for `oneshot`. The handshake carries the declared operation set of the executable (§17.2), which MUST equal the operations the manifest routes to it, so a drifted adapter fails with `E_PROTOCOL` here as well. `run` sends the single request, waits for the response, and then closes descriptor 3 and applies the unload rules of §17.6 (five seconds, then §13.7). This adapter is private to the command: it is not the daemon's shared adapter, and it does not outlive the command. |
| `delegate` | Follows the delegation (§13.4) and runs the target core's entry the same way. The target MUST be discovered and valid; it need not be enabled or loaded. The target operation MUST be in the same subsystem, a cycle is `E_DELEGATION`, and so is a chain longer than four hops. Past the first hop, a target that is missing, invalid, or does not support the operation is `E_DELEGATION`. |
| `unsupported`, or no entry | `E_UNSUPPORTED` (exit status 3). There is no fallback (§11.4). |
| `native` | `E_UNSUPPORTED` with the message `native operations need the daemon: use the normal jumpctl commands` (§13.6). `serviceHandler` has no extension manifest to run, so every `serviceHandler` operation is refused this way (`core.*` operations with `serviceHandler does not claim core`). A delegation to `serviceHandler` is refused the same way. |

The process gets the descriptors and attributes of §15.4, the environment of
§7.5 with the `JUMPSTART_CORE_*` variables (§16.2), and, for runners and
oneshot adapters, `JUMPSTART_OP`, `JUMPSTART_UNIT`, `JUMPSTART_REQUEST_ID`,
and the `JUMPSTART_ARG_*` variables. It runs as the `user` option in system
mode and as the user in user mode (§13.1), with the sandbox if the entry
declares one (an unavailable sandbox is `E_SANDBOX`, and the operation does
not run). It joins `jumpctl`'s own cgroup rather than the core's. The user
environment store (§7.5) lives in the user daemon and is therefore empty for
`run`.

`JUMPSTART_CORE_STATE_DIR` and `JUMPSTART_CORE_RUNTIME_DIR` (§16.2) are the
real per-core paths (§19.1). `run` never creates the state directory, which
cores MUST tolerate being absent. The runtime directory exists only while a
daemon has the core loaded. When it does not exist, `run` creates it for the
duration of the command, as a `0700` directory owned by `account` (user mode:
the user), so that an adapter can rely on it as under the daemon, and removes
it afterwards, unless a daemon has replaced it in the meantime. An existing
runtime directory belongs to a loaded core: `run` uses it as it is and never
clears, changes, or removes it, so an adapter then sees the loaded core's
runtime state, such as pid files.

The entry's `timeout` (§11.6, §13.1) bounds the whole operation as an
absolute deadline: the request's `deadline` (§17.3) is computed when it is
sent, and on expiry `run` cancels the operation (§13.7) and fails with
`E_TIMEOUT`. `SIGINT` (Ctrl-C) cancels the request (§13.7) and `run` fails
with `E_CANCELLED` once the process has ended; a second `SIGINT` stops
waiting. Interrupting `log.follow` is its normal end: like `logs follow`
(§21.5), `run` then exits 0 without a diagnostic. If the adapter completes
the operation despite the cancellation, `run` prints its result and exits 0.
The process's standard output and standard error, and a runner's `log`
commands, are written to `run`'s standard error as `core:CORE: LINE`.

#### Output

Result fields are printed as `KEY: VALUE` lines in registry order. Elements
of a list of scalars are joined with `, `. List-of-objects fields (§17.4,
§12) print one line per element, and elements that arrive as `item` events
and `entry` events (§17.4) are printed as they arrive, not at the end:

- `units` elements as `NAME SUBSYSTEM ENABLEMENT`, the line of `units list`
  (§21.4) without its `CORE/` prefix, since only one core is involved;
- log entries (`entries`, and the `entry` events of `log.follow`) as
  `TIME LEVEL UNIT MESSAGE`, the line of `logs read` (§21.5), with `-` for
  the empty unit of a global entry; and
- other object lists (`errors`, `diagnostics`, `failed`) as
  `FIELD: ` followed by the core's object in the form of `units load`
  (§21.4), one line per element: `NAME: CODE: MESSAGE`, then any further keys
  as ` KEY=VALUE`.

Elements delivered in the response itself follow the streamed ones in the
same format. Streamed elements are not printed twice. Only list-of-objects
fields stream. Lists of scalars are printed with the final result.

`run` validates every result against the registry exactly as the daemon does
(§16.3): an unknown or missing field, a value of the wrong type, or a value
outside its closed vocabulary (§12.10 to §12.12) is `E_PROTOCOL`, and nothing
is printed for the result. An `item` or `entry` event that fails validation is
not printed, and fails the operation with `E_PROTOCOL`. The exit status is
given in §21.16.

#### Warning

For an operation whose registry class is **mutating** (§11.6), and only for
those (not for read-only or streaming operations, and not for
`core.verify`), `run` prints one line to standard error before it starts the
backend, but after the operation has been resolved, so a usage error or
`E_UNSUPPORTED` prints no warning:

```text
jumpctl: warning: running OPERATION outside the daemon: no locking, supervision or daemon state; for debugging only
```

`OPERATION` is the name as typed. A delegation counts as mutating when either
the typed or the executed operation is.

#### Caveats

`run` bypasses the daemon. It is for debugging a core's adapter or handlers,
and mutating operations are allowed, with the operator responsible for the
outcome:

- **No locking and no limits.** The per-unit, core, and log locks (§11.6) are
  not taken, so a `run` can overlap with the daemon's own requests to the same
  core, or with another `run`. Admission and resource limits (§19.3) do not
  apply.
- **No routing.** There is no `unit.resolve`, no name index, and no qualified
  names (§14.3). `UNIT` is the core-local name.
- **No sharing and no supervision.** The adapter is private to the command,
  and the core's supervisor (§15.2) and cgroup (§15.1) are not involved. A
  `service.start` run this way starts the service under a transient adapter,
  in `jumpctl`'s cgroup and not in the core's, so the daemon neither sees nor
  supervises it. If the core's manager does not daemonize the service (§13.7),
  its processes can end when the command ends.
- **No daemon state, no log.** Nothing is recorded in the daemon: not
  unit state, not last-known state (§17.5), and not the core's log (§19.2).
  Notifications (§17.5) and `filesystems-ready` messages are ignored.
- **Security is unchanged.** `run` grants nothing the user could not already
  do. System mode requires root, and user mode runs as the user. Discovery
  and ownership checks, verified executables, the sandbox, and `E_SANDBOX`
  failing closed all apply (§24).

#### Examples

A unit-addressed operation, answered by a persistent adapter. The unit name
reaches the adapter as typed:

```console
# jumpctl cores run openrc service.status agetty.tty1
state: active
detail: started
enablement: enabled
main_pid: 412
pids: 412
```

A streaming list operation, whose items print as the adapter sends them:

```console
# jumpctl cores run openrc unit.list
agetty.tty1 service enabled
sshd service disabled
```

A global log operation, with an argument:

```console
$ jumpctl cores run ini log.read lines=2
2026-10-17T10:00:00Z info - started
2026-10-17T10:00:01Z warn - slow reply
```

A mutating operation prints the warning on standard error:

```console
# jumpctl cores run openrc service.start sshd
jumpctl: warning: running service.start outside the daemon: no locking, supervision or daemon state; for debugging only
state: active
```

Usage errors, a missing `UNIT`, and the qualified form:

```console
$ jumpctl cores run openrc service.status
jumpctl: E_USAGE: service.status requires a UNIT (a name without '=')
$ jumpctl cores run openrc service.status openrc/sshd
jumpctl: E_USAGE: UNIT must be a core-local name, not CORE/UNIT: openrc/sshd
$ jumpctl cores run openrc unit.list sshd
jumpctl: E_USAGE: unit.list takes no UNIT: sshd
```

An operation that needs the daemon:

```console
# jumpctl cores run serviceHandler service.start sshd
jumpctl: serviceHandler: E_UNSUPPORTED: native operations need the daemon: use the normal jumpctl commands
```

### 21.16 Exit status

| Status | Meaning |
| --- | --- |
| 0 | Success, including idempotent no-ops |
| 1 | Error (see §18) |
| 2 | Usage error |
| 3 | Operation unsupported |
| 4 | Timed out or cancelled |

## 22. Native targets

`serviceHandler` gains targets, so that native units can be grouped, enabled
into named sets, and booted by target, just like units of extension cores.
This section is a dependency on `jumpstart_dec(5)`, which MUST document it.

### 22.1 Target decfiles

A new decfile type, `target`, declares a group of native units:

```kongue
description: "Multi-user system"
type: "target"
wants:
  - "sshd"
  - "network"
  - "getty::tty1"
```

| Variable | Req. | Meaning |
| --- | --- | --- |
| `wants` | no | Native units (decfile names, optionally with `::` sub-units) that are always members of this target. They may be other targets. |

A target decfile has no function block. Cycles among targets (through `wants`
or enablement links) are reported by `unit.validate` and `unit.load`. A target
in a cycle cannot be started (`E_UNIT_STATE`).

### 22.2 Membership and enablement

The members of a target `T` are its `wants` plus every unit enabled into it:

- `jumpctl enable UNIT --target T` creates the link
  `/etc/jumpstart/targets/T/UNIT.kg` to the unit's decfile, in the same style
  as `/etc/jumpstart/enabled/` today. `disable UNIT --target T` removes it.
- `jumpctl enable UNIT` without `--target` keeps today's behavior: it creates
  `/etc/jumpstart/enabled/UNIT.kg`, which makes the unit a member of the
  built-in `default` target (§22.3).
- Sub-units (`foo::bar`, §22.4) cannot be enabled on their own, with or
  without `--target`, because a link names a whole decfile. `enable` and
  `disable` of a sub-unit return `E_UNSUPPORTED`, naming the decfile. To make
  only one sub-unit a member of a target, list it in that target's `wants`.

Native enablement values (§12.12):

| Value | When |
| --- | --- |
| `enabled` | The unit is linked in `enabled/` or in any `targets/*/` directory. |
| `static` | It is not linked anywhere, but some target lists it in `wants`. |
| `disabled` | Otherwise. |

`serviceHandler` never reports `indirect`, `masked`, or `generated`.

User mode uses the same layout under `~/.config/jumpstart`.

### 22.3 The default target

`default` is a built-in target that always exists. Its members are the units
linked in `enabled/`. A decfile named `default.kg` is rejected, because the
name is reserved.

**Migration.** An installation that already has a `default.kg` keeps
booting, but that unit is no longer loaded: `unit.load` and every daemon
start report it as an error naming the reserved name, and it never starts.
The administrator renames it (and its `enabled/` link) to restore it. Release
notes for the first version implementing this section MUST call this out.

The **boot target** is the target named in `/etc/jumpstart/default-target`, a
file containing one native target name followed by a newline. If the file
does not exist, the boot target is `default`. This preserves today's boot
behavior on existing installations.

`target.set-default` writes the file atomically (a temporary file plus
`rename`). `target.get-default` reads it.

### 22.4 Multi-unit decfiles

An unqualified `multi` decfile (§14.3) resolves to the `target` subsystem. Its
members are its sub-units. `jumpctl start foo` therefore starts every sub-unit
of `foo`, and `jumpctl stop foo` stops them. Each sub-unit (`foo::bar`)
resolves to its own subsystem, as it does today.

### 22.5 Native activation, deactivation, and isolate

- `unit.activate` starts the boot target.
- Starting a target starts every member, transitively, together with every
  unit in their `depends` closure, whether or not those units are enabled.
  Units are ordered by `depends` and `after`. This changes how native boot
  works today: `serviceHandlerInit` currently skips dependencies that are not
  in `enabled/`, and requires dependencies to be already running. Under this
  specification it starts them instead. Starting a single non-target unit
  with `jumpctl start` keeps today's behavior.
- `unit.deactivate` stops every active native unit in the reverse of the
  order in which they were started. It does not use the exemption of §12.2 for
  `mount` and `swap` units: native mounts are stopped (unmounted) like every
  other unit, as Appendix A defines. For the same reason the busy check of
  §20.4 does not exempt `serviceHandler`'s native units: any active native unit,
  a mount included, keeps `serviceHandler` busy. Nothing in `serviceHandler`
  changes.
- `target.isolate T` starts `T`, then stops every active native unit that is
  not a member of `T`, directly or transitively.
- `target.status` reports `members` as the transitive member list. A member
  is **satisfied** if it is `active`, or if it is a oneshot unit whose last
  run, since the target was last started, exited with status 0. A target's
  state is:
  - `failed` if any member is `failed`;
  - otherwise `active` if every member is satisfied, which includes a
    started target with no members;
  - otherwise `inactive` if the target has not been started since boot or
    was stopped, or if no member is satisfied;
  - otherwise `activating`.

  `detail` says how many members are satisfied. Targets and multi-unit
  decfiles have no status file. `jumpctl status` prints their native status
  text as `running`, `stopped`, `starting`, or `failed`, matching `active`,
  `inactive`, `activating`, and `failed`.

## 23. Versioning and compatibility

### 23.1 Manifest version

`manifest_version` fixes the grammar, top-level fields, registry, backends, and
defaults that apply to a manifest. `manifest_version: 1` is defined by this
document and is not yet implemented, so it may still change until Jumpstart
first ships it. After that, Jumpstart MUST reject manifest versions it does not
implement with `E_VERSION`. A Jumpstart release MAY support several manifest
versions at once, and MUST validate each manifest against its declared
version.

### 23.2 Jumpstart version range

`jumpstart` is a comma-separated conjunction of comparators:

```text
range      = comparator *( "," SP comparator )
comparator = ( ">=" / ">" / "<=" / "<" / "=" ) version
version    = 1*DIGIT "." 1*DIGIT "." 1*DIGIT
```

Jumpstart's version is taken from `jumpstartVersion` without its `jumpstart v`
prefix.

### 23.3 Evolution rules

- Once `manifest_version: 1` has shipped, adding verbs, subsystems, backends,
  options, or result fields requires a new `manifest_version`.
- A new manifest version MUST NOT change the meaning of an existing version's
  manifests.
- Error codes MAY be added in any Jumpstart release. Clients MUST treat unknown
  codes as `E_BACKEND`.
- The exec protocol version (`jumpstart-core/N`) is independent of the
  manifest version. Jumpstart offers every version it supports in the
  handshake, and the process picks one (§17.2).

### 23.4 `core_version`

`core_version` describes the extension and follows semantic versioning.
Jumpstart displays it but does not interpret it.

## 24. Security considerations

- **Trust boundary.** System extensions are root-owned code, and run as root
  unless they declare a non-root `account`. Installing one is equivalent to
  installing a root daemon. Discovery checks the ownership and modes of the
  whole extension tree and its roots (§8.3), so a non-root user cannot plant
  or modify a system core.
- **Time-of-check races.** Executables and manifests are opened without
  following symlinks, re-checked on the open descriptor, and executed through
  it (§8.3).
- **User/system separation.** User cores never cross into the system daemon
  (§7.4). The system daemon never reads user core roots, state, or sockets. It
  starts user daemons only with dropped credentials, and never acts on
  anything they say. The system socket is root-only and checks peer
  credentials, and user daemons have no privileged channel. A user core can
  do nothing its user could not already do.
- **Untrusted unit definitions.** System cores MUST ignore unit definitions
  that non-root users can write (§7.4).
- **Control socket robustness.** Malformed, oversize, or idle requests are
  rejected without affecting the daemon, and connection and concurrency limits
  apply (§14.5, §19.3). A client can never stall or crash PID 1.
- **No execution during discovery.** `list`, `show`, and `check` never run
  Kongue code or executables, so inspecting a hostile extension is safe.
- **Kongue isolation.** Kongue always runs in a separate runner process
  (§16.1), never inside the daemon, so a core's script cannot change PID 1's
  working directory, environment, or console.
- **Shell injection.** Request data cannot be interpolated into shell
  commands (§16.4). A unit name that begins with `-` (§14.1) is accepted by
  `jumpctl` only after `--` (§21.1), and handlers pass `--` before it (§16.4).
- **Sandbox scope.** The sandbox is used only for read-only operations. It
  drops every capability, sets no-new-privileges, and applies a seccomp
  filter, so a sandboxed root process cannot remount, load modules, or reboot
  (§13.8). An unavailable sandbox aborts the operation.
- **Process containment.** Every core process runs in the core's cgroup and is
  reaped by Jumpstart or the core's supervisor. Unloading a core kills
  everything left in its cgroup (§15).
- **Fail closed.** A missing, invalid, or failed core stays `failed`, and its
  units are unreachable rather than handled by another core.
- **No implicit fallback.** Unsupported operations are never retried on
  another core, and no operation is substituted for another. Without this
  rule, a request could reach a core whose semantics the administrator did not
  choose.
- **Live changes.** Loading and unloading are serialized per core, are
  root-only in system mode, and never stop units unless explicitly asked.
- **Delegation.** Delegation targets must be enabled and loaded in the same
  daemon, chains must be acyclic, and depth is bounded.
- **Untrusted process output.** Jumpstart treats all protocol input from core
  processes as untrusted. It enforces line size, schema, id, and rate checks.
- **Environment.** Core processes receive only the environment defined in
  §7.5. `JUMPSTART_*` variables cannot be overridden.
- **Privilege.** `account` and `user` let system cores run without root.
  Cores SHOULD use them, and SHOULD use `sandbox: true` for read-only
  operations.
- **Debugging bypass.** `jumpctl cores run` (§21.15) runs any operation of a
  core outside the daemon, so none of the daemon's protections that depend on
  being the single scheduler apply to it: no locks, admission limits,
  routing, or supervision. It adds no privilege. It needs root in system mode
  and runs as the user in user mode, it applies the discovery and ownership
  checks, the verified executables, the account and sandbox rules, and the
  `E_SANDBOX` failure closed, and it never contacts a socket. A mutating
  operation run this way can contend with the daemon and leave processes
  that the daemon does not know, so it prints a warning, and operators SHOULD
  use it for debugging only.

## 25. Packaging responsibilities

- Cores are packaged and managed by kpkg. An extension package installs
  exactly one directory beneath a core root. Jumpstart never invokes or
  queries kpkg or any other package manager.
- kpkg owns upgrades and rollback of core files. A package that upgrades a
  loaded core SHOULD run `jumpctl cores reload CORE` afterwards (§20.5).
- Package managers MUST NOT write to an enabled-state directory. New
  extensions are disabled until an administrator enables them.
- The extension package and its managed payload SHOULD be separate packages,
  with the extension depending on the payload. Both are built and installed by
  kpkg.
- Jumpstart's own package ships `pam_jumpstart.so` and
  `/usr/lib/jumpstart/kongue-runner`. Distributions enable the PAM module in
  their session stacks.
- User cores are installed by the user into the user core root (§7.2). How
  they get there is outside this specification.
- Removing an enabled extension leaves a deliberate, detectable configuration
  error. Package tooling SHOULD refuse such a removal or warn about it, and
  SHOULD unload a loaded core first. Jumpstart fails closed regardless.

## 26. Implementation structure

The logic is separate from CLI presentation, and every path is injectable. The
code that an external core implementation also needs is a reusable library
(§26.1); the rest is the daemon's:

```text
libs/
  jumpstartcore/                     core library (§26.1), source only
    jumpstartcore.nimble
    src/
      jumpstartcore.nim              umbrella module re-exporting the library
      jumpstartcore/
        errors.nim       typed error model and codes (§18)
        names.nim        core names, unit names, unit refs (§6, §14.1, §14.2)
        registry.nim     operation registry for manifest_version 1 (§12)
        registry_types.nim  registry data types (§12)
        protocol.nim     exec protocol codec and validation (§17)
        adapterloop.nim  adapter end of the exec protocol (§17)
        jsonlines.nim    JSON-lines reader and writer for descriptors 3 and 4 (§17.1)
        cgroups.nim      per-core and nested unit cgroups, attribution (§15.1)
        reaper.nim       central child reaper (§15.3)
        spawn.nim        process attributes, standard and passed descriptors (§15.4)
        sandbox.nim      sandbox (§13.8)
        environment.nim  closed core process environments (§7.5, §16.2)
        coremode.nim     `CoreMode` (§7)
        peercred.nim     peer credentials, pidfds, /proc identity (§7.4, §15.3)
        verified_file.nim  verified reads of core files (§8.3)
jumpstart/
  cores/                             daemon side
    modes.nim        mode selection, paths, isolation checks (§7)
    sessions.nim     user daemon sessions and linger (§7.6)
    manifest.nim     manifest model and static validation (§10, §11, §13)
    discovery.nim    discovery and ownership checks (§8)
    state.nim        enabled state (§9)
    routing.nim      resolution and request mapping (§14.3, §14.4)
    dispatch.nim     backends, timeouts, cancellation (§13)
    lifecycle.nim    core states, load, unload, reload, activation (§20)
    shutdown.nim     system and user daemon shutdown (§20.8, §20.9)
    control.nim      control socket protocol and peer checks (§14.5)
    local_run.nim    one operation of one core without the daemon (§21.15)
  kongue_runner.nim  Kongue runner executable (§16); installed as kongue-runner
  pam_jumpstart.nim  PAM session module (§7.6)
  jumpctl.nim
kongue/              structured headers and manifest mode (§10.2, §10.3)
tests/
  jumpstartcore/     tests of the library
  jumpstart/         tests of the daemon, `jumpctl`, and library plus daemon together
nim.cfg              adds libs/jumpstartcore/src to the compiler path
```

A hyphen cannot appear in a Nim module name, so the source file is
`kongue_runner.nim`; the installed binary is still named `kongue-runner`.

Conceptual API (Nim names are non-normative):

```nim
# jumpstart/cores/modes.nim
type CorePaths* = object
  mode*: CoreMode                  # cmSystem or cmUser
  coreRoots*: seq[string]          # system root, then user root in user mode
  configRoot*, stateRoot*, runtimeRoot*, cgroupRoot*, socketPath*: string

proc pathsFor*(mode: CoreMode, uid: Uid): CorePaths

# jumpstart/cores/discovery.nim
proc discoverCores*(paths: CorePaths): DiscoveryResult

# jumpstart/cores/state.nim
proc enabledCores*(d: DiscoveryResult, paths: CorePaths): seq[Core]
proc enableCore*(name: string, paths: CorePaths)
proc disableCore*(name: string, paths: CorePaths)

# jumpstart/cores/manifest.nim
proc loadManifest*(path, expectedName: string): CoreManifest
proc resolveOps*(m: CoreManifest, mode: CoreMode): seq[ResolvedOp]

# jumpstart/cores/routing.nim, daemon.nim
proc resolveUnit*(d: var Daemon, unitRef: string): Resolution
proc dispatch*(d: var Daemon, core: Core, op: string, unit: string,
               args: JsonNode, deadline: Time): OpResult

# jumpstart/cores/lifecycle.nim, daemon.nim
proc loadCore*(d: var Daemon, name: string)
proc unloadCore*(d: var Daemon, name: string, stopUnits: bool)
proc reloadCore*(d: var Daemon, name: string, stopUnits: bool)

# libs/jumpstartcore/src/jumpstartcore/protocol.nim (the stable contract, §26.1)
proc parseLine*(line: string): ProtocolMessage
proc encodeLine*(message: ProtocolMessage): string
proc helloReply*(role: CoreRole, operations: seq[string] = @[],
                 protocol = CoreProtocol): ProtocolMessage
proc validateResult*(operation: string, value: JsonNode,
                     manifestVersion = CurrentManifestVersion)

# libs/jumpstartcore/src/jumpstartcore/adapterloop.nim
proc newAdapterSession*(role = crAdapter, inputFd = 3.cint,
                        outputFd = 4.cint): AdapterSession
proc handle*(session: AdapterSession, operation: string,
             handler: OperationHandler)
proc serve*(session: AdapterSession): int
```

The registry is data, not code: one table keyed by `manifest_version`. The
generic implementation MUST NOT contain branches on specific core names other
than `serviceHandler`.

**Required changes to existing code.** Today the daemon is a single-threaded
accept loop that blocks on each client, parses requests without error
handling, and lets each native service wait on its own process. This
specification requires an event-driven daemon instead:

- concurrent connections with per-request cancellation;
- a central reaper;
- per-unit serialization; and
- responsive signal handling while operations run.

The other existing code this specification requires to change:

- process creation moves from Nim's `osproc` to a spawn layer built on
  `fork` or `clone3`, because `osproc` cannot start a process inside a cgroup
  or set descriptors and subreaper attributes before `execve` (§15);
- PID 1's `setControlCHook` handler, which today removes the socket and exits
  on `SIGINT` (and so would panic the kernel), is replaced by the shutdown
  triggers of §20.8;
- `jumpctl` stops normalizing unit names (§14.1) and waits for responses
  (§21.2);
- `status` moves into the daemon (§21.2);
- mode selection is tightened (§7.1);
- the user socket moves (§7.2);
- user-mode logging gets its own paths (§19.2); and
- `serviceHandler` gains targets (§22) and the fixes listed in Appendix A.

### 26.1 Core library

`libs/jumpstartcore` is a source-only Nim package (`jumpstartcore`, version
2.1.0, licensed LGPL-3.0-or-later, requiring only Nim 2.2.0 or later) holding the modules listed above. It
exists so that a core implemented in its own repository, such as the systemd
compatibility core, shares the daemon's implementation of the exec protocol,
the registry, the error codes, and the process, cgroup, and reaper layers
instead of copying them. The daemon, `jumpctl`, the Kongue runner, and the
tests import the same modules (`import jumpstartcore/protocol`); there is one
copy of the code.

**Boundary.** The library contains what a core's own processes (adapter,
supervisor, executor helper) and the daemon both need. It never imports
anything under `jumpstart/`. Discovery, the manifest loader, lifecycle,
sessions, routing, the control socket, the daemon-side adapter, supervisor and
backend drivers, logging, and the command line stay in `jumpstart/cores/`.
`CoreMode` is in the library (`coremode.nim`) so that `environment.nim` has no
dependency on `modes.nim`, which re-exports it.

**Stability.** The exec protocol codec (`protocol.nim`), the registry
(`registry.nim`, `registry_types.nim`), and the error codes (`errors.nim`) are
the stable contract: they implement §17, §12, and §18 and change only with
those sections. Every other module, including `adapterloop.nim` and
`jsonlines.nim`, is unstable until the 2.x line settles, and consumers pin an
exact version or commit.

**Adapter helper.** `adapterloop.nim` implements the adapter end of §17 for
cores written in Nim: the hello reply with the registered operations (§17.2),
request dispatch to per-operation handlers, `item` and `entry` events (§17.4),
`cancel` and deadline handling (§17.3), result validation with
`validateResult` before sending, exactly one `response` per request, and exit
on EOF of descriptor 3. A core that multiplexes requests keeps its own event
loop and uses `handshake`, `LineReader`, and the protocol constructors.

**Spawn and cgroup helpers.** A core's own processes are started with
`spawn.spawn` and live in cgroups made with `cgroups.nim`, which extend the
defaults of §15.1 and §15.4 without any knowledge of a particular manager:

- `ProcessSpec.explicitStdio: Option[array[3, cint]]` makes three of the
  caller's descriptors the child's 0, 1, and 2 (no pipes, `stdoutFd` and
  `stderrFd` are -1; exclusive with `nullStdio` and `inheritStdio`).
- `ProcessSpec.passFds: seq[PassedFd]`, `PassedFd = tuple[source, target:
  cint]`, places further descriptors at fixed numbers with `FD_CLOEXEC`
  cleared, and the child closes every other descriptor above 6 except the
  targets. Targets are 3 and 4 (only without `protocol`), or 7 to
  `MaxPassedFd` (1023); 0 to 2, 5, and 6 are refused, as are duplicates and
  invalid descriptors (`ValueError`). Every source is first copied to a
  number that is no target, so sources may alias targets; sources are not
  consumed. The
  descriptor rules are §15.4's. The rest of the spawn layer (the start gate,
  `cgroupProcs`, the verified executable, the environment) is unchanged.
- `cgroups.validUnitCgroupName` implements §15.1 rule 7. `unitCgroupName`, the
  name of a native unit's directory, is unchanged.
- `ensureNestedUnitCgroup(layout, parts, sys, enableLast = false)` creates or
  reuses `units/PART0/.../PARTn` (each part valid under rule 7, at most
  `MaxNestedUnitDepth` = 64 parts) and enables controllers in every directory
  but the last (also the last with `enableLast`, for a slice).
  `removeNestedUnitCgroup(layout, parts, sys)` removes the last directory and
  then each ancestor with `rmdir` only, stopping silently at the first ancestor
  still in use, and returns how many it removed; failing to remove the last
  one raises `CgroupError`. `killCgroup` and `removeCgroupTree` already take a
  whole subtree. `unitOfPid(layout, procCgroupText)` returns the chain of
  `.unit` components below `units/` for a process (`none` if it is not below
  this core's `units/`), as rule 9 describes, and `delegateDir` is exported for
  delegating one nested directory.
- `names.validUnitName` accepts a leading `-` (§14.1), and the registry and
  `protocol.validateResult` accept the state `reloading` and the optional
  `since`, `substate`, and `result` fields of every `status` result (§12.11).

**Consumption.** An external core repository depends on the package with a
nimble git requirement that selects the subdirectory,
`requires "https://github.com/kreatolinux/src?subdir=libs/jumpstartcore#<commit>"`,
or, for local development against a checkout of this repository, adds the
directory to the compiler path (`--path:"<checkout>/libs/jumpstartcore/src"` in
its `nim.cfg`) or runs `nimble develop`. This repository's `nim.cfg` adds the
same path for every project below the root.

## 27. Tests

All tests use temporary roots. They MUST NOT read or modify
`/usr/lib/jumpstart/cores`, `/etc/jumpstart`, `/var/lib/jumpstart`,
`/run/jumpstart`, `/sys/fs/cgroup`, or any real home directory. Tests that
need cgroups or namespaces run in a disposable container or VM.

### 27.1 Discovery and naming

- An absent root yields only `serviceHandler`.
- Arbitrary extension names are discovered without any allowlist.
- Ordering is native first, then byte order within each root.
- Installing and removing an extension changes results.
- Invalid and reserved names are rejected. A unit name MAY begin with `-`
  (`-.mount`, `-.slice`); it is accepted by validation, routing (qualified and
  unqualified), the name index, and the protocol.
- Non-directory entries and symlinks anywhere in the tree are rejected.
- Wrong owners and group- or world-writable entries are rejected anywhere in
  the tree, in the roots, and in their parents.
- Unexpected top-level entries are rejected.
- A file whose ownership changes after discovery is refused at execution.

### 27.2 Modes, isolation, and sessions

- `--user` as root, and system mode as non-root, exit with status 2.
- `~` comes from the password database even when `HOME` differs.
- System mode never opens a user path. Tests verify this with a user root that
  fails loudly if read.
- User mode discovers system cores that list `"user"`, plus user cores, in the
  specified order. A system core without `"user"` is invisible there.
- A user core without `"user"`, or with a name matching a system core, is
  rejected.
- In user mode, `user` and `account` are ignored and shown as such by
  `show --resolved`.
- The system socket is `0600` and rejects non-root peers. User sockets reject
  peers with a different UID.
- The environment matches §7.5 in both modes, including `jumpctl env`
  changes and their effect only on processes started afterwards.
- Session open and close start and stop user daemons with dropped
  credentials, a delegated cgroup, and a size-capped tmpfs on
  `/run/user/UID`. Sessions close when their login process exits. UID 0
  sessions are ignored. A pre-existing `/run/user/UID` that is not a Jumpstart
  mount fails the session start. Lingering keeps daemons running and starts
  them only after activation completes. Restarts follow §7.6 step 7.
- A hand-started user daemon loads only `serviceHandler`, and extension cores
  fail with `E_CGROUP`.
- Enabled sets in the two modes are independent.

### 27.3 Manifest

- Missing, malformed, and mismatched-name manifests are rejected.
- Missing, duplicate, unknown, and wrongly typed top-level fields are rejected.
- Manifest mode rejects style-folded keys, underscore qualifiers, `+:` and
  `-:`, unquoted strings, qualified `init`/`shutdown`, and `init_`/`shutdown_`
  helper names. Ordinary Kongue documents still parse as before.
- `manifest_version` other than 1 is rejected.
- Valid and invalid `core_version` and `jumpstart` ranges are handled.
  Incompatible ranges are rejected.
- Missing and duplicate `init` functions and duplicate `shutdown` functions
  are rejected.
- The structured-header parser handles nesting, typed scalars, tabs,
  misindentation, duplicate keys, and trailing comments.
- `activation` is required exactly when a unit-addressed subsystem is claimed,
  and the `unit.activate`/`unit.deactivate` rules for each strategy are
  enforced.
- `supervisor` keys are validated.
- Body variable references are exact, and collisions between `name qualifier`
  and `name_qualifier` functions are rejected.

### 27.4 Operations

- Unknown, reserved, malformed, and duplicate operation names are rejected.
- The completeness rule rejects a missing verb in a claimed subsystem and
  entries in an unclaimed subsystem.
- Claiming a unit-addressed subsystem without `unit.list`, `unit.resolve`,
  and its `status` is rejected.
- Each backend's required, optional, and forbidden keys are enforced,
  including `functions` only for `custom`.
- `sandbox` is required for read-only operations with `custom` or oneshot
  `exec`, and rejected elsewhere, including all persistent adapters.
  `writable_paths` and `network` are honored, and rejected without the
  sandbox.
- A missing `handler`, unknown roles, and undefined functions are rejected.
- `exec` paths outside `bin/`, missing executables, and non-executable files
  are rejected. Inconsistent persistent-adapter options are rejected.
  Persistent mode is rejected for `core.*`.
- `user` other than `"root"` or `account` is rejected in system mode.
- Delegation to an undiscovered, disabled, or unsupporting core is rejected,
  as are cycles and chains of more than four hops.
- Duration parsing and its limits are enforced.
- Reserved subsystem names are rejected.
- Every `status` result accepts the optional `since` (RFC 3339), `substate`,
  and `result` (1 to 64 bytes of lowercase letters, digits, and `-`) fields,
  and the state `reloading`; malformed values are `E_PROTOCOL`.
- `show --resolved` prints every default and marks it.

### 27.5 State

- Absent state yields `{serviceHandler}`.
- The first mutation materializes state atomically.
- `enable` and `disable` are idempotent and atomic per marker.
- Undiscovered or invalid cores cannot be enabled.
- The last enabled core and delegation targets cannot be disabled.
- An empty state directory and malformed or symlinked markers are rejected.
- An injected write failure leaves old or new state intact.
- Concurrent `jumpctl` invocations are locked.

### 27.6 Routing and dispatch

- Qualified and unqualified resolution behave as in §14.3:
  - ambiguity between native and extension units;
  - waiting while cores load;
  - name-index handling of failing and degraded cores; and
  - the separate `unit.resolve` budget.
- `jumpctl` sends unit names unmodified, and `serviceHandler` still accepts
  legacy suffixes.
- A verb that is unsupported or undefined for the subsystem returns
  `E_UNSUPPORTED` and never falls back.
- A `custom` handler runs in a runner process, fails on the first failing
  command, and validates `respond`, `respond-item`, `fail`, and type
  conversion.
- `spawn` passes arguments without a shell. `${spawn(…).exit()}` and
  `.output()` never fail the handler, and output beyond 1 MiB does.
- With the unit name `a",\x2d)` (and variants containing `", "`, `-`, and
  `)`), both `spawn` forms pass exactly one argument for the name, unchanged.
- Manifests with computed function or method names in expressions are
  rejected.
- With a `failed` or `unloaded` enabled extension whose index lists
  `sshd.service`, an unqualified `sshd.service` returns that core's error and
  never reaches `serviceHandler`. Without an index, a name `serviceHandler`
  matches only by stripping a legacy suffix fails the same way. Indexes of
  cores that are not enabled are ignored.
- A loading core's `init` that calls `jumpctl` with a name, qualified or
  not, does not deadlock.
- `exec` anywhere reachable from a handler is rejected, while `init` and
  `shutdown` may use it.
- `cd` and `env` in a handler do not affect the daemon.
- A oneshot `exec` adapter completes a handshake, request, and response.
- A persistent adapter multiplexes requests, restarts after failure, and
  makes the core degraded after exceeding its restart budget.
- The protocol rejects malformed JSON, unknown ids, oversize lines, and
  handshake mismatches. It accepts `item` events and notifications, and drops
  notifications over the rate limit.
- Timeouts send `SIGTERM`, then `SIGKILL`, to the operation's process group
  only. A daemon that called `setsid` survives.
- `sandbox: true` on an unsupported platform returns `E_SANDBOX` and never runs
  the operation. Sandboxed processes have no capabilities and are denied the
  listed syscalls.
- Unit, core, and log locks behave as in §11.6. A long `unit.activate` or
  `log.follow` never blocks `stop` or `status`.
- Concurrency and connection limits return `E_LIMIT` or queue as specified.

### 27.7 Processes

- Every core process starts in `control/` with stdin from `/dev/null`, logged
  stdout and stderr, descriptors 3 and 4 only where specified, and its own
  session.
- Supervisors start as child subreapers. A supervisor that does not complete
  its handshake in time fails the load.
- A non-root `account` can move processes from `control/` into `units/` but
  not into `control/`, and cannot change its own cgroup limits.
- The user daemon runs in `users/UID/daemon/`, and controllers are available
  to its cores.
- Cgroup directory names below `units/` without the `.unit` suffix are
  rejected, and names with backslash, `%`, `@`, and spaces are accepted, up to
  255 bytes (a table, including `foo@a\x2db.service`). Native names are
  unchanged.
- Nested unit cgroups: creation with controllers in each slice, `rmdir`-only
  bottom-up removal that stops at a busy ancestor, subtree kill, and
  `unitOfPid` (including sub-cgroups of a delegated service and a cgroup
  namespace prefix).
- `explicitStdio` and `passFds`: descriptors appear at their targets with
  `FD_CLOEXEC` cleared, sources are not consumed and may alias targets,
  everything else is closed, 3 and 4 are available only without `protocol`, and
  invalid targets and sources are refused before anything starts.
- A session whose peer PID was reused before `pidfd_open` is rejected. A
  hand-started user daemon holding the socket is reported and does not use
  up the restart budget.
- A supervisor exit makes the core degraded, and services under it survive
  adapter restarts.
- Orphans of cores without supervisors are reaped by Jumpstart, and no
  zombies remain.
- Unloading kills everything left in the core's cgroup.
- Native services run in per-unit cgroups, and native `stop` removes their
  whole process tree.

### 27.8 Daemon and live lifecycle

- Startup succeeds with only `serviceHandler`, and with invalid enabled state
  it loads exactly `serviceHandler` and reports `E_STATE`.
- With `mount_fstab = false`, `/` is not remounted and nothing from
  `/etc/fstab` is mounted.
- With `mount_fstab = false` and a read-only `/`, cores still load. No state
  directory or log file is written until a core sends `filesystems-ready`,
  or until every core has finished activation. Nothing is written into a
  directory that a later mount shadows. Buffered log lines are flushed in
  order, with a marker if any were dropped.
- `filesystems-ready` is honoured once in system mode and ignored in user
  mode.
- An unload of a degraded core is refused while its `units/` cgroup is
  populated, unless stopping is requested.
- User daemon shutdown stops within 25 seconds.
- The socket accepts requests while cores are still loading.
- A missing, invalid, or failing core becomes `failed` while the other cores
  load. Persistent state is unchanged.
- Delegation targets load before their dependents, and a dependent of a failed
  core fails.
- Loading a core at runtime does not restart, pause, or reorder other cores,
  and a failed load at any step rolls back fully.
- Activation runs asynchronously, by strategy, and its failures do not unload
  the core.
- Unload refuses with `E_BUSY` when units are active, stops them by strategy
  with `--stop-units`, refuses to break delegations or unload the only core,
  closes the core before stopping units, and cancels streams immediately.
- A core whose only active units are `mount` or `swap` units is not busy, a
  service next to them is, and a `reloading` unit (of any subsystem) is.
- A degraded core unloads and reloads using last-known states.
- Reload picks up changed files, clears staleness, and leaves the core
  `failed` when the new version is invalid.
- Requests routed during each state get the specified error.
- System shutdown follows §20.8 for every trigger, within its limits, and
  always reaches `reboot(2)`. User daemon shutdown follows §20.9.
- Existing service, mount, and timer behavior through `serviceHandler` is
  unchanged, apart from the fixes in Appendix A.

### 27.9 Native targets

- Target decfiles, `wants`, `--target` enablement links, the built-in
  `default` target, and the reserved `default.kg` name work as specified.
- The boot target comes from `default-target`, or is `default` when the file
  is absent.
- Target cycles are reported.
- Starting a target starts `static` members and their `depends` closure.
- An existing `default.kg` is reported and not loaded.
- Target status follows §22.5: a target whose oneshot members exited 0 is
  `active`, a started empty target is `active`, a target still starting
  members is `activating`, and any failed member makes it `failed`.
- Enabling a sub-unit returns `E_UNSUPPORTED`.
- `isolate` stops exactly the non-members.
- Multi-unit decfiles behave as targets.
- Enablement values follow §22.2.

### 27.10 CLI

- Exact output and exit status for every command in §21.1.
- Help is available at the top level and in every nested command.
- Unknown options and subcommands, and mode-specific commands in the wrong
  mode, produce exit status 2.
- Local commands never connect to the socket.
- Existing `start`, `stop`, `enable`, `disable`, and `status` keep their output
  text and now report failures. Extension `status` appends `since`,
  `substate`, and `result` lines in that order when present, and native output
  is byte-identical.
- A unit name that begins with `-` is accepted after `--` by every command that
  takes a `UNIT` and is `E_USAGE` before it, qualified or not as specified in
  §21.1.
- `enable --now` and `disable --now` persist only after the live change
  succeeds.
- Socket requests and responses work, including cancellation by closing the
  connection.
- `cores run` (§21.15):
  - `UNIT` is required for `unit` operations, optional for `log.*`, and
    rejected for `core` and `global` operations. An argument with `=` is never
    the `UNIT`, `CORE/UNIT` and invalid unit names are `E_USAGE`, and usage
    errors start no process and print no warning;
  - `KEY=VALUE` conversion follows the registry, and unknown keys, duplicate
    keys, and bad values are `E_USAGE`;
  - a `custom` entry runs in the runner; a `oneshot` and a `persistent` exec
    entry each run through a real adapter, and the persistent adapter
    completes the handshake with the declared operation set, answers one
    request, and has ended when the command ends; a drifted operation set is
    `E_PROTOCOL`;
  - `UNIT` reaches the adapter exactly as typed, with the environment of
    §15.4, §7.5, and §16.2;
  - `unit.list` `item` events and log `entry` events print as they arrive in
    the line formats of §21.4 and §21.5 without `CORE/`, are not printed twice,
    and an invalid item is not printed and fails with `E_PROTOCOL`;
  - an invalid result is `E_PROTOCOL`; an adapter error keeps its code;
  - a `delegate` chain is followed, a cycle, a chain over four hops, and a
    missing target are `E_DELEGATION`, and the target need not be enabled;
  - `unsupported`, undeclared, `native`, and `serviceHandler` operations are
    `E_UNSUPPORTED` (exit status 3) with the message of §21.15 for the native
    ones;
  - the entry's `timeout` ends the run with `E_TIMEOUT`, and `SIGINT` cancels
    it with `E_CANCELLED` (exit status 4), except that it ends `log.follow`
    normally;
  - the warning is printed for mutating operations and for no others;
  - a missing runtime directory exists for the run and is removed afterwards,
    and an existing one is left untouched;
  - the core need not be enabled or loaded, and no socket is contacted.

## 28. Acceptance criteria

The core implementation is complete when:

1. A stock installation lists only `serviceHandler`, which is enabled by
   default without a manifest on disk, and boots exactly as before.
2. No systemd, dinit, OpenRC, or other manager name or behavior is compiled
   into the generic subsystem.
3. Any valid extension appears in `jumpctl cores list` once it is installed,
   without restarting the daemon or querying a package manager.
4. New extensions are disabled by default.
5. Enabled state persists atomically and is never empty.
6. Every operation Jumpstart performs for a core matches that core's resolved
   manifest, and `jumpctl cores show --resolved` displays it exactly.
7. Unsupported operations fail with `E_UNSUPPORTED` and never fall back.
8. Unit names are core-defined. Qualified and unqualified routing work through
   `unit.resolve`, including template-style names that no file matches.
9. The `custom`, `exec` (both modes), `delegate`, and `unsupported` backends
   work as specified, with timeouts, cancellation, and fail-closed sandboxing.
10. Every core process the daemon starts runs in its core's cgroup, is
    reaped, and ends when the core unloads. (`jumpctl cores run` is the
    exception: it runs one operation of a core without the daemon, in its own
    cgroup and for debugging, §21.15.) Supervisors keep services alive across
    adapter restarts.
11. Any single core can be loaded, unloaded, or reloaded in a running daemon
    without restarting the others, and activation follows each core's declared
    strategy.
12. User-mode cores work from the user core root, are unreachable from the
    system daemon, and user daemons start and stop with sessions.
13. The system shuts down through the sequence in §20.8.
14. `serviceHandler` supports native targets (§22).
15. The commands in §21 behave as specified. Existing `jumpctl` service
    commands keep their output for native units.
16. The tests in §27 pass.

## 29. Open questions

1. **Unit handover on reload.** Reloading a core with running units currently
   requires stopping them (§20.5). Should a later revision let a reloaded core
   re-adopt its running units, so that upgrades do not interrupt services? The
   supervisor (§15.2) already outlives adapters. Letting it also outlive a
   reload would need an explicit handover step in the protocol.
2. **Core hooks for power events.** SysV `inittab` (`ctrlaltdel`,
   `powerfail`) and runit stage 3 let the manager react to Ctrl-Alt-Del,
   power failure, and shutdown in its own way. In `manifest_version: 1`,
   triggers are native (§20.8), and `unit.deactivate` during unload is a
   core's only shutdown hook. Should a later version let a core handle power
   triggers itself?

---

## Appendix A: `serviceHandler` resolved manifest

`serviceHandler` has no file on disk. Its manifest is compiled in and is shown
by `jumpctl cores show serviceHandler` as:

```kongue
manifest_version: 1
name: "serviceHandler"
description: "Jumpstart native service handler"
core_version: "1.3.0"  # tracks jumpstartVersion
version: "1.3.0"
jumpstart: "=1.3.0"
modes:
  - "system"
  - "user"

subsystems:
  - "unit"
  - "service"
  - "mount"
  - "timer"
  - "target"
activation: "core"

supported_ops:
  - name: "unit.list"
    backend: "native"
  - name: "unit.resolve"
    backend: "native"
  - name: "unit.load"
    backend: "native"
  - name: "unit.validate"
    backend: "native"
  - name: "unit.translate"
    backend: "unsupported"
  - name: "unit.activate"
    backend: "native"
  - name: "unit.deactivate"
    backend: "native"
  - name: "service.start"
    backend: "native"
  - name: "service.stop"
    backend: "native"
  - name: "service.restart"
    backend: "native"
  - name: "service.reload"
    backend: "unsupported"
  - name: "service.kill"
    backend: "native"
  - name: "service.enable"
    backend: "native"
  - name: "service.disable"
    backend: "native"
  - name: "service.status"
    backend: "native"
  - name: "mount.start"
    backend: "native"
  - name: "mount.stop"
    backend: "native"
  - name: "mount.enable"
    backend: "native"
  - name: "mount.disable"
    backend: "native"
  - name: "mount.status"
    backend: "native"
  - name: "timer.start"
    backend: "native"
  - name: "timer.stop"
    backend: "native"
  - name: "timer.enable"
    backend: "native"
  - name: "timer.disable"
    backend: "native"
  - name: "timer.status"
    backend: "native"
  - name: "target.start"
    backend: "native"
  - name: "target.stop"
    backend: "native"
  - name: "target.isolate"
    backend: "native"
  - name: "target.enable"
    backend: "native"
  - name: "target.disable"
    backend: "native"
  - name: "target.status"
    backend: "native"
  - name: "target.set-default"
    backend: "native"
  - name: "target.get-default"
    backend: "native"
```

The versions shown are for the first release that implements this
specification. In practice `core_version`, `version`, and `jumpstart` all equal
the running Jumpstart version.

**Native semantics:**

- `unit.resolve` accepts a decfile name with an optional `::` sub-unit and the
  legacy suffixes `.kg`, `.service`, `.mount`, and `.timer` (§14.1). It
  returns the normalized name and the subsystem from the decfile `type`, and
  tells the router internally whether the match relied on stripping a legacy
  suffix (§14.3). A
  `multi` decfile resolves to `target` (§22.4), and the built-in `default`
  target always resolves.
- `unit.list` lists the decfiles in the mode's config directory and the
  built-in `default` target, with enablement as in §22.2.
- `unit.load` rereads and reparses every decfile and reports parse errors and
  target cycles. It starts and stops nothing.
- `unit.validate` parses one decfile with `configParser` and reports its
  errors.
- `unit.activate`, `unit.deactivate`, and `target.*` behave as in §22.5.
- `service.restart` is `service.stop` followed by `service.start`.
- `service.kill` signals every process in the unit's cgroup (§15.1). Without
  cgroup v2, it signals the service's main process only. It never signals a
  process group, because `terminal: true` services share PID 1's process
  group.
- `unit.translate` is unsupported because native units are already decfiles.
  `service.reload` is unsupported because decfiles have no reload command.

**Status mapping.** Native status files map to §12.11 states as follows:

| Native status file | State | Extra fields |
| --- | --- | --- |
| `running` | `active` | `main_pid` |
| `mounted` | `active` | — |
| `stopped` | `inactive` | — |
| `stopped with an exit code N`, after a stop that Jumpstart requested | `inactive` | `exit_code: N` |
| `stopped with an exit code N`, otherwise | `failed` | `exit_code: N` |
| file absent | `inactive` | — |
| target or multi unit: `running` | `active` | `members` |
| target or multi unit: `stopped` | `inactive` | `members` |
| target or multi unit: `starting` | `activating` | `members`, `detail` |
| target or multi unit: `failed` | `failed` | `members`, `detail` |
| anything else | `unknown` | `detail` holds the raw text |

No native status maps to `reloading`, and native results never carry
`since`, `substate`, or `result` (§12.11).

A requested stop usually ends with signal exit code 143 (SIGTERM), which is
why the requested-stop row must not be read as a failure. `serviceHandler`
MUST record, in memory, whether the last exit of each service followed a
requested stop.

**Fixes this specification depends on.** Restart cannot work until these
existing defects are fixed:

- `stopService` does not remove the service's runtime directory, and
  `startService` refuses to start while that directory exists ("already
  running"), so start-after-stop fails. The fix is that start tolerates a
  stale runtime directory, and stop keeps the status file so the
  requested-stop mapping below keeps working;
- the loop in `stop.nim` runs one index past the end of `services`; and
- native code waits on its own processes rather than through the central
  reaper (§15.3):
  - `status.nim` waits on the main process and on `execPost`;
  - `start.nim` waits on `execPre`;
  - `stop.nim` polls with `running()`, which calls `waitpid`; and
  - `main.nim` runs `mount -a` through `execDirectWait`.

## Appendix B: example systemd manifest (illustrative)

This appendix is non-normative. A real systemd core will be specified
separately.

In this example, `systemd-compat-manager` is the supervisor: it owns every
service process and tracks systemd's dependency graph. `systemd-compat` is a
persistent adapter that answers requests by talking to the manager through a
socket in the core's runtime directory. Unit names are systemd's own (such as
`sshd.service` and `getty@tty1.service`), and the core reads them from
systemd's unit directories for the current mode (`/etc/systemd/system` and
friends in system mode, `~/.config/systemd/user` and friends in user mode).

```kongue
manifest_version: 1
name: "systemd"
description: "systemd unit compatibility"
core_version: "0.1.0"
version: "255"
jumpstart: ">=1.3.0, <2.0.0"
modes:
  - "system"
  - "user"

subsystems:
  - "core"
  - "unit"
  - "service"
  - "mount"
  - "timer"
  - "target"
  - "log"
activation: "core"

supervisor:
  executable: "bin/systemd-compat-manager"
  ready_timeout: "30s"

supported_ops:
  # Managed payload
  - name: "core.verify"
    backend: "custom"
    options:
      sandbox: true
      timeout: "10s"
    functions:
      - handler: "verify_payload"

  # Unit definitions and activation
  - name: "unit.list"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "10s"
  - name: "unit.resolve"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "2s"
  - name: "unit.load"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "30s"
  - name: "unit.validate"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "10s"
  - name: "unit.translate"
    backend: "unsupported"
  - name: "unit.activate"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "10m"
  - name: "unit.deactivate"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "5m"

  # Services
  - name: "service.start"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "90s"
  # service.stop, restart, reload, kill, enable, disable, status: same shape
  # as service.start with their own timeouts (elided here, but REQUIRED in a
  # real manifest by the completeness rule)

  # Mounts, timers, and targets: same shape, elided
  # (mount.*, timer.*, target.start/stop/isolate/enable/disable/status/
  #  set-default/get-default)

  # Journal
  - name: "log.read"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "30s"
  - name: "log.follow"
    backend: "exec"
    options:
      executable: "bin/systemd-compat"
      mode: "persistent"
      timeout: "24h"
  - name: "log.write"
    backend: "unsupported"
  - name: "log.rotate"
    backend: "unsupported"
  - name: "log.vacuum"
    backend: "unsupported"

init {
  print "systemd core ${core.core_version} targeting systemd ${core.version}"
}

func verify_payload {
  if "${spawn(\"/usr/bin/test\", \"-x\", \"/usr/lib/systemd/systemd-executor\").exit()}" == "0" {
    respond installed true
    respond version "$version"
  } else {
    respond installed false
    respond version ""
  }
}
```

## Appendix C: manifest grammar (informative)

This is a summary of the structured-header subset (§10.2) that manifests use,
under manifest mode (§10.3). `kongue(5)` is normative once it is extended.

```text
header      = *( field / comment-line / NL )
field       = key ":" SP value [ comment ] NL
            / key ":" [ comment ] NL INDENT block DEDENT
block       = mapping / sequence
mapping     = 1*( key ":" ( SP value [ comment ] NL
                          / [ comment ] NL INDENT block DEDENT ) )
sequence    = 1*( "-" SP ( value [ comment ] NL / inline-map ) )
inline-map  = key ":" ( SP value [ comment ] NL / [ comment ] NL INDENT block DEDENT )
              *( ALIGN key ":" ( SP value [ comment ] NL / [ comment ] NL INDENT block DEDENT ) )
value       = quoted-string / "true" / "false" / 1*DIGIT
key         = ALPHA *( ALPHA / DIGIT / "_" )
comment     = SP "#" *( any character except NL )
comment-line = *SP "#" *( any character except NL ) NL
```

`INDENT` and `DEDENT` are increases and decreases in leading spaces. `ALIGN`
returns to the column of the first key after `- `. Keys are compared exactly
(§10.3).

## Appendix D: mapping other service managers (informative)

This appendix sketches how common managers fit the model. It is not
normative, and each core will need its own specification.

| Manager | Unit names (§14.1) | Processes (§15) | Activation (§11.5) | Targets and runlevels (§12.7) |
| --- | --- | --- | --- | --- |
| systemd | `sshd.service`, `getty@tty1.service`, aliases; read from systemd unit directories, resolved on demand | Compatibility manager as supervisor | `core`: reach `default.target` | `.target` units; `isolate` changes target; `set-default` |
| OpenRC | Service names from `/etc/init.d` | No supervisor needed: scripts daemonize with `start-stop-daemon` (which calls `setsid`), or use `supervise-daemon`; Jumpstart reaps | `core`: run `sysinit`, `boot`, then the default runlevel | Runlevels from `/etc/runlevels/*` as target units; `enable --target boot` maps to `rc-update add … boot` |
| dinit | Service names from its service directories | `dinit` itself (or a wrapper) as supervisor | `core`: start the `boot` service | Services with `waits-for`/`depends-on` act as targets |
| runit | Service directory names under `/etc/sv` | `runsvdir` as supervisor | `per-unit` or `core` via the active service directory | Runsvdirs (runlevels) as targets; `isolate` switches runsvdir |
| s6 / s6-rc | Service names from the compiled s6-rc database | `s6-svscan` as supervisor; readiness through the s6 notification descriptor | `core`: `s6-rc` bundle for boot | s6-rc bundles as targets |
| SysV init | Script names in `/etc/init.d`, plus `inittab` entries (`generated`) | Core supervisor respawns `inittab` entries; scripts daemonize with `setsid` | `core`: run `rcN.d` `S` scripts in order | Runlevels `0`–`6` and `S` as targets; `isolate` is `telinit N` |
| launchd-style | The job `Label` inside each plist, not the file name | Core supervisor | `core`: load jobs with `RunAtLoad` | None; `target` subsystem not claimed |

## Appendix E: revision history

### Revision 8

Resolutions of the points where the systemd compatibility core's specification
(its section 16.2) found this one too narrow, plus the generic library
extensions that core needs. `manifest_version: 1` has not shipped, so §23.3
does not require a new manifest version for the additive changes. Each item
says whether it only clarifies the specification or changes behavior.

1. **`unit.deactivate` and mounts** (behavior change). A core MAY leave `mount`
   and `swap` units `active` after `unit.deactivate` (§12.2), and the unload
   busy check ignores `active` units of those subsystems but counts
   `activating`, `deactivating`, and `reloading` ones (§20.4); per-unit stopping
   (§11.5, §20.4 step 2) leaves them alone as well. `serviceHandler` is
   unchanged (§22.5): its native units, mounts included, still count and are
   still stopped. Code: `lifecycle.nim`, `daemon.nim`.
2. **cgroup names** (behavior change). §15.1 rule 7 now accepts any name of 6
   to 255 bytes that ends in `.unit` and has no `/`, NUL, control character,
   `.` or `..`, so backslash and `%` are valid; the mapping from unit names
   MUST be injective. `serviceHandler`'s native names are unchanged and still
   reject a backslash. Code: `cgroups.validUnitCgroupName`; the path checks no
   longer treat `\` as a separator.
3. **User environment changes** (clarification, §7.5). A running supervisor or
   persistent adapter sees `jumpctl env` changes after a reload; a core's own
   command (such as `import-environment`) may push variables into it and
   bypasses the store.
4. **Leading `-` in unit names** (behavior change). §14.1 no longer forbids it;
   `jumpctl` accepts such a name only after `--` (§21.1), and unknown options
   before `--` remain `E_USAGE`. cligen already ends option parsing at `--` in
   every command, so only validation changed (`names.validUnitName`). §16.4
   and §24 now require handlers to pass `--`.
5. **State vocabulary** (behavior change). New state `reloading` (§12.11), with
   the busy, deactivation, activation, and notification rules stated there,
   and the optional `since`, `substate`, and `result` fields of every `status`
   result. `jumpctl status` prints them after the unchanged first line (§21.2).
   Code: `protocol.nim`, `registry.nim`, `registry_types.nim`, `supervisor.nim`,
   `types.nim`, `lifecycle.nim`, `daemon.nim`, `cli.nim`. Native behavior and
   output are unchanged (Appendix A).
6. **Compatibility commands** (clarification, §4.2). A shim MAY talk to the
   core's own socket, and then bypasses Jumpstart's locks, admission limits,
   and operation log; the core provides equivalent behavior.
7. **Reserved subsystems** (clarification, §12.13). `slice`, `scope`, and
   `device` stay reserved; a core MAY model a slice as a `target` unit; scopes
   and devices are not addressable.
8. **Library extensions** (behavior change, additive; library 2.1.0). §15.4
   and §26.1: `explicitStdio` and `passFds` in `spawn.nim` (protocol descriptors
   only with `protocol`, `FD_CLOEXEC` cleared on exactly the passed targets,
   everything else closed); §15.1 rule 9 and §26.1: `ensureNestedUnitCgroup`,
   `removeNestedUnitCgroup`, `unitOfPid`, and the exported `delegateDir` in
   `cgroups.nim`. The sandbox and the other modules are unchanged.
9. **Other notes of that section** (clarifications; no behavior changes):
   an admitted mutating operation holds its admission slot until it returns,
   is cancelled, or times out (§19.3); a persistent adapter's read-only
   operations run unsandboxed (§13.1, already required); a supervisor MAY send
   `filesystems-ready` (§17.5, already allowed); and the supervisor
   handshake's parent side is Jumpstart's while the core implements the child
   side (§15.2). A core that maps its own reloading state to `active` (as the
   systemd core does) remains conforming (§12.11).
- Other sections were adjusted to match: §4.2, §7.5, §11.5, §11.6, §12, §12.2,
  §12.11, §12.13, §14.1, §15.1, §15.4, §16.4, §20.4, §20.6, §21.1, §21.2,
  §22.5, §24, §26, §26.1, §27, and Appendix A.

### Revision 7

The code an external core implementation needs moved into a reusable library,
so the systemd compatibility core and the daemon share one copy. No behavior
changed.

- **Library:** `libs/jumpstartcore` holds `errors`, `names`, `registry`,
  `registry_types`, `protocol`, `cgroups`, `reaper`, `spawn`, `sandbox`,
  `environment`, `peercred`, and `verified_file`, moved unchanged from
  `jumpstart/cores/`, plus `coremode` (the `CoreMode` type, split out of
  `modes.nim`) and the new `jsonlines` and `adapterloop` (§26.1). The
  `readLineFd` and `writeAllFd` helpers of the Kongue runner moved into
  `jsonlines`.
- **Stability:** §17, §12, and §18 as implemented by `protocol`, `registry`,
  and `errors` are the library's stable contract; the rest is unstable until
  2.x settles.
- **Build:** `nim.cfg` at the repository root adds the library to the compiler
  path. `sh build.sh -T jumpstartcore` runs the library's tests, which moved
  from `tests/jumpstart/cores/` to `tests/jumpstartcore/`.
- Other sections were adjusted to match: §26 only.

### Revision 6

`jumpctl cores run` runs any operation, for debugging a core's adapter or
handlers without going through the daemon (§21.15). Until now it accepted only
`core.*` operations, so the only way to see what an adapter received and
answered for `unit.list`, `service.status`, or `service.start` was to load the
core in a daemon and read its log.

- **Syntax:** `jumpctl cores run CORE OPERATION [UNIT] [KEY=VALUE…]`. `UNIT`
  is required for unit-addressed operations, optional for `log.*`, and
  rejected for the others. It is the core-local name, passed exactly as typed:
  no `unit.resolve`, and `CORE/UNIT` is `E_USAGE`.
- **Semantics:** the same backend layer as the daemon, in a private
  transient runtime: `custom` in a runner, `oneshot` and `persistent` exec
  adapters started by `jumpctl` itself (a persistent adapter is closed
  again after the single request), `delegate` followed with the §13.4 limits,
  and `unsupported` and `native` entries (and `serviceHandler`) refused with
  `E_UNSUPPORTED`. The entry's timeout, the sandbox, and cancellation by
  `SIGINT` apply.
- **Output:** result fields as before, with `item` and `entry` events printed
  as they arrive in the line formats of `units list` and `logs read`.
- **Safeguards:** mutating operations print a warning that locking,
  supervision, and daemon state are bypassed. §21.15 lists what is bypassed,
  and §24 states that no privilege is added.
- **Runtime directory:** `run` provides the core's runtime directory for the
  command when no daemon has created it, and never touches an existing one.
- **Unchanged:** the core still needs only to be discovered and valid, the
  command still never contacts a socket, and `core.*` operations remain local
  only (§12.1). `core.verify` prints exactly what it printed before.
- Other sections were adjusted to match: §12.1, §13.3, §17.6, §21.1, §24,
  §26, §27.10, and §28.

### Revision 5

Clarifications made while implementing this specification; no behavior was
added.

- **Sandbox `working_directory` (§13.1):** the default `"/"` is accepted with
  `sandbox: true`, as Appendix B uses it. An explicit `working_directory`
  under `sandbox: true`, including an explicit `"/"`, MUST be within a
  writable path.
- **Runner log (§16.3, §17.4):** `log LEVEL MESSAGE` in a runner is reported
  as an `entry` event whose `data` is `{time, level, unit, message}`.
  Jumpstart logs it with the request id and never adds it to the result.
- **§7.6 step 4.5:** the system daemon itself never connects to a user
  socket. The readiness wait is a probe made by a short-lived child that has
  already dropped to the user's credentials.
- **§8.3 ordering:** when a manifest both requires an incompatible Jumpstart
  version and lacks a mode, `E_VERSION` is reported first. The checks run in
  this order: version range, `name`, user mode of a user-root manifest,
  referenced paths.
- **Packaging (§25):** the `Makefile` defaults `LIBDIR` to `$(PREFIX)/lib`.
  The paths in this specification assume `LIBDIR=/usr/lib`. `LIBDIR` is
  compiled into the binaries.

### Revision 4

Fixes from the third independent review:

- **Early boot without `mount_fstab`:** defined when filesystems are ready
  (§20.2), with a new `filesystems-ready` message and Kongue command. State
  directories are created lazily and never fail a load, nothing is written
  into a directory a later mount would shadow, and the daemon log is
  buffered in memory until it can be written.
- **Routing:** name indexes persist under `/run`, and enabled cores that are
  `failed` or `unloaded` cannot answer, so their names never fall through to
  `serviceHandler` via a legacy suffix. Requests from a loading core's own
  processes no longer wait for that core.
- **Kongue:** `spawn` arguments are parsed before interpolation and never
  re-tokenized. The multi-argument call grammar is listed as a `kongue(5)`
  dependency, and computed function names are rejected. `init` and
  `shutdown` should not pass handler-written data to `exec`.
- **Targets:** oneshot members that exited 0 count as satisfied, empty
  started targets are `active`, and partly started targets are
  `activating`. Sub-units cannot be enabled on their own.
- **Processes and sessions:** cgroup directories below `units/` use the
  fixed `.unit` suffix. Session pidfds avoid PID reuse, and hand-started
  user daemons are handled.

### Revision 3

- Unit names are now defined by each core. Jumpstart treats them as opaque
  and routes through the new `unit.resolve` operation, and the declarative
  `units` source list is removed. `jumpctl` no longer normalizes names.
- Added process ownership:
  - per-core cgroup v2 subtrees;
  - optional supervisors, which make themselves subreapers;
  - a central reaper;
  - fixed descriptors and stdio, with the protocol moved to descriptors 3
    and 4; and
  - notifications from adapters and supervisors.
- Added activation strategies (`core` and `per-unit`), with `unit.activate`
  and `unit.deactivate`. Activation now runs asynchronously after the socket
  opens.
- Added enablement states, a `target` argument on `enable`/`disable`,
  `target.enable`, `target.disable`, `target.set-default`, and
  `target.get-default`. Runlevels are modelled as targets. Added native
  targets to `serviceHandler` (§22).
- Added user daemon sessions with a PAM module and lingering, a defined
  operation environment with a user environment store, and the
  `jumpctl users` and `jumpctl env` command groups.
- Defined the system and user daemon shutdown sequences, with
  `jumpctl poweroff`, `reboot`, and `halt`.
- Tightened mode selection, moved the user socket, made the system socket
  root-only, and specified socket robustness and resource limits.
- Required recursive ownership checks and descriptor-based execution of core
  files.
- Restricted the sandbox to read-only operations, and made it drop all
  capabilities and apply a seccomp filter.
- Moved all Kongue execution into a separate runner process. Added
  `respond-item` and `spawn`, typed result conversion, and a static
  shell-injection check.
- Defined Kongue manifest mode: exact keys, no underscore qualifiers, no
  `+:`/`-:`, and typed scalars only in manifests.
- Read-only and streaming operations are no longer serialized with mutations.
- Invalid enabled state now loads exactly `serviceHandler`.
- Degraded cores can be unloaded and reloaded.
- Unload closes the core before stopping units, and load rollback covers every
  step.
- Added the `account` and `supervisor` manifest fields. Added `E_CORE_STATE`,
  `E_UNIT_STATE`, `E_CGROUP`, `E_LIMIT`, and `E_SHUTDOWN`, and renamed
  `E_CORE_DISABLED` to `E_CORE_UNLOADED`.
- Corrected statements about existing code: the mode flag, `status` being
  client-side, the native status mapping, the restart defects, and the
  description of `initSystem`.
- Compatibility commands are an explicit non-goal.
- Added Appendix D.
- Changes made after an independent review of the first draft of this
  revision:
  - **Routing:** unqualified routing waits for loading cores, and a per-core
    name index stops degraded or failing cores from causing misrouting.
  - **Kongue handlers:** `custom` handlers may not use `exec` at all. Added
    the `${spawn(…).exit()}` and `.output()` expression forms. Body variable
    references are exact in manifest mode, and qualified/underscore function
    collisions are rejected.
  - **Cgroups:**
    - non-root accounts get `cores/CORE/` delegated;
    - every process lives in a leaf cgroup, and the user daemon runs in
      `users/UID/daemon/`;
    - cgroup2 is mounted with `nsdelegate`, and v1/hybrid hierarchies are left
      alone;
    - Jumpstart sets supervisors' subreaper attribute itself.
  - **Sessions:**
    - `/run/user/UID` is a size-capped tmpfs that is unmounted, never deleted
      recursively;
    - sessions follow the login process's pidfd;
    - UID 0 sessions are ignored;
    - restart rules and linger timing are defined.
  - **Locks and unload:** split into unit, core, and log locks, and activation
    takes none. Unload busy checks use the `units/` cgroup.
  - **Timeouts:** user daemon shutdown fits inside the system daemon's 30
    seconds, and oneshot adapter exit waits are bounded.
  - **Native boot:** `mount_fstab` in `main.conf` lets cores do their own
    early mounting. Native targets start static members and their
    dependencies. Target status text is defined, and there is a migration note
    for an existing `default.kg`.
  - **Code fixes:** completed the list (all `waitForExit` users, the `osproc`
    spawn layer, the `SIGINT` hook).

### Revision 2

- `supported_ops` changed from a list of strings to structured operation
  entries with `backend`, `options`, and `functions`.
- Added qualified operation names (`subsystem.verb`), a full operation
  registry, the completeness rule, and explicit `unsupported`.
- Added backends (`custom`, `exec`, `delegate`, `unsupported`, `native`) with
  complete option tables and no implementation-defined defaults.
- Added `manifest_version`, `jumpstart`, `modes`, `subsystems`, and
  `description`, plus an optional `shutdown` function.
- Added system and user operating modes, with user cores wholly isolated from
  the system daemon.
- Replaced whole-daemon core initialization with per-core load, unload, and
  reload in a running daemon.
- Added the `swap` and `path` subsystems, explained the reserved subsystem
  names, and added the `writable_paths` and `network` sandbox options.
- Added the control socket protocol, the exec protocol, the error model, and
  per-core directories.
- Added the `restart`, `reload`, `kill`, and `isolate` verbs, and the `units`,
  `logs`, and `cores` command groups.
- Made package management, core-file upgrades (owned by kpkg), and cross-core
  ordering explicit non-goals.
