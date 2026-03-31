Claude Code: How Your Prompt Becomes an API Call
The journey from typing in the terminal to hitting the Anthropic API.
Step 1 — You Type Something
screens/REPL.tsx → PromptInput component
You type "fix the login bug" and press Enter. The React-based terminal UI (Ink) captures your text.
Like typing into a chat box, but it's your terminal.
Step 2 — Input Processing
utils/processUserInput/processUserInput.ts
Your text is checked for:
Slash commands (/help, /commit, /bug)
File attachments (dragged images, pasted screenshots)
@-mentions (referencing files or agents)
Plain text passes through unchanged.
Like a mail room sorting letters before delivery.
Step 3 — Three Things Fetched in Parallel
constants/prompts.ts + context.ts
While your message waits, three things are gathered simultaneously:
The System Prompt is fetched via getSystemPrompt() in constants/prompts.ts:444. This is the giant instruction manual — over 10 sections telling Claude who it is and how to behave.
The User Context is fetched via getUserContext() in context.ts:155. This loads the contents of your CLAUDE.md files and today's date.
The System Context is fetched via getSystemContext() in context.ts:116. This grabs your current git branch, recent commits, and working tree status.
Like a chef gathering the recipe, your dietary preferences, and today's fresh ingredients — all at once.
Step 4 — The System Prompt Is Assembled
constants/prompts.ts:444 → getSystemPrompt()
A huge instruction document is built from 10+ sections stitched together:
asciidoc

┌─────────────────── STATIC (cached across turns) ────────────────────┐
│                                                                      │
│  1. "You are Claude Code, Anthropic's official CLI for Claude."      │
│  2. System rules: markdown rendering, tool permissions, hooks        │
│  3. How to do software engineering tasks                             │
│  4. Be careful with risky/destructive actions                        │
│  5. Which tools you have and how to use them                         │
│  6. Tone: concise, no emojis, use file_path:line_number format       │
│  7. Output efficiency: "Go straight to the point"                    │
│                                                                      │
├─────────────────── CACHE BOUNDARY ──────────────────────────────────┤
│                                                                      │
│  8. Session guidance: agent tool hints, skill hints                  │
│  9. Auto-memory: things you asked Claude to remember                 │
│ 10. Environment: macOS, CWD, git repo info, model name              │
│ 11. Language preference                                              │
│ 12. MCP server instructions                                         │
│ 13. Token budget / effort guidance (if set)                          │
│                                                                      │
└─────────── DYNAMIC (changes per turn, not cached) ──────────────────┘
The boundary between static and dynamic sections enables prompt caching — the static half is reused across turns without re-processing.
Like writing a very detailed job description before someone starts work.
Step 5 — Your Context Is Wrapped and Prepended
utils/api.ts:449 → prependUserContext()
Two things happen:
A. Your CLAUDE.md files and today's date are wrapped in a <system-reminder> tag and injected as a fake "user message" before your real conversation:
xml
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Contents of /Users/you/project/CLAUDE.md:
- Use TypeScript strict mode
- Prefer functional components
...
# currentDate
Today's date is 2026-03-31.
</system-reminder>
B. Git status is appended to the end of the system prompt.
Where CLAUDE.md files come from
getUserContext() walks the directory tree looking for these files:
CLAUDE.md in your project root — shared with your team via git
.claude/CLAUDE.md — also project-scoped, shared with team
.claude/CLAUDE.local.md — local and private, gitignored by default
~/.claude/CLAUDE.md — your global config, applies to every project
~/.claude/projects/*/memory/*.md — auto-memory files that Claude writes on its own when you ask it to remember things
Like pinning a sticky note to the top of a stack of papers: "Remember, this project uses TypeScript and prefers tabs."
Step 6 — Context Management
query.ts:219 → queryLoop()
Before sending, the conversation is trimmed to fit the context window:
Trim big tool outputs — if a tool returned 50,000 chars, it gets cut down
Microcompact — old tool results are summarized to save tokens
Auto-compress — if the whole conversation is still too long, older turns are compressed
Like summarizing yesterday's meeting notes so today's agenda fits on the whiteboard.
Step 7 — The API Call Is Made
services/api/claude.ts:1017 → queryModel()
Everything is packed into the final request:
What the API actually receives:
javascript
anthropic.messages.stream({

  system: [
    // TextBlockParam[] with cache_control markers
    "You are Claude Code, Anthropic's official CLI for Claude.",
    "...how to do tasks...",
    "...be careful with destructive actions...",
    "...list of tools and how to use them...",
    "...tone and style rules...",
    // --- cache boundary ---
    "...session guidance, skills, memory...",
    "...environment: macOS arm64, CWD, git info...",
    "...MCP instructions...",
    "...git status appended here...",
  ],

  messages: [
    // Synthetic user msg with CLAUDE.md + date
    { role: "user", content: "<system-reminder>...CLAUDE.md + date...</system-reminder>" },
    // Synthetic user msg with deferred tool list
    { role: "user", content: "<available-deferred-tools>...</available-deferred-tools>" },
    // ... older turns (possibly compacted) ...
    // Your actual message
    { role: "user", content: "fix the login bug" },
  ],

  tools: [
    // ~30+ tool schemas
    { name: "Bash", ... },
    { name: "Read", ... },
    { name: "Write", ... },
    { name: "Edit", ... },
    { name: "Glob", ... },
    { name: "Grep", ... },
    { name: "Agent", ... },
    { name: "WebFetch", ... },
    { name: "ToolSearch", ..., defer_loading: true },
    // + MCP tools, task tools, etc.
  ],

  model: "claude-opus-4-6",
  thinking: { type: "enabled", budget_tokens: ... },
  max_tokens: ...,
  temperature: 1,
  speed: "fast",  // if fast mode
  metadata: { user_id: "..." },
})
The response streams back → tools are executed → results added to messages → the loop repeats until Claude responds without tool use.
The chef finally puts the dish in the oven.
Key Files Reference
If you want to dig into the source yourself, here's where everything lives:
screens/REPL.tsx — captures user input and orchestrates the 3 parallel fetches
utils/processUserInput/processUserInput.ts — handles slash commands, attachments, and @-mentions
constants/prompts.ts:444 — builds the full system prompt from 10+ sections
constants/system.ts — defines the CLI prefix string and attribution header
context.ts:155 — getUserContext() loads all your CLAUDE.md files and today's date
context.ts:116 — getSystemContext() fetches git branch, status, and recent commits
utils/claudemd.ts:1153 — walks the directory tree to find every CLAUDE.md file
utils/systemPrompt.ts — priority selection logic: agent prompt vs custom vs default
utils/api.ts:437-449 — wraps your context as a <system-reminder> and appends git status
query.ts:219 — the main loop: compact context, send to API, execute tools, repeat
QueryEngine.ts:209 — the SDK/headless path (same pipeline, different entry point)
services/api/claude.ts:1017 — final API request assembly and streaming
The Big Picture
asciidoc

  You type: "fix the login bug"
       │
       ▼
  ┌─────────────┐
  │ Process      │  slash commands? attachments? @-mentions?
  │ User Input   │
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────────────────────────────┐
  │          Fetch 3 things in parallel          │
  │                                             │
  │  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
  │  │ System   │ │ CLAUDE.md│ │ Git Status  │ │
  │  │ Prompt   │ │ + Date   │ │ + Branch    │ │
  │  └──────────┘ └──────────┘ └─────────────┘ │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
  ┌─────────────────────────────────────────────┐
  │         Assemble Everything                  │
  │                                             │
  │  system = [prefix + prompt sections + env]  │
  │  messages = [<system-reminder> CLAUDE.md]   │
  │           + [older turns, compacted]        │
  │           + [your message]                  │
  │  tools = [Bash, Read, Edit, Glob, ...]      │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
  ┌─────────────────────────────────────────────┐
  │    Context Management                        │
  │    trim big outputs, compact old turns,      │
  │    auto-compress if too long                 │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
  ┌─────────────────────────────────────────────┐
  │    anthropic.messages.stream(params)          │
  │                                             │
  │    → Response streams back                   │
  │    → Tools executed if needed                │
  │    → Results added to messages               │
  │    → Loop repeats until done                 │
  └─────────────────────────────────────────────┘