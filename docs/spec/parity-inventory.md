> **Clean-room spec input.** Written by spec writers from a study of PI-Desktop v0.16.1 (LGPL-3.0). It describes behaviour in our own words and contains no PI-Desktop code or prose. Implementers build from this and [../PRD.md](../PRD.md), never from PI-Desktop source. See [../../CLEANROOM.md](../../CLEANROOM.md).

# PI-Desktop functional feature inventory (clean-room spec input for Orphus Desktop)

Subject: PI-Desktop v0.16.1 ("Early Preview" release line), studied as a shallow clone of the public repository.
Purpose: a behavioural inventory, so an independent MIT app can reach feature parity. Everything here is paraphrased. No code, type definitions or documentation prose was copied. Identifiers appear only where a user or plugin author would see them: UI labels, file extensions, slash commands, shortcut keys and the CLI name.

## How to read this

- Each feature is a bullet with an ID, `F-<AREA>-<NN>`.
- Maturity tags:
  - **[S]** stable / shipped and on by default;
  - **[O]** shipped but opt-in or off by default;
  - **[X]** experimental, or gated behind developer mode or a dev build;
  - **[P]** partial: some code exists, but parts are missing or stubbed;
  - **[PL]** planned or specified but not built;
  - **[R]** removed. Recorded so we do not rebuild it by accident.
- Where the docs and code disagree, the code (renderer, locale strings, build config) wins. The disagreement is noted under §47.
- The numbers it uses (limits, timeouts, sizes) are PI-Desktop's observed values. Treat them as reference defaults, not requirements.

---

## 1. Product framing

- **F-PROD-01** [S] A desktop "workspace for AI agents" for macOS, Windows and Linux. It is organised as Project → Session → Agent → Work, not as throwaway chat threads. Sessions persist across app restarts.
- **F-PROD-02** [S] Local-first by default:
  - projects, sessions, settings and logs stay on the machine;
  - no account is needed;
  - no vendor relay sits in the path;
  - model requests go straight to the provider the user configured.
- **F-PROD-03** [S] Model-agnostic: cloud vendors, OpenAI- or Anthropic-compatible gateways, and local servers (Ollama, LM Studio and similar). Each session can use a different model, and a session can switch model at any time.
- **F-PROD-04** [S] Three working modes per session: Agent (executes), Plan (investigates, then proposes a plan for approval) and Goal (agrees an outcome plus acceptance criteria, then works autonomously).
- **F-PROD-05** [S] Two levels of delegation:
  - in-turn background subagents;
  - full "worker sessions", coordinated through an official orchestrator plugin.
- **F-PROD-06** [S] Extensible through plugins, distributed as `.piplug` packages or from a marketplace. Plugins can add commands, panels, floating widgets, work-panel views, agent tools, skills, themes, MCP servers, background services and an inter-plugin message bus.
- **F-PROD-07** [S] Can import existing local sessions from Claude Code, Codex, OpenCode and the Pi CLI. Can import provider configs from those tools and from CC Switch.
- **F-PROD-08** [S] The agent engine is built on the open "pi" agent libraries. The desktop adds persistence, permissions, plugins and orchestration around it. For Orphus this means choosing an equivalent engine, or writing one.

## 2. App shell, windowing and process model

- **F-SHELL-01** [S] One main window with three columns:
  - a left sidebar;
  - a main pane (chat or a "destination" page);
  - an optional right work panel.
- **F-SHELL-02** [S] Destinations replace the main pane, as pages rather than modals:
  - Chat/Home;
  - Scheduled;
  - Extensions (Plugins);
  - Settings, which takes over the whole window.
  Back and forward history covers both sessions and pages (Mod+[ / Mod+]). There are no visible back/forward buttons.
- **F-SHELL-03** [S] Title bar per platform:
  - macOS: inset traffic lights and a native app menu.
  - Windows/Linux: a frameless ~46px row with drawn Minimize, Maximize/Restore and Close buttons, and no menu bar. Windows keeps native edge resizing and rounded corners.
- **F-SHELL-04** [S] Window behaviour:
  - minimum window size of about 800×560, capped to the display's work area;
  - size and position are remembered across launches, including moves between displays;
  - an option starts the window maximized.
- **F-SHELL-05** [S] Single instance per data directory. A second launch focuses or restores the running window, including from the tray. A separate data directory allows a side-by-side profile.
- **F-SHELL-06** [S] Boot sequence:
  - a splash ("Starting…") with a progress bar;
  - after about 30 s, a soft "Still starting…" notice with Copy diagnostics and Open logs;
  - after about 180 s, a hard recovery screen with Retry, Copy diagnostics, Open logs and Quit, stating that no data was lost.
- **F-SHELL-07** [S] Named fatal boot states, each with plain-language guidance:
  - Linux glibc older than 2.35;
  - data written by a newer app version (no downgrade migration);
  - the wrong CPU-architecture build, e.g. the Intel build on Apple Silicon, with a dismissible banner naming the right download;
  - app components out of sync (asks the user to reinstall).
- **F-SHELL-08** [S] Process isolation. The UI, the agent loop (a background helper process) and a privileged native backend (file/command tools, persistence, permissions, secrets) are separate processes.
  - Each helper is supervised and restarted with backoff, at most 3 restarts in 2 minutes; after that a "degraded" banner appears.
  - Transcript writes are buffered in a local outbox while the backend is down, then replayed without duplicates.
- **F-SHELL-09** [S] Backend status indicator: Ready, Limited, Reconnecting…, "Connection restored" toast, or "Can't reach the local service". It carries an Open logs action. Sends are rejected with readable errors while the backend is down.
- **F-SHELL-10** [S] Crash handling:
  - a renderer crash triggers an automatic reload, with a cooldown to prevent loops;
  - a UI crash shows an in-app fallback screen;
  - main-process exceptions are logged instead of raising stock error dialogs.
- **F-SHELL-11** [S] Up to 3 recently viewed sessions stay mounted, so switching back is instant and keeps the scroll position.
- **F-SHELL-12** [S] Orderly shutdown:
  - stop accepting prompts;
  - flush streaming checkpoints;
  - interrupt pending plans;
  - unload plugins;
  - stop the helpers.
  Partial replies survive as "aborted" messages.
- **F-SHELL-13** [S] Window zoom from 50% to 300% in roughly 10% steps (Mod+= / Mod+- / Mod+0). Full screen is F11 on Windows/Linux and Cmd+Ctrl+F on macOS.
- **F-SHELL-14** [S] Power options: "Keep computer awake" blocks idle system sleep while the app runs. A separate "Prevent screen sleep" option keeps the display on too.

## 3. Sidebar, navigation and session list

- **F-NAV-01** [S] Sidebar size:
  - default width 275px, drag-resizable from 240 to 520px;
  - dragging below 160px collapses it, and double-clicking the handle resets it;
  - the handle is keyboard-resizable;
  - Mod+B collapses it to an icon rail, with animation.
- **F-NAV-02** [S] Sidebar sections:
  - a global Pinned section of pinned conversations from all projects;
  - "Sessions"/"Temporary chats" for project-less conversations, showing about 5 rows and then scrolling;
  - "Projects": one collapsible group per retained project.
- **F-NAV-03** [S] Project group header:
  - name, a current-workspace dot and a chevron;
  - hover actions: New task (+) and an overflow menu (…);
  - a full-path tooltip;
  - a star icon when the project is pinned.
- **F-NAV-04** [S] Each project group shows its 10 most recent sessions, then "Load N more…". The expanded list is grouped by time (Today, Yesterday, Previous 7/14 days, Older). Pinned rows always sit above the fold.
- **F-NAV-05** [S] Sort modes: Recently updated, Created date, Oldest first, Name. A "Show archived" toggle sits in the sort menu.
- **F-NAV-06** [S] Manual project ordering:
  - press and drag the project title (8px threshold), with an insertion line showing the drop position;
  - or use Arrow Up/Down on a focused title;
  - Escape cancels.
- **F-NAV-07** [S] Drag an idle session onto another project group to move it there. Running sessions cannot be dragged.
- **F-NAV-08** [S] Drop an OS folder onto the project list to add it as a project, or switch to it if it is already known.
- **F-NAV-09** [S] Row status markers:
  - selection ring;
  - running (a breathing dot);
  - completed/unread (a check);
  - failed/needs attention (an alert).
  They stay static under reduced motion.
- **F-NAV-10** [S] Session hover card, after about 500ms:
  - title, mode and permission chips, live status;
  - model name, workspace name and live git branch;
  - "Updated <when>" and a "Temporary" label;
  - collaboration details: creator session, current task, recent exchanges, result.
- **F-NAV-11** [S] Session row actions:
  - Rename (1–80 characters);
  - Pin/Unpin;
  - Archive/Restore;
  - Branch/Fork;
  - Delete, with a two-click armed confirm;
  - Copy conversation link;
  - developer mode adds Copy conversation ID and Open session path.
- **F-NAV-12** [S] Multi-select with Cmd/Ctrl-click and Shift-click, for batch Archive or batch Delete with confirmation.
- **F-NAV-13** [S] Sidebar footer:
  - Settings;
  - Extensions;
  - Scheduled (clock);
  - Notifications (a bell with an unread-failure badge);
  - a profile menu with Settings, Logs and a theme cycle;
  - Help, which goes to Settings → Info.
- **F-NAV-14** [S] Background runs never steal focus or change the selection. Switching projects never cancels background runs.
- **F-NAV-15** [S] Right-clicking a section header or empty list space opens a create menu.
- **F-NAV-16** [S] Overlay scrollbars that appear only on hover or while scrolling.

## 4. Global search and command palette

- **F-SRCH-01** [S] One dialog opens with Mod+K (Search) or Mod+Shift+P (Command palette). It groups sessions, pages, settings rows and commands. The query is kept when the dialog is reopened, and the dialog is safe for IME input.
- **F-SRCH-02** [S] With an empty query it shows recent sessions, grouped Today / Yesterday / Previous 7 days / Previous 30 days / Earlier. Archived sessions appear only when you type a query.
- **F-SRCH-03** [S] Full-text search over session titles, project names and message bodies (user and assistant text, not reasoning or tool output):
  - substring matching that works for CJK text;
  - special characters are treated literally;
  - case-insensitive.
- **F-SRCH-04** [S] Results are grouped per session with a hit count, up to 2 snippets each (with sender and timestamp, match highlighted), and "Load more results". Clicking a snippet opens the session scrolled to that message, loading a window of history around it.
- **F-SRCH-05** [S] Command entries:
  - built-in commands (New task, Compact conversation context, Switch to Agent/Plan/Goal mode);
  - plugin commands;
  - trusted-extension commands.
  Each has a source badge. Matching covers titles, keywords, category and plugin name, including Chinese. Ordering is recent, then exact prefix, then built-in, then alphabetical.
- **F-SRCH-06** [S] Selecting a settings row navigates to it and flashes it.
- **F-SRCH-07** [S] Imported native Pi CLI sessions are merged into results. Scheduled-run transcripts are excluded.
- **F-SRCH-08** [R] A standalone command palette overlay used to exist. It was merged into search.

## 5. Projects and workspaces

- **F-PROJ-01** [S] A project is a logical group of one or more local folders. The first folder is "Primary" and cannot be removed. Chats, project instructions and project memory are shared across the group.
- **F-PROJ-02** [S] Ways to open a project:
  - Mod+O;
  - File → Open Project…;
  - the "+" on the sidebar Projects header;
  - the home-hero project switcher;
  - dragging a folder in;
  - Settings → Projects → Add project.
- **F-PROJ-03** [S] Create Project dialog:
  - a name, defaulting to the primary folder's name;
  - the source: either "This computer" (one or more folders from one picker, with one marked primary), or "Git repository" (a URL plus a destination parent folder; the app clones it into a subfolder).
  In strict network mode only public hosts are allowed.
- **F-PROJ-04** [S] Edit project:
  - rename;
  - add or remove secondary folders. Removing a secondary folder that has chats splits it off as its own project.
- **F-PROJ-05** [S] Several projects can be retained as sidebar groups, but only one is the "current workspace". Selecting a project's session activates that project. Selecting a temporary chat clears the active workspace.
- **F-PROJ-06** [S] Close removes a project from the sidebar only; it stays in the archive. Archive hides it; it can be restored.
- **F-PROJ-07** [S] Delete removes the project and all its sessions, transcripts, scratch, review files and memory. The folder on disk is never touched.
  - It is refused while a session is running.
  - The confirm dialog lists the sessions that are running and offers "Stop tasks and delete".
- **F-PROJ-08** [S] Settings → Projects (project archive workbench):
  - one toolbar with Recent/Name sort, search (also matches session titles, and shows "N of M") and Add;
  - sections Pinned, All projects and Archived, with counts;
  - each row shows a glyph or star, name, status tag (Active / Open / Archived), shortened path, session count and last-active time;
  - clicking a row opens an inline inspector with actions, folders, git branch and paged sessions;
  - double-click or Enter activates the project.
- **F-PROJ-09** [S] Project overflow menu:
  - Open folder in the OS file manager;
  - Edit project;
  - Project instructions;
  - Project memory;
  - Pin;
  - Archive/Restore;
  - Close;
  - Delete.
- **F-PROJ-10** [S] Temporary (project-less) chats use a per-session scratch folder as their workspace. Their tools are limited, and a "No project — tools limited" banner says so. Plan and Goal need a real project.
- **F-PROJ-11** [S] "New task" reuses the most recent empty session in the group instead of creating duplicates. The empty view appears instantly.
- **F-PROJ-12** [S] Dropping a folder onto the composer asks "Open as project" or "Reference folder". The second inserts the folder path.
- **F-PROJ-13** [S] Home hero: a "What can I help you build?"-style headline. When a project is active, its name is a clickable project switcher with:
  - a list of open projects;
  - search;
  - clone a public git repo;
  - open another folder.
  The copy differs for temporary chats. An animated mascot logo shows (a still frame under reduced motion).

## 6. Sessions: lifecycle, persistence and history

- **F-SESS-01** [S] Each session stores its own mode, provider/model, thinking level, permission mode and project binding. Status is idle, running, waiting for permission, aborted or error.
- **F-SESS-02** [S] Different sessions can run in parallel. Within one session only one turn runs at a time.
- **F-SESS-03** [S] Prompt queue:
  - Send while a turn is running queues the prompt;
  - the queue is per session, durable (it survives restart) and first-in first-out;
  - queued prompts are released one after each turn ends.
  After a restart, a restored queue is held and does not auto-run.
- **F-SESS-04** [S] Queue row actions: Move up, Move down, Send now, Edit (back into the composer when it is empty) and Remove. "Send now" promotes the item, gracefully stops the current turn after its current reply/tool batch, then sends it.
- **F-SESS-05** [S] Steering: Alt/Option+Enter injects the draft into the running turn. It is delivered at the next model-request boundary without aborting the in-flight tools, and it also wakes a parent that is waiting on subagents. If steering is no longer accepted, the draft is kept.
- **F-SESS-06** [S] Stop (button or Mod+.) aborts immediately with no confirmation:
  - it cancels interruptible tools, but completed writes are not rolled back;
  - the partial reply is kept with an "(aborted)" suffix;
  - Stop never clears the queue.
- **F-SESS-07** [S] Smart Stop: stopping before any assistant output appeared removes the just-sent prompt and restores it to the composer, attachments included.
- **F-SESS-08** [S] Mode, model, thinking or permission changes made during a run are staged and apply at the next turn.
- **F-SESS-09** [S] Titles:
  - an immediate fallback title from the first prompt;
  - after the first turn, a one-shot model call writes a short title (thinking off, failures silent);
  - manual renames (1–80 characters) are never overwritten.
- **F-SESS-10** [S] Fork a whole idle session into an independent copy. It inherits project, model, mode, thinking and permission mode, but not grants, pins or live state.
- **F-SESS-11** [S] Branch from a specific assistant reply (inclusive). This is allowed even while the source session is running, as long as the copied prefix is complete. Pasted files referenced in the prefix are copied into the child's scratch folder.
- **F-SESS-12** [S] Edit and resend a user message:
  - the transcript is truncated from that point and the old tail is archived;
  - Retry works even when the text is unchanged;
  - Mod+Enter retries and Escape cancels.
- **F-SESS-13** [S] Regenerate a reply in place. Discarded replies are archived, and a "current / total" pager on the user turn flips between variants. Archived variants are never rewritten.
- **F-SESS-14** [S] Delete an individual message.
- **F-SESS-15** [S] The full visible transcript is kept forever, even after compaction. On restart the model context is rebuilt from saved rows: text, reasoning, and paired tool calls/results.
- **F-SESS-16** [S] Crash safety: the in-progress reply is checkpointed about every 1.5 s, and leftovers are promoted on restart. Pending approvals and running executions are marked interrupted on launch, and nothing is replayed automatically.
- **F-SESS-17** [S] Long sessions open with the newest ~100 messages. Older pages load on scroll ("Loading earlier messages…"), only a trailing window of rows stays mounted, and an offset index allows fast seeking.
- **F-SESS-18** [S] Deleting a session removes its transcript, variants, scratch folder, review snapshots, queue and notifications.
- **F-SESS-19** [S] Session checklist (agent-maintained todo list), persisted per session. Forks start with an empty checklist.
- **F-SESS-20** [S] Conversation links:
  - "Copy conversation link" produces an app-internal link;
  - pasting one into a draft attaches a bounded excerpt of that other conversation, about the newest 40 messages and 16 KB, shown as a card;
  - it only works within the same project.
- **F-SESS-21** [S] Per-message usage is recorded: token usage, model, stream duration and estimated tool-token footprint, rolled up per turn.
- **F-SESS-22** [S] Artifact tracking: every workspace file a session wrote, edited or deleted is recorded.
- **F-SESS-23** [S] Exports never contain secrets.
- **F-SESS-24** [X] Native Pi CLI sessions are listed with a source badge. They can be browsed, continued in their original file (text-only, no tools yet) and forked. A continuation fails closed if the file changed on disk.
- **F-SESS-25** [S] Empty or silent-turn recovery:
  - if a turn ends with no visible text and no tool call, it is silently re-run once with a nudge;
  - a second silent turn becomes a retryable "empty model response" error.
- **F-SESS-26** [S] In approved Plan/Goal execution, if the model announces its next step but makes no tool call, it is nudged once to continue.
- **F-SESS-27** [S] Completion signalling: exactly one notification per unseen completed or failed turn. This drives the unread marks. The session currently on screen never gets one, and aborts produce none.

## 7. Composer

- **F-COMP-01** [S] Floating pill composer: centred on the empty home, docked at the bottom of a conversation, with content up to about 768px wide.
- **F-COMP-02** [S] Textarea:
  - 3 lines tall when empty, auto-grows to 7 lines, then scrolls;
  - autocorrect and spellcheck are off.
- **F-COMP-03** [S] Rotating placeholder hints: ask anything; "/" for commands and "@" for files; Shift+Enter for a newline.
- **F-COMP-04** [S] Left toolbar:
  - "+" Add files (native picker; files are copied to the session scratch folder);
  - the mode chip (click cycles Agent → Plan → Goal, and it pulses while planning);
  - the permission chip (Ask every time / Accept edits / Auto; Goal shows a fixed Auto label).
- **F-COMP-05** [S] Right toolbar:
  - the context-usage ring;
  - a combined model × reasoning chip;
  - Enhance prompt;
  - one Send/Stop slot.
  At narrow widths the labels collapse progressively.
- **F-COMP-06** [S] Model × reasoning menu:
  - a Model submenu with search, "Recently used" (3) and "Other models", sticky provider headings, and reasoning/vision badges;
  - a Reasoning submenu listing "omit/do not send" plus the levels the model supports;
  - a drag slider with one stop per level.
  Level names are shown untranslated: off, minimal, low, medium, high, xhigh, max.
- **F-COMP-07** [S] Enter-to-send is a setting. When it is off, Mod+Enter sends. Shift+Enter always inserts a newline.
- **F-COMP-08** [S] Drafts (text and chips) are kept per session across session and page switches. The home composer has its own draft.
- **F-COMP-09** [S] On an empty composer, Arrow Up/Down recalls this conversation's earlier prompts, with their attachments. History persists: up to 100 prompts per conversation, for 20 conversations.
- **F-COMP-10** [S] Send clears the box immediately, and a failed send restores it. Send is disabled with "Configure a model first" when no model exists.
- **F-COMP-11** [S] Slash menu (/ at the start of the input):
  - groups in order: Prompt templates (project before user-global, with argument-hint ghost text), App commands, Plugin commands, Extension commands, Skills;
  - collision priority: built-in > project template > user template > plugin > extension;
  - a skill never shadows a command;
  - an unknown /foo is sent as literal text.
- **F-COMP-12** [S] Built-in slash commands (complete list): `/new`, `/compact`, `/agent-mode`, `/plan-mode`, `/goal-mode`. A mode command accepts trailing text: it switches mode, then sends the text. `/new` and `/compact` alone run locally and send nothing.
- **F-COMP-13** [S] Skills can also be invoked mid-message by typing "/" after whitespace. Several are allowed per message. Each sent skill shows as a chip and asks the model to load that skill. A sent template shows as a monospace command chip, not its expanded text.
- **F-COMP-14** [S] CJK IME convenience: an ideographic comma (、) typed as the first character of an empty draft becomes "/".
- **F-COMP-15** [S] @ file mentions:
  - "@" after whitespace or a quote opens fuzzy search over the project's file index (git-tracked plus untracked-not-ignored, or a bounded walk when there is no git);
  - quoted `@"..."` tokens and path-spanning queries work;
  - results are capped at 8000 with a notice;
  - a selected file becomes an inline chip; a selected directory continues completion;
  - completion covers all folder roots of a multi-folder project.
- **F-COMP-16** [S] Chips show the leaf name with the full path in a tooltip. Backspace on an empty textarea removes the last chip.
- **F-COMP-17** [S] Paste:
  - OS clipboard files and images are saved to scratch and shown as removable chips (limits 20 files, 64 MiB each);
  - Word-style rich paste keeps the text, not the rendered image.
- **F-COMP-18** [S] Large paste: text over a threshold (default 600 characters, configurable from 1 to 1,000,000) becomes a "pasted-text" file chip. Clicking the chip expands it back into editable text.
- **F-COMP-19** [S] Drag and drop of files onto the composer, with an accent outline. Folders trigger the project-or-reference prompt.
- **F-COMP-20** [S] Image chips show a hover preview. Click opens a modal viewer with zoom, fit, pan, download and previous/next.
- **F-COMP-21** [S] Enhance prompt (sparkles):
  - a one-shot LLM rewrite of the draft, with no history and no tools; chips stay in place;
  - disabled when the draft is empty, starts with "/", or no model is ready;
  - shows "Enhancing…" with a spinner, then a single "Undo enhancement" that is cleared by any edit, send or switch;
  - 60 s timeout; a failure shows a dismissible error bar.
  Built-in rules: keep code, paths and identifiers verbatim; keep the user's language; do not more than about double the length.
- **F-COMP-22** [S] Running-turn controls:
  - Send with a non-empty draft queues it;
  - with an empty draft the slot becomes Stop;
  - Alt/Option+Enter steers.
- **F-COMP-23** [S] A capability warning (non-blocking) appears when a model without tool support is used in Agent mode.
- **F-COMP-24** [S] Plugins can add composer toolbar controls and trigger lists (see §24).
- **F-COMP-25** [O] A Live Voice icon appears once Live Voice is enabled (see §28).
- **F-COMP-26** [P] A dictation microphone button exists but its UI is hidden in current builds (see §28).

## 8. Transcript rendering

- **F-TRAN-01** [S] Message layout:
  - user turns are right-aligned compact plates;
  - assistant turns are full-width prose;
  - hovering reveals copy chips;
  - a streaming reply shows an accent rule and a pulsing cursor.
- **F-TRAN-02** [S] Markdown rendering:
  - GitHub-flavoured Markdown, with HTML sanitized;
  - KaTeX math (copying a formula copies its TeX);
  - syntax-highlighted code with a copy button.
  It works offline.
- **F-TRAN-03** [S] Mermaid diagrams:
  - rendered lazily once the code fence closes;
  - toggle between diagram and source, and Copy diagram source;
  - strict sandboxing with size limits, falling back to source.
- **F-TRAN-04** [S] Table actions: Copy as Markdown, Download as CSV, and an expand-to-preview view.
- **F-TRAN-05** [S] One "process" disclosure per assistant turn holds the thinking, tool calls, searches and intermediate text in order. The final answer sits outside it.
  - Groups of two or more activities get nested disclosures.
  - Completed turns fold by default.
  - Manual open/close choices are respected.
- **F-TRAN-06** [S] Thinking display setting:
  - Detailed: the process is open and reasoning is visible;
  - Compact: collapsed, showing only a live thinking indicator;
  - a per-message Show/Hide thinking toggle overrides it.
  Labels read "Thinking · time" / "Thought for time" and "Processing · time" / "Processed for time".
- **F-TRAN-07** [S] Tool rows:
  - collapsed by default;
  - a verb summary (Read, Listed, Searched, Wrote, Edited, Ran, Fetched, Delegated, Used), present tense while running;
  - chips for exit code, match count, file count, replacement count, truncated, scratch, line range and size;
  - failed and denied calls are flagged in the header.
- **F-TRAN-08** [S] Expanded tool sections: Input, Output, File content, Written content, Command, stdout, stderr, Changes (diff), Files, Matches, Details, Note, Error. Each is a bounded scroll area with a "N more hidden" note.
- **F-TRAN-09** [S] Shell output streams live into the tool row, with stdout and stderr tagged separately.
- **F-TRAN-10** [S] Inline review card after each successful write or edit:
  - added/modified/deleted status and +/− counts;
  - expandable hunks;
  - a guarded Rollback button.
- **F-TRAN-11** [S] Live status row while a turn runs: Working (with elapsed time), Starting, Waiting for model, Preparing next request, Compacting context, Recovering empty response, "Retrying in Ns · attempt k/10" (hovering shows the provider error cause), waiting for named subagents and their current actions, and Planning.
- **F-TRAN-12** [S] Assistant messages longer than 50 lines collapse to 20, with an expand link.
- **F-TRAN-13** [S] A model badge appears under each completed answer.
- **F-TRAN-14** [S] A compaction divider row ("Context compacted · #N", with the summary size) or a failed-summary note.
- **F-TRAN-15** [S] Errors:
  - one card, "Couldn't finish this reply", with Show details, Copy details (redacted) and Dismiss, naming the provider and model;
  - typed errors can carry an action: Open settings, Try again or Continue;
  - an unfinished turn shows a "needs attention" card with Continue.
- **F-TRAN-16** [S] Message menu: Copy, Select message text, Regenerate, Branch from this reply, Edit and resend, Delete message. Conversation menu: Copy conversation, Select conversation text, Scroll to top, Jump to latest. There are no like/dislike buttons.
- **F-TRAN-17** [S] Minimap rail:
  - one marker per turn;
  - hovering magnifies the markers Dock-style and previews the turn;
  - clicking jumps to it.
  Also: Mod+↑ / Mod+↓ jump to the top or bottom, and a floating "Jump to latest" button appears. Auto-follow while streaming stops once the user scrolls up.
- **F-TRAN-18** [S] Resizable conversation column: invisible drag handles on both edges, default about 760px, persisted.
- **F-TRAN-19** [S] File references in prose and tool rows:
  - become links that open in the work panel;
  - a `path:line` reference scrolls to that line;
  - `.html` files open in the side browser;
  - the right-click menu offers Show in file manager, Copy full path, Copy relative path and Open with default app;
  - partial names resolve against the project, then scratch, then attachments.
- **F-TRAN-20** [S] URLs follow the "link open destination" setting. Right-click offers Open in default browser, Open in work panel and Copy link address.
- **F-TRAN-21** [S] Smooth streaming (on by default) releases text typewriter-style. It turns off automatically under reduced motion.
- **F-TRAN-22** [S] A hosted web-search row: "Searching the web", the source count, "+N more sources", and open-page and find-in-page steps.
- **F-TRAN-23** [S] Generated-image cards (see §27).
- **F-TRAN-24** [S] Subagent topology: "Coordinating N delegated tasks", per-delegate status, "What <agent> did", and a click to open the delegate's tab.
- **F-TRAN-25** [S] Session checklist dock above the composer:
  - header "X/Y completed · Current: …";
  - up to 8 visible rows;
  - states Pending, In progress, Completed, Cancelled;
  - expand/collapse.
- **F-TRAN-26** [S] Images referenced in chat (workspace, scratch, attachments) render inline, with size bounds.
- **F-TRAN-27** [S] Transcript text is selectable; app chrome is not.
- **F-TRAN-28** [S] A voice-origin tag on messages that came from Live Work.
- **F-TRAN-29** [R] Removed and not to be rebuilt: message Quote action, response annotations, side chats, and the floating text-selection toolbar.

## 9. Interactive cards (in-chat decisions)

- **F-CARD-01** [S] Permission card:
  - "Allow <tool> to run?" with a risk level (High/Medium/Low), a short reason, an argument preview, and the project or "Temporary chat";
  - buttons: Allow once, Allow for this chat, Deny. There is no "always allow".
- **F-CARD-02** [S] Permission card behaviour:
  - it waits forever, with no timeout, and the composer is disabled meanwhile;
  - Stop denies everything pending;
  - only the head of the queue shows, with "N more requests are waiting";
  - a subagent's request says which subagent asked.
- **F-CARD-03** [S] Question card ("A few questions"), docked above the composer:
  - one question at a time with progress dots (answered, unanswered, skipped, current), clickable to revisit;
  - radio buttons for single-select, checkboxes for multi-select;
  - every question also offers "Enter another answer";
  - Skip, Next/Submit answers, Decline all;
  - Markdown in question text (links in options are inert);
  - long option lists scroll;
  - no timeout; several requests queue.
  Stopping the turn marks the remaining questions as skipped.
- **F-CARD-04** [S] Plan/Goal approval bar, docked above the composer:
  - the title plus Open plan / Open goal (shows the Markdown in the work panel);
  - a split Approve button with Approve (Ask) / Approve (Accept edits) / Approve (Auto). The last choice is remembered on the device, and Auto carries a shell warning;
  - Reject.
  While pending, the draft is read-only and the mode/model/permission controls are locked.
- **F-CARD-05** [S] Plan history card in the transcript: status Awaiting approval / Approved / Rejected / Expired / Interrupted / Superseded. It shows the exact snapshot and a file opener. Deleting the file does not lose the stored text.
- **F-CARD-06** [S] Prompts raised by trusted extensions (text input, Yes/No, OK) are labelled with the extension's name and time out after 5 minutes.
- **F-CARD-07** [S] Questions and approvals in background sessions raise native notifications and a soft chime.

## 10. Agent / Plan / Goal modes

- **F-MODE-01** [S] Per-session mode selector: Agent | Plan | Goal. The default for new sessions is a setting. Switching mode keeps the same agent and context.
- **F-MODE-02** [S] You enter Plan or Goal by choosing it while idle, by using a slash command, or because the agent asked to switch (it must be the only tool call in that step).
- **F-MODE-03** [S] Tools in Plan/Goal:
  - allowed: read, search, browser preview, read-only browser actions, plan-safe plugin actions, and the shell (still governed by the permission mode, so this is not a strict sandbox);
  - always denied, even under Auto: write/edit, non-plan-safe plugin tools, unknown tools, delegation and image generation.
  Denied tools stay visible to the model but are marked unavailable.
- **F-MODE-04** [S] Plan output: the agent investigates (impacted files, risks, validation, migration, open questions) and submits once. The plan is saved as an immutable Markdown file inside a hidden per-project folder (separate subfolders for plans and goals), with a recorded hash and size.
- **F-MODE-05** [S] Goal contract: an outcome, objectively checkable acceptance criteria and boundaries, with no steps. After approval the agent works autonomously and ends with a per-criterion report backed by evidence.
- **F-MODE-06** [S] Approving picks the permission mode for execution and switches the session to Agent. Rejecting, an absolute 30-minute expiry, or an interruption returns the session to editable planning. A revision creates a new artifact.
- **F-MODE-07** [S] Execution after a restart is never replayed. Unattended (scheduled) runs in Plan or Goal are refused.
- **F-MODE-08** [S] A live "Planning" indicator: the mode chip pulses and a compact row appears in the transcript.
- **F-MODE-09** [R] A separate "Chat" mode was removed.

## 11. Context budget and compaction

- **F-CTX-01** [S] The visible transcript and the model context are separate. Compaction summarises older context, but every original row stays visible.
- **F-CTX-02** [S] Automatic compaction is always on and cannot be disabled. It triggers at about 90% of a computed hard limit, at a turn boundary, before the next request. The hard limit is the window minus output and safety reserves.
- **F-CTX-03** [S] Manual compaction with `/compact` or the palette command while idle.
- **F-CTX-04** [S] An agent-callable "new context" tool lets the model ask for a fresh window at the next boundary.
- **F-CTX-05** [S] Hidden budget nudges tell the agent to wrap up when about 15% of the budget remains, and to record what must survive near the end.
- **F-CTX-06** [S] What a summary keeps:
  - while mid-task: the summary plus the latest user message (truncated if huge);
  - for a completed turn: no tail.
  The summary records the files read and modified.
- **F-CTX-07** [S] If the summary input is too large, the app tries once with reduced input, then splits it into up to 16 chunks with a rolling summary. Transient failures are retried 3 times (2/4/8 s).
- **F-CTX-08** [S] If compaction fails, a deterministic fallback keeps the previous summary and a recent window, labels the row, and shows a warning toast. There is also a long-thread warning suggesting a new session.
- **F-CTX-09** [S] On a provider context overflow, the app drops the reply, compacts once and retries once. A second overflow is a terminal error.
- **F-CTX-10** [S] Token estimates are calibrated against the provider's reported usage. The output cap respects both the user's override and the published limit.
- **F-CTX-11** [S] Reasoning replay is preserved across compaction for providers that need it (DeepSeek-style).
- **F-CTX-12** [S] Context usage ring:
  - percent remaining or used (a setting), with warning colours;
  - hidden until usage exists;
  - reflects the most recent request.
- **F-CTX-13** [S] Context inspector (click the ring):
  - remaining and window tokens;
  - this turn's usage;
  - generation speed in tokens/s (≈ when estimated);
  - provider input, output, cache read, cache write, cache hit rate and reasoning tokens;
  - tool context (types, calls, approximate tokens);
  - compaction count and summary size.
- **F-CTX-14** [S] Subagents compact against their own model's budget, in memory only. A degraded report is flagged, and there is a specific context-overflow error.
- **F-CTX-15** [R] Background pre-computed compaction (at 70%) was tried and removed.

## 12. Attachments and multimodal input

- **F-ATT-01** [S] Images arrive by paste, drag, picker or project reference. They are stored content-addressed and sent as real images only when the model has vision and the image is under 10 MB; otherwise the model gets a file-path reference.
- **F-ATT-02** [S] Vision support comes from the catalog and can be overridden per model. Unknown or custom models default to no vision.
- **F-ATT-03** [S] SVG is always treated as a file, never rasterised, even when mislabelled.
- **F-ATT-04** [P] There is a per-model PDF/document capability flag, but PDFs are currently passed as file references.
- **F-ATT-05** [S] Any other file is attached as a path the agent can read. Attachments must lie inside the project, scratch or the attachment store.
- **F-ATT-06** [S] Images inside tool results (from MCP or plugins) are forwarded to vision models.
- **F-ATT-07** [S] Image history restore budget: 10 MB per image and 30 MB total, newest first. Images over budget degrade to path references.
- **F-ATT-08** [S] Reference-only or image-only drafts can be sent.

## 13. Built-in agent tools

- **F-TOOL-01** [S] Core tools: read file, write file, line-anchored edit, shell command, glob (list files), grep (content search). There is also a skill loader, a delegation family, an on-demand tool search, and others.
- **F-TOOL-02** [S] Lazy tool activation:
  - the first request carries the core set;
  - other tools (browser preview, plugin tools, MCP tools, image generation, plugin dev tools and so on) are listed by name and one line each;
  - the agent activates up to 4 at a time through a tool-search tool;
  - activation survives prompts, mode switches, compaction and restarts.
- **F-TOOL-03** [S] Read:
  - line-numbered output with a short whole-file version tag;
  - 2,000-line default window (4,000 max), paged by offset;
  - 128 KB budget, with very long lines clipped;
  - binary files refused;
  - a directory input suggests glob instead.
- **F-TOOL-04** [S] Glob: pattern listing, newest first, 100 entries by default and 1,000 max, respecting ignore rules unless a path is given explicitly.
- **F-TOOL-05** [S] Grep:
  - uses system ripgrep when present, otherwise a built-in searcher;
  - modes: matching lines, files-with-matches, counts;
  - an include filter, and a default limit of 200;
  - each hit carries a version tag so the matched line can be edited directly.
- **F-TOOL-06** [S] Write: create or overwrite a file and return a new version tag. It strips accidentally pasted line-number prefixes and records a review diff.
- **F-TOOL-07** [S] Edit, line-anchored and hash-verified. It addresses lines by number against a version tag; it never matches old text.
  - Operations: replace a range, replace a syntactic block, insert before/after, append, delete or cut into a named or anonymous register, paste a register, delete the file, move/rename the file.
  - Block-aware languages: Rust, TS/TSX, JS/JSX, JSON, Markdown, CSS, Python, TOML, YAML, HTML, Bash.
- **F-TOOL-08** [S] Edit safety:
  - a version gate;
  - a "you must have seen these lines" gate, which reveals up to 40 unseen lines in the error so an honest retry works;
  - drift recovery with a consistent offset;
  - mistyped-path recovery;
  - after 3 failures on the same file in one prompt, the turn stops with "mutation retry budget exhausted".
- **F-TOOL-09** [S] Shell:
  - non-interactive, run in the session workspace;
  - 60 s default timeout, overridable from 1 s to 6 h;
  - stdout and stderr stream separately;
  - results are bounded (96 KB / 4,000 lines each), with the overflow spilled to a scratch log file;
  - timeout or abort kills the whole process tree;
  - the login-shell PATH is probed so Homebrew/nvm tools work from a GUI launch;
  - an environment variable points to the session scratch folder.
- **F-TOOL-10** [S] Tool-call behaviour:
  - calls run sequentially, except that several delegation calls in one batch run in parallel;
  - writes to the same path are serialised, even across subagents;
  - common argument aliases are accepted;
  - success, failure and denial are all reported back to the model;
  - tool calls written as prose are not executed.
- **F-TOOL-11** [S] Checklist tool: replaces the session's visible checklist. Up to 50 items, each with a status and priority; only one item may be in progress.
- **F-TOOL-12** [S] Ask-user tool: multi-question cards (see F-CARD-03). Answers return as question/answer lines.
- **F-TOOL-13** [S] Mode tools: enter Plan, enter Goal, submit plan, submit goal.
- **F-TOOL-14** [S] Browser preview tool: opens a workspace HTML file in the live-reloading side browser. Read-only, allowed in every mode. Needs the bundled Browser plugin enabled.
- **F-TOOL-15** [S] Delegation tools: start, wait, list and stop subagent tasks (see §15).
- **F-TOOL-16** [S] Image generation tool (see §27).
- **F-TOOL-17** [S] Plugin dev tools: scaffold from a template, check, pack (see §24).
- **F-TOOL-18** [S] Hosted web search: a per-model opt-in that uses the provider's own search.
  - Supported on Anthropic Messages and OpenAI Responses wires (including Azure and Codex).
  - For DeepSeek, xAI and OpenAI chat endpoints it transparently reroutes to the vendor's search-capable API.
  - Search rounds render in the transcript and survive restart and compaction.
- **F-TOOL-19** [S] Every tool call is audited (session, turn, tool, argument hash, decision, wait time, duration, outcome), and the audit is kept for 90 days.
- **F-TOOL-20** [S] Backend admission limits bound concurrent tools: 4 shell commands globally (2 per session), 2 mutations (1 per session), and so on. Overload returns a retryable "host overloaded".
- **F-TOOL-21** [P] Scheduled-task tools (list/create/update/delete automations from chat): the user guide describes them, but the runtime notes say they are defined and not registered. See §47.

## 14. Permissions, approvals and sandboxing

- **F-PERM-01** [S] Permission modes:
  - Ask every time (the default): confirm writes, the shell, plugin tools and MCP tools;
  - Accept edits: auto-allow writes and edits inside the workspace;
  - Auto: allow everything.
  There is a global default plus a per-session override, and the model is never told the mode.
- **F-PERM-02** [S] Decisions are Allow once, Allow for this chat (per tool name, in memory, never shared with other sessions) or Deny. There is no auto-deadline.
- **F-PERM-03** [S] Read, glob and grep inside the session roots are always allowed. Scratch-folder writes never prompt.
- **F-PERM-04** [S] Workspace sandbox:
  - paths resolve against the session's own project root (or scratch);
  - ".." and symlink escapes count as outside access;
  - symlinks are re-resolved after approval.
- **F-PERM-05** [S] An explicit path outside the workspace in a file tool is allowed silently only in Auto. Otherwise it needs a confirmation card, and the results are marked "external". Implicit directory walks never escape the workspace.
- **F-PERM-06** [S] An always-on security denylist applies even inside the workspace and is not lifted by any approval: the VCS object store, private keys and certificates, `.env` files (not the example/template variants), credential JSON, and token-bearing package-manager rc files. Glob and grep drop these silently; read, write and edit fail. The shell is not filtered.
- **F-PERM-07** [S] Ignore layers: built-in defaults (dependency, build, venv, cache and log directories), a workspace ignore file in gitignore syntax, and a user-global ignore file.
- **F-PERM-08** [S] Per-session scratch folder outside the project:
  - created lazily;
  - deleted with the session;
  - swept at startup when orphaned or older than 7 days.
- **F-PERM-09** [S] Review and rollback: every successful workspace write or edit records a diff. Rollback works only if the file still matches the post-edit hash; otherwise it reports a conflict and touches nothing. Moves roll back both sides.
- **F-PERM-10** [S] MCP tools are always treated as medium risk, whatever the server declares, and need approval in Ask and Accept-edits modes.
- **F-PERM-11** [S] Plan mode is explicitly not a security boundary for the shell.
- **F-PERM-12** [S] No OS-level process sandbox. Containment relies on permissions, path checks and killing the whole process tree.

## 15. Subagents (in-turn delegation)

- **F-SUB-01** [S] Five built-in subagents, each of which can be switched off:
  - Explorer: read, search and shell; reports with path:line;
  - Code reviewer: read-only;
  - Test runner;
  - Fixer: writes in the workspace;
  - UI designer: writes, and inspects its result with the browser preview.
- **F-SUB-02** [S] User-defined subagents are Markdown files with front-matter, kept in a global user folder. They can be edited outside the app. Up to 16 definitions; malformed files show diagnostics instead of failing.
- **F-SUB-03** [S] Settings → Subagents:
  - a Built-in group (with "Copy as mine") and a user group;
  - an editor with "Start from template" (Explorer, Code reviewer, Test runner, Fixer, UI designer, Start blank);
  - fields: name (slug), "When to delegate to it", the tool grant (or "Inherit parent tools", with a warning when the delegate can change files), and Instructions;
  - Advanced: model ("Same as the session" or a pinned model), reasoning (inherit / do not send / a level), output limit, and an ordered list of fallback models.
- **F-SUB-04** [S] Delegation runs in the background:
  - starting a task returns an id at once, with at most 10 concurrent per session, in Agent mode only;
  - wait for all or any N, with a timeout (default 600 s, max 900 s) that acts as a heartbeat;
  - list (status, elapsed time, turns, last tool);
  - stop some or all.
- **F-SUB-05** [S] If the parent ends its turn while delegates are still running, the turn stays open and the parent is re-prompted with their reports.
- **F-SUB-06** [S] Each delegate has its own context and prompt. It cannot see the user, ask questions or delegate further. It goes through the same permission checks and has no turn limit. Reports are bounded at about 12k characters and spill to scratch beyond that.
- **F-SUB-07** [S] Model selection order: the per-call model, then the definition's pin, then the session model. The agent may pick ad hoc only among models the user marked "Available for AI delegation".
- **F-SUB-08** [S] Ordered fallback models: when one model's retries are exhausted, the delegate continues on the next, keeping completed tool results. A model whose window is too small is skipped.
- **F-SUB-09** [S] Resumable delegations:
  - a completed or failed delegate can be resumed with its full prior transcript, including after a restart;
  - up to 2 resumable chains per subagent type;
  - the parent prompt lists resumable chains with their objective and the files read.
- **F-SUB-10** [S] Statuses: running, completed, failed, aborted, stopped, interrupted (app closed), denied. A fatal error in the parent aborts its delegates and marks them resumable.
- **F-SUB-11** [S] UI:
  - nested delegate rows under the task card;
  - "Waiting for N subagents" with each one's current tool;
  - each delegate opens as a read-only live transcript tab in the work panel, with its composer disabled;
  - the effective model and thinking level are shown.
- **F-SUB-12** [S] Permission scope: built-in and user definitions may set ask / accept-edits / auto / inherit. Project-provided definitions cannot escalate. Plan/Goal denials and the outside-path gate always apply.
- **F-SUB-13** [R] Subagent peer messaging, an agent-to-agent broker, and subagent watchdogs or turn limits were all removed.

## 16. Session orchestration and collaboration (worker sessions)

- **F-ORCH-01** [S] An official "Session Orchestrator" marketplace plugin gives the agent one tool that can:
  - spawn real, durable worker sessions that inherit the parent's project, model, thinking and permission mode;
  - send tasks or messages to existing sessions;
  - supervise, check status, wait, collect results, accept or cancel;
  - list targetable sessions (up to 100).
- **F-ORCH-02** [S] Limits: up to 4 active workers per parent and 16 overall. Waiting polls (about 1 s; default 25 s, max 45 s).
- **F-ORCH-03** [S] An "Agents" work-panel view lists workers with Refresh, Open session and Stop.
- **F-ORCH-04** [S] Collaboration guarantees:
  - the sender identity is authenticated by the host;
  - deliveries are idempotent;
  - a completion callback fires at most once;
  - the permission ceiling is rechecked at admission;
  - a hop limit applies;
  - messages are framed as agent data, not human authorisation;
  - nothing is replayed after a restart.
- **F-ORCH-05** [S] Sidebar and hover card provenance: "Created by", "From/To <name>", message kinds (Session task, Completion report, Agent message), a status card (Idle, Queued, Working, Permission required, Completed, Failed, Cancelled, Interrupted), and a list of created sessions with Open. A deleted target shows "That session no longer exists".
- **F-ORCH-06** [S] Collaboration operations are available only to first-party, authenticated plugins during their own tool call. They are not available to external control clients.

## 17. Models and providers

- **F-PROV-01** [S] Provider kinds: native vendor, OpenAI-compatible gateway, and custom endpoint.
- **F-PROV-02** [S] Wire formats the user can choose: OpenAI Chat Completions, OpenAI Responses, Anthropic Messages, Google Generative AI. Account-only wires (Codex/ChatGPT responses) and an OpenCode Go preset also exist.
- **F-PROV-03** [S] Auth kinds: API key, key plus base URL, bearer token, Azure key, AWS default credentials, custom headers, OAuth vendor account, and none (local).
- **F-PROV-04** [S] Named API-key presets with the endpoint prefilled:
  - OpenAI, Anthropic, Google Gemini, OpenRouter, Groq, xAI, Mistral, Together AI, Fireworks, DeepSeek;
  - Qwen/DashScope, Qwen Token Plan (international and China), Moonshot/Kimi (international and China), Kimi For Coding;
  - Zhipu, Zhipu Coding Plan, Z.AI, Z.AI Coding Plan;
  - SiliconFlow, Volcengine Ark, MiniMax (Anthropic- and OpenAI-format variants), StepFun Plan, Ant Ling;
  - Baseten, Cerebras, Hugging Face, NVIDIA NIM, OpenCode Zen, OpenCode Go, Meta, Vercel AI Gateway;
  - Xiaomi token plans (several regions).
- **F-PROV-05** [S] Not presets, but reachable through a custom endpoint: Azure OpenAI, Bedrock, Vertex AI, the Cloudflare gateways, Cohere, Perplexity, Ollama, LM Studio, vLLM, TGI, LocalAI and LiteLLM. Bedrock and a generic custom HTTP profile are marked experimental.
- **F-PROV-06** [S] "Sign in with a subscription" OAuth vendor accounts:
  - Claude Pro/Max: browser PKCE with a loopback callback, or a copy-code fallback;
  - ChatGPT Plus/Pro via Codex;
  - GitHub Copilot: device code;
  - OpenRouter: PKCE;
  - Kimi: device code;
  - xAI: device code;
  - one further gateway vendor.
  Sign-in flows show a browser link with copy and reopen, device codes with copy, and text prompts (e.g. an enterprise URL).
- **F-PROV-07** [S] Several accounts per vendor, each its own row with a non-secret label. Tokens refresh per request under a per-account lock, and refresh tokens never leave the trusted process. Account model lists come from the vendor's own endpoint.
- **F-PROV-08** [S] Add-provider dialog:
  - a searchable Service picker with "Sign in with a subscription", "Connect with an API key" and "Use a custom endpoint";
  - a custom endpoint asks for name, base URL (path auto-trimmed), API key and API format;
  - the format is auto-detected from a pasted URL, with "Apply suggested format";
  - up to 4 candidate URLs on the same origin are probed sequentially within 12 s, so the key is never sprayed across servers; cross-origin redirects are refused.
- **F-PROV-09** [S] Model selection when adding a provider:
  - a two-pane picker (discovered models with search, select-all and Fetch list, beside the chosen models);
  - custom model IDs, with limits prefilled from the catalog;
  - "Connected · N models found" or "This service doesn't list models";
  - recommended models are auto-picked, but beta and alpha models never are.
- **F-PROV-10** [S] Per-model settings:
  - alias;
  - context window (follows the catalog until edited) and max output;
  - thinking levels (7 chips), default thinking level, and thinking protocol (Standard/Adaptive);
  - image and PDF input capability;
  - native web search;
  - "Available for AI delegation";
  - set as image-generation model;
  - drag to reorder.
- **F-PROV-11** [S] Provider list actions:
  - drag or keyboard reorder;
  - enable/disable;
  - Edit;
  - Copy configuration (a duplicate without secrets or headers);
  - Test connection (latency plus a structured error);
  - Make default;
  - Delete with confirm, which removes its credentials.
  Plugin-provided providers carry a read-only "Plugin" badge.
- **F-PROV-12** [S] Advanced provider settings: display name and custom HTTP headers (up to 32; presets including User-Agent; copy or import as JSON; Latin-1 validation; full-width characters folded). Reserved auth headers are refused. Headers apply to discovery, tests and token refresh too.
- **F-PROV-13** [S] API keys are write-only: never shown again after saving. A blank field on edit keeps the existing key.
- **F-PROV-14** [S] Import from other tools: scans the local configs of Claude Code, Codex, OpenCode, Pi and CC Switch. API keys are copied into the secure store; OAuth grants are not. Results are grouped by source with select all, and duplicates are skipped. Distinct credentials at the same endpoint become separate providers.
- **F-PROV-15** [S] Adding a provider never silently takes over the app's default model.
- **F-PROV-16** [S] Vendor quirks handled: Zhipu/Z.AI thinking format, DeepSeek reasoning replay, developer vs system role, Copilot identity headers, Anthropic token-endpoint rate limiting.
- **F-PROV-17** [S] Provider TLS trusts the OS certificate store plus extra CAs. Certificate errors are terminal and come with trusted-root advice.

## 18. Model catalog, selection and reasoning

- **F-MODEL-01** [S] A bundled offline snapshot of the public models.dev catalog supplies limits, modalities, reasoning options, prices and display names. "Refresh model catalog" updates it in memory. Catalog requests never carry credentials.
- **F-MODEL-02** [S] Live endpoint discovery decides which IDs are actually available. Unknown models get conservative defaults (128k context, ~8k output, text-only). Values the user entered are never overwritten by catalog corrections.
- **F-MODEL-03** [S] The composer model menu shows only configured models, recent first. New chats inherit the last-used model (up to 20 remembered locally). Existing sessions keep their pinned model.
- **F-MODEL-04** [S] Reasoning levels: off, minimal, low, medium, high, xhigh, max, plus "omit" (send nothing).
  - The level is stored per session.
  - An unsupported level clamps to the nearest supported one.
  - Non-reasoning models resolve to off.
  - Switching model resets to that model's default level.
  - Anthropic adaptive vs budget thinking is chosen from metadata or an override.
- **F-MODEL-05** [S] Image-generation and speech models are excluded from the chat pickers.
- **F-MODEL-06** [S] Limits are formatted compactly: 1M, 1.05M, or an em dash when unknown.

## 19. Error handling and retries

- **F-ERR-01** [S] Rate limits: up to 10 silent retries. Server delay hints are honoured; otherwise backoff is exponential from 2 s with jitter, capped at 30 s.
- **F-ERR-02** [S] Other transient errors (network, timeout, dropped stream, 5xx gateway) have their own 10-retry budget (1, 2, 4, 8, 8… s). Both budgets reset after each successful response. Retries reuse the same bubble, and completed tool calls are never re-run.
- **F-ERR-03** [O] "Retry until success" removes the retry ceiling, with a warning about continued billing.
- **F-ERR-04** [S] An opaque 400/422 is retried once without the output-limit fields. A dead connection pool is rebuilt (at most once per 30 s) while keeping the proxy route.
- **F-ERR-05** [S] Non-retryable errors: auth, model not configured, malformed request, context too large, certificate failure.
- **F-ERR-06** [S] User-facing typed errors:
  - local service unavailable;
  - model not configured;
  - key missing or rejected;
  - rate limited;
  - provider error;
  - network error;
  - certificate failure;
  - blocked by address policy;
  - timeout;
  - stream interrupted;
  - empty response twice;
  - edit retry budget exhausted;
  - context too large;
  - compaction failed;
  - subagent overflow;
  - agent busy;
  - stopped;
  - agent runtime crashed;
  - agent runtime out of memory;
  - shell not found;
  - path outside workspace;
  - plan approval expired, stale or interrupted.
- **F-ERR-07** [S] Error diagnostics include the phase, HTTP status, wait time, network category, request size and compaction generation. They never include credentials or raw bodies. The stable code and trace id are copyable.

## 20. Work panel (right dock)

- **F-WP-01** [S] Hidden at startup. Toggled with Mod+J or a fixed toggle button. Its width persists; its tabs live per session and in memory only.
- **F-WP-02** [S] It opens automatically only when the user opens a file, URL, preview or plan/goal artifact, never because the agent wrote something.
- **F-WP-03** [S] Tab strip:
  - horizontally scrollable, with a "+" that opens a "New" launcher (Review, Files, Browser, plugin views);
  - close button, middle-click, or Delete/Backspace to close;
  - Arrow/Home/End to navigate;
  - drag or Alt+Arrow to reorder.
- **F-WP-04** [S] Resizing: the divider has a minimum of 244px and a default of 360px; double-click resets it and Escape cancels a drag. The panel squeezes the chat down to a 450px floor, then auto-collapses the sidebar.
- **F-WP-05** [S] A maximize toggle lets the panel take over the client area beside the sidebar.
- **F-WP-06** [S] Native surfaces (browser, plugin views) hide automatically while overlays or dialogs are open.
- **F-WP-07** [S] Tab types: Review, Browser, Files / File Manager, file preview tabs, plan/goal artifact, Subagent transcript, and plugin views.

## 21. Review (agent change history), files and preview

- **F-REV-01** [S] Review tab:
  - a per-conversation chronological list of the changes the agent's write/edit tools recorded ("N files changed", "N recorded changes");
  - per-file status: Added, Modified, Deleted, Renamed, New;
  - a hunk diff;
  - binary and too-large placeholders;
  - per-change "Rollback this change", guarded against later edits.
  This is not a git diff.
- **F-FILE-01** [S] File preview tab: text, images, an MP4 "Open with default application" option, binary and too-large placeholders, Reveal in file manager, and Open with the default app. It is limited to the project plus scratch and attachment paths that chat opens.
- **F-FILE-02** [S] The bundled "File Manager" plugin view is a third-party MIT plugin, vendored, enabled by default and updatable:
  - a lazy directory tree with type icons and a resizable or collapsible split;
  - a context menu: new file, new folder, rename, move, open, refresh, open with default app, show in folder;
  - filename search;
  - respects .gitignore/.ignore, with a hidden-files toggle;
  - a multi-root folder switcher remembered per project.
- **F-FILE-03** [S] File Manager editor:
  - highlighting for about 30 languages, find/replace, multi-cursor, folding;
  - Mod+S saves atomically, does nothing when the file is unchanged, and asks before overwriting a file that changed on disk;
  - reload from disk;
  - Markdown edit/preview.
- **F-FILE-04** [S] File Manager viewers:
  - images: fit or actual size, checkerboard, dimensions, up to 8 MiB;
  - audio and video with no autoplay, up to 24 MiB;
  - paged and sortable CSV/TSV tables;
  - a JSON tree;
  - a read-only SQLite browser with overview, tables, schema and a read-only query box.
- **F-FILE-05** [S] File Manager limits: no delete, no live file watching, one root at a time, no archive inspection. Credential paths are refused.
- **F-BRW-01** [S] The bundled "Browser" plugin is an embedded browser view:
  - Back, Forward, Reload/Stop, an address box ("Search or enter a URL"), and Open externally;
  - states: "No page open", "Loading stopped", "Could not open this page";
  - the tab title follows the page title;
  - each tab keeps its own live page, and hidden tabs are throttled;
  - it uses the app's proxy settings.
- **F-BRW-02** [S] Live reload of workspace HTML when the file or files next to it change.
- **F-BRW-03** [S] Agent "Browser" tool, driving the view through DevTools:
  - actions: navigate, accessibility snapshot (element uids), click/fill by uid, screenshot (optionally full page), evaluate JS, read the console, raw allowlisted protocol calls;
  - cookie, storage, target and network-interception methods are blocked;
  - read-only actions are allowed in Plan/Goal; mutating actions only in Agent mode.
- **F-BRW-04** [S] Disabling the Browser plugin removes the view and the tool. URL chips then fall back to the OS browser, and the preview tool fails closed.
- **F-BRW-05** [S] Link-open setting: Work panel browser (the default) or the OS browser.

## 22. Shell, terminal and git

- **F-SHX-01** [S] Command shell picker:
  - Windows PowerShell 5.1, PowerShell 7, Command Prompt and Git Bash on Windows; Bash on Unix;
  - each choice shows whether it is available;
  - the default is Windows PowerShell on Windows and Bash elsewhere;
  - if the configured shell is missing, the app falls back and says "Configured shell unavailable; using …";
  - the shell is pinned per turn, and its dialect drives how commands are run;
  - no bash is bundled on Windows (Git for Windows is required).
- **F-SHX-02** [R] An interactive terminal panel was removed. There are no terminal tabs today. The remote-host protocol defines terminal operations, but no desktop screen uses them. Orphus may choose to add a terminal as a differentiator.
- **F-GIT-01** [S] Git integration is limited to:
  - cloning a repository during project creation;
  - showing the current branch on hover cards and in the project inspector;
  - git-aware file indexing for @ mentions;
  - a working-tree diff read exposed only through the automation API.
- **F-GIT-02** [R] A Pull Requests page and a PR-listing tool were removed. There is no commit, staging or branch UI. "GitHub workspace" is pitched as something a plugin could provide.

## 23. Instructions, memory and system prompt

- **F-INST-01** [S] Global instructions: a Markdown editor in Settings → Instructions, applied to every session before project instructions. It shows the file path.
- **F-INST-02** [S] Project instructions, compatible with common conventions:
  - for each directory from the root down to the target, the first file found among an override file, AGENTS.md, CLAUDE.md and a nested Claude folder file;
  - concatenated so the nearest wins;
  - re-resolved lazily per target path;
  - capped at 32 KiB;
  - editable from the project menu;
  - changes apply from the next prompt.
- **F-INST-03** [S] Pi-CLI-compatible prompt overrides: a file that replaces the persona line, and a file that appends to the system prompt. Project files override global ones, and each is capped at 64 KiB.
- **F-INST-04** [S] Project memory: a dialog of memory cards (optional title plus content, up to 32 KiB) with add, remove and save. Memory is injected as durable, lower-priority context for all chats in the project group.
- **F-INST-05** [S] Prompt layering, in order:
  - base product rules (reply in the user's language, narrate before tools, give a visible final answer, finish the job);
  - the mode prompt;
  - workspace info;
  - tool guidance;
  - the skill catalog;
  - project instructions;
  - user instructions.
- **F-INST-06** [S] System-state changes (tools added or removed, skill catalog edits) are journaled chronologically as hidden rows, so the prompt and caches stay stable across restarts.
- **F-INST-07** [S] Prompt templates: user-global and per-project templates appear in the slash menu, with argument hints.

## 24. Plugin system

### 24.1 Concepts

- **F-PLUG-01** [S] A plugin is a folder with a JSON manifest at its root. It may also hold an entry script, panel HTML, skill docs, theme CSS, an icon and a README. Users see every extension type as a "plugin".
- **F-PLUG-02** [S] Plugin identity:
  - the id is a lowercase reverse-domain string, and settings, data, grants and updates key off it;
  - the version is semver;
  - optional author, homepage, repository and icon;
  - an optional per-locale name/description/safety-notes block (English and Simplified Chinese);
  - an optional range of supported app versions.
- **F-PLUG-03** [S] Plugin origins:
  - installed (copied into app data);
  - development (loaded in place from a folder);
  - marketplace;
  - bundled (cannot be uninstalled, but can be disabled and updated).
- **F-PLUG-04** [S] Each plugin's code runs in its own OS process and reaches the host only through a broker. The broker enforces an API allowlist and a permission gateway, and the plugin never gets host objects. Panels and views run in sandboxed, context-isolated web pages with no Node, reaching the host through a single bridge object.
- **F-PLUG-05** [S] Known trust gap, disclosed: the permissions gate the host API, but the plugin process can still use raw runtime file and network access. Third-party plugins are therefore effectively user-privileged code until a capability sandbox exists.
- **F-PLUG-06** [S] Plan-mode rule: plugin agent tools are hidden and cannot run in Plan/Goal (unless an action is declared plan-safe). Plugin commands and panels stay usable as explicit user actions.
- **F-PLUG-07** [S] Activation scope, shared by plugins, user MCP servers and user skills: Off / These projects (with a project picker; subfolders match) / Everywhere.
  - Project-scoped extensions are inactive in project-less chats.
  - Themes are never scoped.
  - It is a three-segment control with a warning when the project list is empty.

### 24.2 Contribution types (complete list)

- **F-PLUG-10** [S] Commands: title, keywords, category, icon. They appear in search/palette Commands and the slash menu.
- **F-PLUG-11** [S] Panel: a detached plugin window with HTML entry, size, resizable flag and localized title. Minimum 360×280.
- **F-PLUG-12** [S] Floating widget: a transparent, frameless window (orb style), always-on-top option, minimum 120×120. Its context menu offers Always on top, Minimize and Close widget.
- **F-PLUG-13** [S] Work-panel views: id, localized title, an icon from a fixed host icon set, HTML entry and order. A view can receive a "location" subject when opened. Requires the view permission.
- **F-PLUG-14** [S] Agent tools: name, description, risk (low/medium/high), JSON-schema input, timeout, and optional plan-safe actions. Shown to the model under a namespaced name.
- **F-PLUG-15** [S] Skills: Markdown docs with name/description front-matter, up to 32 per plugin and 128 KiB each. They need the "adjust agent instructions" permission and are silently inert without it.
- **F-PLUG-16** [S] Themes:
  - a CSS file with a light/dark base and optional assets;
  - typed runtime-adjustable variables (length, number, colour, select);
  - CSS is sanitized: no imports, no undeclared URLs, no script; 256 KiB cap;
  - assets may be images or fonts, up to 4 MiB in total.
- **F-PLUG-17** [S] Scenic theme picker: a data-only, host-rendered Settings → Extensions page of 1–12 theme cards with preview images, a blur slider and Apply.
- **F-PLUG-18** [S] Window appearance: native window background colour per light/dark, and a Windows-only corner radius (0–24). Active only while that plugin's theme is selected.
- **F-PLUG-19** [S] MCP servers: stdio (a PATH command or a plugin-relative one, args, env from literals or plugin-setting references) or HTTP (URL and headers). Up to 8 per plugin.
- **F-PLUG-20** [S] Resident background services: up to 4 per plugin, with auto-restart on by default.
- **F-PLUG-21** [S] Message bus: declared publish topics and subscribe patterns, with single-segment and trailing multi-segment wildcards.
- **F-PLUG-22** [S] Model provider rows: up to 8 per plugin, each with a wire format, key or no auth, and 1–64 models with their limits and thinking levels. They show as read-only "Plugin" rows in Settings. OAuth is not allowed yet; local/dev plugins only.
- **F-PLUG-23** [S] Session sources: declared origins for sessions the plugin imports.
- **F-PLUG-24** [S] Global shortcuts: up to 8, each mapped to one of the plugin's own commands, with an optional default chord.
- **F-PLUG-25** [S] Generated settings: string, number, boolean, select, JSON and shortcut fields, rendered by the host as a Plugin settings sheet with Save and Reset. Secret fields are refused.
- **F-PLUG-26** [S] Agent extensions ("trusted extensions"): up to 8 modules that run inside the agent process (see 24.10).
- **F-PLUG-27** [S] Renderer UI slots: an ES module evaluated inside the host window (see 24.4).
- **F-PLUG-28** [P] Lifecycle hooks: only load and unload fire. Install, enable, disable and uninstall hooks are declared but not fired. Activation events: only "on startup" and "on command" are implemented.

### 24.3 Host APIs exposed to plugins

- **F-PLUG-30** [S] App: version, locale, current appearance, and setting the app theme.
- **F-PLUG-31** [S] Themes at runtime: upsert, remove or list the plugin's own themes, and set declared variables. Changes apply instantly.
- **F-PLUG-32** [S] Plugin self: own id, manifest, settings get/update, and a private data directory.
- **F-PLUG-33** [S] UI: open/close the panel, toasts, in-app notifications, native OS notifications (with a permission probe).
- **F-PLUG-34** [S] Workspace and files, scoped by permission:
  - workspace info, including all folder roots and the project id;
  - read text, stat, byte-range read (8 MiB), classified preview (text/image/binary/too large);
  - write;
  - glob (500 matches) and list directory (1000 entries);
  - remove to the OS trash (non-recursive);
  - open with the default app, reveal in the file manager;
  - request a folder picked by the user;
  - turn a file dropped by the user into a one-file read grant.
  Paths follow the project of the calling tool's session.
- **F-PLUG-35** [S] Project creation: create or reuse a durable project record without switching workspace.
- **F-PLUG-36** [S] Agent:
  - register agent tools;
  - one-shot completions through the user's own models, with the plugin never seeing secrets;
  - limits: 8 per minute, 90 s budget, 32 KiB system prompt, 200k characters of messages;
  - optional inclusion of the current session context.
- **F-PLUG-37** [S] Models list: enabled and authenticated models, with no secrets.
- **F-PLUG-38** [S] Session read: a bounded projection of the in-flight session's model context, available only during the plugin's own tool call.
- **F-PLUG-39** [S] Session import:
  - import one session or a batch into a declared source, idempotent by external id;
  - list, get, page messages, rename and trash/purge the plugin's own imported sessions;
  - limits on messages per session, batch size and rates.
- **F-PLUG-40** [S] Usage read: completed-turn usage rows (tokens by type, model, project, time) over up to 365 days, paged. This is how dashboards are built.
- **F-PLUG-41** [S] Desktop control: a reviewed, risk-tagged operation catalog (see §40). Dangerous operations need a confirm flag plus a native consent dialog owned by the host.
- **F-PLUG-42** [S] Clipboard: read, write, and host-retained history (text and images; only explicit app writes and composer pastes are recorded; in memory; 30 days / 500 entries / 256 MiB).
- **F-PLUG-43** [S] Open external URLs, limited to http, https and mailto.
- **F-PLUG-44** [S] Network: fetch and WebSocket, restricted to the plugin's declared hostname allowlist, with redirects re-checked. Up to 4 sockets and 1 MiB frames.
- **F-PLUG-45** [S] Browser control: drive the work-panel browser (navigate, snapshot, screenshot, click/fill, evaluate, console, allowlisted raw protocol calls), with per-session tabs.
- **F-PLUG-46** [S] Speech adapters: register custom speech-to-text and text-to-speech protocol handlers.
- **F-PLUG-47** [S] Services: start/stop handlers for declared services.
- **F-PLUG-48** [S] Bus: publish and subscribe. A message carries the topic, sender id, payload and host timestamp.
- **F-PLUG-49** [S] Events (no permission needed): workspace changed, settings changed, appearance changed, session model changed, bus messages, WebSocket events, and session turn ended (the last is very new).
- **F-PLUG-50** [S] Global keyboard: register, unregister and list OS-wide shortcuts, with conflict checks against the OS, the app and other plugins. They are released on disable or crash.
- **F-PLUG-51** [P] Audio capture/playback: the API exists and is permission-gated, but there is no device backend; calls return "unsupported". Panels with the microphone permission can use browser audio capture.
- **F-PLUG-52** [S] Panel bridge: a fixed set of host channels (toast, settings, workspace, appearance, themes, models, file read/write, clipboard, open external, fetch, skill management, dropped files). Any other channel is forwarded to the plugin's own main process as custom RPC.
- **F-PLUG-53** [S] Typed error codes: permission denied, not found, invalid argument, timeout, unsupported, limit exceeded, rate limited, confirmation required, internal, crashed.
- **F-PLUG-54** [S] Time budgets: load 15 s, hooks 5 s, commands 30 s, tools about 110 s.

### 24.4 Renderer UI slots (in-window plugin UI)

- **F-PLUG-60** [S] Slots available:
  - the user-message action bar and the assistant-reply action bar: at most 3 visible items per side, the rest in "⋯"; host buttons are never hidden or reordered;
  - an extra block under assistant replies (collapsed at 320px, expandable to 600px);
  - a custom tool card for the plugin's own tools (display only);
  - a code-block renderer for fences tagged with the plugin's namespace;
  - composer toolbar controls;
  - composer trigger lists on @, # or /: one plugin per symbol, the selection becomes a chip, up to 8 chips per draft.
- **F-PLUG-61** [S] Actions a renderer extension can take:
  - call its own main process (64 KB, 2 s, rate-limited, with a circuit breaker);
  - insert text into the composer;
  - read a draft snapshot;
  - replace the draft (compare-and-swap, only when the composer is unfocused);
  - add, list or remove its own attachments (up to 16 MiB).
- **F-PLUG-62** [S] Self-drawn overlays can never sit above host security layers (permission and approval cards). A throwing slot collapses only its own block.
- **F-PLUG-63** [PL] A host-shaped "ask" confirmation card for plugins, and a runtime hook catalog: before-send interception, tool gating, turn watch, turn facts/recap, compaction write-back.

### 24.5 Permissions and consent

- **F-PLUG-70** [S] A permission must be both declared and granted, and the runtime uses the intersection. Default deny. Risk tiers are low, medium and high.
- **F-PLUG-71** [S] Permission catalogue (our wording):
  - panel; floating widget;
  - work-panel view;
  - microphone (panel audio only);
  - theme; settings surface; window appearance;
  - clipboard read (with history); clipboard write;
  - notifications;
  - file read/write/delete within declared globs;
  - register agent tools; adjust agent instructions (activates skills);
  - one-shot completions on the user's models;
  - run code inside the agent; draw UI in the host window;
  - add model providers; desktop control; list models; create projects;
  - read the current session;
  - import sessions; read, rename or trash own imported sessions;
  - read usage statistics;
  - network fetch; network to any host (install-time only, cloud-metadata IPs still blocked); WebSocket;
  - open external links;
  - local MCP server; remote MCP server;
  - background service;
  - bus publish; bus subscribe;
  - browser control;
  - background audio capture; audio playback;
  - speech adapter;
  - global shortcuts.
- **F-PLUG-72** [S] Install, upgrade and dev-load review dialog:
  - permissions grouped High / Medium / Low with plain-language explanations;
  - file globs and network domains shown beside their permissions;
  - developer info and source;
  - upgrades mark newly requested permissions as "New";
  - the user can uncheck permissions.
- **F-PLUG-73** [S] Runtime file consent: access outside the declared scope opens a native dialog (Allow once / Allow until quit / Deny). A runaway-delete brake allows 50 deletes per minute, then asks once. Deletes go to the trash; deleting a plugin's own output is tracked through a write ledger.
- **F-PLUG-74** [S] Dangerous desktop operations open a native per-call dialog. It shows only the catalog's own text and an argument preview, never text the plugin wrote.
- **F-PLUG-75** [S] Legacy workspace-wide file permissions are downgraded automatically, and the Plugins page flags it.
- **F-PLUG-76** [S] Users can view and revoke grants. A running plugin must reload to see the change.
- **F-PLUG-77** [S] An unconditional file denylist for plugins: env files, package-manager rc files, git credentials, SSH keys and certificates, cloud/SSH/GPG/kube/docker config directories, and the app's own data directory.
- **F-PLUG-78** [S] Network egress: one hostname allowlist per plugin governs panels, fetch, WebSockets and remote MCP. A bare "*" is refused, and the dev kit warns about local or metadata hosts.
- **F-PLUG-79** [PL] First-use (per-call) confirmations for clipboard and external links are specified but not built. Today everything is granted at install.

### 24.6 Isolation and storage

- **F-PLUG-80** [S] Each plugin gets its own private data directory, settings namespace and log file. Plugins cannot read the host database, secrets or other plugins' data. Uninstall deletes code, data and logs.
- **F-PLUG-81** [S] The plugin process gets a minimal environment, with no provider keys.
- **F-PLUG-82** [S] Crash containment:
  - pending calls fail;
  - contributions are deregistered;
  - panels close;
  - a toast appears;
  - the crash report never includes plugin output.
- **F-PLUG-83** [S] Panel windows:
  - window chrome lives in a closed shadow DOM, so plugin CSS cannot restyle host controls;
  - per-plugin persisted browser partition;
  - popups and webviews denied.
- **F-PLUG-84** [S] Docked views: up to 4 cached live views (LRU), destroyed on disable, uninstall or crash.
- **F-PLUG-85** [PL] Missing so far: a capability sandbox inside the plugin process, and CPU/memory limits.

### 24.7 Lifecycle

- **F-PLUG-90** [S] Plugin states: discovered, validated, installed, enabled, loaded, running, plus error states. Row groups: Needs attention / Updates available / Active / Turned off.
- **F-PLUG-91** [S] Enable registers every contribution, then starts services. A failure auto-disables the plugin and shows the error. Disable tears everything down, and an active plugin theme falls back to System. Load is roughly transactional.
- **F-PLUG-92** [S] At app startup only enabled plugins load, and one failure doesn't affect the others. On quit, plugins get bounded parallel unload (1.5 s each, 3 s total).
- **F-PLUG-93** [S] Service supervision:
  - per-service state chips (Starting, Running, Stopped, Failed) and a restart count;
  - a crash restarts the whole plugin with backoff 1/2/4/8/16 s, up to 5 attempts;
  - healthy after 60 s.
- **F-PLUG-94** [S] Uninstall removes files and data. Plugin-declared providers and their credentials are deleted on uninstall and only disabled on disable.
- **F-PLUG-95** [S] Plugin lifecycle events (install, uninstall, enable, disable, load, crash, service restarts and so on) are audited.

### 24.8 Packaging and installation (`.piplug`)

- **F-PLUG-100** [S] A `.piplug` file is a ZIP archive with entries stored uncompressed and the manifest at the root.
  - No symlinks and no path traversal.
  - At most 2,000 files and 50 MiB extracted.
  - VCS folders, dependency folders and build folders are excluded when packing.
  - Credential files are left out with a warning.
  - Output is named `<id>-<version>.piplug`, and its SHA-256 is printed.
- **F-PLUG-101** [S] Install flow:
  - safety checks, extract to a temp folder, validate;
  - permission review;
  - move into place and record it;
  - optionally enable.
  A failure leaves nothing behind. The install is classed as new, upgrade or reinstall.
- **F-PLUG-102** [S] Install sources: a directory, a `.piplug`/zip file, a marketplace download, or a dev folder. Nothing is compiled or dependency-installed at install time; plugins must ship bundled output.

### 24.9 Updates, signing and marketplace

- **F-PLUG-110** [S] Integrity: SHA-256 is verified for marketplace installs. The catalog is refreshed just before downloading. A mismatch shows "integrity check failed" and nothing is installed.
- **F-PLUG-111** [S] Update discovery:
  - a silent check from cache when the Extensions page opens;
  - an explicit "Check for updates".
  Manual updates show a changelog and need confirmation. Per-plugin opt-in auto-update is applied through "Apply automatic updates", and only when the update requests no new permissions.
- **F-PLUG-112** [S] Version handling:
  - only strictly newer versions count as updates;
  - yanked versions are excluded and installed copies are flagged;
  - a minimum app version blocks the install early;
  - provenance (repo, commit, builder) is shown.
- **F-PLUG-113** [S] Extensions page, two tabs: Installed and Marketplace.
  - The overflow menu holds Load local plugin, Install package, New plugin from template, Import pi extension, Check for updates, Apply automatic updates and Refresh marketplace.
  - Installed rows have search, Local/Bundled tags, an expandable details section (capabilities, service status, risk-tinted permission chips, inline load errors), Enable/Disable, Reload (dev), Plugin settings, Open panel, auto-update toggle and Uninstall.
- **F-PLUG-114** [S] Marketplace tab:
  - search, categories, a card grid with monogram glyphs (no remote image loads), a Verified badge and install counts;
  - a detail sheet with About, Repository/Homepage (opened in the in-app browser), Safety notes, Permissions by risk, Versions, README and Source commit / Built by;
  - trust levels: Verified publisher / Community plugin / Unverified source.
- **F-PLUG-115** [S] Catalog channels: Official (the default), GitHub backup, a mirror on a Chinese git host, or a Custom catalog URL. A package is always downloaded from the channel that served the catalog. Downloads are HTTPS-only and limited to allowlisted hosts.
- **F-PLUG-116** [S] Official-channel downloads ask the center for mirrors first, trying each mirror and verifying hash and size, and fall back to the catalog URL. They send an anonymous device identifier (a hash of the machine id) for counting and rate limiting. Backup and custom channels send nothing.
- **F-PLUG-117** [S] Install progress dialog:
  - phases: resolve, download, verify, install, enable;
  - mirror n of N, bytes and speed;
  - Cancel (during download), Retry, Copy details with the list of mirrors tried;
  - closes automatically on success.
- **F-PLUG-118** [PL] Not built yet: ed25519 publisher signatures, beta/dev plugin channels, rollback after a failed upgrade, private enterprise sources, ratings and reviews.
- **F-PLUG-119** [P] Publisher-side Plugin Center:
  - a GitHub-App login with ownership verification;
  - pins the publisher's repo and commit;
  - builds, scans, runs two AI review passes and a policy gate;
  - publishes to a distribution repo mirrored to the backup host.
  Contracts are done; the service is still in progress. This is an ecosystem service, not part of the desktop app.

### 24.10 Trusted agent extensions (pi-compatible)

- **F-PLUG-120** [S] A plugin can contribute modules written against the upstream pi coding-agent extension API.
  - They run inside the agent process with full agent trust, behind a separate high-risk grant.
  - They are loaded from local or dev plugins only; marketplace distribution is disabled until signing exists.
  - TypeScript works without a build step.
- **F-PLUG-121** [S] Supported extension capabilities:
  - register tools, slash commands, and custom agents/providers;
  - subscribe to events (session, before-agent-start prompt edits, context replacement, provider request/headers, turn, message, tool call blocking, tool result replacement, compaction);
  - run subprocesses;
  - get/set active tools, model, thinking level and session name;
  - send user messages;
  - UI: notify, confirm, select, input, status line;
  - context usage, compact, new session, fork.
  Unsupported terminal-UI APIs are inert and reported as diagnostics.
- **F-PLUG-122** [S] Handlers have a 30 s budget. A reload happens at the next turn boundary, never mid-turn. Load errors never fail a session; a one-line notice appears instead.
- **F-PLUG-123** [S] "Import pi extension": pick a file or folder, which is copied into app data as a generated plugin.
  - Skill-only packages become skill plugins.
  - npm dependencies install in locked-down mode: registry only, no lifecycle scripts, isolated cache, egress limited to the public registry.
  - If npm or Node is missing, a "Choose npm" picker appears and the choice is remembered.
- **F-PLUG-124** [S] The plugin row shows the agent-extension chip, state, registered names, and diagnostics: failed to load, setup failure, unsupported API, terminal-only symbol, rejected registration, handler error, handler timeout.

### 24.11 Dev kit

- **F-PLUG-130** [S] `pi-plugin` CLI:
  - `init <template> <dir>`;
  - `check <dir>`;
  - `pack <dir>`;
  - `publish <dir>`, which pins the git ref, rejects dirty or credentialed remotes, writes a submission file, and offers a stable/beta channel.
- **F-PLUG-131** [S] Templates: basic panel, basic agent tool, skill pack, full demo. In-app "New plugin from template" scaffolds a folder, shows the review, loads it as a dev plugin and opens the folder as the project.
- **F-PLUG-132** [S] `check`:
  - errors: manifest problems, missing files, path escapes, unknown permissions, missing required permissions, symlinks, size limits;
  - warnings: no icon, unknown view icon, skill without front-matter, legacy file permissions, high-risk permissions, skills declared without the instructions permission, local hosts in the allowlist, declared-but-unused permissions.
  Passing check guarantees the plugin will install.
- **F-PLUG-133** [S] Hot reload of dev plugins:
  - files are watched with a 300 ms debounce;
  - a reload is unload → validate → load, and panels are recreated;
  - a broken reload keeps watching;
  - up to 16 plugins can be watched;
  - a reload can never widen permissions; widening requires a manual Reload with a fresh review.
- **F-PLUG-134** [S] The same dev kit is exposed as agent tools (check in every mode; scaffold and pack in Agent mode). A bundled "plugin development" skill activates automatically when the workspace looks like a plugin.
- **F-PLUG-135** [S] Plugin logs are filterable by plugin id from Settings → Info → Logs.
- **F-PLUG-136** [S] The SDK has no dependencies: manifest and API types, validators, a theme CSS sanitizer and a mock host for tests.

### 24.12 Global plugin launcher

- **F-PLUG-140** [S] A system-wide shortcut (Alt/Option+Space, rebindable) opens a centred 620×440 floating launcher even when the app is unfocused. It is prewarmed hidden at boot.
  - It lists enabled plugins that have panels, in most-recently-used order.
  - Fuzzy search covers name, id, description and Chinese pinyin (full and initials).
  - Arrow keys choose, Enter opens, Esc dismisses.

### 24.13 Bundled and example plugins

- **F-PLUG-150** [S] Browser (bundled, on): see F-BRW-*.
- **F-PLUG-151** [S] File Manager (bundled, vendored MIT third-party, updatable): see F-FILE-02..05.
- **F-PLUG-152** [S] Official marketplace plugins mentioned:
  - Session Orchestrator (§16);
  - Token Insights, a usage dashboard with heatmap, KPIs and streaks, which replaced a removed built-in Usage page;
  - session importers for Claude Code / Codex / OpenCode / Pi / CC Switch.
- **F-PLUG-153** [S] Repo examples:
  - a "hello" demo exercising every contribution type;
  - a "roundtable" multi-role orchestration skill and tool;
  - a "UI slots lab" exercising every renderer slot and failure mode.

## 25. Skills

- **F-SKILL-01** [S] A skill is a Markdown doc (optionally a folder with resources) with name and description. The system prompt lists a catalog of skills (id, name, a short description), and the agent loads a skill's body on demand with a skill tool. Skills are not available in Plan/Goal.
- **F-SKILL-02** [S] Skill sources: user global, user per-project, plugin-contributed, and bundled. Bundled skills cover image generation and plugin development.
- **F-SKILL-03** [S] Settings → Skills workbench:
  - filters All / Global / Project, with counts;
  - search and a project picker;
  - New, Import (file or folder), "Scan other tools" and Market.
- **F-SKILL-04** [S] Skill rows show name, level badge, description, an enable switch and Edit. The overflow menu has Show in folder, Move to Global / Move into project, and Delete (click twice to confirm).
- **F-SKILL-05** [S] Skill editor: Name (with its slug shown), "When to use it", and the Markdown body with a size meter (128 KB cap).
- **F-SKILL-06** [S] Import from a single file, or from several folders that contain a skill file (sibling files are kept). Non-ASCII names fall back to the folder name. "Scan other tools" finds skills from Claude, Codex and similar tools, with an import mode of Copy or Symlink and a summary of imported, skipped and failed.
- **F-SKILL-07** [S] Skill Market:
  - categories All, Workflow, Writing, Coding, Data, Docs;
  - search and paging;
  - full-document preview before installing;
  - user sources: a GitHub skill repo or a JSON catalog;
  - built-in offline picks when a source is down;
  - errors explain proxy, fake-IP and DNS problems.
- **F-SKILL-08** [S] Users invoke skills by typing "/<skill>" in the composer (see F-COMP-13).
- **F-SKILL-09** [S] Activation scope follows F-PLUG-07.

## 26. MCP (Model Context Protocol)

- **F-MCP-01** [S] User MCP servers live in Settings → MCP:
  - the same workbench layout (All/Global/Project, search, project picker);
  - Add, Market and Scan other tools;
  - project records shadow global ones.
- **F-MCP-02** [S] MCP server rows:
  - a status dot: Not connected yet / Connecting / Connected / Failed;
  - transport badge, tool count, OAuth badge;
  - overflow: Test connection ("Connected · N tools"), Move, Remove, Authorize/Re-authorize.
- **F-MCP-03** [S] MCP editor:
  - Name, and an immutable Identifier used as the tool prefix;
  - transport "Local program" (command with npx/uvx/custom launcher helpers, arguments, env vars masked while typing) or "HTTP endpoint" (URL, headers; plain HTTP shows a warning);
  - description;
  - connection timeout (1–600 s, default 10);
  - scope.
- **F-MCP-04** [S] Paste a standard JSON block of MCP server definitions to add several at once (up to 32). Several input shapes are accepted, and bad entries are reported individually.
- **F-MCP-05** [S] "Scan other tools" imports MCP configs from Claude Desktop, Cursor, Codex, Claude Code, opencode and the ChatGPT desktop app.
- **F-MCP-06** [S] MCP runtime:
  - protocol versions 2025-06-18 and 2025-03-26 over stdio and streamable HTTP/SSE;
  - lazy connect, with tool lists cached;
  - up to 16 active processes and 64 tools per server;
  - transparent re-handshake after the transport is lost, without replaying the failed call;
  - stopping a session cancels its in-flight MCP request.
- **F-MCP-07** [S] MCP tools appear namespaced, are deferred behind tool search, and count as medium risk (see F-PERM-10). They are denied in Plan/Goal.
- **F-MCP-08** [S] Remote OAuth 2.1 for HTTP servers:
  - metadata discovery and dynamic client registration;
  - PKCE with a loopback callback;
  - tokens encrypted in the secret store;
  - a 401 marks the server "Authorization required";
  - manual authorize from the server menu.
- **F-MCP-09** [S] Stdio servers get a minimal environment (no provider keys). npx/uvx are resolved across version managers and Windows shims, and the login-shell PATH is used on Unix.
- **F-MCP-10** [S] MCP Market:
  - merges a built-in offline catalog, the official MCP Registry and user-added registry or catalog sources (the official one can't be removed);
  - categories All, Dev tools, Web, Docs, Data, Productivity;
  - search and Load more;
  - an install dialog shows the exact command or URL, prerequisites and masked required values;
  - an "Installed" badge;
  - nothing runs until the first connection;
  - "Registry unreachable — showing built-in picks" when a source fails.
- **F-MCP-11** [S] Market fetches use public HTTPS only, with private and loopback ranges refused, DNS pinning, an 8 s / 4 MiB cap and at most 5 redirects.
- **F-MCP-12** [S] Plugin-declared MCP servers (F-PLUG-19) use the same client. Their budgets are a 10 s handshake and 100 s per call, and a server that breaks the catalog limits is refused entirely.
- **F-MCP-13** [O] Desktop as an MCP server: see §40.

## 27. Image generation

- **F-IMG-01** [S] An agent tool generates or edits images. It works in Agent mode only, is high risk (so it is approved according to the permission mode), and is denied in Plan/Goal.
- **F-IMG-02** [S] Inputs and outputs:
  - each item has a prompt (up to 32k characters), a count, and 1–4 optional source images for editing;
  - 1–10 output images per call;
  - a previously generated image can be the source for the next edit;
  - originals are never overwritten, and outputs go to scratch with unique names.
- **F-IMG-03** [S] Supported backends: OpenAI-compatible Images APIs (generate and edit) and OpenRouter native images. There is no mask editor.
- **F-IMG-04** [S] Batch behaviour:
  - 2 workers in parallel;
  - 180 s per output;
  - no automatic retries, for billing safety;
  - partial successes are kept;
  - cancellable.
  Outputs must be PNG, JPEG or WebP and at most 16 MiB. URL downloads are SSRF-safe.
- **F-IMG-05** [S] Settings: mark models as image models and pick the default "Image generation model" (Set/Clear). Image models are excluded from the chat pickers, and OAuth accounts are not eligible.
- **F-IMG-06** [S] Results UI:
  - a thumbnail strip;
  - a full-window viewer with keyboard navigation, zoom/fit and "N of M";
  - download and show in folder;
  - a failure message per image;
  - a setup deep link when no image model is configured.

## 28. Voice

- **F-VOICE-01** [O] Live Voice: real-time voice calls with an AI provider. It is off by default and enabled in Settings → Voice, which is a regular page.
- **F-VOICE-02** [O] Live Voice providers:
  - Codex account (reuses the vendor login);
  - Gemini Live (API key);
  - OpenAI Realtime-compatible (API key plus endpoint, with a "Realtime GA" or "compat v1" profile).
  Each binding has a model ID, a voice, and a readiness status (Ready / Provider disabled / Credentials missing). Bindings cannot be edited during a call, and there is never an automatic fallback to another provider.
- **F-VOICE-03** [O] Composer preparation popover:
  - the selected binding and whether it is ready;
  - an opt-in "Share limited recent conversation context" checkbox;
  - Start.
  Nothing touches the mic or the provider until Start, and a call starts muted.
- **F-VOICE-04** [O] Call bar: a floating, draggable, always-on-top mini window over all apps that remembers its position.
  - Connecting/Cancel, Mute/Unmute, End, Resume sound, Details.
  - Phases: Preparing, Waiting for microphone, Negotiating, Connecting, Connected, Reconnecting, Ending, Ended.
  - Mod+Shift+V starts or ends a call; Esc cancels startup only.
- **F-VOICE-05** [O] Details panel: a transient transcript (in memory only, never stored), provider identity, the bound work session, and work status with Stop task / Cancel queued.
- **F-VOICE-06** [O] Safety:
  - one call at a time, with barge-in;
  - the call ends automatically on suspend, lock, window loss, a provider change, disable or quit;
  - if the mic cannot be confirmed released, new calls are blocked until a restart;
  - no recordings are kept;
  - audio, SDP and transcript content are never logged;
  - specific errors for mic denied, unavailable or in use.
- **F-VOICE-07** [X] Live Work (in progress): during a call the user can delegate work to a session by voice.
  - The default target is the current composer session. Targets can be switched by voice ("list sessions") or a new session created in a project.
  - A classifier routes each utterance: chat/status, steer the active turn, queue a task, ask about a past result, stop, or answer a pending question.
  - Work is tracked through stages from received to completed.
  - Progress is announced at quiet moments.
  - It fails closed on ambiguity or a workspace change.
- **F-VOICE-08** [S] Voice can never approve permissions or plans. It shows "Waiting for your decision" and an "Open the session" button. It may answer an open question only by choosing one of that question's offered options.
- **F-VOICE-09** [P] Local dictation (offline speech-to-text) is implemented but its UI is hidden.
  - Downloadable models: Whisper Small, Medium and Large V3 Turbo, and SenseVoice Small.
  - Language list, a Chinese variant (Simplified, Taiwan, Hong Kong), and a mic picker.
  - A listening/transcribing overlay.
  - It shares a microphone lease with Live Voice.
- **F-VOICE-10** [S] Host speech service (no UI): transcribe or synthesise through configured OpenAI-audio-compatible providers. It is consumed by plugins, and plugins can register their own adapters.
- **F-VOICE-11** [R] A Voice/speech settings card under AI was removed. There is no wake word.

## 29. Scheduled tasks (automations)

- **F-AUTO-01** [S] The "Scheduled" destination opens from the sidebar clock or a "Scheduled" search. It has a task rail and a task page.
- **F-AUTO-02** [S] Task form:
  - name;
  - instruction, with inline pickers for project, permission mode (with an Auto warning) and model/reasoning;
  - conversation mode: a new conversation per run, or reuse one.
- **F-AUTO-03** [S] Cadence options:
  - Manual (the default);
  - an interval of 5 min to 24 h;
  - Hourly;
  - Daily: Morning 09:00 / Afternoon 14:00 / Evening 19:00 / Night 22:00, or Custom HH:mm;
  - Weekly: several weekdays plus a time period.
  Schedules use the local timezone.
- **F-AUTO-04** [S] Runs happen only while the app is running:
  - missed runs are skipped, with no backlog;
  - a task never overlaps its own run;
  - Plan/Goal tasks cannot run unattended.
- **F-AUTO-05** [S] Task actions: Pause/Resume, Run now, Edit, Delete (which also deletes the history). The rail shows Next run and Last run.
- **F-AUTO-06** [S] Run history (latest 100):
  - status: Running/awaiting input, Completed, Interrupted, Failed;
  - start time, duration, error code;
  - the transcript can be read inline;
  - "Open conversation" opens it in chat with a "Back to Scheduled · <task>" link;
  - Retry.
  Run transcripts are hidden from the sidebar, tray and search.
- **F-AUTO-07** [P] Managing automations from chat: see F-TOOL-21.

## 30. Notifications

- **F-NOTIF-01** [S] Notification center (sidebar bell):
  - All / Unread filters, Mark all as read, Clear all, and "You're all caught up";
  - the inbox lists only failures ("<title> needs attention", with an error code), capped at 200;
  - clicking an item opens its project and session.
- **F-NOTIF-02** [S] Native OS notifications:
  - task finished or needs attention, only when the window is unfocused;
  - questions ("A question needs your answer"), tool approvals and plan approvals, unless that exact session is focused.
  They never navigate on their own.
- **F-NOTIF-03** [S] Toasts:
  - top-centre stack of up to 4;
  - info, success, warning, error;
  - 4 s, or 8 s for errors;
  - hover pauses them;
  - dismiss button.
- **F-NOTIF-04** [S] Unread badge: the dock or launcher badge on macOS and Linux, a taskbar overlay with digits on Windows (99+ cap).
- **F-NOTIF-05** [S] An update notice card in the top-right of the main pane (see §41).

## 31. Settings

- **F-SET-01** [S] Settings is a full-window page:
  - a left rail with "Search settings…" at the top and "Back to app" at the bottom;
  - groups Preferences / Agent / Workspace / System / Extensions;
  - each row has a "?" help tooltip;
  - developer-only pages disappear from the rail and from search when developer mode is off.
- **F-SET-02** [S] General:
  - Theme (§33), Language (§34), Font, Font size;
  - Network: proxy and relaxed mode (§46);
  - Storage (§36);
  - Power (F-SHELL-14);
  - Close behavior (Windows/Linux): Close to tray or Quit app; the first close asks once.
- **F-SET-03** [S] AI:
  - Permission mode default;
  - Default mode for new sessions;
  - Command shell;
  - Link open destination;
  - Thinking display mode;
  - Context usage readout (Remaining/Used);
  - Enter to send;
  - Retry until success;
  - Smooth streaming;
  - Large paste threshold;
  - a Prompt enhancement card: custom template editor with an insert-draft-variable button and Restore default, a model (follow current, or pinned), and reasoning (Off by default).
  There are no context-management settings.
- **F-SET-04** [S] Voice (§28).
- **F-SET-05** [S] Shortcuts (§32).
- **F-SET-06** [S] Instructions (F-INST-01).
- **F-SET-07** [S] Models:
  - Default model;
  - Image generation model;
  - Vendor accounts card (Add account, Edit, Test connection, Remove);
  - AI services list (§17);
  - "Import from other tools";
  - catalog status and "Refresh model catalog".
- **F-SET-08** [S] Skills (§25), MCP (§26), Subagents (§15).
- **F-SET-09** [S] Projects (F-PROJ-08).
- **F-SET-10** [S] Cloud sync (§35).
- **F-SET-11** [X] Remote hosts (§38): developer mode and dev builds only.
- **F-SET-12** [S] Info:
  - app, protocol and host versions;
  - Open logs;
  - "Report a problem", which opens a GitHub issue form prefilled with version and OS;
  - Updates: Automatic/Manual, Check for updates, the status, View release, What's new, and Release notes;
  - Developer card: a Developer mode switch and Open console.
- **F-SET-13** [S] Plugin-contributed Extensions group: scenic theme pages, plus per-plugin settings sheets reachable from Extensions.
- **F-SET-14** [S] Settings export never includes secrets. Settings import recreates provider shells and asks for keys again.
- **F-SET-15** [R] Removed settings pages: Import (its scans now live in each page), Usage (moved to a plugin), and the standalone Voice card under AI.

## 32. Keyboard shortcuts, menus and tray

- **F-KEY-01** [S] Customizable shortcuts (Mod = Cmd on macOS, Ctrl elsewhere):

  | Action | Default |
  |---|---|
  | Go back | Mod+[ |
  | Go forward | Mod+] |
  | New task | Mod+N |
  | Open project | Mod+O |
  | Open settings | Mod+, |
  | Search | Mod+K |
  | Command palette | Mod+Shift+P |
  | Plugin launcher (system-wide) | Alt/Option+Space |
  | Toggle sidebar | Mod+B |
  | Toggle side panel | Mod+J |
  | Stop active task | Mod+. |
  | Show/hide window (system-wide) | Alt+Shift+W |
  | Reset zoom | Mod+0 |
  | Zoom in | Mod+= |
  | Zoom out | Mod+- |
  | Full screen | F11, or Cmd+Ctrl+F on macOS |
  | Toggle Live Voice | Mod+Shift+V |
  | Cancel Live Voice setup | Esc |

- **F-KEY-02** [S] Shortcut editor:
  - grouped Navigation / Agent / Window, with platform-native labels;
  - click to record a chord, Esc to cancel;
  - per-row Restore default and Unbound, plus Restore defaults for all.
  Validation:
  - a chord needs a modifier, unless it is F1–F12;
  - OS- and editor-reserved chords are refused;
  - a duplicate shows "Already assigned to X".
  Changes apply instantly, including to menu accelerators and global registrations.
- **F-KEY-03** [S] Fixed keys:
  - Enter / Mod+Enter to send, Shift+Enter for a newline, Alt+Enter to steer;
  - Esc closes the innermost overlay;
  - Arrow Up/Down for prompt history;
  - Mod+↑/↓ for transcript top/bottom;
  - list navigation with arrows, Home/End, Enter/Tab, Esc;
  - work-panel tab keys;
  - sidebar reorder keys;
  - F12 and Ctrl+Shift+I open devtools in developer mode only.
  A browser-style reload key is swallowed on Windows/Linux.
- **F-MENU-01** [S] macOS app menu:
  - app menu: About, Check for Updates…, Settings…, Services, Hide, Hide Others, Show All, Quit;
  - File: New Task, Open Project…, Show/Hide Window;
  - Edit: the standard items;
  - View: Search…, Sidebar, Reload, Actual Size, Zoom In, Zoom Out, Toggle Full Screen, and Developer Tools in developer mode;
  - Window: Minimize, Zoom, Bring All to Front;
  - Help: Help, Open Logs, Refresh marketplace.
  Menu items are localized and follow custom bindings.
- **F-MENU-02** [S] Windows/Linux have no visible menu bar; the same commands work through shortcuts. The right-click system menu on the Linux drag strip is suppressed.
- **F-MENU-03** [S] Tray:
  - an icon on every platform (a template image on macOS);
  - menu: Open app; Running / Unread / Pinned session groups (up to 9 sessions, 3 per group, then "View more…"); Quit;
  - clicking a session opens it;
  - single click restores the window on Windows/Linux, double-click on macOS.
- **F-MENU-04** [S] Quit (menu, Cmd+Q or tray) asks for confirmation and warns that running sessions will stop. Hiding to the tray keeps agents running. On macOS, closing the window keeps the app in the Dock.
- **F-MENU-05** [S] Context menus open at the pointer, are clamped to the viewport, and close on outside press, Esc, Tab, blur or scroll.

## 33. Appearance, themes and fonts

- **F-THEME-01** [S] Theme picker:
  - searchable;
  - System (follows the OS, with no white flash), Light and Dark pinned at the top;
  - then plugin themes, each labelled "Provided by the X plugin";
  - an unavailable plugin theme falls back to System.
- **F-THEME-02** [S] Plugin themes can set background images and the native window background. On Windows they can also set the window corner radius.
- **F-THEME-03** [S] macOS sidebar vibrancy.
- **F-THEME-04** [S] Visual style:
  - a restrained neutral palette with one accent family and semantic colours;
  - Apple-style corner radius hierarchy;
  - code highlighting in a One-Dark-like theme.
- **F-THEME-05** [S] Font picker: searchable, with "System default" plus installed system fonts, each rendered in its own face. A "Saved" entry covers a stored font that is no longer installed. A CJK fallback is always appended, and no fonts are bundled.
- **F-THEME-06** [S] Font size presets named after coffee-cup sizes: Tall 85%, Grande 100%, Venti 115%, Trenta 125%, plus an 80–150% slider. This is independent of window zoom.
- **F-THEME-07** [S] Reduced motion is honoured: no pulses, transitions or smooth scroll, and smooth streaming turns off. Focus rings show only for keyboard focus.
- **F-THEME-08** [S] Every icon-only button has a localized tooltip that includes its shortcut.

## 34. Internationalization

- **F-I18N-01** [S] Nine UI languages:
  - English (the source language);
  - Simplified Chinese;
  - Traditional Chinese;
  - German, Spanish, French;
  - Korean, Turkish;
  - Brazilian Portuguese.
  "Match system" detects the OS locale, and the Hong Kong/Macau/Hant variants map to Traditional.
- **F-I18N-02** [S] Searchable language picker with native names, English first.
- **F-I18N-03** [S] Localized beyond the main UI: native menus, tray, window controls, notifications, splash, permission copy, and the in-app changelog (every release ships notes in every locale).
- **F-I18N-04** [S] Plugins read the app locale. Plugin manifests localize to English or Simplified Chinese, and other locales fall back to English.
- **F-I18N-05** [S] Pinyin search in the plugin launcher and the command search. Reasoning-level names are deliberately left untranslated.
- **F-I18N-06** [S] The documentation site is bilingual (English and Chinese).

## 35. Configuration sync (encrypted WebDAV)

- **F-SYNC-01** [S/F] Settings → Cloud sync is configuration-only sync to a WebDAV server the user chooses. Notes conflict on whether its page is gated for all users; see §47.
- **F-SYNC-02** [S] Setup:
  - server URL, username, app password, remote directory, device label;
  - a vault password (8+ characters, never leaves the device);
  - server mode: Strict (conditional writes, recommended) or an append-only Compatibility mode, which comes with a warning;
  - Test server.
- **F-SYNC-03** [S] Categories:
  - on by default: app preferences, providers and models, MCP definitions, skills (with bounded resources), subagents, instructions and templates, projects and folder mappings, plugin install intent, automation definitions;
  - opt-in: project memory and portable credentials.
  Never synced: conversations, source files, OAuth tokens, cookies, the WebDAV password and machine keys.
- **F-SYNC-04** [S] Encryption is client-side (a password-derived key, per-object authenticated encryption, an opaque remote layout). HTTPS is required; plain HTTP is allowed only for LAN hosts after acknowledgement, and redirects are refused.
- **F-SYNC-05** [S] Merge and safety:
  - a three-way merge per entity, with conflicts shown side by side;
  - deletions sync as tombstones;
  - imported executable content (MCP, skills, subagents, plugins, automations) needs local approval on each device: Approve / Reject / Map folder, with a stated reason;
  - imported automations stay disabled until approved.
- **F-SYNC-06** [S] Operation:
  - "Sync now", with phases collect, download, merge, upload, apply, cleanup;
  - automatic sync on startup, 30 s after a local change, and on a 5-minute poll;
  - retry with backoff.
- **F-SYNC-07** [S] Status values: Not configured, Locked, Up to date, Local changes pending, Syncing, Offline, Unsupported server, Conflict, Awaiting activation, Paused, Needs attention.
- **F-SYNC-08** [S] Maintenance:
  - revision history and restore (30 kept);
  - change the vault password;
  - unlock;
  - pause this device;
  - disconnect, which keeps the data on both sides;
  - a setup preview with counts.

## 36. Data directory and storage

- **F-DATA-01** [S] All data lives in one per-user data folder, with a separate folder for dev builds. It holds:
  - a local database index;
  - one human-readable transcript file per session;
  - variants;
  - streaming checkpoints;
  - encrypted secrets;
  - content-addressed attachments;
  - plugins, logs, caches and crash dumps;
  - review snapshots;
  - scratch folders.
  An environment variable can override the location.
- **F-DATA-02** [S] Settings → Storage → "Change location…": the app restarts and runs a migration window (scan, copy, verify, update paths). The old location is kept as a backup until "Remove backups…".
  - A failed migration keeps the current location and offers Retry.
  - A missing drive blocks startup instead of creating an empty profile.
- **F-DATA-03** [S] "Clear cache…" shows the reclaimable size and deletes only caches. Transcripts, secrets, settings, logs and plugins are never touched.
- **F-DATA-04** [S] Storage controls are disabled, with an explanation, when the location is managed by an environment variable. Database migrations take an automatic backup first.
- **F-DATA-05** [S] Retention:
  - transcripts are never auto-pruned;
  - audit logs 90 days;
  - run history 100 per task;
  - notification inbox 200.

## 37. Importing from other tools

- **F-IMP-01** [S] Session import from Claude Code, Codex, OpenCode, Pi and CC Switch:
  - scan, then group by project path or source, then select and import;
  - idempotent;
  - bound to projects by path, restoring an archived project if needed;
  - origin badges.
  Delivered through the plugin session-import API; there is no longer a Settings page for it.
- **F-IMP-02** [S] Provider/model import (F-PROV-14), skill import (F-SKILL-06) and MCP import (F-MCP-05).

## 38. Remote hosts and remote agent control

- **F-REM-01** [X] Settings → Remote hosts is marked "Experimental", needs developer mode, and appears only in dev builds.
  - It is a host list showing Online/Offline with Remove.
  - The Add form has SSH and Pair tabs, and keeps typed values when switching tabs.
- **F-REM-02** [X] SSH install form:
  - Label, Host, User, Port;
  - auth: Key (identity file) or Password (show/hide);
  - "Install" → "Installing…" → success or "Install failed: <reason>".
  It uses the system ssh client, so the user's ssh config, agent and jump hosts apply.
- **F-REM-03** [X] Bootstrap steps:
  1. Detect the remote OS and architecture. Linux x64 and arm64 only.
  2. Fetch the headless host bundle at exactly the desktop's version and verify its published SHA-256.
  3. Install under the user's home with a "current" link. Node 22+ is required on the remote.
  4. Start the host on loopback with a one-time pairing token.
  5. Open a persistent SSH port forward.
  6. Exchange the pairing token for a device token, stored encrypted.
  Typed failure steps cover: missing tool, old Node, download failed, checksum mismatch, extract failed, host exited, ready timeout. A bootstrap can be re-run safely.
- **F-REM-04** [X] Password authentication:
  - goes through a private, short-lived askpass file, never argv or env;
  - is stored encrypted with OS secure storage;
  - allows exactly one attempt;
  - is refused on Windows OpenSSH;
  - a wrong password shows the host Offline with ssh's last error line.
- **F-REM-05** [X] Manual Pair tab: label, WebSocket URL and pairing token, for a host started by hand.
- **F-REM-06** [X] Remote sessions:
  - appear in the normal list with a "Remote host" badge;
  - the remote machine owns the transcript, tools, workspace, permissions, secrets, skills, MCP and schedules;
  - the work panel lists and reads files and diffs from the remote;
  - the browser preview stays local;
  - a failed host falls back cleanly, and local sessions are never affected.
- **F-REM-07** [X] Control protocol, as users experience it:
  - a turn keeps running when the client disconnects;
  - reconnect resumes from a cursor without duplicates, or does a full resync after a host restart;
  - pending approvals and questions are visible to a client that attaches late;
  - the first decision wins across all clients;
  - remote approvals expire after 30 min;
  - remote-started turns are capped at a host-defined maximum permission mode (Ask by default), except for the SSH-paired owner;
  - up to 8 queued turns per session.
- **F-REM-08** [X] Session roles: viewer, controller, approver, owner. A controller cannot approve its own requests unless it also holds the approver role.
- **F-REM-09** [PL] Not built:
  - reverse tool relay (desktop MCP/plugin tools offered to remote sessions);
  - a remote terminal client;
  - provider-config push over SSH;
  - outbound chat/webhook notifications (Telegram, Slack, WeChat, webhook) with chat-command stop/approve;
  - a self-hosted gateway, browser client or gRPC binding;
  - mobile apps.

## 39. Headless host CLI (`pi-host`)

- **F-CLI-01** [X] `pi-host` runs the agent host without any UI and serves the remote-control protocol over WebSocket on loopback. Options:
  - data directory;
  - port (0 picks a free one);
  - bind address (loopback only unless TLS);
  - print a one-time pairing token, and set its lifetime (default 10 min);
  - paths to the backend binary, agent bundle and Node;
  - root for the remote folder browser;
  - log level;
  - help.
  Each option has an environment-variable equivalent.
- **F-CLI-02** [X] Machine-readable stdout lines report ready (host id, address, port, version), the pairing token, and startup failure. Everything else goes to stderr and a log file, with secrets masked.
- **F-CLI-03** [X] Supervision:
  - restarts its child processes on crash;
  - recovers in-flight messages;
  - orderly SIGINT/SIGTERM shutdown;
  - advertises terminal support only if the optional PTY module is present.
- **F-CLI-04** [X] Distributed as Linux x64/arm64 tarballs with checksums and an install script that keeps versioned folders plus a current link.
- **F-CLI-05** [S] The desktop app itself has no user-facing CLI subcommands.

## 40. Local automation endpoint (desktop as MCP server)

- **F-CTRL-01** [O] An optional loopback-only MCP server (streamable HTTP) lets external agents and scripts drive the running app. It is enabled by an environment variable, with a configurable port.
  - Bearer token plus a connection file written owner-only.
  - Origin validation against DNS rebinding.
  - Supports MCP 2025-06-18 and 2025-03-26.
- **F-CTRL-02** [O] It exposes named tools plus a generic "describe" and "invoke" over a risk-tagged operation catalog:
  - sessions: list, create, fork, open, rename, configure, move, revisions, scratch path, title, delete;
  - agent: prompt, compact, abort, stop, status;
  - instructions read/save;
  - permission, question and plan resolution;
  - notifications;
  - updates;
  - providers and catalog;
  - projects;
  - workspace diff and file reads;
  - usage history;
  - browser state;
  - scheduled tasks;
  - plugins, views, themes, services;
  - MCP, skills, subagents;
  - marketplace and palette search;
  - run an extension command.
- **F-CTRL-03** [O] Dangerous or destructive operations require an explicit confirm flag. Secret access, settings writes, native pickers and dev-plugin loads are never exposed. Mutations refresh the visible UI.

## 41. Application updates

- **F-UPD-01** [S] The updater uses a fixed GitHub Releases feed, stable channel only (prereleases are never offered). It is disabled in dev builds. Checks run about 15 s after launch and then every 6 h.
- **F-UPD-02** [S] Delivery by install type:
  - in-app download plus "Restart to update" (install-on-quit as a fallback): macOS, Windows installer, Linux AppImage;
  - notify and link to the release: Linux deb/rpm, Windows portable ZIP.
- **F-UPD-03** [S] Per-install preference, Automatic or Manual:
  - portable builds default to Manual, with a warning before opting in;
  - Manual mode reminds once per version;
  - dismissing an update cancels its download and stays dismissed until a newer version appears.
- **F-UPD-04** [S] States: checking, up to date, available, downloading N%, ready, error. Automatic failures stay quiet; manual checks report errors.
- **F-UPD-05** [S] "What's new": localized release notes ship inside the build. There is a full changelog dialog with Current and Available badges.
- **F-UPD-06** [PL] Not built: rollback, staged rollout, beta/RC channels.

## 42. Packaging and platforms

- **F-PKG-01** [S] macOS:
  - Apple Silicon and Intel;
  - a branded drag-to-Applications DMG and a ZIP, with architecture in the file names;
  - hardened runtime;
  - Developer ID signed and notarized for tagged releases (per the README and ADRs, though the local build config shows no signing identity);
  - microphone and local-network usage strings, the latter so LAN model servers work.
- **F-PKG-02** [S] Windows x64:
  - an NSIS installer that is not one-click and lets you choose the install directory;
  - a portable ZIP (no admin needed);
  - a legacy portable EXE.
  ARM64 runs through emulation. No VC++ redistributable is needed.
- **F-PKG-03** [S] Linux x64 and arm64: AppImage, deb, rpm, plus a raw app archive for repackaging with a system runtime. Requires glibc 2.35+ (Ubuntu 22.04, Debian 12, Fedora 36).
- **F-PKG-04** [S] No separate runtime is shipped: the agent runs on the desktop shell's own runtime. Bundled pieces:
  - the native backend binary;
  - the agent bundle;
  - the built-in skills and plugins;
  - a model catalog snapshot;
  - native audio and transcription modules.
- **F-PKG-05** [S] A dev build has its own app identity and data folder, so it can run beside the release build.
- **F-PKG-06** [S] Releases are mirrored to a Chinese git host for users in that region.

## 43. Logging and diagnostics

- **F-DIAG-01** [S] Local structured logs:
  - split by process (app, backend, agent) and by category (lifecycle, session, tool, permission, plugin, provider, persistence, updater, diagnostics, runtime);
  - each file rotates at 5 MB, keeping 2;
  - info level in release builds;
  - correlation ids on every record.
- **F-DIAG-02** [S] Redaction: secrets, auth headers, tokens, URL credentials and file contents are never logged, and home and app-data paths are replaced with placeholders.
- **F-DIAG-03** [S] "Open logs" is available from Settings → Info, the Help menu, error screens and the backend banner. "Copy diagnostics" exists on boot problems. Errors show a stable code and a copyable trace id.
- **F-DIAG-04** [S] Crash dumps for every process type are stored locally and never uploaded. New dumps are summarised in the log on the next launch.
- **F-DIAG-05** [S] Per-plugin log files, and an audit trail in the local database.

## 44. Privacy and telemetry

- **F-PRIV-01** [S] Zero telemetry and no analytics SDK. No account is needed.
- **F-PRIV-02** [S] Content leaves the machine only to the providers, plugins and MCP servers the user configured.
- **F-PRIV-03** [S] Other network contacts:
  - the model catalog refresh;
  - the marketplace;
  - GitHub for updates;
  - OAuth endpoints;
  - the skill and MCP markets.
  The official plugin channel sends an anonymous hashed device id.
- **F-PRIV-04** [S] Disclosed trade-off: Markdown written by the model may auto-load remote images or media, which reveals the user's IP.
- **F-PRIV-05** [S] A published privacy policy covers what is stored locally, retention, and how to ask for removal.

## 45. Security model

- **F-SEC-01** [S] Hardened renderer:
  - context isolation, no Node, sandboxed;
  - a whitelisted bridge;
  - no window.open, no in-app navigation, no webview tags;
  - only http, https and mailto are handed to the OS;
  - a strict CSP;
  - no provider traffic from the renderer.
- **F-SEC-02** [S] Secrets are held only by the trusted backend: AES-256-GCM files with an owner-only machine key.
  - The UI sees only "configured" flags.
  - The agent gets a key in memory per run, or a short-lived token.
  - A decrypt failure is treated as a missing key and asks for re-entry.
  - Disclosed limitation: same-user processes could decrypt. An OS keychain backend is reserved but not built, even though the README says keychain. Remote-host secrets do use OS secure storage.
- **F-SEC-03** [S] Sanitized Markdown, strict Mermaid, and sanitized SVG.
- **F-SEC-04** [S] Security reporting policy: private email or advisory, acknowledgement within 7 days, assessment within 14 days, latest release only, no bounty.
- **F-SEC-05** [PL] Planned: hardened runtime fuses, a plugin capability sandbox, and publisher signatures.

## 46. Networking

- **F-NET-01** [S] Proxy: System / Direct / Custom (HTTP, HTTPS or SOCKS5 URL with credentials; a bypass list that defaults to localhost and private LANs; Test). It applies to model calls, the marketplace, updates, the in-app browser and Live Voice sockets. Credentials go through a local relay and are hidden from shell commands.
- **F-NET-02** [S] Relaxed network mode, on by default:
  - user-typed LAN, loopback and plain-HTTP endpoints are allowed;
  - fake-IP proxies are tolerated;
  - a one-time notice appears on the first plaintext connection.
  Strict mode allows public HTTPS only.
- **F-NET-03** [S] Market fetches (skills, MCP) use safe public HTTPS only, with DNS pinning and per-hop redirect checks.

## 47. Discrepancies observed (decide deliberately for Orphus)

- Scheduled-task tools usable from chat: the user guide says they exist, but the runtime notes say they are not registered (F-TOOL-21).
- Credentials: the README says "OS keychain", but the shipped store is encrypted files with a local key (F-SEC-02).
- Cloud sync: one source says it is a regular page for everyone, another says it sits behind a dev gate (F-SYNC-01).
- macOS signing: the README and ADRs say signed and notarized, but the build config has no signing identity, so signing presumably happens in CI.
- Plugin theme assets inside the package: one decision re-allowed them, but the dev-kit check still rejects them.
- The "Pull Requests"/GitHub workspace and the terminal appear in marketing but have been removed from the core. They are framed as plugin territory.
- Remote hosts are described as shipped in specs, but are dev-build only in practice.

## 48. Features explicitly removed (do not rebuild unless intentionally)

- **F-GONE-01** Message quotes, annotations, side chats, and the selected-text toolbar.
- **F-GONE-02** The interactive terminal panel.
- **F-GONE-03** The Pull Requests page and PR-listing tool.
- **F-GONE-04** Subagent peer messaging, the agent-to-agent broker, and subagent watchdog and turn limits.
- **F-GONE-05** Chat mode.
- **F-GONE-06** The standalone command palette overlay.
- **F-GONE-07** Built-in commands beyond the core five: delete session, abort, settings, plugin, logs, rename, reload, devtools.
- **F-GONE-08** Settings Import page, Settings Usage page, and the Voice card under AI.
- **F-GONE-09** Bundled fonts, diagnostic timing logs, and the unsigned-macOS quarantine helper.
- **F-GONE-10** Background pre-computed compaction, and growing the native window for the work panel.
- **F-GONE-11** Empty-home starter cards, the composer context rail, and an in-window menubar on Windows/Linux.
- **F-GONE-12** A first-party "Advisor" reviewer plugin (temporarily not shipped).

## 49. Planned / not yet built (parity optional)

- **F-PLAN-01** An OS keychain secret backend, plus a one-time warning when falling back.
- **F-PLAN-02** Native PDF document input.
- **F-PLAN-03** Plugin publisher signatures, beta channels, rollback, private sources, ratings, a capability sandbox, resource limits and secret settings.
- **F-PLAN-04** Plugin lifecycle hooks beyond load/unload, and extra activation events.
- **F-PLAN-05** An audio device backend for plugins.
- **F-PLAN-06** A runtime hook catalog for plugins, and a host "ask" card.
- **F-PLAN-07** Remote reverse tool relay, a remote terminal, outbound messaging notifications, a gateway/browser client, and macOS remote hosts.
- **F-PLAN-08** App update rollback, staged rollout and an RC channel; a native Windows ARM64 build.
- **F-PLAN-09** Live Work device qualification (every manual acceptance case is still unrun).
- **F-PLAN-10** A per-plugin log panel, panel DevTools and a mock tool invoker for plugin developers.
