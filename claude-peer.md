https://g.co/gemini/share/b75cd1c277c8

If you're running 3–4 sessions on versions 0.20.70–0.28, and you want them to talk without the modern "Channel" flag, you are right to worry about a "shared JSON" becoming a mess. If everyone writes to and reads from one file, they’ll just keep reading their own messages or getting confused by "cross-talk."
The solution is to use Session-Specific Mailboxes (unique IDs) and a Dispatcher. Since you don't have built-in channels, you have to build a "Poor Man's Channel" using Named Pipes (FIFOs).
The "Session ID" Pipe Strategy
Instead of one big JSON file, you create a dedicated "inbox" pipe for each session.
1. Assign each session an ID
When you start your sessions, give them a simple name (e.g., frontend, backend, db).
2. Create the Pipes
In your terminal, before starting Claude, create a folder for your "mailboxes":
mkdir -p /tmp/claude_pipes
mkfifo /tmp/claude_pipes/frontend /tmp/claude_pipes/backend /tmp/claude_pipes/db

3. Start the Sessions with "Listeners"
This is the trick to making it work like a real channel. You start Claude so it is always listening to its specific pipe:
 * Terminal 1 (Backend): tail -f /tmp/claude_pipes/backend | claude
 * Terminal 2 (Frontend): tail -f /tmp/claude_pipes/frontend | claude
4. How they "Talk" (The Tool)
You create a simple MCP tool (or even just a shell alias) called send_peer.
 * When Backend wants to talk to Frontend, it just runs:
   echo "Hey Frontend, I updated the API schema. Check it out." > /tmp/claude_pipes/frontend
 * What happens: Because the Frontend session is running with tail -f, that text is "typed" into its terminal instantly. It responds immediately because it thinks you just typed it.
Why this is better than a Shared JSON:
| Problem | Shared JSON Method | Named Pipe Method |
|---|---|---|
| Routing | Everyone sees everything (Chaos). | Only the target session sees the message. |
| Speed | Requires "Polling" (waiting to check). | Instant. The message is pushed to stdin. |
| Conflicts | File locking issues with 4 sessions. | No locks; it's a stream. |
| History | Hard to tell who said what. | Appears as a normal User message in the chat. |
Summary for your 4-Session Setup:
 * Don't use the modern channel flag (since it's not supported).
 * Use tail -f [pipe_name] | claude for every session you launch.
 * Give each session a unique pipe in /tmp/.
 * Tell Claude in its instructions: "You have access to a folder of pipes. To talk to the 'db' session, write your message to /tmp/claude_pipes/db."
This effectively creates a physical pipe between your sessions that works exactly like the modern "Channel" concept, but uses standard Linux/Mac terminal features that your version of Claude Code understands perfectly.