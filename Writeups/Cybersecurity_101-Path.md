                                                       TryHackMe — Cyber Security 101 Path

## Module 1: Start Your Journey

### Useful Information Websites
- **Shodan** — search engine for IoT and internet-connected devices
- **VirusTotal** — compiles results of multiple antivirus scanners for a specific file
- **CVE Database** — list of known vulnerabilities and attacks

## Module 2: Linux Fundamentals

### Linux Fundamentals Part 1
- Connecting to a cloud machine using `ssh`
- Basic commands: `echo`, `whoami`
- Navigation: `cd`, `ls`, `pwd`, `cat`
- File searching with `find` and `grep`
- Shell operators:
  - `&` — run in background
  - `&&` — run next command only if previous succeeds
  - `>` — redirect output (overwrite)
  - `>>` — redirect output (append)

### Linux Fundamentals Part 2
- File system interaction: `mkdir`, `rmdir`, `cp`, `mv`
- Creating and removing files: `touch`, `rm`
- User switching and permissions: `chmod`, `su`

### Linux Fundamentals Part 3
- Built-in text editors: `nano`, `vim`
- `wget` to download files from a URL (and key flags like `-q`, `-O`)
- `scp` (secure copy) to transfer files between host and remote servers
- Process management:
  - `ps` to view processes
  - `systemctl` to manage services
  - Backgrounding with `&`, foregrounding with `fg`
- Automation of the terminal with `crontab`:
  - `crontab -e` to edit, `-l` to list, `-r` to remove
  - Format: `MIN HOUR DOM MON DOW CMD`
- Package management with `apt`
- Accessing server logs (e.g., `/var/log/apache2/`)

## Module 3: Windows and AD Fundamentals

### Windows Fundamentals 1  
- Layout of the windows GUI system
- File systems used (NFTS, FAT, HPFS)
- %windir% and System32 files and their importance
- Administrator and User account privileges
- User Account Control (UAC) and what it does
- Layout of control panel
- Task manager and its various functions

### Windows Fundamentals 2

**MSConfig**
  - General tab: Normal / Diagnostic / Selective boot modes
  - Boot tab: boot options, safe mode config
  - Services tab: all background services (running/stopped)
  - Startup tab
  - Tools tab: shortcut launcher for admin utilities

**Advanced System Settings**
  - Page file (pagefile.sys): hard drive used as overflow RAM 
  - Crash dumps: RAM snapshots

**Computer Management (compmgmt.msc)**
  - Task Scheduler (taskschd.msc) — schedule automated tasks
  - Event Viewer (eventvwr.msc) — system/security/application logs
  - Shared Folders — network shares including hidden defaults (C$, ADMIN$, IPC$)
  - Local Users and Groups (lusrmgr.msc) — user/group management
  - Performance Monitor (perfmon) — real-time system metrics
  - Device Manager (devmgmt.msc) — hardware management
  - Disk Management (diskmgmt.msc) — partition/drive admin
  - Services (services.msc) — manage background services
  - WMI — scripting interface for managing Windows remotely

**Other tools covered**
  - msinfo32 — system information panel (hardware, software, components)
  - Resource Monitor (resmon) — detailed CPU/memory/disk/network usage
  - Command Prompt (cmd) — Windows CLI
  - Registry Editor (regedit) — database of all Windows configuration settings
