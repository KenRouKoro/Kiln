<script lang="ts">
  import { onMount } from "svelte"
  import { _ } from "svelte-i18n"

  export let tag: string | null = null
  export let on_select: (tag: string) => void = () => {}
  export let on_escape: () => void = () => {}
  export let focus_on_mount: boolean = false
  let error: string | null = null
  let id = crypto.randomUUID()

  function handle_keyup(event: KeyboardEvent) {
    if (event.key === "Enter") {
      if (tag === null || tag.length === 0) {
        error = $_("tags.cannot_be_empty")
      } else if (tag.includes(" ")) {
        error = $_("tags.no_spaces")
      } else {
        on_select(tag)
        error = null
      }
    } else if (event.key === "Escape") {
      on_escape()
    }
  }

  onMount(() => {
    if (focus_on_mount) {
      document.getElementById(id)?.focus()
    }
  })
</script>

<div class="w-full">
  <input
    {id}
    list="tag_options"
    type="text"
    class="w-full input input-bordered py-2 {error ? 'input-error' : ''}"
    placeholder={$_("tags.add_tag")}
    bind:value={tag}
    on:keyup={handle_keyup}
  />
  <datalist id="tag_options">
    <option value="needs_rating"></option>
    <option value="golden"></option>
    <option value="eval_set"></option>
    <option value="fine_tune_data"></option>
  </datalist>
  {#if error}
    <div class="text-error text-sm mt-1">{error}</div>
  {/if}
</div>
