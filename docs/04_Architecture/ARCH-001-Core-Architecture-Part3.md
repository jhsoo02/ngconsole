# ARCH-001 — NGConsole Core Architecture
## Part 3: Console Engine Architecture

**Project:** NGConsole  
**Document ID:** ARCH-001  
**Part:** 3 of 6  
**Version:** 1.0  
**Status:** Baseline  
**Target Platform:** macOS  
**Language:** Swift  
**UI:** SwiftUI

---

## 1. Purpose

Part 3 defines the architecture of the NGConsole Console Engine.

The Console Engine is responsible for providing the professional terminal experience expected from a SecureCRT-class application.

Primary goals:

- Reliable terminal emulation
- ANSI/VT processing
- Fast rendering
- Large scrollback support
- Text selection
- Search
- Highlighting
- Configurable colors
- Configurable fonts
- Keyboard input
- Terminal resizing
- High-volume output handling
- Complete separation between terminal state and transport

The Console Engine must be independent of SSH, Telnet, Serial, SFTP, FTP, TFTP and Python implementations.

---

## 2. Design Principles

### 2.1 Transport Independence

The Console Engine receives terminal bytes through a generic transport.

```text
SSH
Telnet
Serial
Local Shell
     |
     v
TerminalTransport
     |
     v
Console Engine
```

The Console Engine must not know whether bytes originated from SSH, Telnet, Serial or a local process.

### 2.2 State Before Rendering

Terminal state is the source of truth.

```text
Input Bytes
    |
    v
Parser
    |
    v
Terminal State
    |
    v
Renderer
    |
    v
SwiftUI
```

SwiftUI views must never be the authoritative storage for terminal contents.

### 2.3 High Performance

Terminal output can be extremely large:

- `show configuration`
- `show log`
- `show tech-support`
- debug output
- large routing tables
- large interface tables
- script output

The Console Engine must avoid:

- One SwiftUI update per byte
- One Task per output chunk
- Rebuilding the entire terminal for every character
- Unbounded memory growth
- Excessive String copying

---

## 3. Console Engine Responsibilities

The Console Engine owns:

- Terminal state
- Screen buffer
- Scrollback
- Cursor
- ANSI parser
- VT control sequences
- Character attributes
- Colors
- Selection
- Search
- Highlighting
- Keyboard input mapping
- Terminal resize
- Rendering model

It does not own:

- SSH connection
- Telnet connection
- Serial connection
- SFTP
- FTP
- TFTP
- Python execution
- Workspace persistence
- AI processing

---

## 4. Terminal Architecture

```text
+----------------------------------------------------------+
|                    SwiftUI Terminal View                 |
+----------------------------------------------------------+
|                 Terminal Presentation Model              |
+----------------------------------------------------------+
|                    Terminal Renderer                     |
+----------------------------------------------------------+
|                    Terminal State                        |
|                                                          |
| Screen Buffer | Cursor | Modes | Attributes | Selection |
+----------------------------------------------------------+
|                    ANSI / VT Parser                      |
+----------------------------------------------------------+
|                     Byte Stream                          |
+----------------------------------------------------------+
|                  TerminalTransport                       |
+----------------------------------------------------------+
| SSH | Telnet | Serial | Local Shell                      |
+----------------------------------------------------------+
```

---

## 5. Terminal State

The terminal state is the authoritative representation of the current terminal.

Conceptual model:

```swift
public struct TerminalState: Sendable {
    public private(set) var screen: ScreenBuffer
    public private(set) var cursor: CursorPosition
    public private(set) var modes: TerminalModes
    public private(set) var attributes: TextAttributes
}
```

The actual implementation may use a reference-backed or actor-isolated model for performance.

The public architecture must remain independent of the storage mechanism.

---

## 6. Screen Buffer

The screen buffer represents the visible terminal area.

```text
+---------------------------------------+
| user@switch# show version             |
|                                       |
| ExtremeXOS version 32.x               |
|                                       |
| Switch uptime: 12 days                |
|                                       |
| user@switch#                          |
|                                       |
+---------------------------------------+
```

The buffer is defined by:

```text
columns
rows
cells
cursor
```

Each cell may contain:

- Character or grapheme
- Foreground color
- Background color
- Bold
- Dim
- Italic
- Underline
- Blink
- Inverse
- Hidden
- Strike-through where supported

---

## 7. Cell Model

Conceptual model:

```swift
public struct TerminalCell: Sendable {
    public var text: String
    public var attributes: TextAttributes
}
```

For performance, the final implementation should avoid allocating a separate Swift String for every cell.

Possible implementation strategies:

- UTF-8 storage
- Character IDs
- Grapheme references
- Interned strings
- Compact cell structures

The final representation must be selected through benchmarking.

---

## 8. Unicode

NGConsole must support modern network-device output containing Unicode.

The architecture should account for:

- UTF-8
- Unicode scalar values
- Combining characters
- Wide characters
- Full-width characters
- Emoji where terminal behavior permits

Terminal column width is not equivalent to Swift `String.count`.

A wide CJK character may occupy two terminal columns.

The rendering layer must use terminal display width rather than naive character count.

---

## 9. Grapheme and Display Width

Terminal parsing should distinguish:

```text
Unicode Scalar
      |
      v
Grapheme Cluster
      |
      v
Terminal Cell Width
      |
      +--> 0 columns
      +--> 1 column
      +--> 2 columns
```

This must be handled consistently by:

- Cursor movement
- Line wrapping
- Selection
- Search
- Rendering
- Copy

---

## 10. Cursor Model

Conceptual cursor position:

```swift
public struct CursorPosition: Sendable {
    public var row: Int
    public var column: Int
}
```

Cursor state also needs:

- Visible / hidden
- Blink
- Shape
- Current attributes

Possible cursor styles:

- Block
- Underline
- Bar

Cursor style should be configurable if supported by the terminal protocol.

---

## 11. Terminal Modes

Terminal modes must be represented separately from visual state.

Examples:

- Insert mode
- Auto-wrap
- Origin mode
- Application cursor keys
- Application keypad
- Bracketed paste
- Mouse reporting
- Alternate screen
- Cursor visibility

Conceptual model:

```swift
public struct TerminalModes: Sendable {
    public var autoWrap: Bool
    public var insertMode: Bool
    public var originMode: Bool
    public var applicationCursorKeys: Bool
    public var bracketedPaste: Bool
    public var alternateScreen: Bool
}
```

Only modes required by supported terminal behavior should be implemented initially.

---

## 12. ANSI / VT Parser

The parser converts incoming bytes into terminal actions.

```text
Bytes
  |
  v
UTF-8 Decoder
  |
  v
ANSI / VT Parser
  |
  +--> Print Character
  +--> Cursor Movement
  +--> Erase
  +--> Scroll
  +--> Set Attribute
  +--> Set Mode
  +--> Save/Restore Cursor
  +--> OSC
  |
  v
Terminal State
```

The parser must support fragmented input.

A control sequence may be split across multiple transport packets.

Example:

```text
Packet 1:
ESC [

Packet 2:
31 m
```

The parser must preserve parser state between calls.

---

## 13. Parser State Machine

The parser should use an explicit state machine.

Conceptually:

```text
Ground
  |
  +--> Escape
  |
  +--> CSI Entry
  |
  +--> CSI Parameter
  |
  +--> CSI Intermediate
  |
  +--> OSC
  |
  +--> Other String State
```

A parser must never assume that one network read equals one terminal command.

---

## 14. CSI Sequences

The parser should support commonly used CSI operations required by network equipment.

Examples:

- Cursor movement
- Erase display
- Erase line
- Insert/delete characters
- Insert/delete lines
- Scroll
- Select graphic rendition
- Save/restore cursor

Examples:

```text
ESC [ H
ESC [ 2 J
ESC [ K
ESC [ 31 m
ESC [ 0 m
ESC [ ? 25 l
ESC [ ? 25 h
```

Implementation priority should be based on sequences commonly emitted by network devices and terminal shells.

---

## 15. SGR / Text Attributes

SGR controls text appearance.

Examples:

```text
ESC [ 0 m
ESC [ 1 m
ESC [ 3 m
ESC [ 4 m
ESC [ 30 m
ESC [ 31 m
ESC [ 32 m
ESC [ 40 m
```

The Console Engine must maintain current attributes:

```text
Current Attributes
        |
        +--> Foreground
        +--> Background
        +--> Bold
        +--> Dim
        +--> Italic
        +--> Underline
        +--> Inverse
        +--> Blink
```

---

## 16. Color Model

NGConsole should support at least:

- Default foreground/background
- ANSI 16 colors
- 256 colors
- True Color where supported

Conceptual model:

```swift
public enum TerminalColor: Sendable {
    case `default`
    case ansi(index: UInt8)
    case indexed(index: UInt8)
    case rgb(red: UInt8, green: UInt8, blue: UInt8)
}
```

The final color rendering should be resolved by the theme layer.

---

## 17. User Color Customization

Required user customization:

- Background color
- Text color
- Highlight color
- Font size

Terminal semantics and appearance must remain separate.

```text
TerminalCell
      |
      v
Semantic Color
      |
      v
Theme Resolver
      |
      v
SwiftUI Color
```

A terminal theme can therefore be changed without modifying parser logic or terminal state.

---

## 18. Font Model

The terminal font must be configurable.

Settings should include:

```text
Font Family
Font Size
Line Height
Letter Spacing
```

A monospace font is required for correct terminal alignment.

The renderer must not assume a specific font.

---

## 19. Scrollback

Scrollback is separate from the visible screen buffer.

```text
+----------------------+
| Scrollback Buffer    |
| Line N-10000         |
| Line N-9999          |
| ...                  |
| Line N-1             |
+----------------------+
| Screen Buffer        |
| Current terminal     |
+----------------------+
```

The architecture should support a configurable scrollback limit.

Example settings:

```text
10,000 lines
50,000 lines
100,000 lines
Unlimited*
```

`Unlimited` must not mean unrestricted memory growth. A safe memory policy is still required.

---

## 20. Scrollback Storage

Avoid repeatedly concatenating the entire terminal output into one large String.

Preferred conceptual model:

```text
Ring Buffer
     |
     +--> Line
     +--> Line
     +--> Line
     +--> ...
```

A ring buffer can bound memory while preserving recent terminal history.

---

## 21. Alternate Screen

Some terminal applications use an alternate screen buffer.

Examples:

- `top`
- `less`
- Full-screen terminal applications

The architecture should support:

```text
Primary Screen
     |
     +--> Alternate Screen
```

Switching to alternate screen must preserve the primary screen state.

---

## 22. Selection Model

Text selection should be independent of SwiftUI's native text selection where necessary.

The terminal has a two-dimensional coordinate system.

```swift
public struct TerminalPoint: Sendable {
    public var row: Int
    public var column: Int
}

public struct TerminalSelection: Sendable {
    public var start: TerminalPoint
    public var end: TerminalPoint
}
```

Selection should support:

- Character selection
- Line selection
- Rectangular selection where practical
- Copy
- Clear selection

---

## 23. Copy Behavior

```text
Terminal Cells
      |
      v
Selection Extractor
      |
      v
Logical Text
      |
      v
Clipboard
```

The copy operation must not blindly copy cell padding.

Wide characters and line boundaries must be handled correctly.

---

## 24. Search

Search should operate against terminal history.

Required behavior:

- Search text
- Next match
- Previous match
- Case-sensitive option
- Case-insensitive option
- Highlight matches
- Clear search

The search system must not modify terminal state.

---

## 25. Highlight

Highlighting is a presentation feature.

Possible uses:

- Search matches
- User-defined patterns
- Important CLI output
- Error messages
- Warnings
- Selected text

Potential future pattern model:

```text
Pattern
├── Expression
├── Match Type
├── Foreground
├── Background
└── Enabled
```

Regex-based highlighting is optional and should not be required for the first implementation.

---

## 26. Keyboard Input

Keyboard input must be translated into terminal input sequences.

Example:

```text
Arrow Up
   |
   +--> Application Mode?
           |
           +--> ESC [ A
           |
           +--> ESC O A
```

The input layer must consider terminal modes.

Supported input should include:

- Standard characters
- Enter
- Backspace
- Tab
- Escape
- Arrow keys
- Home
- End
- Page Up
- Page Down
- Function keys
- Ctrl combinations
- Option/Meta combinations where appropriate

---

## 27. Paste

Paste must support normal terminal input.

Bracketed paste mode must be supported when enabled by the remote application.

```text
Paste Text
    |
    v
Bracketed Paste Enabled?
    |
    +---- Yes ---> ESC [ 200 ~
    |              Text
    |              ESC [ 201 ~
    |
    +---- No ----> Text
```

Paste must not unexpectedly execute multi-line configuration unless the user explicitly pastes it.

---

## 28. Terminal Resize

The terminal view determines current dimensions.

```text
SwiftUI View Size
      |
      v
Calculate Columns / Rows
      |
      v
Terminal State
      |
      v
Connection Manager
      |
      v
Protocol Driver
```

For SSH, the PTY window size must be updated.

Resize events should be debounced/coalesced during continuous window resizing.

---

## 29. Rendering Strategy

Rendering should be based on terminal state.

```text
Terminal State
      |
      v
Visible Rows
      |
      v
Line / Cell Grouping
      |
      v
Attributed Runs
      |
      v
SwiftUI Rendering
```

Adjacent cells with identical visual attributes should be grouped.

```text
red red red red | white white | green green
       |
       v
Run 1           Run 2          Run 3
```

This is more efficient than creating a SwiftUI view for every cell.

---

## 30. Rendering Performance

Avoid:

```text
1 cell = 1 SwiftUI View
```

for large terminals.

Preferred conceptual rendering:

```text
Terminal Row
     |
     +--> Attribute Run
     +--> Attribute Run
     +--> Attribute Run
```

Possible rendering technologies:

- SwiftUI `Text`
- SwiftUI `Canvas`
- Core Text
- TextKit
- Metal-backed rendering

The final renderer must be selected based on measured performance.

---

## 31. High-Volume Output

High-volume output must be handled through batching.

```text
Incoming Data
      |
      v
Transport Buffer
      |
      v
Parser
      |
      v
State Updates
      |
      v
Frame / Batch Boundary
      |
      v
UI Update
```

The important requirement is:

> UI update frequency must not be directly proportional to network packet frequency.

---

## 32. Output Throttling

If a device produces extremely high output:

```text
Device
  |
  v
High-rate output
  |
  v
Transport
  |
  v
Parser
  |
  v
State
  |
  v
Coalesced Rendering
```

The parser must continue processing data correctly even if UI rendering temporarily falls behind.

The UI may render the newest state while preserving required scrollback.

---

## 33. Auto-Scroll

The terminal should automatically scroll to the bottom during normal output.

If the user scrolls upward:

```text
User scrolls up
      |
      v
Auto-scroll disabled
      |
      v
New output arrives
      |
      v
Show "new output" indicator
```

Returning to the bottom should re-enable automatic scrolling.

This prevents the user's reading position from being unexpectedly moved.

---

## 34. Terminal Bell

The Console Engine should recognize terminal bell events.

Possible behavior:

- Visual bell
- System notification
- Optional sound

This must be configurable.

---

## 35. Terminal Profiles

Terminal behavior should be configurable through a terminal profile.

Conceptual model:

```swift
public struct TerminalProfile: Codable, Sendable {
    public var fontName: String
    public var fontSize: Double
    public var themeID: String
    public var scrollbackLimit: Int
    public var cursorStyle: CursorStyle
    public var cursorBlink: Bool
}
```

A Session may reference a terminal profile.

---

## 36. Theme Architecture

Themes should be data-driven.

Example:

```text
Theme
├── background
├── foreground
├── cursor
├── selection
├── ansiBlack
├── ansiRed
├── ansiGreen
├── ansiYellow
├── ansiBlue
├── ansiMagenta
├── ansiCyan
├── ansiWhite
└── bright variants
```

A theme must be loadable without changing terminal parser code.

---

## 37. Terminal Engine API

Conceptual public interface:

```swift
public protocol TerminalEngine: Sendable {
    func receive(_ data: Data) async
    func sendInput(_ data: Data) async
    func resize(columns: Int, rows: Int) async
    func clear() async
    func reset() async
}
```

The implementation may expose state through an observable adapter rather than making the engine itself a UI object.

---

## 38. Terminal State Observation

The UI should observe state changes through an adapter.

```text
TerminalEngine
      |
      v
TerminalStatePublisher / AsyncStream
      |
      v
@MainActor ViewModel
      |
      v
SwiftUI
```

This keeps the engine independent of SwiftUI.

---

## 39. Testing Strategy

The Console Engine must be highly testable.

### Unit Tests

Test:

- ANSI parsing
- Cursor movement
- Erase operations
- SGR
- Wrapping
- Scrollback
- Selection
- Search
- Unicode width
- Alternate screen
- Resize
- Input encoding

### Golden Tests

A sequence of terminal input should produce an expected terminal state.

Example:

```text
Input:
ESC [ 31 m
HELLO
ESC [ 0 m

Expected:
HELLO
foreground = red
```

### Fragmentation Tests

The same input must produce the same result whether delivered as:

```text
"ESC[31mHELLO"
```

or:

```text
"ESC"
"["
"31"
"m"
"HEL"
"LO"
```

This is critical for real network operation.

---

## 40. Fuzz Testing

The parser should eventually support fuzz testing.

Random and malformed byte sequences must not:

- Crash
- Hang
- Consume unbounded memory
- Produce invalid cursor positions

Malformed terminal sequences should be safely ignored or handled according to parser rules.

---

## 41. Security Considerations

The Console Engine may display sensitive information:

- Password prompts
- Authentication output
- Network configuration
- IP addresses
- Private keys
- Tokens

Therefore:

- Do not write terminal contents to logs by default.
- Do not send terminal output to analytics.
- Do not persist terminal output unless the user explicitly requests logging.
- Ensure copied data is controlled by the user.
- AI plugins must not automatically receive terminal contents.

---

## 42. AI Extension Boundary

AI is not part of the Console Engine.

Future AI integration should consume explicitly selected terminal content.

```text
Terminal
   |
   v
User Selects Text
   |
   v
AI Plugin
   |
   v
Analysis
```

The AI plugin must never silently capture the entire terminal stream.

This is an important privacy and security boundary.

---

## 43. Console Engine Development Order

### Phase 1

- Terminal cell model
- Screen buffer
- Cursor
- Basic text output

### Phase 2

- ANSI parser
- SGR
- Cursor movement
- Erase
- Line wrapping

### Phase 3

- Scrollback
- Selection
- Copy
- Search
- Highlight

### Phase 4

- Unicode
- Wide characters
- Combining characters
- Alternate screen

### Phase 5

- Keyboard input
- Terminal modes
- Bracketed paste
- Resize

### Phase 6

- SwiftUI rendering
- High-volume output optimization
- Performance benchmarks

### Phase 7

- Golden tests
- Fragmentation tests
- Fuzz tests
- Regression suite

---

## 44. Performance Targets

Exact targets should be verified through benchmarks, but the architecture should aim for:

- Smooth interactive terminal rendering
- No visible UI freeze during large output
- Stable memory usage with bounded scrollback
- Correct handling of sustained high-volume output
- No unbounded task creation
- No main-thread blocking during network I/O

Performance must be measured on representative macOS hardware.

---

## 45. Definition of Done

Console Engine Part 3 is complete when:

- Terminal state is independent of UI.
- Transport is independent of terminal rendering.
- ANSI/VT parser supports required network-device sequences.
- Parser handles fragmented input.
- Screen buffer and scrollback are bounded.
- Unicode width is handled correctly.
- Selection and copy work correctly.
- Search and highlight work correctly.
- User can configure font, background, foreground and highlight colors.
- Terminal resize works.
- High-volume output does not freeze the UI.
- Unit tests cover parser and state transitions.
- Golden tests exist for important sequences.
- Malformed input cannot crash the parser.
- AI has no direct dependency on the Console Engine.

---

## 46. Next Part

**ARCH-001 Part 4 — Connection Manager Architecture**

will define:

- Connection Manager
- SSHv2 architecture
- Host-key verification
- Password authentication
- Public-key authentication
- SSH PTY
- Telnet
- Serial Console
- Local Shell
- Connection state machine
- Reconnect strategy
- Keepalive
- Session lifecycle
- Credential handling
- Keychain integration
- Transport abstraction
- Mock transports
- Connection testing strategy
