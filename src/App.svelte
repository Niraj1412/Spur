<script>
  import Chat from "./lib/Chat.svelte";
</script>

<svelte:head>
  <title>Spur AI Support Chat</title>
</svelte:head>

<main class="stack">
  <header class="headline">
    <div>
      <p class="eyebrow">Spur support</p>
      <h1>Ask questions. Get clear answers.</h1>
      <p class="lede">
        A friendly virtual agent that remembers your session, shares store policies, and keeps the
        conversation flowing.
      </p>
    </div>
    <div class="pill-group">
      <span class="pill">Instant</span>
      <span class="pill">Saved</span>
      <span class="pill">Reliable</span>
    </div>
  </header>

  <section class="chat-panel">
    <Chat />
  </section>
</main>

<style>
  .stack {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .headline {
    display: flex;
    flex-wrap: wrap;
    align-items: flex-end;
    justify-content: space-between;
    gap: 10px;
    background: #fff;
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 10px 12px;
    box-shadow: 0 12px 32px rgba(15, 23, 42, 0.06);
  }

  h1 {
    margin: 2px 0 6px;
    letter-spacing: -0.02em;
    font-size: 1.4rem;
  }

  .eyebrow {
    text-transform: uppercase;
    letter-spacing: 0.12em;
    font-size: 0.78rem;
    color: var(--accent);
    margin: 0;
  }

  .lede {
    margin: 0;
    color: var(--muted);
    max-width: 520px;
    font-size: 0.95rem;
  }

  .pill-group {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
  }

  .pill {
    border-radius: 999px;
    padding: 6px 10px;
    border: 1px solid var(--border);
    background: #fff;
    color: var(--text);
    font-weight: 600;
    font-size: 0.9rem;
  }

  .chat-panel {
    min-width: 0;
  }

  @media (max-width: 720px) {
    .headline {
      align-items: flex-start;
    }
  }
</style>
