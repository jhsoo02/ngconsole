# ARCH-001 — NGConsole Core Architecture
## Part 1: Architecture Foundation

**Project:** NGConsole  
**Document ID:** ARCH-001  
**Part:** 1 of 6  
**Version:** 1.0  
**Status:** Baseline  
**Target:** macOS  
**Language:** Swift  
**UI:** SwiftUI

---

## 1. Purpose

This document defines the architectural foundation of NGConsole.

NGConsole is a native macOS application intended to combine the daily tools most useful to a network engineer:

- SSHv2
- Telnet
- Serial Console
- Local Shell
- SFTP Client / Server
- FTP Client / Server
- TFTP Client / Server
- Python Script Execution
- Workspace
- Future optional plugins

The central architectural principle is:

> **Build a small, solid core. Everything else is optional.**

The initial product must remain practical and achievable. Future capabilities such as AI Agent, vendor-specific analysis, inventory, configuration history, and advanced automation must be attachable without making them Core dependencies.

---

## 2. Product Direction

NGConsole v1 is not a network management platform.

The primary objective is:

> **SecureCRT + SFTP/FTP/TFTP + Python + Workspace in one native macOS application.**

The following are explicitly outside the initial Core scope:

- Device inventory
- Device tree
- Customer/site/rack hierarchy
- Live configuration history
- Network monitoring
- Configuration management database
- Cloud management
- AI Agent
- Vendor-specific network analysis

These may be implemented later as optional modules or plugins.

---

## 3. Architecture Principles

### 3.1 Small Core

Core contains only functionality required for the primary NGConsole product.

A feature must not be added to Core merely because it may be useful in the future.

### 3.2 Stable Interfaces

Core exposes stable interfaces at boundaries where future functionality is expected.

Implementations may evolve without requiring consumers to change.

### 3.3 Plugin-Oriented Extension

Optional functionality should be implemented outside Core whenever practical.

Potential future plugins include:

- AI Assistant
- AI Agent
- Extreme Networks helper
- Linux helper
- Cisco helper
- Git integration
- Configuration analysis
- Device inventory
- Advanced automation

### 3.4 Core Independence

Core must not depend on a particular vendor, AI provider, cloud service, or external management platform.

### 3.5 Native macOS

Preferred technologies:

- Swift
- SwiftUI
- Swift Concurrency
- Swift Package Manager
- Apple Keychain
- macOS system APIs where appropriate

### 3.6 Security First

Credentials, private keys, session data, and server-side file-transfer services must be designed with security as a primary concern.

---

## 4. High-Level Architecture

```text
+------------------------------------------------------------+
|                       NGConsole App                        |
+------------------------------------------------------------+
|                         SwiftUI                            |
|                 Application / Presentation                 |
+------------------------------------------------------------+
|                       Core Services                        |
|                                                            |
| Workspace Manager     Session Manager     Settings        |
| Connection Manager    Transfer Manager    Python Manager |
| Plugin Manager        Logging / Diagnostics               |
+------------------------------------------------------------+
|                     Console Engine                        |
|                                                            |
| Terminal State | ANSI Parser | VT Parser | Scrollback    |
| Input Handling | Selection   | Clipboard | Rendering      |
+------------------------------------------------------------+
|                    Protocol Drivers                        |
|                                                            |
| SSHv2 | Telnet | Serial | Local Shell                       |
| SFTP  | FTP    | TFTP                                    |
+------------------------------------------------------------+
|                  macOS / System Services                   |
|                                                            |
| Network | File System | Keychain | Process | PTY | TTY   |
+------------------------------------------------------------+

                    Optional Extensions
                             |
                             v
+------------------------------------------------------------+
|                       Plugin SDK                           |
+------------------------------------------------------------+
| AI | AI Agent | Extreme | Linux | Git | Future Extensions |
+------------------------------------------------------------+
```

---

## 5. Core Modules

| Module | Primary Responsibility |
|---|---|
| Console Engine | Terminal emulation and terminal state |
| Connection Manager | Connection lifecycle and protocol drivers |
| Transfer Manager | File-transfer clients and servers |
| Session Manager | Saved connection/session definitions |
| Workspace Manager | Workspace persistence and organization |
| Python Manager | Python execution and automation |
| Plugin Manager | Loading and managing optional extensions |
| Settings Manager | Application and terminal preferences |

These modules form the initial architectural boundary.

---

## 6. Console Engine

The Console Engine is the heart of the terminal experience.

### Responsibilities

- Terminal screen state
- ANSI escape-sequence processing
- VT-style terminal behavior
- Cursor state
- Character attributes
- Scrollback buffer
- Selection
- Copy/paste
- Search
- Highlighting
- Terminal resizing
- Keyboard input translation
- Output rendering
- Terminal color handling

### Non-Responsibilities

The Console Engine must not:

- Open SSH connections
- Implement Telnet
- Open serial ports
- Execute Python
- Transfer files
- Store workspace data
- Call AI services

The Console Engine receives terminal input/output through defined interfaces.

```text
Protocol Driver
      |
      | bytes / terminal events
      v
Console Engine
      |
      | terminal state
      v
SwiftUI Terminal View
```

This separation allows the same Console Engine to work with SSH, Telnet, Serial, and Local Shell.

---

## 7. Connection Manager

The Connection Manager owns terminal connection lifecycle.

### Supported Connection Types

- SSHv2
- Telnet
- Serial
- Local Shell

### Responsibilities

- Create connection
- Open connection
- Authenticate
- Connect terminal to Console Engine
- Disconnect
- Reconnect
- Connection state
- Connection errors
- Connection timeout
- Keepalive
- Terminal resize propagation

### Connection Lifecycle

```text
Created
   |
   v
Connecting
   |
   +----> Failed
   |
   v
Authenticating
   |
   +----> Authentication Failed
   |
   v
Connected
   |
   v
Interactive
   |
   +----> Reconnecting
   |
   v
Disconnected
```

The Connection Manager knows about protocol drivers. The Console Engine does not.

---

## 8. Protocol Drivers

Protocol drivers provide the actual communication mechanism.

### SSH Driver

Responsible for:

- SSH version 2
- Authentication
- Password authentication
- Public-key authentication
- Host-key verification
- PTY allocation
- Shell channel
- Terminal resize
- Keepalive
- Disconnect handling

SFTP functionality remains under Transfer Manager rather than being embedded into the terminal engine.

### Telnet Driver

Responsible for:

- TCP connection
- Telnet negotiation
- Input/output stream
- Disconnect handling

### Serial Driver

Responsible for:

- Serial port enumeration
- Port opening
- Baud rate
- Data bits
- Stop bits
- Parity
- Flow control
- Input/output streams

### Local Shell Driver

Responsible for:

- Local process creation
- PTY allocation where required
- stdin/stdout/stderr handling
- Process lifecycle
- Environment handling
- Working directory

---

## 9. Transfer Manager

The Transfer Manager is independent from the Console Engine.

### Client Features

- SFTP Client
- FTP Client
- TFTP Client

### Server Features

- SFTP Server
- FTP Server
- TFTP Server

### Responsibilities

- Connection lifecycle
- Directory listing
- Upload
- Download
- File deletion
- Rename
- Directory creation
- Transfer progress
- Transfer cancellation
- Transfer errors
- Server lifecycle
- Server configuration

```text
Console
   |
   +---- Connection Manager
   |
   +---- Console Engine

File Transfer
   |
   +---- Transfer Manager
           |
           +---- SFTP
           +---- FTP
           +---- TFTP
```

A terminal session must not be required to perform a file transfer.

---

## 10. Session Manager

The Session Manager represents saved connection definitions.

A session may contain:

- Name
- Protocol
- Host
- Port
- Username
- Authentication method
- Serial settings
- Terminal settings
- Connection options

Sensitive values must not be stored as plain text when secure system storage is available.

Passwords and private credentials should use macOS Keychain.

```text
Session
├── id
├── name
├── protocol
├── host
├── port
├── username
├── authentication
└── terminalProfile
```

The Session Manager does not own terminal rendering.

---

## 11. Workspace Manager

Workspace is deliberately simple in v1.

It is not a device inventory or network management hierarchy.

### Workspace Contents

```text
Workspace
├── Sessions
├── Scripts
├── Logs
├── Downloads
├── Uploads
├── Bookmarks
└── workspace metadata
```

### Responsibilities

- Create workspace
- Open workspace
- Close workspace
- Save workspace
- Load workspace
- Manage workspace-relative paths
- Manage workspace preferences
- Restore session/layout state where supported

### Explicitly Excluded

The Workspace Manager does not manage:

- Customers
- Sites
- Racks
- Devices
- Device inventory
- Network topology

Those concepts may be added later outside Core.

---

## 12. Python Manager

The Python Manager provides lightweight automation.

### v1 Responsibilities

- Select Python script
- Execute script
- Pass arguments
- Capture stdout
- Capture stderr
- Display exit code
- Stop running script
- Store scripts in Workspace

### Future Extension Points

Possible future capabilities:

- Script scheduler
- Macro recorder
- Session automation
- Device command automation
- Python API for NGConsole
- AI-generated scripts

These are not required for the initial implementation.

---

## 13. Plugin Manager

The Plugin Manager provides the extension boundary.

### Core Rule

> **Core must work correctly when zero plugins are installed.**

A plugin is optional.

### Future Plugin Examples

```text
plugins/
├── AI
├── AI-Agent
├── Extreme
├── Linux
├── Git
├── Inventory
└── Future
```

The initial release does not need to implement these plugins. It only needs to preserve a clean extension boundary.

---

## 14. Settings Manager

The Settings Manager stores application preferences.

### Required Settings

- Theme
- Background color
- Font family
- Font size
- Text color
- ANSI color scheme
- Highlight color
- Keyboard shortcuts
- Terminal behavior
- Scrollback size
- Default transfer directory
- Python preferences

### Theme Model

```text
Background
Foreground
Cursor
Selection
ANSI 16 colors
Search highlight
Match highlight
Error/output emphasis
```

The terminal color system should not be hard-coded into SwiftUI views.

---

## 15. Dependency Direction

The dependency direction must remain controlled.

```text
                    Plugin
                       |
                       v
+------------------------------------------------+
|                    Core                        |
|                                                |
| Workspace  Session  Settings  Python  Plugin   |
|       \        |       |        |       /      |
|            Connection / Transfer               |
|                     |                          |
|              Console Engine                   |
+------------------------------------------------+
                       |
                       v
                macOS Services
```

### Mandatory Rules

1. Core must not import optional plugins.
2. Plugins may depend on public Core interfaces.
3. Console Engine must not depend on SSH implementation.
4. Console Engine must not depend on SFTP/FTP/TFTP.
5. Connection Manager owns terminal transport drivers.
6. Transfer Manager owns file-transfer drivers.
7. Workspace Manager owns workspace persistence.
8. Session Manager owns session definitions.
9. Settings Manager owns preferences.
10. UI must not directly implement protocol logic.

---

## 16. Application Layer

SwiftUI views should not contain networking or protocol implementation.

Recommended flow:

```text
SwiftUI View
     |
     v
ViewModel / Application State
     |
     v
Core Service
     |
     v
Driver / Engine
```

This prevents UI code from becoming tightly coupled to protocol implementations.

---

## 17. Error Boundary

Errors should be represented at module boundaries.

Examples:

```text
ConnectionError
├── invalidAddress
├── timeout
├── authenticationFailed
├── hostKeyRejected
├── transportFailure
└── disconnected

TransferError
├── permissionDenied
├── fileNotFound
├── connectionFailed
├── transferCancelled
└── protocolFailure

PythonError
├── interpreterNotFound
├── executionFailed
├── scriptNotFound
└── cancelled
```

UI should display user-friendly descriptions while retaining structured error information for logging and diagnostics.

---

## 18. Logging Boundary

NGConsole should provide centralized application logging.

Logs should distinguish:

- Application
- Connection
- Terminal
- Transfer
- Python
- Plugin
- Security
- Error

Sensitive data must never be logged.

Never log:

- Passwords
- Private keys
- Keychain secrets
- Authentication tokens

---

## 19. Architecture Baseline

The following decisions are baseline for NGConsole v1:

- Native macOS
- Swift / SwiftUI
- Small Core
- SSHv2, Telnet, Serial and Local Shell in Core
- SFTP/FTP/TFTP functionality in Transfer Manager
- Python in Core
- Workspace in Core
- Plugin boundary in Core
- AI is not a Core dependency
- Device inventory is not Core
- Device tree is not Core
- Live configuration history is not Core
- Vendor-specific functionality is not Core

Changes to these decisions should be documented as an ADR.

---

## 20. Next Part

**ARCH-001 Part 2** will define:

- Module dependency graph
- Swift Package structure
- Directory structure
- Public protocols
- Event model
- Thread and concurrency model
- Data flow
- Connection lifecycle
- Terminal I/O pipeline
- Transfer lifecycle
- Plugin lifecycle
- Error propagation
- Logging architecture
- Mermaid architecture diagrams
