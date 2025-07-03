# Gemini CLI Terminal UI Reference

## Project TUI Overview

The Gemini CLI uses [React Ink](https://github.com/vadimdemedes/ink) to render an interactive terminal application.  The UI is composed from numerous components under `packages/cli/src/ui`.  This document describes every visible element, state management pattern and user interaction so that the interface can be reimplemented in Python using a framework such as [Textual](https://textual.textualize.io/) or [Rich](https://rich.readthedocs.io/).

## Global UX Philosophy & Principles

* **Retro Terminal Feel** – The app renders ASCII art banners and simple boxes with rounded or single borders.  Text is kept compact with modest padding.
* **Themeable Colors** – All colors come from a theme system.  Light and dark palettes are provided and users may switch with `/theme`.
* **Minimal Flicker** – Long outputs are streamed and broken into static sections to avoid terminal flicker.
* **Keyboard Focus** – Interaction is driven entirely by keyboard input; mouse events are not used.
* **Accessibility** – A `NO_COLOR` mode disables colored output; ASCII art adjusts to terminal width; layout adapts on resize.

## Component‑by‑Component Reference

### App Wrapper (`App.tsx`)

* Creates contexts (`SessionStatsProvider`, `StreamingContext`, `OverflowProvider`).
* Renders a static area containing the banner, tips, update notifications and previous history.
* Maintains state for dialogs (theme, auth, editor), help overlay, debug console and streaming spinner.
* Handles ctrl‑key shortcuts globally (Ctrl+C/D to quit, Ctrl+S to disable height constraint, Ctrl+O toggles debug console, Ctrl+T toggles tool descriptions).
* Uses `useInput` from Ink to receive keystrokes even when components are not focused.
* Monitors terminal size and recalculates layout width.
* Provides the footer with model info, branch name and memory stats.

#### Python mapping
In Textual a similar structure can be achieved with a root `App` class that sets up reactive state and sub‑widgets.  Global shortcuts map to `on_key` handlers.  `Static` areas can be emulated with `Static` widgets or by writing directly to the screen buffer.

### Header (`Header.tsx`)

* Shows large or small ASCII art depending on terminal width.
* When theme defines gradient colors, art is printed through `ink-gradient`.

**Python Notes:** Use `rich.console.Console` with gradient text (Rich supports gradient styles) or static color output.

### Input Prompt (`InputPrompt.tsx`)

* Wraps a custom text buffer that supports multiline editing, external editor integration and cursor control.
* Handles keybindings: Ctrl+A/E move to start/end, Ctrl+L clears screen, Ctrl+P/N traverse input history, Ctrl+K/U kill line, Ctrl+X open in editor, Enter submits, Ctrl+Enter inserts newline, Esc cancels suggestions/shell mode, Tab cycles auto‑completion.
* When first character typed is `!`, shell mode toggles; executed command is stored in shell history.
* Autocomplete suggestions appear below using `SuggestionsDisplay` and support scrolling with arrows or Tab.
* Cursor is drawn with inverse highlighting via `chalk.inverse`.

**Python Notes:** Textual's `Input` widget or custom text buffer can implement similar editing.  Key handling logic maps to `on_key` events.  Suggestions can be shown in a dropdown `ListView`.

### SuggestionsDisplay

* Shows up to eight suggestions with optional description column.
* Displays ▲ and ▼ markers when the list is scrollable, along with `(index/total)` indicator.

**Python Equivalent:** Use a vertical list inside a `Panel`; highlight active item with color.

### LoadingIndicator & GeminiRespondingSpinner

* Displays a spinner or witty loading phrase while Gemini is responding.
* When waiting for confirmation the spinner stops and a minimal symbol is shown.
* Uses `usePhraseCycler` to rotate phrases every 15s.

**Python Notes:** Textual `Spinner` or `ProgressSpinner` can mimic this.  Phrase rotation can be implemented with timers.

### AutoAcceptIndicator & ShellModeIndicator

* Shown in the footer when auto‑accept or shell mode is active.
* Provide short explanatory text (“accepting edits (shift+tab to toggle)”, “shell mode enabled (esc to disable)”).

### Help Overlay (`Help.tsx`)

* Rendered when `/help` is triggered.
* Lists basic usage instructions, available slash commands, and keyboard shortcuts (Enter, Shift+Enter, Up/Down, Alt+Left/Right, Esc, Ctrl+C).

### ThemeDialog, AuthDialog, EditorSettingsDialog

* Modal dialogs opened via slash commands.
* Use `RadioButtonSelect` (custom component) to choose an option.  Tab switches between sections (e.g., theme list and “Apply To” scope).  Escape closes dialog without selecting.
* Each dialog shows help text under the list (“Use Enter to select...” etc.) and may display an error message in red.

**Python Equivalent:** Use Textual `Dialog` or `Modal` with `ListView` for options and key handlers for Tab/Esc/Enter.

### Tool Messages (`ToolMessage`, `ToolGroupMessage`, `ToolConfirmationMessage`)

* Represent tool execution suggestions/results inside the conversation history.
* `ToolGroupMessage` wraps multiple tools in a bordered box.  Tools may be pending, executing, confirming, success, error, or canceled.  Status is shown with colored symbols (`o`, `?`, `✔`, `x`).
* Confirmation prompts use `RadioButtonSelect` to choose from options like “allow once”, “allow always”.
* Tool diffs or output displayed via `DiffRenderer`, optionally truncated with `MaxSizedBox`.

### MarkdownDisplay

* Parses Markdown to Ink `<Text>` components with bold, italic, strikethrough, underline and links.  Code fences are syntax highlighted via `lowlight` using the active theme.  Lists, headers and horizontal rules are supported.
* Code blocks and diff content are constrained by available terminal height to avoid overflow.

**Python Equivalent:** `rich.markdown.Markdown` for markup and `rich.syntax` for highlighting.  Use a container with height limits to manage overflow.

### Footer

* Shows target directory, git branch, model name and percentage of context used.  In debug mode displays memory usage and debug messages.  When “corgi mode” is enabled, prints ASCII corgi face.
* Middle area shows sandbox status or seatbelt profile when running in a sandbox.

### Tips & UpdateNotification

* Tips component provides a short list of onboarding messages.  UpdateNotification shows when a newer CLI version is available.

### DetailedMessagesDisplay (Debug Console)

* Appears when Ctrl+O is pressed.  Shows patched console output (`console.log`, `console.error`, etc.) inside a `MaxSizedBox` with icons for info/warn/error/debug.

### StatsDisplay & SessionSummaryDisplay

* Displays token usage statistics.  StatsDisplay shows current turn and cumulative numbers.  SessionSummaryDisplay is shown when quitting and includes a gradient title “Agent powering down. Goodbye!”

### Overflow Handling (MaxSizedBox & ShowMoreLines)

* `MaxSizedBox` constrains content to a maximum height and reports when content overflows via `OverflowContext`.
* `ShowMoreLines` prompts users with “Press ctrl-s to show more lines” when overflow occurs and height constraint mode is active.

### ConsolePatcher

* Overrides console methods to capture output and forward to the debug console.  Messages are stored with counts for repeated lines.

## User Interaction Flows & CLI Commands

### Keyboard Shortcuts

* `Ctrl+C` or `Ctrl+D` – pressed twice exits the program (prompt appears on first press).
* `Ctrl+S` – toggles height constraint, allowing overflow.
* `Ctrl+O` – toggles detailed debug console.
* `Ctrl+T` – toggle display of tool descriptions in the context summary and `/mcp` command.
* Arrow keys – navigate input history and suggestions.
* `Ctrl+P/N` – also navigate input history when suggestions are hidden.
* `Tab` – switch focus in dialogs or accept autocompletion.
* `Esc` – cancel suggestions, exit dialogs, exit shell mode, or cancel streaming.

### Slash Commands

* `/help` – show help overlay.
* `/clear` – clear the screen and conversation history.
* `/theme` – open ThemeDialog for theme selection.
* `/auth` – open AuthDialog for choosing auth method.
* `/editor` – open EditorSettingsDialog for preferred editor.
* `/stats` – show session statistics.
* `/mcp` – list configured MCP servers and tools (with optional `desc`, `nodesc`, `schema`).
* `/memory add|show|refresh` – manage hierarchical memory.
* `/restore` – restore from saved tool checkpoints.
* `/quit` or `/exit` – quit application.
* `/about` – display CLI version and environment info.
* `/bug` – open bug report URL.
* Additional built-in commands exist (see `slashCommandProcessor.ts`).

### At Commands

* `@path` – include file or directory content in query, with autocompletion of filesystem paths.
* `@` alone sends a literal `@` to Gemini.

### Shell Mode

* Typing `!` on an empty prompt toggles shell mode.  Commands typed in this mode are executed locally.  History is stored and accessible with up/down arrows when shell mode is active.
* `!cmd` as a single line executes immediately without toggling mode.

### Streaming Feedback

* While Gemini is responding, spinner and phrase are shown.  Pressing `Esc` cancels the request.  After completion, messages and tool results are added to history with timestamps.

## UI State Management & Feedback Patterns

* **Contexts**: `StreamingContext` indicates Idle, Responding or WaitingForConfirmation.  `SessionContext` tracks token statistics.  `OverflowContext` tracks which history items exceeded height constraints.
* **Hooks**: Many custom hooks manage logic:
  - `useGeminiStream` orchestrates streaming, tool scheduling and message handling.
  - `useSlashCommandProcessor`, `useShellCommandProcessor`, `useAtCommandProcessor` parse commands and return actions.
  - `useInputHistory`, `useShellHistory` maintain user input history.
  - `useCompletion` handles asynchronous suggestions for slash and at commands.
  - `useLoadingIndicator` ties streaming state to spinner phrases and elapsed time.
  - `useTerminalSize` listens for terminal resize events.
* **Static vs Dynamic Rendering**: Ink’s `<Static>` component is used to write previously finalized messages, preventing them from rerendering.  Only the latest pending history item rerenders as new data streams in.
* **Overflow Handling**: `MaxSizedBox` measures content and truncates with “first X lines hidden” or “last X lines hidden”.  `ShowMoreLines` invites the user to disable height constraints with Ctrl+S when needed.

## Visual / Styling Reference

* **Width Constants** – UI width typically 90% of terminal width.  Boxes use `BOX_PADDING_X` of 1 from `constants.ts`.
* **ASCII Banners** – Short and long variants exist; width around 59 characters.  Example snippet:

```
 ███            █████████  ██████████ ...
```

* **Color Palette** – Provided by theme objects (`theme.ts`).  Colors include Foreground, Background, LightBlue, AccentBlue, AccentPurple, AccentCyan, AccentGreen, AccentYellow, AccentRed, Gray and optional GradientColors.
* **Gradient Usage** – When a theme has `GradientColors`, the header title and quitting message use gradient text via `ink-gradient`.
* **Icons & Prefixes** – Messages use prefixes: user `>`, shell `!`, Gemini `✦`, info `ℹ`, error `✕`.  Tool statuses use `o`, `?`, `✔`, `x` etc.
* **Spinners** – `dots` spinner is standard; toggling spinner uses `toggle` type for tool execution.
* **Dialogs** – Rounded borders, gray border color, minimal padding (1).  Content inside uses two columns when space permits.

## Python TUI Implementation Guide

Below is a high level approach for recreating the interface using Textual (or Rich):

1. **Application Skeleton**
   ```python
   from textual.app import App, ComposeResult
   from textual.widgets import Static, Input, Footer

   class GeminiApp(App):
       CSS_PATH = "styles.css"  # define theme colors via CSS variables

       def compose(self) -> ComposeResult:
           yield Static(id="header")
           yield Static(id="history")
           yield Input(id="prompt")
           yield Footer()
   ```
   Manage global state using dataclass models or Textual’s `reactive` attributes.  Replace Ink contexts with Python class attributes or custom context managers.

2. **Keyboard Handling**
   ```python
   async def on_key(self, event: events.Key) -> None:
       if event.key == "ctrl+c":
           if self.exit_pending:
               await self.shutdown()
           else:
               self.exit_pending = True
               await self.show_status("Press Ctrl+C again to exit")
       elif event.key == "ctrl+l":
           self.clear_history()
       # add other shortcuts following the InputPrompt logic
   ```

3. **Prompt Editing & History**
   Implement a custom input widget with multiline editing and history recall.  Use `TextArea` (Textual 0.50+) or build with a `DataTable` for full control.  Autocomplete suggestions can appear in a `ListView` positioned below the input.

4. **Dialogs & Menus**
   Recreate `ThemeDialog`, `AuthDialog`, and `EditorSettingsDialog` using Textual’s `ModalScreen` or `Dialog`.  Radio button selection corresponds to `OptionList` or `ListView` with highlighted row.

5. **Rendering Messages**
   Use `Markdown` from Rich for model output, `Panel` for tool groups, and `Syntax` for code blocks.  Maintain a container widget where new message widgets are appended.  For streaming, update the last message widget incrementally.

6. **Spinners and Timers**
   Textual’s `LoadingIndicator` or `Spinner` widget can show while streaming.  Manage timers with `set_interval` to cycle phrases.

7. **Overflow Management**
   Use `ScrollableContainer` with `max_height` or compute truncated text manually.  Provide a control key (Ctrl+S) to toggle scroll behaviour.

8. **Theme Support**
   Map the theme structure to Textual CSS variables and allow dynamic switching.  Provide a command to change theme and persist selection using config files.

9. **Logging & Debug Console**
   Capture stdout/stderr and print to a separate panel.  This can be toggled by keybinding similar to Ink’s debug console.

## Tips, Gotchas, and Migration Recommendations

* **Streaming Efficiency** – Use Textual’s `Screen.update` carefully to avoid flicker.  Append static messages and update only the current streaming message.
* **External Editors** – When integrating an external editor (e.g., Vim) the TUI must temporarily suspend and restore the terminal modes (Textual `run_subprocess` with `standalone` flag or `console.screen.enter_alt_screen()`/`console.screen.restore()` when using Rich).
* **Terminal Resize** – Listen for resize events to recompute layout and wrap text accordingly.
* **No Color Mode** – Provide an environment variable to disable all color styling so output works in monochrome terminals.
* **Path Handling in `@` Commands** – Use Python’s `glob` and `os` modules to replicate cross-platform path expansion and ignore patterns.
* **Context Files & Memory Refresh** – Implement hooks for refreshing hierarchical memory from `GEMINI.md` files; watch for file count and display summary in footer.
* **Extensibility** – Slash commands are parsed before sending prompts to the model.  Implement a command dispatcher to allow new commands easily.

---

This reference aims to capture the entire UX implemented by the original TypeScript/Ink code so that it can be reproduced or extended in Python without direct access to the codebase.
