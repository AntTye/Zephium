<script lang="ts">
  import { onMount } from "svelte";
  import { faultView } from "../lib/faults";
  import { loadModelPicker } from "$features/work";
  import { commands } from "$shared/ipc/bindings";
  import * as m from "$shared/i18n/messages";
  import LazyView from "$shared/ui/LazyView";

  let {
    profile,
    text,
    selection,
    onclose,
  }: {
    profile: string;
    text: string;
    selection: boolean;
    onclose: () => void;
  } = $props();

  let field: HTMLTextAreaElement;
  let question = $state("");
  let answer = $state("");
  let error = $state("");
  let pending = $state(false);
  // The native command checks the same UTF-8 byte limit before model resolution.
  const tooLarge = $derived(new TextEncoder().encode(text).length > 64 * 1024);
  const ready = $derived(!!question.trim() && !tooLarge && !pending);

  onMount(() => field.focus());

  async function send() {
    if (!ready) return;
    pending = true;
    error = "";
    answer = "";
    try {
      const result = await commands.noteAsk(profile, question.trim(), text);
      if (result.status === "ok") answer = result.data;
      else if (result.error.reason === "model" && result.error.fault === "too_large")
        error = m.note_ask_too_large();
      else error = faultView(result.error).message;
    } catch {
      error = m.assist_unavailable();
    } finally {
      pending = false;
    }
  }
</script>

<form
  class="note-ask"
  aria-label={m.panel_ask_zephium()}
  onsubmit={(event) => {
    event.preventDefault();
    void send();
  }}
>
  <details>
    <summary>{selection ? m.assist_selection() : m.note_document()}</summary>
    <pre>{text}</pre>
  </details>
  <textarea
    bind:this={field}
    bind:value={question}
    rows="2"
    maxlength="2048"
    placeholder={m.assist_placeholder()}
    aria-label={m.assist_placeholder()}
    readonly={pending}
    onkeydown={(event) => {
      if (event.isComposing) return;
      if (event.key === "Escape") {
        event.preventDefault();
        onclose();
      } else if (event.key === "Enter" && !event.shiftKey) {
        event.preventDefault();
        void send();
      }
    }}></textarea>
  <div class="actions">
    <LazyView
      loader={loadModelPicker}
      loadingLabel=""
      failureLabel={m.surface_render_failed()}
      retryLabel={m.surface_retry()}
    >
      {#snippet children(Picker)}<Picker {profile} role="page" />{/snippet}
    </LazyView>
    <button type="button" onclick={onclose}>{m.assist_menu_close()}</button>
    <button type="submit" disabled={!ready}>{m.assist_send()}</button>
  </div>
  {#if tooLarge}<p role="alert">{m.note_ask_too_large()}</p>{/if}
  {#if error}<p role="alert">{error}</p>{/if}
  {#if pending}<p role="status">{m.assist_phase_thinking()}</p>{/if}
  {#if answer}<div class="answer" role="status">{answer}</div>{/if}
</form>

<style>
  .note-ask {
    display: grid;
    gap: 10px;
    margin-block-end: 16px;
    padding: 12px;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-control);
    background: var(--color-fill);
    color: var(--color-text);
    font-size: var(--text-body);
  }

  summary {
    color: var(--color-label-secondary);
    cursor: pointer;
  }

  pre,
  .answer {
    max-height: 30vh;
    overflow: auto;
    overflow-wrap: anywhere;
    user-select: text;
    white-space: pre-wrap;
  }

  .answer {
    padding-block-start: 10px;
    border-top: 1px solid var(--color-border);
    line-height: 1.5;
  }

  textarea {
    box-sizing: border-box;
    width: 100%;
    padding: 8px;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-inset);
    background: var(--color-page);
    color: inherit;
    font: inherit;
    resize: vertical;
  }

  .actions {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 6px;
  }

  button {
    padding: 5px 10px;
    border: 0;
    border-radius: var(--radius-inset);
    background: var(--color-control);
    color: inherit;
    font: inherit;
  }

  button[type="button"] {
    margin-inline-start: auto;
  }

  button[type="submit"]:enabled {
    background: var(--color-accent);
    color: var(--color-on-accent);
  }

  button:disabled {
    color: var(--color-faint);
  }

  :is(button, textarea, summary):focus-visible {
    outline: 2px solid var(--color-ring);
  }

  p {
    margin: 0;
  }

  p[role="alert"] {
    color: var(--color-warning);
  }
</style>
