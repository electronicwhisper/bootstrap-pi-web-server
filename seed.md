# Overview

- This machine is administered by the pi coding agent
- This document specs a web server that runs on the machine
- The web server is the primary interface into the machine and is intended to be a convenient and richly interactive alternative to SSH'ing into the machine and talking to pi through the terminal
- It provides a chat interface for talking to pi, similar to how you can talk to Claude, ChatGPT, etc., on the web. The difference is there's a full, persistent machine that the agent has access to, the chat history is shared by a small team, and the web interface itself can be extended by the agent!
- The web server also provides a navigable file browser for viewing, downloading, editing, and uploading files on the machine

# Setup

1. Download and read this entire document
2. Write `~/AGENTS.md`
3. Install Bun and any other necessary software
4. Initialize a Bun project in `~/web-server`
5. Implement the server as specified
6. Set it to run on startup, port 8000
7. Copy this document to `~/web-server/seed.md` for future reference

# AGENTS.md

Copy this content to `~/AGENTS.md` so that new pi sessions know how to use this system:

## Overview

- This is a machine administered by the pi coding agent
- Users interact with pi through the web interface that's implemented in `~/web-server`
- The machine is already secured at the network level so no auth is needed on the web server
- Assume anyone who has access to the web server (our small team) has full privileges on the machine

## File Server

The web server serves all files in the home directory via the `/raw/**` route. There is also a browseable route `/files/**` with uploading, editing, and pretty rendering of markdown and code.

## Chat

- The web server has a chat interface at `/chat` which is where users chat with pi. Pi's responses are markdown-rendered.
- This setup lets pi make _artifacts_ in response to user queries. For example, pi can make a self-contained HTML page at `~/artifacts/2026-04-26-recipe-calculator.html` and then include a link in the response (leveraging the raw file server), e.g., `[Recipe Calculator](/raw/artifacts/2026-04-26-recipe-calculator.html)`.

## Things we might build

Here are some examples of things users may ask to build or do on this system:

- Downloading files (code repos, research papers, etc.) into folders and then asking questions about them in chat
- Writing scripts to scrape data from the web
- Writing scripts to access data from our team's internal systems
- Writing cron jobs
- Writing scripts that can notify us about things
- Writing scripts that call APIs to get work done
- Adding APIs to the web server that can be accessed by artifacts
- Setting up web-hook-triggered scripts
- Adding custom dashboard pages to the web server
- Adding custom features to the chat interface

## Preferences

- Prefer TypeScript with Bun (Bun has many [batteries included](https://bun.com/llms.txt)) but use other languages when they're a better fit for the job
- Place one-off artifacts in `~/artifacts` like the example above. When users ask to make more involved projects on the machine, like cron jobs, data scrapes, or reusable CLI tools, put these appropriately in folders in `~`.
- Keep implementations as simple as possible so that they can easily evolve
- Minimize external dependencies unless they're robust and a great fit for the problem
- All pages should work well on mobile and desktop browsers
- This system is for the team's internal use. Things don't need to be super polished or even optimal performance. Keep things usable and nice looking, but not at the expense of making the system harder to maintain. Optimize for simplicity, development speed, learning, and flexibility.

# Stack

- [Bun](https://bun.com/llms.txt) and TypeScript
- Use Bun's [integrated dev server](https://bun.com/docs/bundler/fullstack.md) with `development: true`. This lets us bundle client-side JS without a build step. (The system philosophy is that it's always under development, so these kinds of settings are often preferred over "production".)
- pi coding agent SDK (your system prompt should indicate where you can find docs)
- Client side: [marked](https://github.com/markedjs/marked), [CodeMirror](https://codemirror.net/), [highlight.js](https://github.com/highlightjs/highlight.js)

# Web Server Routes

```
GET /                 -> home SPA   (HTMLBundle)
GET /chat, /chat/*    -> chat SPA   (HTMLBundle)
GET /files, /files/*  -> files SPA  (HTMLBundle)
GET /raw/**           -> raw bytes  (function handler)
*   /api/**           -> JSON/WS    (function handlers)
```

# Chat

- Serve at `/chat`
- Use pi's SDK
- Use WebSockets to stream data
- Use markdown rendering where appropriate
- You should be able to start a new conversation or browse a list of old conversations in a left side panel
- Each chat conversation is served at `/chat/**` where `**` mirrors pi's JSONL storage scheme. Pi stores sessions at `~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl` (where `<path>` is the working directory with `/` replaced by `-`) so our URLs should be `/chat/--<path>--/<timestamp>_<uuid>`.

## Layout and UX

- No header bar, just a main scrolling area with all the chat messages and a bottom bar where you can type a message and send it
- Bottom bar has a text field and a button on either side of it.
  - On the left is a ⋯ button which opens up a list of additional actions:
    - Switch Model: opens a modal to let you pick a model
    - Switch Thinking Level: opens a modal to let you pick thinking level
    - Upload Image: opens browser file picker
  - On the right is a ↑ button to send. The send button turns into ■ while pi is responding and that lets the user stop pi (use `session.abort()`).
  - On mobile: Pressing enter in the chat field starts a new line. You have to press ↑ to send.
  - On desktop: Press shift-enter for a new line. Pressing enter sends.
  - If you paste an image it should add that to the message you're composing
  - Show image attachments as thumbnails above the text input, with `x` buttons to remove them
- Messages include:
  - User messages: 👤 You, blue background, markdown render
  - Assistant messages: 🤖 Assistant, green background, markdown render
  - Assistant thinking: 🧠 Thinking, gray background, scrolling text field
  - Assistant tool calls: e.g. 🔧 bash, gray background, scrolling `pre` field
  - Tool responses: 📋 Result or ❌ Error, gray background, scrolling `pre` field
  - System messages: ⚙️ Model or ⚙️ Thinking Level, gray background
- We want to emphasize the chat (user and assistant messages) but we also want visibility into all the messages so the user understands what's going on behind the scenes
- All messages from pi get streamed in on the WebSocket for real-time feedback
  - When streaming into a scrolling field, scroll it so we can see the data coming in
  - Scroll the main messages area as messages stream in
  - But don't scroll any area where the scroll is not already at the bottom (e.g., the user has scrolled up to look at something)
- User messages have a small Edit button to branch the conversation. When there's a branch it shows e.g., `← 1 of 3 →` to let you switch branch.
- On mobile there's a floating hamburger menu icon in the top-left which opens the left side panel. On desktop the left side panel is always open. The chat scrolling area has enough padding-top so that the hamburger icon doesn't cover it up when scrolled to the top.
  - The left side panel lets you go back to the homepage 🏠, start a new chat, and has a list of old chats, recent on top. You can click these to open and continue them.

## Implementation notes

- State is persisted by the pi SDK itself in `~/.pi/agent/sessions/**` JSONL files. The server is stateless across restarts.
- Use the home directory as the `cwd` for all new pi sessions
- Use `createAgentSession`. We want to mirror the semantics of opening a pi session in the terminal. Make sure to load extensions with `session.bindExtensions({})`.
- **New chat:** call `createAgentSession({ sessionManager: SessionManager.create(homedir) })`
- **Resume chat:** call `createAgentSession({ sessionManager: SessionManager.open(path) })`
- **List chats:** call `SessionManager.listAll()`
- Session multiplexing: The server maintains at most one `AgentSession` per conversation (JSONL file). Multiple WebSocket connections can attach to the same session. On connect, the server sends the current session entries. After that, all observers see the same event stream in real time. The session is created when the first WebSocket connects and disposed when the last WebSocket disconnects and pi is done responding. If a user sends a message while pi is already responding, it is queued as a follow-up.

## Gotchas

- Client-server abstraction boundary: The server should have an API that mirrors pi's SDK as much as possible, then the client should have code to e.g., render messages as markdown. This way we can iterate on presentation without needing to restart the server. See appendix for suggested protocol.
- CSS: Getting the layout to work right on iOS is harder than it should be. The main messages area must never scroll horizontally but individual elements (e.g., tables, code blocks) can scroll horizontally if necessary. The bottom input bar must always be at the bottom, even if iOS has the URL bar or keyboard open. See appendix for working example code.

# Raw File Server

- Serve all raw files at `/raw/**` which mirrors the file system at `~/**`

# Browseable File Server

- Serve a browseable view at `/files/**` which mirrors the file system at `~/**`
- On top is
  - 🏠 button (goes back to top-level home page)
  - breadcrumbs (e.g., `~/folder/sub-folder`) and you can click any to navigate back up the hierarchy
  - ⋯ button that opens a menu
    - For folders:
      - Upload: opens a file picker
      - New folder: prompts for a name
      - New file: prompts for a name, makes a blank file
      - Show hidden: default off, persist this preference in `localStorage`
    - For files:
      - Copy to clipboard (use a maximally compatible implementation)
      - Edit (for textual files, puts the contents in a codemirror, has a save button)
      - Raw
- A folder shows a listing of subfolders and files. Each entry as a ⋯ button along a rightmost column:
  - Rename
  - Delete: confirm this
- For files
  - `.md` - render the markdown
  - other text content - syntax highlight the content
  - images - show the image
  - pdf - show the pdf in an iframe

# Top-level home page

- Serve at `/`
- This is a convenience for quickly accessing the functionality of the server
- Show a text box for starting a new chat (entry to `/chat`)
- Show the directory listing of the home directory (entry to `/files`)
- Users will likely ask to further customize this page

# Appendix: Suggested Chat Protocol

The guiding principle is that the server should be a **thin relay** between the client and pi's SDK. The server doesn't transform, render, or interpret pi's output - it passes SDK events to the client with minimal wrapping, and the client handles all presentation. This keeps the server stable as pi evolves and lets us iterate on the UI without restarting the server.

## REST Endpoints

- **`POST /api/chat/new`** - Creates a new session. Returns `{ path }` where path is the session's URL path (e.g., `--root--/2026-...-abc123`). The client navigates to `/chat/{path}` to open it.
  - Hint: An empty session has no on-disk file until something is appended; write the header yourself or append a no-op entry if you need an addressable empty session
- **`GET /api/chat/sessions`** - Returns the session list for the sidebar. Each entry includes the path, timestamp, and first message or session name.
- **`GET /api/chat/models`** - Returns available models and thinking levels.
  - Hint: Available models, providers, and credentials may come from extensions, not just auth.json. Before serving `/api/chat/models`, create a throwaway session and call `bindExtensions({})` so providers like the exe.dev gateway register themselves into the shared `ModelRegistry`.

## WebSocket

Each WebSocket connection targets a specific conversation: `/api/chat/ws/{sessionPath}`. The server maps this to a JSONL file and either creates a new `AgentSession` or attaches to an existing one (per the session multiplexing rules).

### Server → Client

- **`history`** - Sent once on connect. Contains all session entries from `sessionManager.getEntries()` and the current leaf ID from `sessionManager.getLeafId()`. This is the raw session tree - every entry has `id` and `parentId`, so the client can reconstruct the active path (walk from leaf to root), detect fork points (multiple entries sharing a `parentId`), and render branch navigation. No filtering or annotation by the server.
- **`event`** - A forwarded `AgentSessionEvent` from `session.subscribe()`, wrapped minimally as `{ type: "event", event: <AgentSessionEvent> }`. The server forwards **all** event types rather than filtering. New event types added to the SDK automatically flow through. The client ignores any it doesn't recognize.
- **`leaf`** - Sent after a branch operation. Contains just the new leaf ID. The client already has the full tree and re-renders the active path from the new leaf.
- **`entries`** - New session entries appended to the session. Sent whenever entries are added - user messages, model changes, completed agent turns, etc. Contains the entries with their tree metadata (`id`, `parentId`, etc.) so all connected clients can maintain the full session tree.

### Client → Server

Client messages are user actions, not SDK calls. Every message is a JSON object with a `type` field.

- **`prompt`** - Send a message. Includes text and optionally images. The server calls `session.prompt()` or `session.followUp()` depending on whether the agent is currently streaming.
- **`abort`** - Stop the current response. The server calls `session.abort()`.
- **`set_model`** - Change model. Include provider and model ID.
- **`set_thinking_level`** - Change thinking level.
- **`branch`** - Switch to a different point in the session tree. Includes the target entry ID. The server calls `sessionManager.branch()`, then sends a `leaf` message with the updated leaf.

### Design Notes

- **Don't invent a rendering protocol.** Send the raw SDK types and let the client render them.
- **Session state changes flow through `entries`.** When the client sends `set_model` or `set_thinking_level`, the server calls the SDK method, and the client learns it worked by receiving the corresponding entry. Don't add separate ack messages.
- **Events vs entries.** `event` messages drive real-time display (streaming text, tool activity). `entries` messages update the client's tree (completed messages with IDs). The client uses events during streaming for live feedback, then integrates entries into its tree when they arrive.
- **Branching sends a `leaf` message.** After a `branch`, the server sends `{ type: "leaf", leafId }`. The client already has the full tree and just re-renders the active path from the new leaf.
- **Editing a message is branching + prompting.** The client sends `branch` to the parent of the message being edited, then sends `prompt` with the new text. No special "edit" message type needed.

# Appendix: Example HTML and CSS for a chat UI that works right on iOS

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1, maximum-scale=1, interactive-widget=resizes-content"
    />
    <title>Chat 2</title>
    <style>
      *,
      *::before,
      *::after {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
      }

      html,
      body {
        height: 100%;
        overflow: hidden;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
          sans-serif;
        background: #1a1a2e;
        color: #e0e0e0;
      }

      #app {
        display: flex;
        flex-direction: column;
        height: 100dvh;
      }

      #messages {
        flex: 1 1 0;
        min-height: 0;
        overflow-y: auto;
        -webkit-overflow-scrolling: touch;
        padding: 16px;
        display: flex;
        flex-direction: column;
        gap: 12px;
      }

      .message {
        border-radius: 8px;
        padding: 12px 16px;
        max-width: 100%;
        overflow-wrap: break-word;
      }

      .msg-user {
        background: #2a2a4e;
        border-left: 3px solid #64b5f6;
      }

      .msg-assistant {
        background: #1e2e1e;
        border-left: 3px solid #81c784;
      }

      .msg-role {
        font-weight: 600;
        font-size: 0.8em;
        margin-bottom: 6px;
        color: #aaa;
      }

      .msg-content {
        white-space: pre-wrap;
        word-break: break-word;
        line-height: 1.5;
      }

      #input-bar {
        flex: 0 0 auto;
        display: flex;
        gap: 8px;
        padding: 12px 16px;
        padding-bottom: max(12px, env(safe-area-inset-bottom));
        background: #16162a;
        border-top: 1px solid #333;
      }

      #input-bar textarea {
        flex: 1;
        min-height: 44px;
        max-height: 200px;
        padding: 10px 14px;
        border: 1px solid #444;
        border-radius: 8px;
        font-size: 16px;
        line-height: 1.4;
        resize: none;
        outline: none;
        overflow-y: auto;
        font-family: inherit;
        background: #222244;
        color: #e0e0e0;
      }

      #input-bar textarea:focus {
        border-color: #64b5f6;
      }

      #input-bar textarea::placeholder {
        color: #666;
      }

      #input-bar button {
        width: 44px;
        height: 44px;
        background: #2a2a4e;
        border: 1px solid #444;
        color: #a0c4ff;
        border-radius: 50%;
        cursor: pointer;
        font-size: 1.2em;
        flex-shrink: 0;
        font-family: inherit;
        display: flex;
        align-items: center;
        justify-content: center;
        align-self: flex-end;
      }

      #input-bar button:hover {
        background: #333366;
      }
    </style>
  </head>
  <body>
    <div id="app">
      <div id="messages">
        <div class="message msg-user">
          <div class="msg-role">👤 You</div>
          <div class="msg-content">Here's a message</div>
        </div>
        <div class="message msg-assistant">
          <div class="msg-role">🤖 Assistant</div>
          <div class="msg-content">Here's a response</div>
        </div>
      </div>
      <div id="input-bar">
        <button>⋯</button>
        <textarea
          id="input"
          rows="1"
          placeholder="Send a message..."
        ></textarea>
        <button>↑</button>
      </div>
    </div>

    <script>
      const messages = document.getElementById("messages");
      messages.scrollTop = messages.scrollHeight;

      // Auto-grow textarea
      const input = document.getElementById("input");
      function autoGrow() {
        input.style.height = "auto";
        input.style.height = Math.min(input.scrollHeight, 200) + "px";
      }
      input.addEventListener("input", autoGrow);

      // Keep scrolled to bottom on iOS keyboard / URL bar changes
      if (window.visualViewport) {
        window.visualViewport.addEventListener("resize", () => {
          requestAnimationFrame(
            () => (messages.scrollTop = messages.scrollHeight)
          );
        });
      }
    </script>
  </body>
</html>
```
