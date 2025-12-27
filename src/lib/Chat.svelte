<script lang="ts">
  import { afterUpdate, onMount } from "svelte";

  type Message = {
    id?: string;
    sender: "user" | "ai" | "system";
    text: string;
    createdAt?: string;
  };

  const API_URL = import.meta.env.VITE_API_URL || "http://localhost:4000";
  const STORAGE_KEY = "spur-session-id";

  const quickPrompts = [
    "What is your shipping timeline to the USA?",
    "How do returns and refunds work?",
    "Can you help me change my order?",
    "What time can I reach human support?"
  ];

  let messages: Message[] = [];
  let input = "";
  let loading = false;
  let loadingHistory = false;
  let error = "";
  let sessionId =
    typeof localStorage !== "undefined" ? localStorage.getItem(STORAGE_KEY) || "" : "";
  let chatWindow: HTMLDivElement | null = null;

  onMount(() => {
    if (sessionId) {
      loadHistory(sessionId);
    }
  });

  afterUpdate(() => {
    if (!chatWindow) return;
    chatWindow.scrollTop = chatWindow.scrollHeight;
  });

  function shortSession(id: string) {
    return id ? `${id.slice(0, 6)}...` : "new";
  }

  function handleKeyDown(event: KeyboardEvent) {
    if (event.key === "Enter" && !event.shiftKey) {
      event.preventDefault();
      send();
    }
  }

  function setSession(id: string) {
    sessionId = id;
    if (typeof localStorage !== "undefined" && id) {
      localStorage.setItem(STORAGE_KEY, id);
    }
  }

  async function loadHistory(id: string) {
    loadingHistory = true;
    error = "";
    try {
      const res = await fetch(`${API_URL}/chat/${id}/history`);
      const data = await res.json();
      if (!res.ok) {
        throw new Error(data.error || "Could not load your chat history.");
      }

      setSession(data.sessionId || id);
      messages = data.messages.map((m: any) => ({
        id: m.id,
        sender: m.sender === "assistant" ? "ai" : m.sender,
        text: m.text,
        createdAt: m.createdAt
      }));
    } catch (err) {
      error = err instanceof Error ? err.message : "Unable to load history.";
      messages = [];
      if (typeof localStorage !== "undefined") {
        localStorage.removeItem(STORAGE_KEY);
      }
      sessionId = "";
    } finally {
      loadingHistory = false;
    }
  }

  function addSystemMessage(text: string) {
    messages = [...messages, { sender: "system", text }];
  }

  async function send() {
    const trimmed = input.trim();
    if (!trimmed || loading || loadingHistory) return;

    let outgoing = trimmed;
    if (trimmed.length > 1000) {
      outgoing = trimmed.slice(0, 1000);
      error = "We trimmed your message to 1000 characters.";
    } else {
      error = "";
    }

    messages = [
      ...messages,
      {
        sender: "user",
        text: outgoing,
        createdAt: new Date().toISOString()
      }
    ];

    input = "";
    loading = true;

    try {
      const res = await fetch(`${API_URL}/chat/message`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ message: outgoing, sessionId })
      });

      const data = await res.json();
      if (!res.ok) {
        throw new Error(data.error || "Unable to reach the agent right now.");
      }

      if (data.sessionId) {
        setSession(data.sessionId);
      }

      messages = [
        ...messages,
        {
          sender: "ai",
          text: data.reply,
          createdAt: new Date().toISOString()
        }
      ];
    } catch (err) {
      error = err instanceof Error ? err.message : "Unable to reach the agent right now.";
      addSystemMessage("We hit a snag talking to the agent. Please try again in a moment.");
    } finally {
      loading = false;
    }
  }

  function resetConversation() {
    messages = [];
    sessionId = "";
    input = "";
    error = "";
    if (typeof localStorage !== "undefined") {
      localStorage.removeItem(STORAGE_KEY);
    }
  }

  function usePrompt(prompt: string) {
    input = prompt;
  }
</script>

<div class="chat-shell">
  <div class="chat-header">
    <div>
      <p class="eyebrow">Live support</p>
      <h2>Spur AI Agent</h2>
      <p class="muted">
        Ask about shipping, returns, or payments. Conversations persist by session, even if
        you refresh.
      </p>
    </div>
    <div class="session-meta">
      <div class="pill">
        <span class="dot"></span>
        {sessionId ? `Session ${shortSession(sessionId)}` : "New session"}
      </div>
      <button
        type="button"
        class="ghost"
        on:click={resetConversation}
        disabled={loading || loadingHistory || messages.length === 0}
      >
        Reset
      </button>
    </div>
  </div>

  <div class="quick-prompts">
    {#each quickPrompts as prompt}
      <button type="button" class="chip" on:click={() => usePrompt(prompt)}>{prompt}</button>
    {/each}
  </div>

  <div class="chat-window" bind:this={chatWindow}>
    {#if loadingHistory}
      <div class="status">Loading your last conversation...</div>
    {:else if messages.length === 0}
      <div class="empty">
        <p>Try asking about shipping timelines, refunds, billing changes, or support hours.</p>
        <small>Hit Enter to send, Shift+Enter for a new line.</small>
      </div>
    {/if}

    {#each messages as msg, index (msg.id || `${msg.sender}-${index}-${msg.text.slice(0, 20)}`)}
      <div class={`message-row ${msg.sender}`}>
        <div class="avatar">
          {msg.sender === "ai" ? "AI" : msg.sender === "user" ? "You" : "!"}
        </div>
        <div class="bubble">
          <div class="bubble-header">
            <span>
              {msg.sender === "ai" ? "Spur AI" : msg.sender === "user" ? "You" : "System"}
            </span>
            {#if msg.createdAt}
              <span class="timestamp">
                {new Date(msg.createdAt).toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })}
              </span>
            {/if}
          </div>
          <p>{msg.text}</p>
        </div>
      </div>
    {/each}

    {#if loading}
      <div class="message-row ai typing">
        <div class="avatar">AI</div>
        <div class="bubble">
          <div class="bubble-header">
            <span>Spur AI</span>
            <span class="timestamp">typing</span>
          </div>
          <div class="dots">
            <span></span>
            <span></span>
            <span></span>
          </div>
        </div>
      </div>
    {/if}
  </div>

  {#if error}
    <div class="error-banner">{error}</div>
  {/if}

  <form class="composer" on:submit|preventDefault={send}>
    <textarea
      rows="3"
      placeholder="Ask a question... Shift+Enter for a new line."
      bind:value={input}
      on:keydown={handleKeyDown}
      maxlength="1200"
    ></textarea>
    <div class="composer-actions">
      <span class="hint">
        {sessionId
          ? `Session saved (${shortSession(sessionId)})`
          : "A session ID will be created with your first message."}
      </span>
      <div class="buttons">
        <button
          type="button"
          class="ghost"
          on:click={resetConversation}
          disabled={loading || loadingHistory || messages.length === 0}
        >
          Clear
        </button>
        <button
          type="submit"
          class="primary"
          disabled={!input.trim() || loading || loadingHistory}
        >
          {loading ? "Sending..." : "Send"}
        </button>
      </div>
    </div>
  </form>
</div>

<style>
  .chat-shell {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 18px;
    padding: 14px;
    box-shadow: 0 30px 70px rgba(15, 23, 42, 0.08);
  }

  .chat-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 12px;
  }

  h2 {
    margin: 6px 0;
    font-size: 1.4rem;
    letter-spacing: -0.01em;
  }

  .muted {
    color: var(--muted);
    margin: 0;
  }

  .eyebrow {
    text-transform: uppercase;
    letter-spacing: 0.12em;
    font-size: 0.72rem;
    color: var(--accent);
    margin: 0;
  }

  .session-meta {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .pill {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    border-radius: 999px;
    border: 1px solid var(--border);
    background: var(--accent-soft);
    color: #1e3a8a;
    font-weight: 600;
  }

  .dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--accent);
    box-shadow: 0 0 0 6px rgba(68, 210, 184, 0.16);
  }

  .quick-prompts {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin: 10px 0;
  }

  .chip {
    border: 1px solid var(--border);
    background: #fff;
    color: var(--text);
    padding: 8px 12px;
    border-radius: 12px;
    cursor: pointer;
    transition: border-color 0.2s, transform 0.2s;
  }

  .chip:hover {
    border-color: var(--accent);
    transform: translateY(-1px);
  }

  .chat-window {
    background: #fff;
    border: 1px solid var(--border);
    border-radius: 14px;
    padding: 14px;
    height: 280px;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .status,
  .empty {
    color: var(--muted);
    text-align: center;
    padding: 12px;
  }

  .empty small {
    display: block;
    margin-top: 4px;
    color: var(--muted);
  }

  .message-row {
    display: flex;
    align-items: flex-start;
    gap: 10px;
  }

  .message-row.user {
    flex-direction: row-reverse;
  }

  .message-row.system .bubble {
    border-color: rgba(234, 179, 8, 0.35);
    background: rgba(255, 237, 213, 0.6);
  }

  .avatar {
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: grid;
    place-items: center;
    font-weight: 700;
    color: #0f172a;
    background: linear-gradient(135deg, #eff4ff, #dbeafe);
    border: 1px solid var(--border);
  }

  .message-row.user .avatar {
    background: linear-gradient(135deg, #fff7ed, #ffe4d5);
  }

  .bubble {
    flex: 1;
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 10px 12px;
    background: #f8fafc;
    color: var(--text);
    box-shadow: 0 6px 18px rgba(15, 23, 42, 0.08);
  }

  .message-row.user .bubble {
    background: linear-gradient(135deg, #e8edfb, #f4f7ff);
    border-color: rgba(37, 99, 235, 0.35);
  }

  .bubble-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 6px;
    font-size: 0.86rem;
    color: var(--muted);
  }

  .timestamp {
    font-size: 0.76rem;
    color: var(--muted);
  }

  .bubble p {
    margin: 0;
    line-height: 1.5;
    white-space: pre-wrap;
  }

  .typing .bubble {
    border-style: dashed;
    border-color: rgba(37, 99, 235, 0.5);
  }

  .dots {
    display: flex;
    gap: 6px;
    align-items: center;
    height: 12px;
  }

  .dots span {
    width: 8px;
    height: 8px;
    background: var(--accent);
    border-radius: 50%;
    animation: pulse 1s infinite ease-in-out;
  }

  .dots span:nth-child(2) {
    animation-delay: 0.15s;
  }

  .dots span:nth-child(3) {
    animation-delay: 0.3s;
  }

  @keyframes pulse {
    0%,
    100% {
      transform: translateY(0);
      opacity: 0.6;
    }
    50% {
      transform: translateY(-3px);
      opacity: 1;
    }
  }

  .error-banner {
    margin-top: 10px;
    padding: 10px 12px;
    border-radius: 10px;
    border: 1px solid rgba(239, 68, 68, 0.35);
    background: rgba(254, 226, 226, 0.8);
    color: #991b1b;
  }

  .composer {
    margin-top: 12px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  textarea {
    width: 100%;
    border-radius: 12px;
    border: 1px solid var(--border);
    background: #fff;
    color: var(--text);
    padding: 12px;
    resize: vertical;
    min-height: 80px;
    font-size: 1rem;
    font-family: inherit;
    transition: border-color 0.2s, box-shadow 0.2s;
  }

  textarea:focus {
    outline: none;
    border-color: var(--accent);
    box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.16);
  }

  .composer-actions {
    display: flex;
    align-items: center;
    gap: 12px;
    justify-content: space-between;
    flex-wrap: wrap;
  }

  .hint {
    color: var(--muted);
    font-size: 0.9rem;
  }

  .buttons {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  button {
    font-family: inherit;
  }

  .ghost,
  .primary {
    border-radius: 12px;
    padding: 10px 14px;
    font-weight: 700;
    cursor: pointer;
    border: 1px solid var(--border);
    background: #fff;
    color: var(--text);
    transition: transform 0.15s ease, border-color 0.15s ease, background 0.15s ease;
  }

  .primary {
    background: linear-gradient(135deg, #2b6def, #1d4ed8);
    border-color: rgba(37, 99, 235, 0.8);
    color: #f8fafc;
  }

  .ghost:hover,
  .primary:hover {
    transform: translateY(-1px);
  }

  .ghost:disabled,
  .primary:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    transform: none;
  }

  @media (max-width: 720px) {
    .chat-shell {
      padding: 12px;
    }

    .chat-window {
      height: 240px;
    }

    .session-meta {
      flex-direction: column;
      align-items: flex-end;
    }
  }
</style>
