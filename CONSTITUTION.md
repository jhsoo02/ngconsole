# NGConsole Constitution

Version: 1.0.0 (Draft)

## Vision

**NGConsole is a native macOS Network Engineering Workspace.**

NGConsole is not merely an SSH client. Its mission is to provide a unified workspace for network engineers by integrating terminal access, file transfer, automation, and AI assistance into a single application.

## Core Principles

### 1. Console First
The terminal emulator is the heart of the application.
All transports (SSH, Telnet, Serial, Local Shell) connect to the Console Engine.

### 2. Native macOS
- Swift
- SwiftUI
- Swift Concurrency
- Swift Package Manager
- Apple Keychain

### 3. Modular Architecture
All protocol implementations and extensions must be modular.
AI must remain an optional plugin.

### 4. Offline First
Core functionality must not depend on cloud services.

### 5. Workspace Driven
A Workspace stores:
- Sessions
- Layouts
- Logs
- Downloads
- Uploads
- Scripts
- Reports
- Bookmarks
- History

### 6. Security
- Store credentials in Apple Keychain.
- Never store plaintext passwords.
- Encrypt sensitive configuration.

### 7. Production Quality
No placeholder code, demo-only implementations, or unfinished TODOs in released code.

## Initial Product Scope

### Console
- SSH v2
- Telnet
- Serial Console
- Local Shell

### File Transfer
- SFTP Client/Server
- FTP Client/Server
- TFTP Client/Server

### Automation
- Python
- Macros
- Scheduler

### AI
Optional assistant for:
- CLI explanation
- Output summarization
- Report generation
- Script generation

## Long-term Goal

NGConsole aims to become the primary daily workspace for professional network engineers.

---
This document is the governing architecture and design constitution for the NGConsole project.
