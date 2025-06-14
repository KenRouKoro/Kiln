<script lang="ts">
  import { _ } from "svelte-i18n"

  export let warning_message: string | undefined | null = undefined
  export let warning_color:
    | "error"
    | "warning"
    | "success"
    | "primary"
    | "gray"
    | undefined = undefined
  export let warning_icon: "exclaim" | "info" | "check" = "exclaim"
  export let large_icon: boolean = false
  export let tight: boolean = false
  export let trusted: boolean = false

  // Default to error if no color is provided
  $: color = warning_color || "error"

  function html_warning_message() {
    let message = warning_message
    if (!message) {
      return ""
    }
    message = message.replace(
      /https?:\/\/\S+/g,
      '<a href="$&" class="link underline" target="_blank">$&</a>',
    )
    const paragraphs = message.split("\n")
    return paragraphs
      .map((paragraph: string) => {
        return `<p>${paragraph}</p>`
      })
      .join("")
  }

  function get_color() {
    switch (color) {
      case "error":
        return "text-error"
      case "warning":
        return "text-warning"
      case "success":
        return "text-success"
      case "primary":
        return "text-primary"
      case "gray":
        return "text-gray-500"
      default:
        return ""
    }
  }
</script>

{#if warning_message}
  <div class="text-sm text-gray-500 flex flex-row items-center mt-2">
    {#if warning_icon === "exclaim" || warning_icon === "info"}
      <svg
        class="{large_icon
          ? 'w-8 h-8'
          : 'w-5 h-5'} flex-none transition-[width,height,transform] duration-500 ease-in-out {get_color()} {warning_icon ===
        'info'
          ? 'rotate-180'
          : 'rotate-0'}"
        style="transform-origin: center; will-change: transform, width, height, rotate;"
        fill="currentColor"
        viewBox="0 0 256 256"
        id="Flat"
        xmlns="http://www.w3.org/2000/svg"
        aria-label={warning_icon === "info"
          ? $_("ui.warning.info_icon_alt")
          : $_("ui.warning.exclamation_icon_alt")}
        role="img"
      >
        <path
          d="M128,20.00012a108,108,0,1,0,108,108A108.12217,108.12217,0,0,0,128,20.00012Zm0,192a84,84,0,1,1,84-84A84.0953,84.0953,0,0,1,128,212.00012Zm-12-80v-52a12,12,0,1,1,24,0v52a12,12,0,1,1-24,0Zm28,40a16,16,0,1,1-16-16A16.018,16.018,0,0,1,144,172.00012Z"
        />
      </svg>
    {:else if warning_icon === "check"}
      <!-- Uploaded to: SVG Repo, www.svgrepo.com, Generator: SVG Repo Mixer Tools -->
      <svg
        class="{large_icon
          ? 'w-8 h-8'
          : 'w-5 h-5'} transition-[width,height,transform] duration-500 ease-in-out flex-none {get_color()}"
        viewBox="0 0 24 24"
        fill="none"
        xmlns="http://www.w3.org/2000/svg"
        aria-label={$_("ui.warning.check_icon_alt")}
        role="img"
      >
        <path
          d="M16 9L10 15.5L7.5 13M12 21C16.9706 21 21 16.9706 21 12C21 7.02944 16.9706 3 12 3C7.02944 3 3 7.02944 3 12C3 16.9706 7.02944 21 12 21Z"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
      </svg>
    {/if}

    <div class="{tight ? 'pl-1' : 'pl-4'} flex flex-col gap-2">
      {#if trusted}
        <!-- eslint-disable-next-line svelte/no-at-html-tags -->
        {@html html_warning_message()}
      {:else}
        {warning_message}
      {/if}
    </div>
  </div>
{/if}
