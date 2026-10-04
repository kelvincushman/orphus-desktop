> **Clean-room spec input.** Written by spec writers from a study of PI-Desktop v0.16.1 (LGPL-3.0). It describes behaviour in our own words and contains no PI-Desktop code or prose. Implementers build from this and [../PRD.md](../PRD.md), never from PI-Desktop source. See [../../CLEANROOM.md](../../CLEANROOM.md).

# Orphus Desktop: clean-room inventory of UX/UI and architecture

**Subject.** PI-Desktop v0.16.1 (vastsa/PI-Desktop, LGPL-3.0). This document describes what it does so that an independent MIT-licensed implementation can match it.

**Clean-room rules.** Everything here is paraphrased. No code, types, config, CSS values or document text is reproduced. Product, package and tool names are used only as vocabulary. Numbers appear only where they set observable behaviour (timeouts, caps, limits). Visual values are described in qualitative terms only.

**Sources surveyed.** Product specs (`docs/spec/01..08`), about 320 ADRs, the renderer (`apps/desktop/src`), the main process and preload, the `packages/*` modules, `crates/host-core`, `apps/pi-host`, CI workflows, scripts, and the three pi patch files.

---

# Part 1: UX / UI

## 1. Product character

**What it is.** A single-window desktop workbench for a local-first coding agent. Chat is the home surface. Tool activity, approvals, diffs and questions all render inline in the conversation or dock above the composer. They are almost never modal.

**Visual lineage.** It openly follows the Codex / ChatGPT desktop look:
- neutral charcoal greys rather than blue-slate,
- one fixed toolbar height across all chrome,
- a sidebar roughly a quarter of the window wide,
- a floating, heavily rounded composer.

The transcript is denser than a consumer chat app.

**Guiding principles:**
- Clarity over decoration, and developer-grade density.
- Dark is primary; light is fully supported.
- One restrained accent; colour carries semantic status only.
- Motion is used only as feedback.
- Keyboard first.
- Titlebar chrome is continuous and borderless.

**Separation rule.** In-flow surfaces are separated by tone and spacing, not lines. Hairline strokes are reserved for floating layers: menus, sheets, dialogs and toasts.

**Copy rule.** Show only text that changes a decision. Never trim risk, scope, path, keyboard or error detail. Empty states always offer a next action.

## 2. Window layout and regions

### 2.1 Three columns: sidebar | main pane | work panel (dock)

**Sidebar**
- Resizable by its right edge.
- Dragging well under its minimum collapses it to an icon rail.
- Toggled with Mod+B.

**Main pane**
- Shows one destination at a time. Destinations replace each other as pages.
- In chat, content sits in a centred reading band whose width the user can drag:
  - invisible edge handles reveal a capsule grip on hover,
  - double-click resets the width,
  - arrow keys step it, and Home/End jump to default/fill.
- User message plates are narrower than assistant content.

**Work panel**
- An in-flow docked column, not an overlay. Closed by default.
- The chat keeps a hard minimum readable width. When the panel needs room, the sidebar auto-collapses first and returns when the panel closes; a manual collapse stays collapsed.
- The panel resizes from its inner edge. Escape cancels a drag; double-click restores the default.
- Opening or resizing the panel never resizes the OS window. Only the preferred panel width persists.

**Window**
- Minimum size is small-laptop sized, clamped to the work area.

**Preview mode**
- The work panel can expand across the whole client area beside the sidebar, temporarily unmounting chat.
- A slim chrome row keeps New Task, the sidebar toggle and the window controls reachable.

**No status bar.** One was planned and deferred. Running state lives in the composer and transcript.

### 2.2 Chrome per OS

One fixed toolbar height is shared by the conversation top bar, the drag bands, the sidebar header, the work-panel header and the window-control band.

**macOS**
- Native inset traffic lights; the expanded sidebar shows only a collapse button beside them.
- The sidebar uses native vibrancy with a faint theme tint and edge sheen. The main pane stays opaque.
- Full application menu: New Task, Open Project, Settings, Command Palette, Sidebar, Edit, View, Window, Help/Logs, Check for Updates.

**Windows / Linux**
- Frameless window with no in-window menubar.
- The sidebar starts with a home brand mark plus a collapse button.
- Custom minimise / maximise-restore / close buttons sit at the top right:
  - they are explicit no-drag zones with localised tooltips,
  - the glyph tracks the real maximise state,
  - when the dock is open they stay pinned over its header.
- Windows uses a slightly rounded window; a plugin theme may adjust the radius.

**Close and quit**
- On Windows/Linux, close behaviour is user-configurable. The first time, a native prompt offers cancel / close to tray / quit; the choice is remembered and editable in Settings.
- Explicit Quit always confirms.

**Conversation top bar (chat only)**
- Left: a sidebar toggle (only while collapsed) and the ellipsised task title; the project appears only in the tooltip.
- Right: New task and Search icons, whose tooltips show the shortcut.
- No model or mode controls here.
- Every chrome icon button is the same small square with a transparent seat. Hover adds a wash; an "on" state changes only glyph and ink, never adds a filled pill.

**Backend status capsule**
- A floating capsule under the titlebar when the backend is restarting or failed, with an "Open logs" action.

### 2.3 Navigation model

- **Routes:** Chat, Scheduled, Extensions (plugins), Settings (full-window takeover). The project archive lives in Settings.
- **History:** linear back/forward with Mod+[ and Mod+]. There are no visible buttons; while on Extensions, the footer Plugins button acts as Back.
- **No navigation by background events.** Background completions, permission prompts and notifications never change the page, project, session or focus. Only explicit user actions navigate.
- **Retained session panes.** Returning to a session reveals it instantly with its own scroll position and its own work-panel tabs.
- **New Task is idempotent per group.** The newest empty session in the current project or temporary group is reused; otherwise a durable empty session is created and appears in the sidebar at once.
- **Projects and temporary chats.**
  - Choosing a session in another project activates that project first.
  - Temporary (path-less) chats never inherit a workspace.

## 3. Sidebar

**Sections, top to bottom:**
1. **Pinned.** Global; shown only when something is pinned.
2. **Sessions.** Path-less chats.
   - Header actions: sort and new chat.
   - Shows about five rows, then scrolls internally.
3. **Projects.** Collapsible groups filling the remaining height.

**Project group header**
- Contents: name, a current-workspace dot, a disclosure chevron, a hover "+" for a new task, and an overflow menu.
- The whole title row toggles the group.
- Press-and-move reorders groups; arrow keys on a focused title reorder too.
- Projects never get a "selected" fill. Only the current conversation does.

**Project actions:** open folder, edit (name and multiple root folders), pin, archive/restore, close tab (deletes nothing), project instructions, project memory.

**Conversation actions:**
- rename, pin, archive/restore, fork, delete
- copy a conversation link, used to reference it from another chat
- developer mode only: copy the ID, open the session file

**Sorting and paging**
- Sort options: recently updated, created, oldest, name.
- Pinned rows come first.
- Ten rows per group, then a "load more" row.

**Row status slot** (always a shape or icon, never colour alone):
- selected: an outlined ring
- running: an orange dot that breathes twice then holds
- completed and unread: a green check
- failed and unread: a red alert mark

Opening the session marks the outcome as read.

**Titles.** "New task" until the first prompt, then a truncated prompt, later replaced by an auto-summary. A user rename is never overwritten.

**Footer**
- Settings, Extensions, Scheduled, and a notification bell with an unread badge.
- A version chip that doubles as the update entry.

**Collapsed rail:** icons only, with tooltips.

## 4. Screens

### 4.1 Empty home (new chat)

- **Layout.** A vertically centred hero, with an optional onboarding checklist below it and the composer reserved at the bottom. The composer never overlaps content.
- **Hero.** A small animated mascot with light and dark art, held still under reduced motion, plus a large "what shall we build" style title.
- **Hero variants:**
  - **In a project:** the project name in the title is dotted-underlined and acts as a switcher. It is searchable and offers open projects, clone a git repo, and open another folder.
  - **Temporary chat:** dedicated copy and no switcher.
  - **No project:** a generic welcome.
- **No starter cards.** Earlier versions had prompt-prefill starter cards; the current design removed them so task entry goes straight to the composer.

### 4.2 Conversation

The transcript and composer, covered in sections 5 and 6.

### 4.3 Scheduled (automations)

**Layout.** A task list column plus a detail page.

**Create/edit form**
- Cadence options: manual, hourly, interval (minutes or hours), daily, weekly.
  - Daily uses preset time-of-day buckets (morning, afternoon, evening, night). A time set by the agent that fits no bucket shows as "custom".
  - Weekly uses a multi-select of weekdays.
- Per-task settings: project, permission mode (choosing Auto shows a warning), model (searchable, grouped by provider), and conversation mode (a fresh chat per run, or one continuing chat).
- The instruction box borrows the composer's styling, with these choices as chips.

**Task detail page**
- last and next run, the configuration, and the instruction behind a disclosure
- run history: status, start time, duration, error code
- the selected run's transcript, readable inline
- "Open conversation", which hands off to chat with a back link
- Run now (works in the background), pause/resume, and delete with confirmation

### 4.4 Extensions (plugins page)

- A compact header and two tabs: **Installed** and **Marketplace**.
- **Installed** groups plugins as soft tiles under: Needs attention, Updates available, Active, Turned off.
- **Marketplace** is a card grid with a source selector. Cards show monograms, not remote icons.
- A detail sheet slides over the page, and a permission review dialog appears on install or upgrade.
- MCP servers, skills and subagents are managed in Settings, not here.

### 4.5 Settings

A full-window takeover, covered in section 10.

### 4.6 Overlays and secondary windows

**Overlays**
- global search (which contains commands)
- the model × reasoning menu
- the notification inbox
- toasts
- the project switcher
- dialogs: project create/edit/delete, instructions, memory, rename, release notes
- the startup splash and startup recovery screen

**Separate windows**
- the global plugin launcher
- detached plugin panels
- floating plugin widgets
- the Live Voice call bar

## 5. Chat surface

### 5.1 Transcript structure

**Layout**
- A centred message band, with a stable scrollbar gutter on the right.
- A **conversation minimap** on the left edge:
  - one dash per turn; hover previews the sender and the start of the text, and nearby dashes magnify;
  - clicking jumps to that turn;
  - shown only when content overflows, with an "earlier history" item while older history is unloaded.

**One assistant turn per user turn.** Everything between two user messages reads as one unit: thinking, tool rows, interim text and the final answer. The turn has a single action bar and one meta row.

**User messages**
- Right-aligned, borderless, on a soft tinted plate with one tighter corner. They shrink-wrap short prompts and keep newlines.
- Inline chips stand in for:
  - file references: the leaf name, with the full path in the tooltip
  - images: a thumbnail on hover
  - links to other conversations: the referenced chat's title
- Bare path text is chipped only if the file really exists.

**Assistant messages**
- Markdown with syntax-highlighted code blocks on an inset surface.
- Tables with their own copy actions.
- Math rendering.
- Mermaid diagrams, rendered only after the fence closes and near the viewport, with the source as fallback.
- Local images contained inside the message.

**Hover actions** (icon-only, revealed on hover or focus):
- Assistant turn: copy, plus fork and regenerate once the turn is complete.
- User message: edit and delete.
- Regenerate archives the previous branch. A "current / total" pager on the user message switches between variants.
- Edit is in place, with retry and cancel. Escape cancels; Mod+Enter retries.

**Right-click menus**
- On a message: copy, select text, and that row's actions.
- On empty transcript space: copy or select the whole conversation, go to top, jump to latest.

**Optional model badge** under the answer.

**Compaction marker**
- A centred, muted, line-free label after the last message that was summarised.
- It shows the compaction count and the token cost of the summary.

**Failure handling**
- A failed turn with no structured error ends in a small outcome card with a single **Continue** action, which sends a localised "carry on" prompt.
- A structured error card instead shows a stable error code with details open by default, copy, dismiss, and Continue.
- An aborted turn keeps its partial text with a muted "aborted" suffix. If nothing had started yet, the user message is withdrawn and the draft restored.

### 5.2 Live status and thinking display

**Working row**
- Before any output, a compact "Working…" row shows an elapsed timer and a small pulse.
- When the runtime reports a phase, the row names it:
  - starting
  - waiting for the model
  - preparing the request
  - compacting
  - recovering an empty reply
  - retrying, with the attempt number and cause
  - waiting for subagents, showing each one's current action
- Its height is reserved while a turn is running, so content never jumps.
- It hides while a permission, question or approval card is pending.
- Only a small marker pulses. The text never shimmers.

**Process disclosure**
- Each turn's reasoning, tool calls, searches and interim text fold into one disclosure labelled with the elapsed time ("processing for…" or "processed for…"). The final answer stays outside it.
- Adjacent activities group under a header with counts.

**Two display modes** (a user setting):
- **Detailed** (the default). The process is open while running. The active group opens, then closes when done, unless the user touched it.
- **Compact.** Everything starts closed and reasoning text is hidden behind a "thinking" indicator. A tool failure keeps the process open during recovery.

**Disclosure rules**
- Manual open and close always win, through streaming and completion.
- Closing a parent preserves the children's state.
- Opening a parent never expands everything.

**Thinking blocks**
- A light inline disclosure: a sparkle cue, a chevron, and secondary-coloured text with a faint left rule.
- Excluded from copy and search.

### 5.3 Tool-call rows

**Row anatomy.** A lightweight transparent row, not a card:
1. a type icon: file, folder, search, edit, terminal, web or generic
2. a plain-language verb ("Read", "Ran", "Searched"; "Reading…" and so on while active)
3. a one-line mono argument hint
4. result chips

**Result chips**
- exit code, shown in the error colour
- match and file counts
- lines read, with their range
- bytes written
- "truncated", shown only when this result was actually cut
- "scratch", for files written to the session's scratch area
- nothing extra on plain success

**Shell rows** add an outcome dot (done or failed) and a copy-command button.

**Expanded body.** Labelled blocks rather than raw JSON:
- file content
- written content
- compact diffs
- stdout and stderr
- clickable file lists
- grep matches grouped by file with line numbers
- errors listed first

Each block has its own copy button and an internal scroll cap.

**States:** running (a pulsing marker), success, error (a compact danger status, left collapsed), and denied (a muted "Denied").

### 5.4 Diffs and review

**Inline review card**
- Appears after every successful workspace Write or Edit.
- Shows a git-style status letter, the path, and added/removed counts.
- Expands to that message's own hunks.
- Offers a guarded per-message rollback. If the file changed since, the host reports a conflict rather than overwriting.
- Backed by a snapshot stored with the message, so it survives commits and restarts.
- Not shown for failed, denied, scratch, unchanged or non-git edits.

**Review tab** (in the work panel)
- The session's change history in chronological order: a summary bar, then collapsible rows that reuse the same cards.
- Opens only on request. Agent edits never pop it open.

### 5.5 Permission modes and approvals

**Two independent composer chips**
- **Operating mode:** Agent → Plan → Goal, cycled by clicking.
- **Permission mode:** Ask → Accept edits → Auto.

**Plan and Goal behaviour**
- Plan and Goal are contract states of the same agent, not a separate planner.
- Writes, edits and most plugin tools are hard-denied.
- Shell commands still follow the permission mode. The UI warns that a shell command can still change files.
- In Goal mode the permission chip shows a fixed "Auto".

**Permission card** (inline, in the originating session only)
- **Contents:**
  - a title, the tool name and its risk level
  - a short reason
  - a redacted preview of the arguments, shown as a command, code, or label/value pairs
  - the workspace
- **Actions:** Allow once (primary), Allow for this session (per tool name), and Deny.
  - There is no "allow always".
  - There is no countdown; the card waits indefinitely.
  - The first click locks the card into its resolved state, marked with a success or error edge.
- **Accessibility:** announced politely without stealing focus. Focus returns to the composer after a decision.
- **Composer:** while a request is pending, the draft stays editable but cannot be sent.

**Parallel subagent requests**
- Requests from several subagents queue; only the oldest is shown.
- The card says which subagent asked and how many more are waiting.
- Aborting denies the whole queue.

**Plan/Goal approval card** (docked in the composer area)
- **Contents:** the plan title and a button that opens the plan's Markdown file in the work panel.
- **Approve** requires picking the permission mode for execution. The last choice is remembered on this device.
- **Reject** stops the run and stays in planning mode.
- **States:** pending, resolving, approved, queued, running, rejected, expired (after 30 minutes), interrupted.
- **Composer:** while the card is pending, the composer is read-only.
- **Planning indicator:** the mode chip pulses twice, and a compact "Planning" row takes the working slot.

### 5.6 AskTool question card

The agent's structured multi-question prompt.

**Placement and look**
- Docks above the composer input, like the approval card, on a slim plate in the composer's style.
- Option rows and the custom-answer input sit on deeper inset fills.

**Header**
- A small label, the question number, faint progress, and clickable step indicators: answered, unanswered, skipped, current.
- "Decline all" is a quiet text link.

**Options**
- Radio semantics for single choice; checkbox semantics for multiple choice.
- Shown at menu-row density, with a small square mark.
- A free-text "other" answer is always visible.

**Actions:** Skip, Next, and Submit on the last question.

**Rich text and limits**
- Questions and options may contain Markdown, but links in options are inert.
- Long option lists scroll within roughly a third of the viewport, while the header and actions stay visible.

**Reset behaviour.** Switching chats resets the card to the first question.

### 5.7 Todo checklist

- **Placement.** A dock in the composer stack, above any approval surface, shown only when the session has items.
- **Collapsed:** progress counts and the current in-progress item.
- **Expanded:** up to eight rows, animated in with a height-and-fade transition.
  - done: success tint with a check
  - in progress: accent tint
  - cancelled: muted
- **Focus:** never steals it.

### 5.8 Subagents

**Delegation card** (in the transcript)
- A small graph: a "main agent" node linked to one node per subagent.
- Each node shows the name, model and reasoning level, a status capsule, elapsed time, a two-line description and a step count.
- The card header summarises completion counts and duration.

**Lifecycle rows** (wait, list, stop)
- Compact rows with a bot icon in a subagent accent colour, rolled up per agent.
- When expanded, they show a per-subagent table: status, runtime, turns.

**Subagent tab in the work panel** (opened by clicking a node; parallel subagents get separate tabs)
- A sticky identity header: avatar with a status dot, name, model caption, status and elapsed time.
- A "Task" card clamped to a few lines, with "show more".
- An "Activity" timeline rendered with the same components as the main chat. It updates live, follows new output, and offers jump-to-latest.

### 5.9 Queued prompts and steering

**Queue**
- While a turn runs, Send queues the draft. Stop shows only when the draft is empty.
- Queued prompts appear as rows above the composer, each with Remove and Send now.
  - Send now lets the current answer finish, then runs that prompt next.
  - The buttons show a saving state until the host has accepted the row.
- The queue runs in order, survives restarts, and is not cleared by Abort.

**Steering**
- Alt/Option+Enter injects the draft into the running turn. It appears as a user message and takes effect at the next model request without interrupting tools.
- If the turn has already ended, the draft is restored and an error is shown.

### 5.10 Context usage inspector

**Trigger.** A small capacity ring with a percentage on the composer toolbar, just before the model chip.
- Hidden until usage data exists.
- Shows remaining capacity by default; a setting switches it to used.
- Turns to warning at about a quarter remaining and to critical at about a tenth.

**Popover.** Click to open a non-modal popover with no internal rules, showing:
- used and window token counts
- this turn's total and its speed
- one compact line of model usage: input, output, cache, reasoning, and cache-hit rate
- one combined line of tool usage
- one muted line for the latest compaction checkpoint

**Compaction**
- Runs automatically, with no user settings.
- Can be triggered manually with /compact.
- Each compaction shows a warning toast suggesting a fresh session, plus the transcript marker.

### 5.11 Streaming and scroll

- **Streaming:** tokens append in place, with a subtle pulsing accent cursor.
- **Following:** the view follows the bottom while pinned there. The first deliberate upward scroll unpins it and shows a floating "jump to latest" button. Sending, retrying or regenerating always re-pins.
- **Long histories:** the newest window paints behind an opaque skeleton until the layout settles (with a short cap), so the view never visibly jumps. Older messages load in windows.
- **Long answers:** collapsed to a preview with "show more".

## 6. Composer

**Look**
- A floating plate with the largest corner radius tier, on a solid surface with a soft shadow.
- No blur, no gradient, no stroke. It lifts slightly on focus.
- On the empty home it sits in a reserved bottom slot. In a conversation it floats over the transcript's bottom edge, and the transcript fades out beneath it.

**Text area**
- About three lines tall when empty, growing to about seven, then scrolling.
- The placeholder rotates between a welcome, a "/ for commands, @ for files" hint, and a keyboard hint. It changes only when you navigate, never on a timer.

**Toolbar**
- **Left:** the operating-mode chip (fixed width, cross-fading labels) and the permission-mode chip.
- **Right:**
  - the context ring
  - a combined **model · reasoning** chip
  - the prompt-enhance button (sparkles)
  - one Send/Stop slot
- **Narrow widths:** the reasoning label drops first, then the model label shortens, then the chip goes icon-only. The toolbar never wraps.

**Model · reasoning menu**
- Opens upward. The root shows two entries, Model and Reasoning level, each with its current value.
- When a model has several reasoning levels, a drag slider sits under the entries.
- Choosing an entry swaps the menu in place to a submenu with a back row:
  - **Model:** search, with sticky provider headers.
  - **Reasoning:** a radio list.
- Picking returns to the root rather than closing.
- The levels (off, minimal, low, medium, high, xhigh, max) are deliberately not translated.

**While a turn runs.** Controls stay editable and apply to the next turn.

**No project open.** The composer warns that tools are limited.

**Slash menu**
- **Opens** when typing "/" at the start. An ideographic comma at the start of an empty draft also opens it, for CJK keyboards.
- **Groups:**
  - prompt templates (project ones first, then user ones, with argument hints)
  - app commands
  - plugin commands
  - skills, always last
- **App commands:** new, compact, agent-mode, plan-mode, goal-mode. The mode commands accept a trailing prompt.
- **"/" mid-text** offers skills only. Several skills per message are allowed and render as chips once sent.
- Matched characters are highlighted, and a footer shows keyboard hints.

**@ menu**
- File and folder names, with the full path in the tooltip.
- Choosing a folder keeps drilling down; choosing a file inserts a chip.
- Without a project open, it explains that one is needed.

**Attachments**
- Pasted or dropped files and images become chips. Images are accepted only if the selected model can take them.
- Pastes longer than a threshold become one "pasted text" chip, which can be expanded back into editable text. The threshold is a setting, a few hundred characters by default.

**Prompt enhancement**
- **What it does:** a one-shot rewrite of the draft using the current model.
- **Disabled** when the draft is empty, starts with "/", or no model is ready.
- **While running:** a spinner and an "enhancing" label.
- **Result:** replaces the text, with a single undo that disappears after you edit, send or switch.
- **Kept / discarded:** file chips are preserved. A stale response is discarded if the draft changed meanwhile.
- **Failure:** a dismissible bar under the composer.
- **Settings:** a template (which must contain a draft placeholder), a model override, and a reasoning level that defaults to off.

**Live Voice.** When enabled, one microphone icon opens call preparation (section 12).

## 7. Work panel (dock)

**Header**
- A horizontally scrolling tab strip, a fixed "+" button, and a quiet icon group on the right: maximise/preview and the panel toggle.
- **Tabs:**
  - each has an icon, an ellipsised label, and a close button (shown on hover or focus, always on the active tab)
  - keyboard: arrow keys, Home/End, Delete to close, Alt+Arrow to reorder
  - mouse: middle-click closes; press-and-drag reorders with drop indicators and auto-scrolls

**"+" launcher.** Opens a **New** tab listing destinations:
- the built-in **Review**
- every plugin-contributed view. The bundled **Files** and **Browser** are themselves plugin views.

Closing the last tab leaves the panel open on the launcher.

**Tab types**
- Review
- File preview: opened from transcript links, jumping to the line when one is given; can also open the file in the system app or reveal it in the file manager
- Browser: a native page embedded under plugin-drawn chrome
- Plugin views
- Subagent transcripts

**State rules**
- Panel state is per session: open/closed, tabs, active tab, browser page.
- Only explicit user actions open the panel. Background artifacts update state silently.
- Toggled with Mod+J. Disabled when there is no active session and inside Settings.

**No built-in terminal.** Shell output lives in the chat.

### 7.1 How plugins add UI

- **Work-panel views.** Declared in the plugin manifest and shown in the "+" launcher. Each is a native web view positioned over a rectangle the renderer measures.
  - The renderer is the visibility authority. It hides native views when their tab is inactive, during animations or drags, and under overlays.
  - Views can be told to open a specific location.
- **Detached panel windows.** Frameless, sandboxed windows.
  - The host draws only a transparent drag band and a small top-right minimise/maximise/close capsule. The capsule is isolated from plugin CSS and adopts the page's colours.
  - The plugin draws its own title and toolbar.
  - There are versioned "chrome contract" markers for how the page handles spacing.
- **Floating widgets.** Transparent, frameless, optionally always on top.
  - The whole surface drags.
  - Right-click shows a host-owned menu: close, minimise, always on top.
- **Settings destinations.** Plugins declare data only. The host renders those pages under a final "Extensions" group in the Settings rail.
- **Themes and theme variables.**
- **Commands**, which appear in search and in the slash menu.
- **Skills.**
- **Renderer UI slots,** for trusted plugins:
  - message action bars
  - a block under the assistant reply
  - custom tool cards for the plugin's own tools
  - code-fence renderers
  - composer controls and trigger lists
  - self-drawn overlays

### 7.2 Global plugin launcher

- **Trigger.** An OS-wide shortcut, Alt/Option+Space by default.
- **Window.** A centred, frameless, fixed-size utility window on the display under the pointer. It has no window controls and closes on blur or Escape.
- **Search.**
  - Covers enabled plugins that have a panel.
  - Matches on name, id, description, and Chinese pinyin (full and initials).
  - An empty query shows recently used plugins.
- **Speed.** Pre-created hidden at startup so it appears instantly.

## 8. Global search = command palette

**One dialog.** The standalone palette was merged into search. Mod+K and Mod+Shift+P open the same top-anchored dialog.

**Result groups:** sessions (by title, project and message content), pages, settings entries, and commands (built-in and plugin).

**Session results**
- Grouped by session with match counts.
- Up to two sentence-level snippets each, with sender, time and highlighting.
- A badge when the match is on the title or project.
- Archived chats appear only when searching.

**Opening a snippet** jumps to the exact message, highlights it, and loads history around it.

**Behaviour**
- Fully keyboard driven.
- Debounced, with stale results dropped.
- The last query is remembered.
- IME-safe: Enter and Escape do nothing while text is being composed.

**Built-in commands:** new task, compact context, and switch to Agent, Plan or Goal.

## 9. Keyboard shortcuts

Every shortcut can be rebound or unbound in Settings → Shortcuts. Mod means Cmd on macOS and Ctrl elsewhere.

**Global and app**

| Shortcut | Action |
|---|---|
| Alt/Option+Space | Plugin launcher (OS-wide) |
| Mod+K, Mod+Shift+P | Search / commands |
| Mod+N | New task |
| Mod+O | Open project |
| Mod+, | Settings |
| Mod+B | Sidebar |
| Mod+J | Work panel |
| Mod+[ / Mod+] | Back / forward |
| Mod+. | Abort the turn |
| Alt+Shift+W | Show/hide to tray (OS-wide) |
| Mod+Shift+V | Live Voice start/end |
| Escape | Cancel a connecting call; never ends a live call |
| F12, and Ctrl+Shift+I off macOS | DevTools (developer mode only) |

**Composer**

| Shortcut | Action |
|---|---|
| Enter | Send. If Enter-to-send is off, Mod+Enter sends instead |
| Shift+Enter | Newline |
| Alt/Option+Enter | Steer the running turn |
| Escape | Clear or blur |

**Transcript**

| Shortcut | Action |
|---|---|
| Mod+Up / Mod+Down | Jump to top / bottom |

**Rules**
- Never hijack clipboard, select-all or save.
- Ignore modifier-only presses and IME keystrokes.
- Reject reserved OS chords.
- Swallow plain reload (Ctrl+R) so it doesn't reload the page.

## 10. Settings structure

**Shell**
- A full-window takeover; the main sidebar is hidden.
- The left rail matches the sidebar's width and material:
  - a "search settings" pill at the top
  - "back to app" pinned at the bottom
  - grouped destinations with icons, separated only by whitespace
- Content: a large page title over elevated rounded cards of rows. Width is fluid. Each page opens scrolled to the top.

**Rail groups and destinations**

- **Preferences**
  - **General**
    - Appearance:
      - theme picker: searchable; System, Light and Dark first, then plugin themes labelled by their provider
      - language picker: searchable; "Auto" shows the detected language; each entry is listed in its own name and in English
      - UI font picker: installed system fonts; no fonts are bundled
      - font scale: four named presets plus a percentage slider; it scales text and icons
    - Storage: data location with a guided move, cache clearing, backup pruning.
    - Power: keep the system awake, keep the screen on.
    - Network: proxy (system, direct, or custom with a bypass list and a test button), and a relaxed mode that lets user-typed LAN/HTTP endpoints through.
    - Close behaviour.
  - **AI**
    - Default permission mode and operating mode.
    - Command shell choice. On Windows: PowerShell 5 or 7, cmd, Git Bash; elsewhere bash.
    - Where links open: the work-panel browser or the system browser.
    - Context display (remaining or used) and thinking display (detailed or compact).
    - Enter-to-send, infinite provider retry, and the paste-to-file threshold.
    - The prompt-enhancement card.
  - **Voice**
    - Live Voice enable switch, provider binding, model, voice and realtime profile.
  - **Shortcuts**
    - Click a binding to record a new one.
    - Unbind, or reset one or all.
    - Conflicts are shown inline. Key labels follow platform conventions.
- **Agent**
  - **Instructions**
    - The global instruction document. Per-project instructions are edited from the project menu.
  - **Models**
    - Default model.
    - Vendor accounts via OAuth: several per vendor, signed in by device code or link.
    - Provider studio:
      - add a provider
      - a two-pane model picker with per-model context window, output limit and thinking levels
      - test connection
      - custom headers and user agent
      - import configuration from other tools
  - **Skills, MCP and Subagents**
    - All three share one "workbench" layout:
      - Toolbar: a scope filter (all, global or project, with counts), search, a project picker, and primary actions.
      - Scope group headers that show the path.
      - Rows: a muted icon, name, scope and source badges, a one-line description and mono meta. Edit and overflow appear on hover; an on/off switch is always visible.
    - MCP rows show a connection dot.
    - Removing asks for confirmation on the second click.
    - "Scan other tools" imports inline.
    - Subagents are global only. There are five built-ins (explorer, reviewer, test runner, fixer, UI designer), and each can be copied to customise it.
- **Workspace**
  - **Projects archive**
    - Sort and search, with Pinned, All and Archived sections.
    - Rows: a colour glyph, name, short path, session count and last active.
    - Clicking a row opens an inline detail card: new task, open, folder/branch facts, the chat list, and the overflow actions.
- **System**
  - **Cloud sync**
    - WebDAV setup, category opt-ins, staged approval of incoming items, and detailed status.
  - **Remote hosts**
    - Developer mode only, marked experimental.
  - **About / Info**
    - Versions, open logs, report a problem, and updates with "what's new".
    - A developer card with the developer-mode switch.
- **Extensions**
  - Plugin-contributed settings pages, shown only when present.

**Shared controls**
- one switch component, with a neutral "on" state rather than green
- one segmented control
- one checkbox
- one anchored, searchable menu-select; native selects are not used
- editor sheets
- confirm dialogs for destructive actions

## 11. Visual system

### 11.1 Themes and palette character

**Themes:** System (the default), Light, Dark, and plugin themes. Following the OS theme never flashes white at launch.

**Dark**
- Near-black charcoal base. On macOS the sidebar is darker still and glassy.
- Raised surfaces step up in small grey increments; code insets sit darker than the page.
- Text runs from white through faded white to mid-grey.
- Borders are very faint white.
- The accent is inverted ink: white accent, dark text. There is no brand blue.

**Light**
- White base with a light-grey sidebar and near-black ink.
- Secondary text and borders are derived from the ink.
- The accent is near-black.
- Scrims are softer than in dark.

**Status colours.** A conventional green, amber/orange, red and indigo, used only for meaning.

**Elevation.** In dark, elevation is shown by lighter surfaces, not shadows. Light uses minimal shadows. Floating layers get a hairline plus a dialog shadow.

**Selection.** A light tint of the ink, not the browser's blue.

**Token families**
- backgrounds: primary, sidebar, secondary, tertiary, inset, composer, tiles, raised, elevated, hover, active
- text: four levels
- borders: two levels
- accent: base, hover, soft
- status colours
- switch colours
- glass tint
- shadows
- motion durations and easings
- radius ladder
- type scale, line heights, weights and tracking
- z-index layers

**Enforcement.** A lint script forbids raw typography and radius literals and arbitrary utility values. Everything must go through tokens.

**Plugin themes.** CSS files that override tokens. They are sanitised in the main process before injection, may declare typed runtime variables, and may set the window background.

### 11.2 Density

- Compact, built on a small base spacing step.
- List rows are short, roughly the height of a dense desktop list.
- The sidebar is tight, the gaps between messages are tight, and panels sit edge to edge.
- There is a hard cap on vertical gaps; no extra "breathing room" whitespace.

### 11.3 Typography

**Typefaces**
- UI: an Inter-like sans with system fallbacks and an always-appended CJK fallback tier.
- Code, paths and tool output: a JetBrains-Mono-like monospace.
- No font files ship. Users can switch the UI font to any installed font; the mono font is fixed.

**Size scale**
- A fine-grained scale, with half-steps from tiny badge text up to large page and hero titles.
- Body and chat text are deliberately a step smaller than the usual web default, a "workstation" size.
- Section labels are small, uppercase and widely tracked. Uppercase and tracking are relaxed for CJK.

**Weights.** Variable-font intermediates between regular and semibold. Nothing bold or heavier.

**Chat prose** has its own calmer heading ladder.

**Scaling.** A global multiplier scales the whole type ramp.

### 11.4 Corner hierarchy

**Ladder.** Radius grows with surface size and elevation, Apple-style. From smallest to largest:
1. inline code
2. chips
3. compact buttons
4. standard buttons, inputs and rows
5. menus
6. cards and code blocks
7. panels
8. dialogs
9. message bubbles and the composer, the most rounded

**Capsules and circles**
- Capsules: pills, badges, segmented controls, switches and scroll thumbs.
- Circles: icon buttons, avatars and dots.

**Nesting.** Nested corners are concentric: the outer radius equals the inner radius plus the inset.

**Edges.** Structural full-width panels stay square at the window edge.

### 11.5 Motion

**Durations.** Three tiers: fast for hover, normal for enter, expand, toasts and dialogs, slow for panels and the splash.

**Easing.** Ease-out on enter, ease-in on exit.

**Entrances**
- Pages, the dock and inline errors enter once with a small fade and rise.
- Dialogs add a slight scale.
- Search drops in from the top.
- Toasts slide and fade.
- Buttons press-scale slightly.

**Width changes.** The sidebar and dock animate their width so chat reflows smoothly.

**Streaming** never replays entrance animations.

**Bounded idle motion.** Status indicators pulse twice and then hold still. Only genuinely live work keeps moving: the streaming cursor, a running tool, the recording mic.

**Banned:** parallax, ambient animation (the mascot is the only exception), long shimmer, bounce, theatrical boot.

**Reduced motion:** near-instant transitions, a still mascot, a static boot bar, no pulses, instant scrolling.

### 11.6 Empty, loading and error states

**Empty states**
- Always pair a title with a next action: no sessions → new task, no provider → add provider, no project → open folder.
- Inbox: "all caught up".

**Loading**
- Shaped skeleton rows that mirror the real layout, with competing controls disabled.
- An opaque skeleton veil for long transcripts.
- Spinners on busy buttons.
- Settings that fail to load show an inline failure with Retry.
- Loading or failure is never shown as "empty".

**Toast vs inline**
- **Toasts** are for one-off results of explicit actions.
  - Top-centre, at most four stacked.
  - Short life, longer for errors. Hover pauses the timer, and there is always a close button.
  - A neutral surface with a tinted status icon.
  - Duplicates restart the existing toast instead of stacking.
- **Inline** is for anything tied to a message, tool or field, or for persistent conditions.
  - Examples: tool errors, denials, turn failures, validation, broken plugins, backend and update banners.

**Interrupted streams** keep the partial text and offer retry.

**Startup**
- A brief branded splash with a soft indeterminate bar. It has a minimum dwell to avoid a flash, then fades.
- A slow-boot hint appears after about 30 seconds.
- A recovery screen appears after a long timeout or a fatal backend state.

## 12. Onboarding, notifications, tray, voice

### 12.1 Onboarding

**Form.** An inline checklist on the empty home, not a wizard. It appears when no provider, no key or no session exists yet.

**Steps**
1. Add a provider.
2. Save a key.
3. Open a project.
4. Send the first prompt.
5. Optional: load a development plugin.

**Links.** Provider and key steps go to Settings → Models; the plugin step goes to Extensions; the project and prompt steps act directly.

**Lifecycle**
- Collapses once the core steps are done.
- Dismissal persists, but an unfinished critical step can return as a banner.

**Without a model**, Send is disabled and its tooltip says why.

**Out of scope:** sign-up, cloud setup, product tours.

### 12.2 Notifications

**Inbox**
- A popover from the bell, with an All/Unread filter, mark-all-read and clear.
- In practice it lists failures only. Successes are hidden from the inbox but still drive the sidebar check mark and native notifications.
- An entry is created only for turns that finish in the background or while the window is unfocused.
- Clicking an entry marks it read and opens the session.

**Native notifications**
- Sent only when the window is unfocused.
- Questions, permission requests and plan approvals each have their own alert.
- Clicking one focuses the app first, then opens the session.

### 12.3 Tray

- Always present; on macOS the icon is a monochrome template image.
- **Menu:**
  - Open
  - grouped Running, Unread and Pinned sessions: about nine in total, a few per group, with empty slots redistributed. Titles are capped by display width, so wide CJK characters count double.
  - View more
  - Quit, which asks for confirmation
- The menu is localised.
- Opening it doesn't mark anything read.

### 12.4 Live Voice

**Enabling.** Off until enabled in Settings → Voice. While off, no voice icon appears anywhere.

**Preparation** (the composer mic icon opens it)
- Shows the provider and whether it is ready, with a link to Settings if not.
- A context-sharing consent checkbox, off by default.
- Start begins muted, targeting the current session.

**During a call: the widget**
- A small separate frameless window, always on top and on every desktop.
- One-row status, growing sideways only.
- Controls: cancel while connecting; once connected, mute/unmute, end and details.
- Remembers its position.
- Shows a persistent "waiting" line when the bound session needs a decision.

**Details panel**
- In the main window: the transcript, the provider, the target, and operations with their results.
- Closing it does not end the call.

**Hard rule.** Voice can never approve permissions or plans. It may only pick one of a question card's own options.

## 13. Accessibility

**Focus**
- Visible, offset accent focus rings, shown for keyboard focus only.
- Search and dialogs trap focus. Escape closes and restores the previous focus.
- The composer is the default focus target after send, switch, abort and decisions.

**Roles and announcements**
- The transcript is a polite live log.
- Permission and approval cards are labelled regions, announced without stealing focus.
- Disclosures expose their expanded state. Hidden content leaves the tab order.
- Tabs, radios, switches and listboxes use proper roles.

**Icon-only buttons** all have localised accessible names and themed tooltips. Only one tooltip shows at a time.

**Status** is never conveyed by colour alone.

**Contrast:** at least 4.5:1 for body text and 3:1 for large text, in both themes.

**Text selection.** Chrome text can't be selected. Messages, code, tool output and toasts can.

**Input**
- Nothing fires during IME composition.
- Spellcheck and autocorrect are off in inputs.

**Reduced motion** is honoured everywhere.

## 14. Internationalisation

**Approach**
- English is the source language. The library is i18next.
- Every string has a stable dotted key.
- Every locale must have identical key sets and identical placeholder variables. Tests enforce this.

**Shipped locales (9):** English, Simplified Chinese, Traditional Chinese, German, Spanish, French, Turkish, Korean, Brazilian Portuguese.

**Locale registry**
- Each entry carries the language's own name and its English name.
- The picker is generated from the registry, so adding a language means adding a catalogue plus one registry row.

**Auto language.** Reads the OS locale through the main process.

**Locale-aware formatting.** Dates follow the app's language, not the OS.

**Not translated:** reasoning-level values, paths and ids.

**Also translated:** window controls, the macOS menu, the tray, the splash and crash screens, and changelog catalogues (one per locale).

**Plugins**
- Plugin names and descriptions are resolved in the host. Plugin manifests support only English and Simplified Chinese; other Chinese variants map to Simplified Chinese and everything else maps to English.
- Plugin-owned UI reads the host's locale.

**Copy tone.** Plain product language: "project", not "workspace". Errors say what happened and what to do next.

**CJK.** Widths are budgeted for both English and CJK labels.

## 15. Renderer organisation

This is a map for orientation, not a prescription.

**Feature areas**
- app shell: chrome, tray sessions, startup watchdog
- chat: composer, transcript, todos
- plugins
- projects
- scheduled
- sessions
- settings: about 60 components
- voice: live
- work panel: review, files, plugin view, subagent tabs

**State.** Zustand stores. They are in-memory projections of host state, never the authority.

**Rendering stack**
- React 19
- Tailwind 4, used as utilities over CSS custom-property tokens
- markdown via a remark/rehype pipeline
- Shiki for code highlighting
- KaTeX for math
- Mermaid for diagrams
- DOMPurify for HTML sanitising

**Styles.** Organised as one stylesheet per surface over a shared token layer, with a theme-override layer for plugin themes.

## 16. Inconsistencies in the source docs (pick one for Orphus)

- Some docs still describe a standalone command palette; it was merged into search.
- Starter cards were added and later removed. A leftover rule remains.
- One accessibility line mentions a permission countdown; elsewhere there is explicitly no countdown.
- The composer is described both with and without a hairline stroke. The latest intent is shadow only.
- Live Voice is described both as generally available and as gated behind developer mode.


---

# Part 2: Architecture

## 17. Process topology

```
 Renderer (React, sandboxed, no Node)
     │  preload bridge: allowlisted invoke / on
     ▼
 Electron Main  ── thin orchestrator ──────────────────────────────┐
  │   windows, tray, updater, IPC routing, child supervision,      │
  │   persistence outbox, logs, OAuth, MCP client, plugin broker,  │
  │   live-voice adapters, remote-host router                      │
  │                                                                │
  ├─ stdio NDJSON JSON-RPC ─►  Rust host-core (no network port)    │
  │                            SQLite, transcripts, tools,         │
  │                            permissions, secrets, plans,        │
  │                            config sync, audit                  │
  │                                                                │
  ├─ stdio NDJSON JSON-RPC ─►  Node agent sidecar (pi loop)        │
  │                            reaches host-core only via a        │
  │                            reverse "proxy" call that Main      │
  │                            filters with an allowlist           │
  │                                                                │
  └─ utilityProcess per plugin ─► plugin hosts (all API calls      │
                                  brokered back through Main)      ┘

 Optional extras:
   • Loopback MCP control server inside Main (env-gated)
   • Headless pi-host on a remote machine, reached over an SSH
     tunnel and spoken to with RACP over WebSocket
   • Live Voice sockets / WebRTC
   • Local dictation (whisper.cpp binding + native mic recorder)
```

### 17.1 Frozen rules

The project treats these as non-negotiable:
- The renderer never touches the database or Main's internals.
- SQLite belongs to the Rust core alone.
- Agent execution never moves into the renderer.
- Main stays thin.
- Shared contract packages never depend on app code.
- Plugin permission and sandbox boundaries cannot be bypassed.

Every cross-boundary message is schema-typed. They use TypeBox schemas in a shared package.

### 17.2 Rationale for the hybrid

The ADRs considered two alternatives and rejected both:
- **All TypeScript in Main.** This gave a weaker privilege boundary.
- **A Rust agent loop.** This cost too much and would have lost the pi ecosystem.

The split of authority that resulted:
- **Rust is authoritative for** OS capability, durable session mode and approval policy.
- **pi is authoritative for** model and agent-loop semantics.

### 17.3 Process responsibilities

**Electron Main** owns:
- window and tray lifecycle
- the IPC fan-in/fan-out
- child supervision with restart backoff
- a file-backed persistence outbox
- NDJSON logs
- OAuth login and refresh
- the MCP client for the user's MCP servers
- the plugin broker and permission gateway
- auto-update, against a fixed GitHub feed
- live-voice adapters
- the remote-host router

**Preload**
- A sandboxed CommonJS bundle that exposes a single object with `invoke` and `on`.
- Both methods are checked against a channel allowlist. Main checks the allowlist again.
- It also has helpers for dropped-file paths, platform and locale.
- It transfers a message port used for voice audio.

**Renderer**
- UI only.
- Its stores are disposable projections of host state.

**Rust host-core** owns:
- SQLite, with exclusive write access
- the transcript file store
- the builtin tools
- permission policy
- the durable per-session operating mode
- plan/goal artifacts and their approvals
- the shell catalogue
- the plugin registry and installer
- secrets
- config sync
- the audit log
- the notification inbox
- the turn queue
- the session collaboration ledger
- tool admission budgets

**Agent sidecar** owns:
- the pi agent loop
- provider streaming
- tool-call planning
- compaction
- system-prompt composition
- subagent delegation
- read/continue of native pi CLI sessions

**Plugin host**
- Runs one plugin's main module.
- It has no capabilities of its own: every API call is proxied to Main's broker.

### 17.4 Boot sequence

1. **Single-instance lock.** The lock is per data directory. A second launch focuses the first window and exits. Dev builds use a separate data root, and an environment variable can force a fully isolated profile.
2. Load English defaults.
3. Spawn host-core.
4. **Handshake.** It requires an exact protocol-version match (currently 11); on a mismatch the app shows the recovery screen. Before host-core serves anything, it runs one transaction that:
   - marks pending approvals and queued or running plan executions as interrupted,
   - aborts running turns.

   Nothing is ever replayed.
5. **Proxy settings.** Read them from host-core and apply them to Chromium sessions, Main's own fetch, and the sidecar's configuration.
6. **Spawn the sidecar.**
   - Packaged builds run the Electron binary itself in run-as-Node mode against a bundled sidecar script. No separate Node ships.
   - Dev builds use the system Node, version 22.19 or newer.
7. Wire the sidecar's tool bridge through Main.
8. **Create the window.** The renderer pings for the version. It shows a slow-boot hint at about 30 seconds and recovery at about 3 minutes, both above the default RPC timeout.

### 17.5 Supervision

This logic lives in a reusable package that the headless host also uses.

- **Restart policy.** Exponential backoff that caps at a few seconds, allowing at most 3 restarts per 2 minutes. After that the process is reported to the UI as fatal.
- **Generation tagging.** Restarts are single-flight and tagged with a generation number. Traffic carrying a stale generation is rejected.
- **Failing in-flight calls.** When a child exits, every in-flight call to it fails immediately with a host-unavailable code.
- **Deadlines.** Every RPC has a deadline, about 2 minutes by default. Bash, plugin tools, MCP and compaction get longer, method-specific deadlines.
- **Frame cap.** Frames are limited to 64 MiB.
- **Persistence outbox.**
  - Message appends made while host-core is down queue in a file.
  - After the next handshake they flush in order, before the host is reported ready.
  - Appends are idempotent by message id.
- **Named fatal boot states.** Each one gets its own recovery screen:
  - database schema newer than the app (a downgrade)
  - wrong CPU architecture (for example running under Rosetta)
  - Linux glibc below 2.35

**Crash policy by component:**

| Component that crashed | Response |
|---|---|
| Renderer | Reload; the backend is unaffected. |
| Host-core | Degrade, fail sessions closed, interrupt approvals, restart. |
| Sidecar | Abort turns, promote the last streaming checkpoint to an "aborted" row, classify the cause (OOM or other) from the stderr tail, restart without replay. |
| Main | App exit. |

Crash dumps are local only and never uploaded.

**Shutdown order.**
1. Reject new prompts.
2. Flush reply checkpoints, abort turns, drain the outbox. This step is bounded to about 2 s.
3. Interrupt plans.
4. Unload plugins.
5. Stop the sidecar.
6. Close the database.
7. Stop host-core.
8. Dispose the updater.
9. Exit.

Install-on-quit goes through the same path.

### 17.6 Host-core internals worth replicating

- **Dedicated stdio threads.** One OS thread reads and one writes, instead of async stdio adapters, so a busy async runtime cannot stall or kill the RPC pipe. Transient pipe errors are retried.
- **Bounded admission.**
  - There are separate concurrency caps for:
    - active RPCs
    - total tool executions
    - shell (global and per session)
    - read/search tools
    - mutating tools (global and per session)
    - plugin tools
    - per-session tools
  - There is also a bounded wait queue with a maximum wait.
  - Overflow returns a retryable "overloaded" error. Main retries only that error, a few times, with backoff.

## 18. Communication channels

### 18.1 Renderer ↔ Main (Electron IPC)

**Channel names**
- Request/response channels are named by application prefix, then domain, then action.
- Push channels add an event segment.
- Every reply is a tagged union: either success with data, or error with a code, message, optional details and a retriable flag.
- Long-running work streams as events and is never returned as one big response.

**Domains**

| Group | Domains |
|---|---|
| Core | app; agent (prompt, steer, graceful stop, abort, answer question, turn queue); plan; session (CRUD, fork, history windows, title summary, collaboration projection); settings |
| Security and projects | secrets (write, delete and exists only; reads never reach the renderer); project and project groups; tool permission responses |
| System and shell | shell; logs; plugin; command palette; workspace; embedded browser; read-only fs; window; menu; notifications; usage stats |
| Composer | file index, pickers, paste and import, prompt enhancement |
| Capabilities and services | speech; live voice; MCP servers; skills; subagents; config sync; remote hosts |

**Agent events pushed to the renderer**

Every event carries an envelope: session id, turn id, timestamp, and subagent attribution where relevant. The event types are:
- run start/end and turn start/end
- message start, update (text and thinking deltas), end
- tool start, update, end
- planning state
- permission request
- compaction start/end
- error
- a status event naming the quiet phase: starting, waiting for model, preparing, compacting, recovering, retrying, waiting for subagents

### 18.2 Host RPC: Main ↔ host-core, and sidecar ↔ Main

**Transport and framing**
- JSON-RPC 2.0 over the child's stdin/stdout, one JSON object per line, newline-delimited.
- Tolerant framing:
  - CRLF line endings are accepted.
  - Unicode line separators inside strings do not split frames.
  - A final frame without a trailing newline is accepted at EOF.
- Malformed frames are logged by size only, never by content.

**Errors and notifications**
- Errors carry a numeric code plus a stable string error code, such as "path outside workspace".
- Notifications (no id) cover permission prompts, streamed command output and sync changes.

**Handshake**
- The caller sends its protocol version, client identity and locale.
- The host replies with its version, feature list and capability list.
- Protocol history: the inbox, fork, compaction checkpoints and the frozen plan contract were each added in a successive version. A2A was added and then removed.

**Host method domains**
- Sessions and conversation:
  - app, workspace, projects
  - sessions (begin/end turn, append message, append compaction, replace, save in-flight checkpoint, fork, configure)
  - review snapshots, todos, stats, plans, scheduled tasks, search
- Tools and policy: tools, shells, permissions, keyboard (Windows hook)
- Extensions and capabilities: plugins and marketplace, providers and models, agent capabilities (skills, subagents, MCP definitions)
- Storage, security and sync: secrets, artifacts, audit, notifications, config sync

**Sidecar ↔ host**
- The sidecar never talks to host-core directly. It sends a reverse "proxy" request to Main.
- Main forwards only an allowlisted set:
  - tool execute, abort, list
  - session get, append, append compaction, replace
  - workspace get
  - plan enter, submit, pending, abort
  - instruction resolution
  - provider auth resolution
  - health
  - a few session and extension operations used by trusted extensions
- Some proxied calls are answered by Main itself: local tools, vendor OAuth binding, the extension bridge.

**Methods the sidecar exposes to Main**
- configure, health
- native-session list, search, get, fork
- prompt, steer, execute approved plan, compact, abort, stop
- answer question, run extension command, status, dispose session

### 18.3 Plugin host channel

Messages are one JSON object per frame.

| Direction | Message kinds |
|---|---|
| Main → plugin | init (with manifest and path); calls (command run, tool execute, service start/stop, unload); per-invocation cancel; event push |
| Plugin → Main | host-API calls, checked by the broker against the API allowlist, then the permission gateway, then executed and audited; results; log lines |

### 18.4 Local MCP control plane (opt-in)

- **Activation.** Off unless an environment flag is set.
- **Transport.** Streamable HTTP MCP on 127.0.0.1. Origin checks prevent DNS rebinding.
- **Auth.** A random bearer token plus a connection manifest (URL, token, PID), written owner-only into the data directory.
- **Tools.** Named tools for common flows, plus a generic "invoke" over a reviewed, risk-tagged catalogue of the same IPC handlers the renderer uses.
- **Restrictions.**
  - Secrets, settings writes and native pickers are excluded.
  - Dangerous operations need an explicit confirm flag.
- **Used by** the E2E harness and by external agents driving the app.

### 18.5 RACP: Remote Agent Control Protocol

**Purpose**
- Lets an external client attach to a running agent host and drive it remotely: another desktop, a phone or a web client.
- The agent host is a headless module that owns session and turn admission, the turn queue, the approval broker, and a sequenced in-memory event log with snapshots.
- RACP is deliberately not:
  - a public form of Electron IPC
  - the sidecar proxy
  - host-core RPC
  - a provider proxy
  - the MCP control plane

  Host-core still binds no port.

**Transports**
- WebSocket is the normative binding: JSON-RPC 2.0, one message per text frame, a dedicated subprotocol name, and keep-alive pings.
- An HTTP+SSE browser profile and a gateway "host link" (multiplexed over an outbound mTLS socket) are specified but not built.
- gRPC is reserved.

**Initialisation**
- Negotiates the protocol version, bindings and capabilities: event replay, approvals, input requests, attachments, turn queue, host events, history, tool relay, terminal, remote-host profile.
- Also negotiates limits: frame, prompt and attachment sizes, subscription count, queued turns, replay window.
- The host advertises a policy: a ceiling on remote permission mode (default "ask") and an approval lifetime.

**Message categories**

| Category | Operations |
|---|---|
| Connection | initialize, ping, pair |
| Discovery | list hosts (gateway only), list/register/browse projects |
| Session | list, get, create, attach (returns a snapshot), history, archive, revoke; the remote-host profile adds configure, fork, rename, delete, compact |
| Events | subscribe from a cursor, unsubscribe, ack |
| Turn | start (immediate or queued), get, graceful stop, interrupt, cancel queued, prioritise |
| Server → client requests | approvals and input questions; clients answer with respond calls |
| Attachments | create, then complete with hash and size verification |
| Tool relay | clients advertise tools they execute themselves |
| Workspace | bounded, ignore-aware list, read and diff |
| Terminal | open, input, resize, close; backed by an optional pty on the host |

**Event cursor semantics**
- A cursor is an epoch plus a sequence number.
- Durable events are replayable. Ephemeral events (deltas, activity) never are.
- Eviction or an epoch change forces a resync rather than silently losing data.
- A slow consumer loses ephemeral events first, then is disconnected.

**Roles:** viewer, controller, approver, owner.

**Auth and pairing**
- A bearer token goes on the WebSocket upgrade. Tokens in URLs are refused.
- There are two token kinds, distinguished by prefix:
  - a long-lived device token
  - a single-use, short-lived pairing token
- Tokens are stored hashed and compared in constant time.
- A pairing connection may only call "pair", which mints an owner device token.
- Plaintext WebSocket is allowed only when both ends are loopback. The SSH tunnel provides confidentiality.

**SSH bootstrap**
1. The desktop runs the system ssh client in batch mode. The app stores no SSH secrets.
2. It uploads a bootstrap script. The script:
   - downloads the pi-host tarball matching the desktop's version from GitHub Releases,
   - verifies its published checksum,
   - installs it under the remote home directory,
   - starts the host on loopback in pairing mode,
   - prints readiness and a pairing token.
3. The desktop opens a local port forward and exchanges the pairing token for a device token.
4. Provider configuration is written over SSH, never over RACP.
5. Published remote targets are Linux x64 and arm64.

**Desktop routing**
- A router inside Main's IPC registration sends session ids carrying a "remote:host:session" prefix to a RACP-backed backend. All other ids use local handlers unchanged.
- Approval ids encode their session, so replies route without a correlation table.
- The renderer is transport-agnostic.

**pi-host CLI options:** data dir, port (0 = any), loopback bind, pair mode, pairing lifetime, host-core path, sidecar entry, browse root, log level. It runs on plain Node.

**Withdrawn:** an agent-to-agent peer messaging layer was built and then removed. Subagents now coordinate only through parent-collected task reports.

## 19. The agent loop

**Where it runs**
- Only in the Node sidecar.
- There is one pi agent per durable session.
- Plan and Goal change that agent's tools and prompts; they do not add a second planner.

**Runtime API**
- prompt, which returns a turn id
- steer, which validates the expected turn and injects at the next model-request boundary using the pi core's native steering queue
- graceful stop at the next turn boundary
- abort
- status
- subscribe to normalised events
- dispose

**Prompt pipeline**
1. Load the session from host-core: mode, provider, model, project.
2. Resolve model metadata and clamp the thinking level to what the model supports.
3. Check the credential.
4. Reject if busy. The renderer uses the host turn queue instead.
5. Main has already validated attachments and stored image bytes content-addressed.
6. Pin the shell for the turn.
7. Run the pi turn, with bounded silent retries: about ten for rate limits and about ten for transient errors.
8. Stream normalised events.
9. Execute every tool call through host-core with the durable session id.
10. Persist failures as visible assistant error rows.

**Features layered on pi**
- **Compaction.** Durable checkpoints, background compaction at turn boundaries, a fallback to a recent window plus a chunked summary, and summary retry.
- **Silent-turn recovery.**
- **Deferred tool loading.** Optional, plugin and MCP tools sit in an on-demand catalogue that the model activates through a tool-search tool.
- **System prompt assembly.**
  - Inputs: a chronological system-state journal, the user's custom prompt files, the project instruction chain (AGENTS.md style, walked up the tree) and project-group memory.
  - A metadata-only skills catalogue. Skill bodies are fetched on demand by a Skill tool.
- **Subagents** behind a Task tool:
  - bounded workers with inherited or scoped permissions
  - parent-judged lifetime, event-driven timeouts, an output token cap and a context budget
  - reports collected through a wait tool
  - resumable delegations
  - background delegation
  - five built-in subagent profiles, each of which can be disabled
- **Session titles.** A one-shot summarisation with thinking off.
- **Provider plumbing.** Per-provider user agent and headers, system certificate trust, and a proxy relay supporting authenticated SOCKS5/HTTP.

**Turn state machine**
- Session status: idle, running, waiting for permission, or a terminal aborted/error state.
- A turn moves through accept, start, stream, an optional tool loop (permission check then execute), and end.
- The terminal reason is decided exactly once, keyed by turn id, so stale terminal events are ignored.
- Ending a turn is transactional and creates at most one inbox notification.

**Plan and Goal ("contract modes")**
1. The agent writes an immutable Markdown artifact into a hidden folder in the workspace.
2. Host-core records its path, hash and size in an approvals row.
3. Approval atomically switches the session to Agent with the chosen permission mode and queues an execution turn.
4. Rejection, a 30-minute expiry, or a crash return the session to planning and grant nothing.

## 20. How the pi packages are used

All are pinned to **exactly 1.0.1** and patched.

| Package | Capabilities it provides | Used where |
|---|---|---|
| pi-ai | Provider transports and adapters (Anthropic Messages, OpenAI Completions/Responses, Azure, Codex, Mistral, Google, more); built-in provider/model registry; OAuth flows and a credential-store interface; message/context types; retries; token estimation; stream framing | Sidecar streaming; in Main for catalogue merge, OAuth credential store, plugin one-shot completions, image generation, the live-voice work bridge |
| pi-agent-core | The agent loop (prompt/continue, events, tool contract, steering queue, system-message handling) | Sidecar runtime and subagents. The desktop supplies its own compaction, prompt templates and context projection |
| pi-coding-agent | Session manager (JSONL v3), native compaction and branch summarisation, trust manager, resource loader, extension API | Transitional. Limited to (a) reading and continuing the user's existing pi CLI sessions, kept opaque and never imported into SQLite, and (b) the trusted-extension compatibility shim. Slated for removal |
| pi-mcp | MCP support | Dev dependency of the desktop |

A durable-storage pi add-on was evaluated and rejected because it conflicts with the rule that only Rust writes SQLite.

### 20.1 Patches to pi 1.0.1: one line each

**pi-agent-core**
- Forwards provider-hosted web-search progress through the loop as streaming updates, so each search round renders live.
- Failures that originate locally, not at the provider, get a structured, fixed-wording error description instead of raw exception text.

**pi-ai**
- **Hosted web search, Anthropic.** Attaches the server-side search tool when the model opts in. Captures search calls and results whole, including encrypted content. Streams the query live, collects citations, and replays the blocks verbatim on later turns.
- **Hosted web search, OpenAI / Azure / Codex Responses.** Attaches the search tool, requests sources, keeps search items in place for replay, emits status transitions, and collects URL citations.
- **Mistral.** No longer replays hosted-search blocks as function calls.
- **Responses streams.** Stop at the terminal event, so a connection the server never closes can't hang a request.
- **Local request-preparation errors** (context resolution, validation, serialisation) are tagged as local with structured details, not mistaken for provider or network errors.
- **DeepSeek-style reasoning replay on OpenAI-compatible endpoints.** Keeps reasoning-only messages, normalises the reasoning field across the whole history, and adds a compat flag for relays that need a placeholder.
- **Token estimation.** Model-aware, covering every role and the hosted-search replay. Timestamp-anchored usage tolerates malformed timestamps. Validation now runs before the fast path.
- **Stream frame codec.** Tolerates gateways that drop empty delta fields, deep-copies tool-call fields to prevent later mutation, and handles hosted-search snapshot frames.
- **Anthropic OAuth token endpoint.**
  - One overall deadline.
  - Retries only on explicit rate limits, honouring the server's wait. Never retries after network errors, 5xx or invalid-grant, since the grant may already be spent.
  - Never logs token bodies.
- **Bundled catalogue.** Marks a DeepSeek flash model as accepting mid-conversation system messages.
- **New exports** for the above: hosted-search helpers and types, citations and error details on messages, a web-search flag on models, the new compat flag.

**pi-coding-agent**
- Compaction, branch summaries, thresholds, cut points and bug-report sizing pass the active model to token estimation.
- Context estimation delegates to pi-ai's estimator. It validates hosted-search history first and throws on unknown roles instead of silently counting zero.
- Conversation serialisation for summaries includes hosted-search blocks as a labelled replay projection.

A CI script verifies that the installed pi packages are exactly the pinned, patched versions by comparing patch hashes.

## 21. Tools and permissions

**Built-in tools (executed by host-core)**
- **Read, Glob, Grep.** Grep uses ripgrep if installed, otherwise an in-process search.
- **Write.**
- **Edit** (high risk). Line-anchored against a content tag returned by Read, not a string search and replace.
- **Bash** (high risk).
  - Non-interactive, using the selected catalogue shell.
  - Default timeout of about a minute; the agent can override it from 1 s to 6 h.
  - stdout and stderr stream separately.
  - The whole process tree is killed on timeout or abort.
  - On Unix, PATH comes from the user's login shell.

**Control tools**
- the question form (AskTool)
- the todo list
- new context
- browser preview
- enter/submit plan and enter/submit goal
- tool search
- Skill
- Task (subagents)

**Extension tools**
- Plugin tools.
- MCP tools, carrying a namespaced name.

**Path policy**
- Paths are canonicalised and must stay within the session's project roots.
- An explicit outside path needs permission.
- Each session has a scratch folder where writes never prompt.
- Every Write/Edit stores a bounded before-snapshot outside the workspace. This snapshot powers per-message review and rollback.

**Risk and decisions**
- Risk tiers: low auto-allows; medium and high confirm by default. MCP tools are always medium, because a server's self-declared risk isn't trusted.
- Decisions: allow once, allow for the session (in memory, keyed by tool name), or deny.
- Local approvals never time out.

**Permission modes**
- Resolution order: the session's mode, then the global default, then Ask.

| Mode | File writes and edits | Shell and plugin tools |
|---|---|---|
| Ask | Confirm | Confirm |
| Accept edits | Auto | Confirm |
| Auto | Auto | Auto |

- Enforced only in host-core. The model is never told which mode is active.
- Plan and Goal hard-deny writes, edits, unknown tools and non-plan-safe plugin tools before mode evaluation. Shell still follows the mode, so Plan is an intent, not a sandbox.

**Audit** records every call: session, turn, call id, tool, an args hash, the decision, timings and the outcome.

## 22. Persistence

**Ownership.** Rust is the sole writer of SQLite and of the transcript files. Everything else goes through RPC.

**Data root**
- Packaged builds use a hidden folder in the user's home. Dev builds use a separate one.
- An environment variable overrides the location.
- Users can move the data to another volume. The move is a cold, journaled migration with checksum-verified copies, plus an offline path-relocation mode.

**Layout** (described, not named):

| Area | Contents |
|---|---|
| Index database | SQLite in WAL mode, with versioned backups taken before destructive migrations |
| Sessions | One JSONL transcript per session: a header line, then one line per message (canonical content blocks), plus compaction checkpoint lines. Alongside it, an append-only regenerate history and a transient in-flight reply file |
| Secrets | Encrypted blobs and a machine key |
| Config sync | Encrypted base and pending bundles |
| Attachments | Content-addressed by SHA-256 |
| Plugins | Installed code, private data, logs, marketplace cache and the registry |
| Logs | Main's log files |
| Cache | Disposable cache |
| Crash dumps | Local crash dumps |
| Review snapshots | Before-snapshots for review and rollback |
| Scratch | Per-session scratch folders, with orphans swept |

**Transcript rules**
- SQLite is an index, not the payload store.
- The JSONL file is written before the database transaction, so a crash loses at most an index row and the next rewrite heals it.
- Appends are fsynced. Rewrites go through a temp file and an atomic rename.
- Readers skip unknown line types and a torn last line.
- Main checkpoints the streaming reply every second or two. A boot sweep turns leftovers into completed or aborted rows.

**SQLite**
- WAL journal, normal sync, foreign keys on, busy timeout, incremental vacuum.
- The schema version lives in the database's user-version field (version 17 at this release).
- Every change requires a migration plus tests.
- Boot maintenance:
  1. Run the interruption fence.
  2. Vacuum.
  3. Prune the audit log.

**Schema areas**
- **Settings store:** a namespaced key/value table. It also holds project groups, memory, instructions and sync settings.
- **Core records:** projects, providers (each can be owned by a plugin), and a cache of models.
- **Sessions:** mode, permission mode, provider, model, thinking level, project binding, soft delete, plugin ownership.
- **Turns.**
- **Plan approvals.**
- **Turn queue:** a host-owned FIFO.
- **Collaboration ledger.**
- **Messages:** an index carrying subagent attribution.
- **Full-text search index.**
- **Message revisions.**
- **Artifacts.**
- **Scheduled tasks and their runs.**
- **Secret metadata.**
- **Audit log.**
- **Notification inbox.**
- **Usage ledger:** one row per physical operation, used for token history.

**Native pi sessions** are a second, read-mostly authority. They stay in pi's own session folder as JSONL, are owned by the sidecar, and are never imported. A fork writes a new file atomically.

**Ephemeral state.** The remote agent host's event log lives in memory only. A restart starts a new epoch.

**Renderer-local state** covers only conveniences, such as launcher recency.

## 23. Credential storage

**Ownership**
- Host-core owns secret write, read and delete.
- The renderer only ever sees whether a key or OAuth login exists, plus a non-secret account label.

**References.** Each provider row has one reference for its API key and a separate one for its OAuth credential. The OAuth credential is pi-ai's serialised credential.

**Runtime access**
- The sidecar fetches a secret per run, internally. Each fetch is audited, and the secret stays in memory only.
- OAuth refresh is serialised per provider inside Main, which implements pi-ai's credential-store interface on top of host secret RPCs.
- Refresh tokens never leave Main. The sidecar only gets short-lived request auth.

**Spec vs shipped code**

| | Approach |
|---|---|
| Spec | OS keychain via Electron's safeStorage first, encrypted file as fallback |
| Shipped code | Encrypted file only. A random 256-bit machine key lives in an owner-only file. Each secret is a separate AES-GCM file named by a hash of its reference, with a fresh nonce on every write. There is no keyring dependency. |

**For Orphus:** decide explicitly. The recommendation is an OS keychain (libsecret, Keychain, DPAPI) with the encrypted file as fallback.

**Redaction**
- Logs mask auth headers, keys, tokens, sensitive query parameters and token-shaped strings.
- Audit writes are redacted in Rust.

## 24. Config sync (portable, encrypted)

**Path:** renderer → Main (bridge) → host-core → a WebDAV server. The sidecar plays no part.

**What syncs**
- preferences
- providers
- MCP definitions
- skills
- global subagents
- the global and per-project instruction files
- projects and groups, by logical id rather than path
- plugin install intent
- automations
- optionally, project memory

**Secrets**
- Synced only on explicit opt-in: API keys and MCP env/header values.
- Never synced: OAuth sessions, cookies, machine keys, WebDAV credentials, plugin binaries, transcripts.

**Encryption**
- Each vault has a data key, wrapped with AES-GCM under an Argon2id key derived from a separate backup password.
- Each object is encrypted with:
  - a purpose-derived key,
  - domain-separated associated data,
  - a fresh nonce.
- Resource ids are keyed content ids.

**Write protocol**
- **Strict mode:** compare-and-swap on a head object using conditional HTTP headers. Server support is verified by a probe.
- **Append-only mode:** an explicit alternative for servers that lack conditional writes. It keeps per-device pointers and merges their tips.
- **Transport rules:** HTTPS is required, except for acknowledged plain HTTP on a LAN. Redirects are refused.

**Merging**
- A three-way merge against the last acknowledged base.
- Explicit tombstones mark deletions. A conflict keeps both versions.
- Imported executable content (MCP, skills, subagents, plugins, automations) is staged until the user approves it on this device. The approval is bound to the content's digest.
- Imported automations arrive disabled.
- A durable encrypted pending bundle lets an interrupted apply recover after a crash.

## 25. Logging and observability

**Channels and categories**
- Main writes NDJSON logs for three channels:
  - app (Main itself)
  - host (host-core's stderr, wrapped into records)
  - agent (sidecar stderr)
- Each channel is split by category: lifecycle, session, tool, permission, plugin, provider, persistence, updater, diagnostics, runtime.
- Audit goes to the database. Each plugin has its own log file.

**Record fields**
- timestamp, level, channel, category, a stable dotted event name, and a message
- correlation ids: trace, request, session, turn, tool call, parent call, agent, plugin
- an error code
- a data payload that is redacted, bounded and capped at a few KiB

**Retention**
- Size-based rotation, keeping two old files.
- A logging failure never fails the operation that was logging.

**Viewer.** An in-app log viewer can filter by plugin.

## 26. Providers and the model catalogue

**Source of truth.** Model metadata comes from models.dev. A snapshot ships with the app, startup reads only that snapshot, and the network is used only on an explicit refresh.

**Listing a provider's models**
1. Query the provider's live models endpoint with the stored key. Results are enriched from models.dev and cached.
2. If that fails or returns nothing, use the vendor's catalogue.
3. As a last resort, use the configured model.

OAuth accounts use the account's entitlement list.

**Picker merge.** Entries are deduplicated by provider plus model id. Priority, highest first: user entries, models.dev, discovered models, recents.

**Metadata rules**
- Capabilities (tools, vision, reasoning levels, context and output limits) come from the catalogue.
- An unknown model id gets generic metadata, never a sibling model's.
- Offline, cached and configured ids stay usable, and custom ids are always allowed.

**Provider tiers:** always-shown vendors, vendors shown when pi supports them, and user-defined providers. Plugins can also declare providers.

**Image generation** is bound to its own model and runs in Main.

## 27. Voice runtime

There are three separate subsystems.

### 27.1 Local dictation

**Where it runs.** In Main.

**State machine:** prepare → start → listen → stop → transcribe → result.

**Components** (all optional dependencies, unpacked from the app archive)
- Capture: a native microphone recorder library.
- Transcription: a whisper.cpp binding using downloaded Whisper models.
- Chinese script conversion.

The same capture path also backs the plugin audio-capture API.

### 27.2 Host speech capability

**Operations**
- transcribe: a file in, text out
- synthesise: text in, an audio file in scratch out

**Bindings.** One per role, each naming a provider, model and protocol.

**Built-in protocols**
- an OpenAI audio REST protocol
- a chat-completions protocol that returns audio

**Plugin adapters.** Plugins can register their own adapters. Their HTTP plans are executed by the host with the bound key and must stay on that provider's origin.

**Constraints**
- Audio bytes never enter the renderer.
- There is no settings UI; it is consumed by IPC and plugins.

### 27.3 Live Voice (realtime)

**Default:** off.

**Adapters**

| Adapter | Media path | Credential |
|---|---|---|
| Codex Live | WebRTC in the renderer; Main negotiates the session | Existing Codex OAuth |
| Gemini Live | WebSocket in Main | Provider key |
| OpenAI Realtime | WebSocket in Main | Provider key |

For the two WebSocket adapters, PCM audio crosses a message port from preload.

**Responsibilities**
- Main owns:
  - the single call slot
  - credentials
  - protocol adapters
  - transport
  - the microphone lease, shared with dictation and quarantined if a release is unconfirmed
  - cleanup on suspend, lock, crash or quit
- The renderer owns the media and an in-memory transcript.
- Nothing is recorded or persisted.

**Delegating work.** The voice model gets a single tool for delegating work:
1. It returns a receipt immediately.
2. A read-only one-shot classifier, running on the target session's model with tools disabled, decides whether the request is:
   - conversation,
   - steering the running turn,
   - queueing a new task,
   - or answering one open question by option label.
3. All resulting actions go through normal admission and permission checks.
4. Approvals stay in the desktop UI.

## 28. Plugin runtime

### 28.1 Package format

**Directory form (development).**
- A manifest at the root, plus optional:
  - main module
  - renderer assets
  - skills
  - themes
  - views
  - icon
  - README

**Distribution form.**
- A zip with its own file extension. Entries must be stored, not compressed.
- An optional per-file checksum list.

**Install guards.**
- The manifest must be at the root.
- No absolute symlinks and no path traversal.
- About 50 MB and 2000 files at most.
- Packages are staged in a temp area and committed atomically, with backup on upgrade.

### 28.2 Manifest

**Core fields.**
- schema version, reverse-domain id, name, semver version
- description, author, homepage, repository, icon
- i18n (EN and zh-CN)
- main module
- ui (panel or widget shape, size, resizable, title, always on top)
- contributions
- permissions
- file scopes and network domains
- engine range, lifecycle entrypoints, activation events
- default-enabled flag (bundled plugins only)
- renderer extension fields

**Contribution points.**
- commands
- agent tools: name, description, risk, JSON schema, timeout
- skills
- agent extensions
- providers
- settings: typed, including shortcut type
- themes and scenic (appearance) themes
- window appearance
- MCP servers: stdio or HTTP
- resident services
- message-bus publish/subscribe topics
- work-panel views
- session sources
- global shortcuts

**Validation.**
- Every contribution requires its matching permission.
- Unknown permissions fail validation.
- Write and delete scopes may not be whole-tree globs.
- Network domains must be bare hostnames, with optional wildcard subdomains.
- Bus topics are bounded, dot-segmented names.
- The validator is mirrored in TypeScript (the SDK) and Rust (the installer).

### 28.3 Isolation model

**Plugin main logic.**
- Each plugin runs in its own Electron utility process, with a minimal allowlisted environment that carries no provider keys.
- An injected API global turns every call into a message to Main's broker.
- The broker checks the API allowlist, then the permission gateway, then executes and writes an audit record.

**Known gap.** The child process can still reach raw Node built-ins, so permissions only bound the brokered APIs.
- There are no CPU or memory limits.
- Marketplace code is effectively user-privileged.
- A true capability sandbox is the stated target but is not built.

**Orphus could improve here,** for example with a Node permission model, a Deno-style runtime, or WASM.

**Timeouts.** Load, lifecycle hooks, commands and tools each have their own budget, ranging from a few seconds to a couple of minutes.

**Crash handling.**
- Pending calls are rejected.
- The plugin's contributions are deregistered and its panel closed.
- The crash is audited and the user sees a toast.
- A clean quit is not counted as a crash.

**Panels and views.**
- Sandboxed web contents with context isolation.
- A per-plugin persisted session partition.
- Network egress filtered to the declared domains.
- A fixed bridge in which each channel maps to a required permission. Channels the host does not own are forwarded to the plugin's own handler.

**File access: four ordered gates.** Each later gate can only refuse, never widen:
1. The permission is both declared and granted.
2. Real-path containment within the root.
3. An unconditional deny-list (secret files, VCS internals, host paths).
4. A scope match. Anything outside the scope triggers a native user prompt.

**File access roots and deletes.**
- Roots are the calling session's project, or a user-picked folder held in memory only.
- Deletes go to the OS trash, are never recursive, and are rate-limited.
- A delete is allowed only for files the plugin created (tracked in a ledger) or for explicitly scoped paths.

**Network.**
- Declared domains bound every host-mediated egress path.
- An "any host" permission lifts the list but still blocks cloud metadata addresses.

**Storage.**
- No access to the host database, secrets or keys.
- A private data folder and its own log file.
- Settings are kept in a per-plugin key/value namespace.

### 28.4 Permissions

There are about 40 flat permission strings, each with a risk tier.

**Low risk.**
- panels, views, themes, window appearance
- notifications

**Medium.**
- clipboard, opening external links
- background services, the message bus
- listing models, reading usage
- the plugin's own sessions
- global shortcuts, the panel microphone

**High.**
- writing and deleting files
- registering agent tools, injecting prompts or skills, agent extensions, registering providers
- network fetch and websockets, the any-host permission
- MCP servers
- browser DevTools protocol (filtered to an allowlist of methods)
- desktop control
- creating projects
- reading and importing sessions
- one-shot completions
- registering speech adapters
- background audio

**Grant flow.**
- Declared permissions are granted at install unless the user unchecks them in the review dialog.
- Any upgrade that adds permissions shows a diff.
- Revocation takes effect on reload.

**Plan mode.** Plugin tools are invisible in Plan mode unless they declare plan-safe actions.

### 28.5 Lifecycle and supervision

**States.**
- discovered, validated, installed, enabled, loaded, running
- disabled
- error variants

**Load and disable.**
- Load is transactional: register commands, tools, skills and themes; connect MCP lazily; commit; start services. Any failure rolls the whole load back.
- Disable reverses everything.
- One failing plugin never blocks the others.

**Resident services.**
- Restart with exponential backoff and a few attempts before "failed". The counter resets after a minute of stability.
- Per-service status chips appear in the UI.

**Message bus.**
- Routed by the broker; fire-and-forget; publishers don't receive their own messages.
- Rate-limited and size-capped.
- Payloads are data only, never capabilities.

**Dev mode.**
- Picking a folder returns a permission review. Confirming sets a permission ceiling.
- A debounced file watcher reloads on change.
- A reload can never widen permissions silently. A broken reload stays watched, so the next good save recovers it.

### 28.6 Agent extensions (trusted)

**What they are.** Modules written against pi-coding-agent's extension API.

**Where they run.** Inside the agent sidecar, through a desktop-owned runner per session, loaded with a just-in-time TypeScript loader.

**API coverage.** Each API member is classified as supported, deferred or unsupported. Unsupported members are inert and emit diagnostics.

**Gating.** They need the high-risk agent-extension grant and are not distributable through the marketplace.

**Importing a pi extension.**
1. Copy it into a managed folder.
2. Generate a manifest.
3. Install its dependencies with npm, lockfile only, with lifecycle scripts disabled.
4. Register it as a dev plugin.

### 28.7 Renderer extensions (trusted, not a security boundary)

**What they are.** An ES module running in the host window's realm.

**What they can do.** Register UI slots and dispatch from a declared action vocabulary:
- insert into the composer
- read or replace the draft
- attach files
- call into the plugin process

**Cleanup.** Every registration returns a disposer.

### 28.8 Bundled plugins

**Policy.**
- Shipped as resources.
- Enabled by default.
- They can be disabled but not uninstalled.
- The marketplace may update them.

**The bundled set.**
- **Browser:** a work-panel view plus a Browser agent tool. It uses the DevTools protocol against a host-owned guest page.
- **Files:** a file manager and preview view.

**Also distributed as plugins.** The session orchestrator and token-usage dashboard are ordinary marketplace plugins.

### 28.9 Marketplace and plugin center

**Marketplace.**
- A distribution layer only. Validation, consent, install and runtime stay in the host.
- Catalogue channels:
  - official
  - a GitHub raw backup
  - a Chinese git mirror
  - a custom URL
  - an environment-variable override
- **Official install flow.**
  1. A device-scoped resolve call returns ordered mirrors.
  2. The client tries each one and verifies sha256 and size.
  3. If every mirror fails, it falls back to the catalogue URL.
- **Download hosts.** Allowlisted, HTTPS only, with redirects rechecked.
- **Progress** is reported in phases: resolve, download, verify, install, enable.
- **Catalogue entries** carry provenance (repo, ref, commit, builder), review verdicts and yank state.

**Plugin center (a separate web service).**
- Built on GitHub App identity.
- Pins sources to a commit.
- Submissions go through a state machine: ownership, pin, scan, build, then AI review (primary plus critic). A deterministic policy evaluator is the only component that approves.
- Artifacts are rehosted in a distribution git repository.
- Trust tiers (verified, community, unknown) are assigned by the center only.

**Signing.**
- Today: sha256 is required for marketplace installs; dev and local installs are unchecked.
- Planned: Ed25519 publisher signatures and release channels.
- Auto-update is opt-in per plugin and never silently accepts new permissions.
- Yanked versions are excluded from installs and updates.

### 28.10 Developer kit

**SDK.** Dependency-light. It provides:
- manifest and API types
- the permission list
- the validator
- naming helpers
- policy helpers
- slot types

**CLI.**
- `init` from four templates: panel, agent tool, skill pack, full demo.
- `check` mirrors every installer rule and adds warnings.
- `pack` produces a stored zip and prints its hash.
- `publish` writes a submission pinned to a clean commit.

**Agent access.** The same kit is exposed as agent tools (check, scaffold, pack) and as a "new plugin from template" action. A built-in plugin-development skill activates in plugin workspaces.


## 29. Build tooling

### Monorepo
- pnpm workspace containing apps, packages and the docs site.
- Hoisting is enabled. Build scripts run only for packages on an allowlist.
- A minimum release age is enforced for dependencies; the pinned pi packages are exempt.
- Several security overrides are applied.
- Required runtime: Node 22.19 or newer, pnpm 10 or newer. CI runs Node 24 to match the Node version Electron bundles.

### Applications

| App | What it is |
|---|---|
| desktop | Electron 43, built with electron-vite (on Vite 7) and packaged with electron-builder 26 |
| pi-host | A headless remote host, shipped as its own tarball |

**Desktop app details**
- The UI uses React 19, Tailwind 4 and Zustand.
- Build entry points:
  - main process
  - a separate plugin-host process entry
  - two fully bundled CJS preloads (main window and plugin panel)
  - the renderer, with a plugin that tightens its content security policy

### Packages

| Package | Role |
|---|---|
| shared | Contracts and the per-locale changelog catalogs |
| i18n | Locale catalogs and the locale registry |
| plugin-sdk | Plugin authoring SDK |
| plugin-devkit | Plugin CLI |
| agent-runtime | The sidecar; bundled into one script by esbuild |
| agent-host | Headless agent-host module |
| host-runtime | Transports and supervisor, with no Electron dependency |
| racp | Remote control protocol server/client cores, WebSocket binding and pairing |
| voice-runtime | Local dictation |

### Rust
- A cargo workspace with one crate, which builds the host-core binary.
- Dependencies:
  - async runtime: tokio
  - storage: rusqlite with bundled SQLite
  - serialisation: serde
  - HTTP: reqwest with rustls
  - crypto: aes-gcm, argon2, sha2
  - file walking: the ignore, globset and walkdir crates
  - allocator: mimalloc
  - logging: tracing
  - platform: windows-sys on Windows, libc elsewhere
- On Windows the C runtime is linked statically, so no VC++ redistributable is needed.

### Lint and format
- **Biome** is used narrowly: the formatter is off and a single rule (no unused variables) is on, applied only to an explicit set of paths.
- **Custom scripts** enforce:
  - architecture line-of-code budgets, with a documented allowlist for exceptions
  - the style-token rules
  - pinned and patched pi packages
  - the marketplace catalog preflight
  - the release-docs gate
  - AGENTS.md / CLAUDE.md policy sync
  - that a PR is based on the latest main
  - the Linux glibc floor

### Tests
- **Vitest** for the TypeScript packages.
- **`cargo test`** for host-core, including migration suites.
- **Node's built-in test runner** for about 465 desktop test files. Most of these are contract and behaviour tests of main-process modules.

### Docs
- A bilingual VitePress site (English plus Chinese), deployed to Vercel.
- The sidebar is generated from the spec, the ADRs and the guides.
- Dead-link and locale-pair checks run in CI.

## 30. Packaging (electron-builder)

**Identity and output**
- A reverse-domain app id, with releases published to GitHub. electron-builder itself never publishes; CI uploads the artifacts.
- Each build stamps a distribution marker (installed, zip or portable) into its metadata. The updater reads this marker to choose its mode (§31).
- Chromium locales are trimmed to the nine shipped languages.

**Resources bundled with every build**
- tray icon
- bundled sidecar script
- built-in skills
- bundled plugins
- models.dev snapshot
- the per-platform host-core binary, placed in a bin folder

**Archive.** Smart unpacking is off. An explicit list unpacks the native addons (whisper binding and recorder) from the app archive.

### Targets

| OS | Targets | Arch |
|---|---|---|
| macOS | DMG and ZIP | arm64 and x64, as separate native runners |
| Windows | NSIS installer, ZIP and portable EXE | x64 |
| Linux | AppImage, deb and rpm | x64 and arm64 |

**macOS**
- Hardened runtime, with entitlements for JIT, unsigned executable memory, disabled library validation and the microphone.
- Usage strings for the microphone and the local network.
- The DMG is branded and shows exactly two icons: the app and an Applications link.

**Windows**
- The NSIS installer is assisted rather than one-click, and lets the user choose the install directory.
- The three Windows flavours are separate builder invocations, each stamping a different marker. The ZIP build writes no update feed.

**Linux**
- The rpm is built without build-id links.
- Runners use an older Ubuntu LTS to hold the glibc 2.35 floor.

### Sidecar and pi-host
- **Sidecar runtime.** No separate Node ships: the sidecar runs on Electron's embedded Node.
- **pi-host.** Not part of the desktop installer. It ships as separate Linux x64 and arm64 tarballs with checksums, attached to the same GitHub release.

## 31. Update and release pipeline

### Auto-update
- **Ownership.** Main exclusively owns electron-updater and the fixed GitHub feed.
  - The renderer can only check, read state, open the release link and install.
  - The renderer can never supply a feed URL.
  - The update path carries no credentials.
- **Mode by distribution:**
  - In-app download plus quit-and-install: macOS (signed), the Windows NSIS installer and Linux AppImage. AppImage is detected from the environment.
  - Notify-and-link only: deb, rpm, Windows ZIP and portable builds.
  - Disabled: dev builds.
- **Prereleases** are never offered, so release-candidate users still move onto stable.
- **Schedule.**
  - The first check runs shortly after startup, then every few hours.
  - Automatic checks use short timeouts and fail silently.
  - Manual checks report their status in the menu and on the About page.
- **Release notes.** "What's new" comes from changelog catalogs bundled with the app, one per locale, not from the GitHub release body.
- **Download cache.** Main relocates and prunes it.

### Release CI (tag-triggered)
1. **Verify.** Runs the same job as CI.
2. **Build matrix.** macOS arm64, macOS Intel, Windows x64, Linux x64 and Linux arm64. Each job:
   - checks that the tag matches the app version,
   - validates the catalog snapshot,
   - builds host-core alongside the JavaScript build,
   - packages with an explicit arch.
3. **macOS signing.**
   - Sign with Developer ID and notarize the app.
   - Notarize the DMG separately, then staple it.
   - Verify both.
   - A watchdog wrapper emits heartbeats and enforces timeouts during notarization.
   - Tag builds fail if signing secrets are missing.
4. **Linux.**
   - Verifies the per-arch updater feeds.
   - Exports the raw app archive as an asset, so distros can repackage it against a system Electron.
5. **Mac feed merge.** The per-arch mac feeds are merged into one.
6. **Publish.** Creates a GitHub release with generated notes. A hyphenated version is marked as a prerelease.

### Other workflows
- **CI, JavaScript job.** Build, typecheck, lint, architecture budgets against the base, a patch-hash test, then package tests.
- **CI, Rust job.** fmt, clippy, test.
- **Linux package inspection.**
- **Docs check.**
- **PR-base freshness check.**
- **Mirror to a Chinese git host.**
- **Dependabot.**

### Pre-tag gate
- Refresh the models.dev snapshot.
- Add changelog entries in every locale, with matching counts.
- Align the version across every package manifest, the Cargo workspace, the lockfile and the protocol app-version constant.
- Update the READMEs.
- Run the release-docs check.
- Generate icons from a single 1024 px master.

## 32. Testing strategy

**Layers**
- **Unit.** Vitest for the TypeScript packages, Rust tests for host-core, and the Node test runner for the desktop main-process modules.
- **Protocol smoke.** A headless script drives the real host-core binary over NDJSON. It covers:
  - providers and secrets
  - sessions
  - tools
  - loading a plugin in dev mode
  - agent-tool dispatch round-trips
  - checks that no secret leaks

  Live model chat runs only when the environment provides real credentials.
- **Electron journeys.** About 50 scenario runners, written in two styles:
  1. **Real app.** Launch the built Electron app against a throwaway data dir and the chosen host binary, with probe flags.
     - Drive it through the DevTools protocol (remote debugging, evaluating in the page) and/or the local MCP control plane.
     - Use deterministic in-process stub servers for models, realtime voice over TLS, and fixtures.
  2. **Component harness.** Bundle a scenario file together with the real production React modules and tokens. Load it in a minimal Electron main that stubs only the IPC it needs, then assert inside the page. Used for composer paste, IME and slash behaviour.

  Shared helpers handle boot, host, plugin, session, fixtures, asserts and waits. There are a few PowerShell probes for Windows. Playwright is planned but not used.

**Process rules**
- **Scenario catalogue.** A very large catalogue holds one templated scenario per user-visible or protocol-visible behaviour, with stable ids that are never reused. A traceability matrix maps acceptance criteria to scenarios.
- **Gating.** Every code change must pass the E2E suites for its surface on a candidate already rebased onto the latest main.
  - A suite that cannot run is reported as NOT RUN, with the reason and the remaining risk, and the gate stays unsatisfied.
  - Weakening assertions or adding retries to hide flakes is forbidden.
- **Acceptance criteria.** The MVP bar: on macOS, configure a model, open a project, complete one permission-gated tool task, and restore the session after a restart. Criteria are grouped as startup, models, sessions and streaming, workspace, tools and permissions, plan and shell, persistence, plugins, and diagnostics. Each carries evidence tagged automated, manual or open.
- **Non-goals.** Testing against hostile plugins, performance testing, and native Windows/Linux qualification.

## 33. Notes for the Orphus implementation

- **Architecture.** The four-process split is well motivated and worth keeping:
  - a thin Main
  - a Rust core as the sole persistence and policy authority
  - a Node sidecar running pi
  - per-plugin hosts

  The two contracts that pin the system down are the stdio JSON-RPC and the TypeBox-typed shared package. Define both early.
- **Credentials.** The shipped code stores credentials in an encrypted file only, which conflicts with the spec. Choose the OS keychain from the start.
- **Plugins.** Plugin isolation is the weakest part of the source system: raw Node is reachable and there are no resource limits. This is an opportunity to do better.
- **pi version.** Pin pi to 1.0.1. The patch list in §20.1 shows which behaviours you will need, either via upstream pi changes or your own adapters: hosted web search, local-error tagging, OAuth retry safety, and model-aware token estimation.
- **Many later ADRs are UI-polish decisions.** Treat Part 1 as the target state, not the history.
