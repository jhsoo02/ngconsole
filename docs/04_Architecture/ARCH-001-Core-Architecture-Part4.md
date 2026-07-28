# ARCH-001 — NGConsole Core Architecture
## Part 4: Connection Manager & SSHv2 Architecture

**Project:** NGConsole  
**Document ID:** ARCH-001  
**Part:** 4 of 6  
**Version:** 1.0  
**Status:** Baseline  
**Target Platform:** macOS  
**Language:** Swift  
**UI:** SwiftUI

---

## 1. Purpose

Part 4 defines the architecture of the NGConsole Connection Manager and its transport implementations.

The most important requirement is reliable SSHv2 connectivity.

NGConsole is primarily intended to connect to:

- Extreme Networks switches
- Linux-based systems
- XIQ-SE systems
- Other network devices and servers exposing SSH
- Serial console devices where applicable
- Telnet devices where required

The Connection Manager must provide a clean boundary between:

```text
Connection Protocol
        |
        v
Terminal Transport
        |
        v
Console Engine
```

The Console Engine must not contain SSH-specific logic.

---

# 2. Design Goals

The Connection Manager must provide:

- SSHv2
- Telnet
- Serial Console
- Local Shell
- Connection lifecycle management
- Authentication
- Host-key verification
- PTY allocation
- Terminal resize
- Keepalive
- Clean disconnect
- Reconnect support
- Error propagation
- Credential isolation
- Testable transport interfaces

The initial implementation should prioritize:

1. SSHv2
2. Local Shell
3. Serial Console
4. Telnet

SFTP, FTP and TFTP are separate transfer functions and must not be implemented inside the Console Transport.

---

# 3. Critical Architecture Rule

The following separation is mandatory:

```text
+------------------------------------------------------+
|                    SwiftUI UI                         |
+------------------------------------------------------+
|                 Session ViewModel                     |
+------------------------------------------------------+
|                Connection Manager                    |
+------------------------------------------------------+
|                 Terminal Transport                   |
+------------------------------------------------------+
| SSH | Telnet | Serial | Local Process                |
+------------------------------------------------------+
|                  Console Engine                      |
+------------------------------------------------------+
```

More precisely:

```text
Remote Device
     |
     v
SSH / Telnet / Serial / Local Process
     |
     v
Transport
     |
     +---- bytes ---->
     |
     v
Console Engine
```

The transport is responsible for moving bytes.

The Console Engine is responsible for interpreting terminal bytes.

---

# 4. Transport Protocol

A common protocol must be defined before implementing SSH.

Conceptual interface:

```swift
public protocol TerminalTransport: Sendable {
    func connect() async throws
    func disconnect() async
    func send(_ data: Data) async throws
    func resize(columns: Int, rows: Int) async throws
    func receive() -> AsyncThrowingStream<Data, Error>
}
```

The exact API may evolve, but the following principles are mandatory:

- No SSH types in the Console Engine
- No SwiftUI types in the transport
- No UI state in SSH implementation
- Network I/O must not run on the main actor
- Transport must be mockable

---

# 5. Connection Lifecycle

Every connection follows a state machine.

```text
                 +---------+
                 |  Idle   |
                 +----+----+
                      |
                      | connect()
                      v
                 +---------+
                 |Connecting|
                 +----+----+
                      |
              +-------+-------+
              |               |
           success           error
              |               |
              v               v
        +-----------+     +---------+
        | Authenticating|  | Failed |
        +------+----+     +---------+
               |
           success
               |
               v
        +-------------+
        | Establishing|
        |    PTY      |
        +------+------+
               |
               v
        +-------------+
        |  Connected  |
        +------+------+ 
               |
       +-------+--------+
       |                |
   disconnect         error
       |                |
       v                v
+-------------+    +---------+
| Disconnecting|   | Failed |
+------+------+
       |
       v
     Idle
```

The actual implementation should represent states explicitly.

Example:

```swift
public enum ConnectionState: Sendable {
    case idle
    case connecting
    case authenticating
    case establishingSession
    case connected
    case disconnecting
    case failed(ConnectionError)
}
```

---

# 6. Connection Configuration

A connection configuration should contain only connection parameters.

```swift
public struct ConnectionConfiguration: Sendable {
    public var host: String
    public var port: Int
    public var username: String
    public var authentication: AuthenticationMethod
    public var terminal: TerminalConfiguration
}
```

Do not store plaintext passwords inside general workspace configuration.

Secrets must be handled separately.

---

# 7. Authentication Model

Authentication must be abstracted.

```swift
public enum AuthenticationMethod: Sendable {
    case password
    case publicKey
    case keyboardInteractive
    case agent
}
```

The initial implementation should prioritize:

1. Password
2. Public key
3. Keyboard-interactive

Additional authentication methods can be added without changing the Console Engine.

---

# 8. Password Handling

Passwords must not be stored in:

- SwiftUI state
- Connection logs
- Terminal logs
- Workspace JSON
- Debug output

Passwords should be stored in the macOS Keychain when persistence is explicitly requested.

Conceptual flow:

```text
User
 |
 v
Connection Dialog
 |
 v
Credential Provider
 |
 v
macOS Keychain
 |
 v
SSH Authentication
```

The SSH implementation should receive the secret only when needed.

---

# 9. Keychain Boundary

Credential storage should be abstracted.

```swift
public protocol CredentialStore: Sendable {
    func save(_ credential: Credential) async throws
    func load(_ identifier: CredentialIdentifier) async throws -> Credential?
    func delete(_ identifier: CredentialIdentifier) async throws
}
```

The macOS implementation can use Keychain Services.

The rest of NGConsole must not depend directly on Keychain APIs.

This allows unit tests to use an in-memory credential store.

---

# 10. SSHv2 Architecture

The SSH implementation should be layered.

```text
+---------------------------------------------------+
|              SSH Transport Adapter                |
+---------------------------------------------------+
|              SSH Session                         |
+---------------------------------------------------+
| Authentication | Channel | PTY | Window Resize   |
+---------------------------------------------------+
|              SSH Library                          |
+---------------------------------------------------+
| TCP / Network                                     |
+---------------------------------------------------+
```

NGConsole should not implement cryptographic primitives itself.

Use a mature SSH implementation/library rather than writing:

- SSH encryption
- Key exchange
- MAC
- Public-key algorithms
- Host-key algorithms
- Cipher implementations

from scratch.

---

# 11. SSH Library Boundary

The external SSH implementation should be isolated behind an adapter.

```text
NGConsole
    |
    v
SSHTransport
    |
    v
SSHAdapter
    |
    v
SSH Library
```

This is important because:

- SSH libraries may change
- macOS compatibility may change
- Testing should not depend on a real server
- Future replacement should not affect the Console Engine

---

# 12. SSH Connection Flow

The SSH connection flow is:

```text
1. Resolve Host
       |
2. Open TCP Connection
       |
3. SSH Protocol Identification
       |
4. Key Exchange
       |
5. Host-Key Verification
       |
6. User Authentication
       |
7. Open Session Channel
       |
8. Request PTY
       |
9. Request Shell
       |
10. Connected
```

NGConsole must treat each step as a separate stage.

---

# 13. Host-Key Verification

Host-key verification is mandatory.

The client must not silently accept every host key.

Conceptual flow:

```text
Server Host Key
      |
      v
Known Hosts Store
      |
      +---- Match --------> Continue
      |
      +---- Unknown -------> Ask User
      |
      +---- Mismatch ------> Security Warning / Reject
```

A host-key mismatch must never be silently ignored.

---

# 14. Known Hosts

NGConsole should maintain a known-hosts abstraction.

```swift
public protocol HostKeyStore: Sendable {
    func lookup(host: String, port: Int) async throws -> StoredHostKey?
    func store(_ key: StoredHostKey) async throws
    func remove(host: String, port: Int) async throws
}
```

The initial implementation can support an OpenSSH-compatible known-hosts format where practical.

The storage mechanism must be isolated from the SSH adapter.

---

# 15. Host-Key Trust Dialog

When a host is unknown, the UI should show enough information for the user to make a decision.

Example:

```text
New SSH Host

Host:
10.10.10.10

Port:
22

Host Key:
ED25519

Fingerprint:
SHA256:xxxxxxxxxxxxxxxx

[Cancel] [Trust Once] [Trust & Save]
```

The UI must not automatically trust the host.

---

# 16. Host-Key Mismatch

If the server presents a different key than the stored key:

```text
WARNING:
REMOTE HOST IDENTIFICATION HAS CHANGED
```

The connection should stop unless the user explicitly resolves the conflict.

This protects against accidental or malicious host-key changes.

---

# 17. SSH Session Channel

After authentication, the client opens a session channel.

Conceptual flow:

```text
SSH Connection
      |
      v
Authenticated Session
      |
      v
Open Session Channel
      |
      v
Request PTY
      |
      v
Request Shell
```

The session channel is the source of terminal data.

---

# 18. PTY Allocation

A terminal application must request a PTY.

Typical configuration:

```text
Terminal:
xterm-256color

Columns:
120

Rows:
40
```

The terminal type should be configurable.

Default recommendation:

```text
xterm-256color
```

The implementation must verify compatibility with the target device.

---

# 19. PTY Request

Conceptual API:

```swift
public struct PTYConfiguration: Sendable {
    public var terminalType: String
    public var columns: Int
    public var rows: Int
}
```

The SSH adapter uses this to request the remote PTY.

The Console Engine should not know how the PTY request is encoded.

---

# 20. Shell Request

After PTY allocation:

```text
PTY Request
    |
    v
Shell Request
    |
    v
Remote Shell / CLI
```

The shell must not be requested before the PTY is established when terminal behavior depends on PTY allocation.

---

# 21. Extreme Networks CLI Considerations

NGConsole is primarily intended for network-device access.

For Extreme Networks devices, SSH sessions may present:

- CLI prompts
- ANSI formatting
- Paging
- Command output
- Configuration output
- Logs
- Routing information
- Fabric/IS-IS related information

The Connection Manager should not implement Extreme-specific parsing.

Device-specific analysis belongs in future plugin modules.

---

# 22. Linux / XIQ-SE Considerations

Linux-based systems may provide:

- Bash
- sh
- zsh
- Other shells
- Interactive applications

The same SSH transport must work without special-case Linux code.

The Console Engine handles terminal behavior.

---

# 23. Terminal Transport Data Flow

Incoming data:

```text
Remote Device
     |
     v
SSH Channel
     |
     v
SSHTransport
     |
     v
AsyncThrowingStream<Data>
     |
     v
Console Engine
```

Outgoing data:

```text
Keyboard
     |
     v
Console Input Encoder
     |
     v
TerminalTransport.send()
     |
     v
SSH Channel
     |
     v
Remote Device
```

---

# 24. Receive Loop

The receive loop must be asynchronous.

Conceptual implementation:

```swift
for try await data in sshChannel.receiveStream {
    await terminalEngine.receive(data)
}
```

The receive loop must not execute on the main actor.

---

# 25. Send Path

Keyboard input should follow:

```text
SwiftUI
  |
  v
Terminal Input Handler
  |
  v
TerminalTransport.send()
  |
  v
SSH Channel
```

The SSH implementation should not interpret terminal keyboard semantics.

The Console Engine/Input layer determines what bytes to send.

---

# 26. Terminal Resize

When the window changes:

```text
SwiftUI Terminal View
       |
       v
columns / rows
       |
       v
Session Manager
       |
       v
SSHTransport.resize()
       |
       v
SSH PTY Window Change
```

The resize request must update:

- Columns
- Rows
- Pixel width where supported
- Pixel height where supported

The initial implementation can prioritize columns and rows.

---

# 27. Resize Debouncing

Window resizing can generate many events.

Do not send:

```text
resize
resize
resize
resize
resize
...
```

for every layout update.

Instead:

```text
Resize Events
      |
      v
Debounce / Coalesce
      |
      v
Latest Size
      |
      v
SSH PTY Resize
```

This reduces unnecessary network operations.

---

# 28. Keepalive

Long-running network sessions need keepalive support.

Keepalive should be configurable.

Conceptual configuration:

```swift
public struct KeepAliveConfiguration: Sendable {
    public var enabled: Bool
    public var interval: TimeInterval
    public var maxFailures: Int
}
```

Keepalive must distinguish:

- Application-level keepalive
- SSH-level keepalive
- TCP keepalive

The SSH library should handle protocol-specific mechanisms where possible.

---

# 29. Connection Timeout

Connection stages need timeout handling.

At minimum:

```text
TCP Connect Timeout
SSH Handshake Timeout
Authentication Timeout
PTY Establishment Timeout
```

A timeout must result in a clear error.

Example:

```swift
enum ConnectionError: Error {
    case timeout(stage: ConnectionStage)
}
```

---

# 30. Error Model

Errors should be categorized.

```swift
public enum ConnectionError: Error, Sendable {
    case invalidConfiguration
    case dnsFailure
    case networkFailure
    case connectionRefused
    case timeout(stage: ConnectionStage)
    case hostKeyUnknown
    case hostKeyMismatch
    case authenticationFailed
    case authenticationCancelled
    case channelOpenFailed
    case ptyRequestFailed
    case shellRequestFailed
    case disconnected
    case protocolError
    case unknown(String)
}
```

The UI should display user-friendly descriptions while retaining structured error information for diagnostics.

---

# 31. Disconnect

A clean disconnect should follow:

```text
User selects Disconnect
        |
        v
Stop Input
        |
        v
Close Shell / Channel
        |
        v
Close SSH Session
        |
        v
Close Network Connection
        |
        v
Release Resources
        |
        v
Idle
```

Disconnect must be idempotent.

Calling disconnect twice must not crash.

---

# 32. Unexpected Disconnect

If the remote system closes the connection:

```text
SSH Channel Closed
      |
      v
Transport detects closure
      |
      v
Connection State = Disconnected
      |
      v
UI notification
```

The terminal contents should normally remain visible after disconnect.

This allows the user to inspect the last output.

---

# 33. Reconnect

Reconnect should be a Session Manager function rather than an SSH protocol function.

```text
Session
  |
  v
Disconnected
  |
  v
Reconnect
  |
  v
Connection Manager
  |
  v
SSHTransport
```

The initial version can provide explicit user-triggered reconnect.

Automatic reconnect can be added later.

---

# 34. Serial Console Transport

Serial Console should use the same transport abstraction.

```text
Serial Port
    |
    v
SerialTransport
    |
    v
TerminalTransport
    |
    v
Console Engine
```

Serial-specific configuration:

```text
Device
Baud Rate
Data Bits
Parity
Stop Bits
Flow Control
```

Serial transport must not affect Console Engine design.

---

# 35. Telnet Transport

Telnet should also use the same abstraction.

```text
Telnet Server
     |
     v
TelnetTransport
     |
     v
TerminalTransport
     |
     v
Console Engine
```

Telnet negotiation belongs inside TelnetTransport.

The Console Engine must receive only terminal data.

---

# 36. Local Shell Transport

macOS local shell support can use a process-backed transport.

```text
Swift Process
     |
     v
LocalShellTransport
     |
     v
TerminalTransport
     |
     v
Console Engine
```

The same terminal engine can therefore display:

- SSH shell
- Telnet session
- Serial console
- Local shell

without knowing the source.

---

# 37. Mock Transport

Every transport should be replaceable by a mock.

```swift
public final class MockTerminalTransport {
    ...
}
```

Example test:

```text
Mock Transport
     |
     | "show version
"
     v
Console Engine
     |
     v
Expected Terminal State
```

This allows Console Engine development without requiring a physical switch or SSH server.

---

# 38. SSH Test Server

The project should maintain a controlled SSH test environment.

Possible targets:

```text
macOS Local SSH
Linux VM
Docker SSH server
Test network device
```

Tests should cover:

- Password login
- Public-key login
- Invalid password
- Unknown host key
- Host-key mismatch
- PTY allocation
- Resize
- Long-running session
- Large output
- Remote disconnect
- Reconnect

---

# 39. SSH Integration Test Sequence

A basic integration test should verify:

```text
1. Connect
2. Verify host key
3. Authenticate
4. Open channel
5. Request PTY
6. Request shell
7. Receive prompt
8. Send command
9. Receive output
10. Resize
11. Send another command
12. Disconnect
```

A failure at any stage must produce a structured error.

---

# 40. SSH Debug Logging

SSH debugging must be available without exposing secrets.

Safe to log:

```text
Connection started
TCP connected
SSH handshake completed
Host key verified
Authentication method selected
Authentication succeeded
PTY requested
Shell requested
Disconnected
```

Never log:

- Passwords
- Private key material
- Authentication tokens
- Complete terminal output by default

---

# 41. Security Boundary

The following must never be mixed:

```text
Credential
Terminal Output
Connection Metadata
AI Context
```

For example:

```text
SSH Password
   X
   |
   +----> Terminal Engine
   |
   +----> AI Plugin
```

The password must never enter either component.

---

# 42. SSH and AI Boundary

AI functionality is not part of the Connection Manager.

Future AI functionality may receive user-selected output:

```text
SSH Session
     |
     v
Console
     |
     v
User Selects Output
     |
     v
AI Plugin
```

The Connection Manager must never automatically send:

- Passwords
- SSH keys
- Credentials
- Complete terminal streams

to an AI provider.

---

# 43. Session Object

A session ties together the transport and Console Engine.

Conceptual model:

```swift
public protocol TerminalSession: Sendable {
    var id: UUID { get }
    var state: ConnectionState { get async }
    func connect() async throws
    func disconnect() async
    func sendInput(_ data: Data) async throws
    func resize(columns: Int, rows: Int) async throws
}
```

Internally:

```text
TerminalSession
   |
   +--> TerminalTransport
   |
   +--> TerminalEngine
   |
   +--> Session Metadata
```

---

# 44. Session Manager

The Session Manager manages active sessions.

```text
Session Manager
 |
 +--> Session 1 --> SSH --> Switch
 |
 +--> Session 2 --> SSH --> XIQ-SE
 |
 +--> Session 3 --> Serial --> Device
 |
 +--> Session 4 --> Local Shell
```

The Session Manager should support multiple simultaneous sessions.

The first version does not require a device tree.

---

# 45. Multi-Session Isolation

Each session must have independent:

- Terminal state
- Scrollback
- Connection
- Authentication state
- Selection
- Search state
- Theme/profile reference

A failure in one session must not corrupt another session.

---

# 46. Connection Manager and Workspace

Workspace stores connection metadata, not live transport objects.

Example:

```text
Workspace
 |
 +--> Connection Profile
 |       |
 |       +--> Host
 |       +--> Port
 |       +--> Username
 |       +--> Protocol
 |       +--> Terminal Profile
 |
 +--> Runtime Session
         |
         +--> SSH Transport
         +--> Console Engine
```

Workspace data must be serializable.

Runtime objects must not be serialized.

---

# 47. Connection Profile

Conceptual model:

```swift
public struct ConnectionProfile: Codable, Sendable {
    public var id: UUID
    public var name: String
    public var protocolType: ConnectionProtocol
    public var host: String
    public var port: Int
    public var username: String
    public var credentialIdentifier: String?
    public var terminalProfileID: String
}
```

Passwords must not be stored in this structure.

---

# 48. Plugin Boundary

The Connection Manager should allow future transport plugins without making the Core dependent on them.

Potential future transports:

```text
Core
 |
 +--> SSH
 +--> Telnet
 +--> Serial
 +--> Local Shell
 |
 +--> Plugin Transport
```

The plugin mechanism should be defined later.

The Core implementation should not be delayed by building a full plugin marketplace or plugin management system.

---

# 49. Recommended Implementation Strategy

The SSH implementation should be built in the following order.

### Step 1 — Transport Protocol

Implement:

```swift
TerminalTransport
```

and a `MockTerminalTransport`.

### Step 2 — Local Shell

Implement:

```text
LocalShellTransport
```

This validates the Console Engine without network complexity.

### Step 3 — SSH Adapter

Integrate a mature SSH library behind:

```text
SSHTransport
```

### Step 4 — Authentication

Implement:

- Password
- Public key
- Keyboard-interactive

### Step 5 — Host-Key Verification

Implement:

- Known host lookup
- Unknown host confirmation
- Host-key mismatch handling

### Step 6 — PTY

Implement:

- PTY allocation
- Terminal type
- Columns
- Rows

### Step 7 — Shell

Implement interactive shell request.

### Step 8 — Resize

Connect Console View size to PTY resize.

### Step 9 — Keepalive

Add configurable keepalive.

### Step 10 — Integration Tests

Test against:

- macOS SSH server
- Linux SSH server
- Extreme switch
- XIQ-SE Linux system

---

# 50. Common SSH Implementation Mistakes

The following mistakes must be explicitly avoided.

### Mistake 1

Implementing SSH directly inside SwiftUI.

Incorrect:

```text
SwiftUI View
   |
   +--> SSH Code
```

Correct:

```text
SwiftUI
   |
Session Manager
   |
SSHTransport
   |
SSH Library
```

### Mistake 2

Treating TCP packets as terminal messages.

Incorrect:

```text
read() == command
```

Correct:

```text
byte stream
    |
    v
SSH channel
    |
    v
terminal byte stream
    |
    v
ANSI parser
```

### Mistake 3

Assuming a complete ANSI sequence arrives in one read.

The parser must support fragmented sequences.

### Mistake 4

Skipping host-key verification.

This creates a serious security weakness.

### Mistake 5

Running network I/O on the MainActor.

Network and SSH processing must remain asynchronous and off the UI path.

### Mistake 6

Using SwiftUI text as the terminal state.

The terminal engine must own terminal state.

### Mistake 7

Building a custom SSH implementation.

Do not implement cryptography or the SSH protocol from scratch.

### Mistake 8

Mixing SFTP with the terminal channel.

SFTP is a separate subsystem.

---

# 51. Definition of Done

Part 4 is complete when:

- `TerminalTransport` is defined.
- SSH is isolated behind `SSHTransport`.
- SSH uses a mature SSH implementation/library.
- Authentication is separated from terminal rendering.
- Host-key verification is mandatory.
- Passwords are isolated from workspace and terminal state.
- PTY allocation works.
- Shell request works.
- Terminal resize works.
- Keepalive is configurable.
- Clean disconnect works.
- Unexpected disconnect is handled.
- Connection errors are structured.
- Mock transport exists.
- SSH integration tests exist.
- Local Shell can reuse the same Console Engine.
- Telnet and Serial can reuse the same transport interface.
- AI has no direct access to credentials or the complete SSH stream.
- The architecture remains compatible with future plugins.

---

# 52. Next Part

**ARCH-001 Part 5 — Transfer Manager & File Services Architecture**

will define:

- SFTP Client
- SFTP Server
- FTP Client
- FTP Server
- TFTP Client
- TFTP Server
- Local File Browser
- File transfer queues
- Progress reporting
- Cancellation
- Resume
- Permissions
- Security boundaries
- Transfer Manager abstraction
- Integration with Workspace
- Future plugin boundary
