<script lang="ts">
  import { _ } from "svelte-i18n"

  export let content: unknown[] = []
  export let content_label: string = "Item"
  export let start_with_one: boolean = true
  export let empty_content: unknown = {}
  export let frozen: boolean = false

  // Unique ID for the list, for scrolling to top after removal
  let id = "form_list_" + Math.random().toString(36).substring(2, 15)

  function remove_item(index: number) {
    let item = content[index]
    let isItemUnedited = false

    if (item === null || item === undefined) {
      return
    } else if (item instanceof Object && empty_content instanceof Object) {
      // Compare item to empty_content to check if it's unedited
      isItemUnedited = Object.keys(item).every((key) => {
        return (
          item[key as keyof typeof item] ===
          empty_content[key as keyof typeof empty_content]
        )
      })
    }

    if (
      isItemUnedited ||
      confirm(
        $_("forms.form_list.remove_item_confirm", {
          values: { content_label, index: index + 1 },
        }),
      )
    ) {
      content.splice(index, 1)
      // trigger reactivity
      content = content
      // Move the page to the top anchor
      const list = document.getElementById(id)
      if (list) {
        // Scroll to the top of the list because removing sets focus to first form element for some reason
        setTimeout(() => {
          list.scrollIntoView()
        }, 1)
      }
    }
  }

  async function add_item(focus: boolean = true) {
    content.push(structuredClone(empty_content))
    // Trigger reactivity
    content = content
    if (focus) {
      // Scroll into view. Async to allow rendering first
      setTimeout(() => {
        const list = document.getElementById(
          "list_item_" + (content.length - 1) + "_" + id,
        )
        if (list) {
          list.scrollIntoView({ block: "center" })
        }
      }, 1)
    }
  }

  // Add one empty item to start so the list isn't empty
  if (start_with_one && content.length === 0) {
    add_item(false)
  }
</script>

{#if content.length > 0}
  <div class="flex flex-col gap-8" {id}>
    {#each content as item, item_index}
      <div id={"list_item_" + item_index + "_" + id}>
        <div class="flex flex-row gap-3 font-medium text-sm pb-2">
          <div class="grow">
            {content_label} #{item_index + 1}
          </div>
          <button
            class="link text-xs text-gray-500 {frozen ? 'hidden' : ''}"
            on:click={() => remove_item(item_index)}
          >
            {$_("forms.form_list.remove")}
          </button>
        </div>
        <slot {item} {item_index} />
      </div>
    {/each}
  </div>
{/if}

<div class="flex place-content-center">
  <button
    class="btn btn-sm {frozen ? 'hidden' : ''}"
    on:click={() => add_item(true)}
    id={id + "_add_button"}
  >
    {$_("forms.form_list.add", { values: { content_label } })}
  </button>
</div>
