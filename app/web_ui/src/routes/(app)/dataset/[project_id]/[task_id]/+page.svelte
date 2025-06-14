<script lang="ts">
  import AppPage from "../../../app_page.svelte"
  import EmptyInto from "./empty_into.svelte"
  import { client } from "$lib/api_client"
  import type { RunSummary } from "$lib/types"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount } from "svelte"
  import { model_info, load_model_info, model_name } from "$lib/stores"
  import { goto } from "$app/navigation"
  import { page } from "$app/stores"
  import { formatDate } from "$lib/utils/formatters"
  import { replaceState } from "$app/navigation"
  import TagDropdown from "../../../run/tag_dropdown.svelte"
  import Dialog from "$lib/ui/dialog.svelte"
  import { _ } from "svelte-i18n"

  let runs: RunSummary[] | null = null
  let filtered_runs: RunSummary[] | null = null
  let error: KilnError | null = null
  let loading = true
  let sortColumn = ($page.url.searchParams.get("sort") || "created_at") as
    | keyof RunSummary
    | "rating"
    | "inputPreview"
    | "source"
    | "outputPreview"
    | "model"
    | "repairState"
    | "created_at"
  let sortDirection = ($page.url.searchParams.get("order") || "desc") as
    | "asc"
    | "desc"
  let filter_tags = ($page.url.searchParams.getAll("tags") || []) as string[]
  let page_number: number = parseInt(
    $page.url.searchParams.get("page") || "1",
    10,
  )
  const page_size = 1000
  $: {
    // Update based on live URL
    const url = new URL(window.location.href)
    sortColumn = (url.searchParams.get("sort") ||
      "created_at") as typeof sortColumn
    sortDirection = (url.searchParams.get("order") ||
      "desc") as typeof sortDirection
    filter_tags = url.searchParams.getAll("tags") as string[]
    page_number = parseInt(url.searchParams.get("page") || "1", 10)
    sortRuns()
  }

  $: project_id = $page.params.project_id
  $: task_id = $page.params.task_id

  const columns = [
    { key: "rating", label: $_("dataset.rating") },
    { key: "repairState", label: $_("dataset.repair_state") },
    { key: "source", label: $_("dataset.source") },
    { key: "model", label: $_("dataset.model") },
    { key: "created_at", label: $_("dataset.created_at") },
    { key: "inputPreview", label: $_("dataset.input_preview") },
    { key: "outputPreview", label: $_("dataset.output_preview") },
  ]

  onMount(async () => {
    get_runs()
  })

  async function get_runs() {
    try {
      load_model_info()
      loading = true
      if (!project_id || !task_id) {
        throw new Error("Project or task ID not set.")
      }
      const { data: runs_response, error: get_error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/runs_summaries",
        {
          params: {
            path: {
              project_id,
              task_id,
            },
          },
        },
      )
      if (get_error) {
        throw get_error
      }
      runs = runs_response
      sortRuns()
    } catch (e) {
      if (e instanceof Error && e.message.includes("Load failed")) {
        error = new KilnError($_("dataset.error_loading"), null)
      } else {
        error = createKilnError(e)
      }
    } finally {
      loading = false
    }
  }

  function sortFunction(a: RunSummary, b: RunSummary) {
    let aValue: string | number | Date | null | undefined
    let bValue: string | number | Date | null | undefined

    switch (sortColumn) {
      case "id":
        aValue = a.id
        bValue = b.id
        break
      case "created_at":
        aValue = a.created_at
        bValue = b.created_at
        break
      case "rating":
        aValue = a.rating?.value ?? -1
        bValue = b.rating?.value ?? -1
        break
      case "source":
        aValue = a.input_source ?? ""
        bValue = b.input_source ?? ""
        break
      case "inputPreview":
        aValue = (a.input_preview ?? "").toLowerCase()
        bValue = (b.input_preview ?? "").toLowerCase()
        break
      case "outputPreview":
        aValue = (a.output_preview ?? "").toLowerCase()
        bValue = (b.output_preview ?? "").toLowerCase()
        break
      case "repairState":
        aValue = a.repair_state
        bValue = b.repair_state
        break
      case "model":
        aValue = model_name(a.model_name || undefined, $model_info)
        bValue = model_name(b.model_name || undefined, $model_info)
        break
      default:
        return 0
    }

    if (!aValue) return sortDirection === "asc" ? 1 : -1
    if (!bValue) return sortDirection === "asc" ? -1 : 1

    if (aValue < bValue) return sortDirection === "asc" ? -1 : 1
    if (aValue > bValue) return sortDirection === "asc" ? 1 : -1
    return 0
  }

  function handleSort(columnString: string) {
    const new_column = columnString as typeof sortColumn
    let new_direction = "desc"
    if (sortColumn === new_column) {
      new_direction = sortDirection === "asc" ? "desc" : "asc"
    } else {
      new_direction = "desc"
    }
    updateURL({
      sort: new_column,
      order: new_direction,
    })
  }

  function sortRuns() {
    if (!runs) return
    runs = runs ? [...runs].sort(sortFunction) : null
    filtered_runs = runs
      ? [...runs].filter((run) =>
          filter_tags.every((tag) => run.tags?.includes(tag)),
        )
      : null

    // Clear the last selected id, as it's moved
    last_selected_id = null
  }

  let filter_tags_dialog: Dialog | null = null

  function remove_filter_tag(tag: string) {
    const newTags = filter_tags.filter((t) => t !== tag)
    updateURL({
      tags: newTags,
      page: 1,
    })
  }

  function add_filter_tag(tag: string) {
    const newTags = [...new Set([...filter_tags, tag])]
    // Selections confusing as filters change
    select_mode = false
    selected_runs = new Set()
    updateURL({
      tags: newTags,
      page: 1,
    })
  }

  $: available_filter_tags = get_available_filter_tags(
    filtered_runs,
    filter_tags,
  )
  function get_available_filter_tags(
    filtered_runs: RunSummary[] | null,
    filter_tags: string[],
  ): Record<string, number> {
    if (!filtered_runs) return {}

    const remaining_tags: Record<string, number> = {}
    filtered_runs.forEach((run) => {
      run.tags?.forEach((tag) => {
        if (filter_tags.includes(tag)) return
        if (typeof tag === "string") {
          remaining_tags[tag] = (remaining_tags[tag] || 0) + 1
        }
      })
    })
    return remaining_tags
  }

  function updateURL(params: Record<string, string | string[] | number>) {
    // update the URL so you can share links
    const url = new URL(window.location.href)

    // we're using multiple tags, so we need to delete the existing tags
    if (params.tags) {
      url.searchParams.delete("tags")
    }

    // Add new params to the URL (keep current params)
    Object.entries(params).forEach(([key, value]) => {
      if (Array.isArray(value)) {
        value.forEach((v) => url.searchParams.append(key, v))
      } else {
        url.searchParams.set(key, value.toString())
      }
    })

    // Update state manually
    if (params.sort) {
      sortColumn = params.sort as typeof sortColumn
    }
    if (params.order) {
      sortDirection = params.order as typeof sortDirection
    }
    if (params.tags) {
      filter_tags = params.tags as string[]
    }
    if (params.page) {
      page_number = params.page as number
    }

    // Use replaceState to avoid adding new entries to history
    replaceState(url, {})

    sortRuns()
  }

  function open_dataset_run(run_id: string | null) {
    if (!run_id) return
    const url = `/dataset/${project_id}/${task_id}/${run_id}/run`
    const list = filtered_runs?.map((run) => run.id)
    goto(url, { state: { list_page: list } })
  }

  let select_mode: boolean = false
  let selected_runs: Set<string> = new Set()
  let select_summary: "all" | "none" | "some" = "none"
  $: {
    if (selected_runs.size >= (filtered_runs?.length || 0)) {
      select_summary = "all"
    } else if (selected_runs.size > 0) {
      select_summary = "some"
    } else {
      select_summary = "none"
    }
  }

  function toggle_selection(run_id: string): boolean {
    const was_selected = selected_runs.has(run_id)
    if (was_selected) {
      selected_runs.delete(run_id)
    } else {
      selected_runs.add(run_id)
    }
    // Reactivity trigger
    selected_runs = selected_runs

    return !was_selected
  }

  let last_selected_id: string | null = null
  function row_clicked(run_id: string | null, event: MouseEvent) {
    if (!run_id) {
      last_selected_id = null
      return
    }
    if (select_mode) {
      const selected = toggle_selection(run_id)

      // Potentially select a range of runs if SHIFT-click
      if (selected) {
        select_range(run_id, event)
      }
      last_selected_id = selected ? run_id : null
    } else {
      open_dataset_run(run_id)
    }
  }

  // Select a range of runs if SHIFT-click
  function select_range(run_id: string, event: MouseEvent) {
    if (!last_selected_id) return
    // return unless shift key is down
    if (!event.shiftKey) return

    // select all runs between last_selected_id and run_id
    const last_selected_index = runs?.findIndex(
      (run) => run.id === last_selected_id,
    )
    const run_index = runs?.findIndex((run) => run.id === run_id)
    if (
      last_selected_index === -1 ||
      run_index === -1 ||
      last_selected_index === undefined ||
      run_index === undefined
    )
      return
    const start_index = Math.min(last_selected_index, run_index)
    const end_index = Math.max(last_selected_index, run_index)
    for (let i = start_index; i <= end_index; i++) {
      const id = runs?.[i]?.id
      if (id) {
        selected_runs.add(id)
      }
    }
    // Reactivity trigger
    selected_runs = selected_runs
  }

  function select_all_clicked(event: Event) {
    // Prevent default checkbox, we're using reactivity
    event.preventDefault()

    // Clear the last selected id, it no longer makes sense
    last_selected_id = null

    if (select_summary === "all" || select_summary === "some") {
      selected_runs.clear()
    } else {
      filtered_runs?.forEach((run) => {
        if (run.id) {
          selected_runs.add(run.id)
        }
      })
    }
    selected_runs = selected_runs
  }

  let delete_dialog: Dialog | null = null

  function show_delete_modal() {
    delete_dialog?.show()
  }

  async function delete_runs(): Promise<boolean> {
    try {
      const { error } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/runs/delete",
        {
          params: {
            path: { project_id, task_id },
          },
          body: Array.from(selected_runs),
        },
      )
      if (error) {
        throw error
      }

      // Close modal on success
      return true
    } finally {
      // Reload UI, even on failure, as partial delete is possible
      selected_runs = new Set()
      select_mode = false
      await get_runs()
    }
  }

  let add_tags: Set<string> = new Set()
  let remove_tags: Set<string> = new Set()
  let show_add_tag_dropdown = false
  let current_tag: string = ""

  let add_tags_dialog: Dialog | null = null

  function show_add_tags_modal() {
    // Show the dropdown
    show_add_tag_dropdown = true

    add_tags_dialog?.show()
  }

  async function add_selected_tags(): Promise<boolean> {
    // Special case for this UI - consider the partly filled tag in the input
    // as a tag to add
    if (current_tag.length > 0) {
      add_tags.add(current_tag)
      current_tag = ""
    }
    // Don't accidentially remove tags
    remove_tags = new Set()
    return await edit_tags()
  }

  let removeable_tags: Record<string, number> = {}
  function update_removeable_tags() {
    let selected_run_contents: RunSummary[] = []
    for (const run of filtered_runs || []) {
      if (run.id && selected_runs.has(run.id)) {
        selected_run_contents.push(run)
      }
    }
    removeable_tags = get_available_filter_tags(
      selected_run_contents,
      Array.from(remove_tags),
    )
  }

  let remove_tags_dialog: Dialog | null = null

  function show_remove_tags_modal() {
    // clear prior lists
    remove_tags = new Set()

    update_removeable_tags()

    remove_tags_dialog?.show()
  }

  async function remove_selected_tags(): Promise<boolean> {
    // Don't accidentially add tags
    add_tags = new Set()
    return await edit_tags()
  }

  async function edit_tags(): Promise<boolean> {
    try {
      const { error } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/runs/edit_tags",
        {
          params: { path: { project_id, task_id } },
          body: {
            run_ids: Array.from(selected_runs),
            add_tags: Array.from(add_tags),
            remove_tags: Array.from(remove_tags),
          },
        },
      )
      if (error) {
        throw error
      }

      // Hide the dropdown (safari bug shows it when hidden)
      show_add_tag_dropdown = false

      // Close modal on success
      return true
    } finally {
      // Reload UI, even on failure, as partial delete is possible
      selected_runs = new Set()
      select_mode = false
      await get_runs()
    }
  }
</script>

<AppPage
  title={$_("dataset.title")}
  sub_subtitle={$_("dataset.read_the_docs")}
  sub_subtitle_link="https://docs.getkiln.ai/docs/organizing-datasets"
  no_y_padding
  action_buttons={[
    {
      label: $_("dataset.add_data"),
      href: `/dataset/${project_id}/${task_id}/add_data`,
    },
  ]}
>
  {#if loading}
    <div class="w-full min-h-[50vh] flex justify-center items-center">
      <div class="loading loading-spinner loading-lg"></div>
    </div>
  {:else if runs && runs.length == 0}
    <div class="flex flex-col items-center justify-center min-h-[75vh]">
      <EmptyInto {project_id} {task_id} />
    </div>
  {:else if runs}
    <div class="mb-4">
      <div
        class="flex flex-row items-center justify-end py-2 gap-3 {select_mode
          ? 'sticky top-0 z-10 backdrop-blur'
          : ''}"
      >
        {#if select_mode}
          <div class="font-light text-sm">
            {selected_runs.size} selected
          </div>
          {#if selected_runs.size > 0}
            <div class="dropdown dropdown-end">
              <div tabindex="0" role="button" class="btn btn-mid !px-3">
                <img alt="tags" src="/images/tag.svg" class="w-5 h-5" />
              </div>
              <ul
                class="dropdown-content menu bg-base-100 rounded-box z-[1] w-52 p-2 shadow"
              >
                <li>
                  <button tabindex="0" on:click={() => show_add_tags_modal()}>
                    {$_("dataset.add_tags_to_runs")}
                  </button>
                </li>
                <li>
                  <button
                    tabindex="0"
                    on:click={() => show_remove_tags_modal()}
                  >
                    {$_("dataset.remove_tags_from_runs")}
                  </button>
                </li>
              </ul>
            </div>
            <button
              class="btn btn-mid !px-3"
              on:click={() => show_delete_modal()}
            >
              <img alt="delete" src="/images/delete.svg" class="w-5 h-5" />
            </button>
          {/if}
          <button class="btn btn-mid" on:click={() => (select_mode = false)}>
            {$_("common.cancel")}
            {$_("common.select")}
          </button>
        {:else}
          <button class="btn btn-mid" on:click={() => (select_mode = true)}>
            {$_("common.select")}
          </button>
          <button
            class="btn btn-mid !px-3"
            on:click={() => filter_tags_dialog?.show()}
          >
            <img alt="filter" src="/images/filter.svg" class="w-5 h-5" />
            {#if filter_tags.length > 0}
              <span class="badge badge-primary badge-sm"
                >{filter_tags.length}</span
              >
            {/if}
          </button>
        {/if}
      </div>
      <div class="overflow-x-auto rounded-lg border">
        <table class="table">
          <thead>
            <tr>
              {#if select_mode}
                <th>
                  {#key select_summary}
                    <input
                      type="checkbox"
                      class="checkbox checkbox-sm mt-1"
                      checked={select_summary === "all"}
                      indeterminate={select_summary === "some"}
                      on:change={(e) => select_all_clicked(e)}
                    />
                  {/key}
                </th>
              {/if}
              {#each columns as { key, label }}
                <th
                  on:click={() => handleSort(key)}
                  class="hover:bg-base-200 cursor-pointer"
                >
                  {label}
                  {sortColumn === key
                    ? sortDirection === "asc"
                      ? "▲"
                      : "▼"
                    : ""}
                </th>
              {/each}
            </tr>
          </thead>
          <tbody>
            {#each (filtered_runs || []).slice((page_number - 1) * page_size, page_number * page_size) as run}
              <tr
                class="{select_mode
                  ? ''
                  : 'hover'} cursor-pointer {select_mode &&
                run.id &&
                selected_runs.has(run.id)
                  ? 'bg-base-200'
                  : ''}"
                on:click={(event) => {
                  row_clicked(run.id, event)
                }}
              >
                {#if select_mode}
                  <td>
                    <input
                      type="checkbox"
                      class="checkbox checkbox-sm"
                      checked={(run.id && selected_runs.has(run.id)) || false}
                    />
                  </td>
                {/if}
                <td>
                  {run.rating && run.rating.value
                    ? run.rating.type === "five_star"
                      ? "★".repeat(run.rating.value)
                      : run.rating.value + " ({$_('rating.custom_score')})"
                    : $_("rating.unrated")}
                </td>
                <td>{run.repair_state}</td>
                <td>{run.input_source}</td>
                <td class="break-words max-w-36">
                  {model_name(run.model_name || undefined, $model_info)}
                </td>
                <td>{formatDate(run.created_at)}</td>
                <td class="break-words max-w-48">
                  {run.input_preview || $_("dataset.no_input")}
                </td>
                <td class="break-words max-w-48">
                  {run.output_preview || $_("dataset.no_output")}
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>

    {#if page_number > 1 || (filtered_runs && filtered_runs.length > page_size)}
      <div class="flex flex-row justify-center mt-10">
        <div class="join">
          {#each Array.from({ length: Math.ceil(runs.length / page_size) }, (_, i) => i + 1) as page}
            <button
              class="join-item btn {page_number == page ? 'btn-active' : ''}"
              on:click={() => updateURL({ page: page })}
            >
              {page}
            </button>
          {/each}
        </div>
      </div>
    {/if}
  {:else if error}
    <div
      class="w-full min-h-[50vh] flex flex-col justify-center items-center gap-2"
    >
      <div class="font-medium">{$_("dataset.error_loading")}</div>
      <div class="text-error text-sm">
        {error.getMessage() || $_("errors.unknown_error")}
      </div>
    </div>
  {/if}
</AppPage>

<Dialog
  bind:this={filter_tags_dialog}
  title={$_("dataset.filter_by_tags")}
  action_buttons={[{ label: $_("common.close"), isCancel: true }]}
>
  {#if filter_tags.length > 0}
    <div class="text-sm mb-2 font-medium">{$_("dataset.current_filters")}</div>
  {/if}
  <div class="flex flex-row gap-2 flex-wrap">
    {#each filter_tags as tag}
      <div class="badge bg-gray-200 text-gray-500 py-3 px-3 max-w-full">
        <span class="truncate">{tag}</span>
        <button
          class="pl-3 font-medium shrink-0"
          on:click={() => remove_filter_tag(tag)}>✕</button
        >
      </div>
    {/each}
  </div>

  <div class="text-sm mt-4 mb-2 font-medium">{$_("dataset.add_filter")}</div>
  {#if Object.keys(available_filter_tags).length == 0}
    <p class="text-sm text-gray-500">
      {$_("dataset.no_more_filters")}
    </p>
  {/if}
  <div class="flex flex-row gap-2 flex-wrap">
    {#each Object.entries(available_filter_tags).sort((a, b) => b[1] - a[1]) as [tag, count]}
      <button
        class="badge bg-gray-200 text-gray-500 py-3 px-3 max-w-full"
        on:click={() => add_filter_tag(tag)}>{tag} ({count})</button
      >
    {/each}
  </div>
</Dialog>

<Dialog
  bind:this={delete_dialog}
  title={selected_runs.size > 1
    ? $_("dataset.delete_runs")
    : $_("dataset.delete_run")}
  action_buttons={[
    { label: $_("common.cancel"), isCancel: true },
    {
      label: $_("common.delete"),
      asyncAction: () => delete_runs(),
      isError: true,
    },
  ]}
>
  <div class="text-sm font-light text-gray-500">
    {$_("dataset.cannot_undo")}
  </div>
</Dialog>

<Dialog
  bind:this={add_tags_dialog}
  title={selected_runs.size > 1
    ? $_("dataset.add_tags_to_runs")
    : $_("dataset.add_tags_to_run")}
  action_buttons={[
    { label: $_("common.cancel"), isCancel: true },
    {
      label: $_("common.add") + " " + $_("common.tags"),
      asyncAction: add_selected_tags,
      disabled: add_tags.size == 0 && !current_tag,
      isPrimary: true,
    },
  ]}
>
  <div>
    <div class="text-sm font-light text-gray-500 mb-2">
      {$_("dataset.tags_organize")}
    </div>
    <div class="flex flex-row flex-wrap gap-2 mt-2">
      {#each Array.from(add_tags).sort() as tag}
        <div class="badge bg-gray-200 text-gray-500 py-3 px-3 max-w-full">
          <span class="truncate">{tag}</span>
          <button
            class="pl-3 font-medium shrink-0"
            on:click={() => {
              add_tags.delete(tag)
              add_tags = add_tags
            }}>✕</button
          >
        </div>
      {/each}
      <button
        class="badge bg-gray-200 text-gray-500 p-3 font-medium {show_add_tag_dropdown
          ? 'hidden'
          : ''}"
        on:click={() => (show_add_tag_dropdown = true)}>+</button
      >
    </div>
    {#if show_add_tag_dropdown}
      <div
        class="mt-3 flex flex-row gap-2 items-center {show_add_tag_dropdown
          ? ''
          : 'hidden'}"
      >
        <TagDropdown
          bind:tag={current_tag}
          on_select={(tag) => {
            add_tags.add(tag)
            add_tags = add_tags
            show_add_tag_dropdown = false
            current_tag = ""
          }}
          on_escape={() => (show_add_tag_dropdown = false)}
          focus_on_mount={true}
        />
        <div class="flex-none">
          <button
            class="btn btn-sm btn-circle text-xl font-medium"
            on:click={() => (show_add_tag_dropdown = false)}>✕</button
          >
        </div>
      </div>
    {/if}
  </div>
</Dialog>

<Dialog
  bind:this={remove_tags_dialog}
  title={selected_runs.size > 1
    ? $_("dataset.remove_tags_from_runs")
    : $_("dataset.remove_tags_from_run")}
  action_buttons={[
    { label: $_("common.cancel"), isCancel: true },
    {
      label: $_("dataset.remove_tags_from_runs"),
      asyncAction: () => remove_selected_tags(),
      disabled: remove_tags.size == 0,
      isError: true,
    },
  ]}
>
  <div>
    <div class="text-sm font-light text-gray-500 mt-6">
      {$_("dataset.selected_tags_remove")}
    </div>
    {#if remove_tags.size == 0}
      <div class="text-xs font-medium">{$_("dataset.no_tags_selected")}</div>
    {:else}
      <div class="flex flex-row flex-wrap gap-2 mt-2">
        {#each Array.from(remove_tags).sort() as tag}
          <div class="badge bg-gray-200 text-gray-500 py-3 px-3 max-w-full">
            <span class="truncate">{tag}</span>
            <button
              class="pl-3 font-medium shrink-0"
              on:click={() => {
                remove_tags.delete(tag)
                remove_tags = remove_tags
                update_removeable_tags()
              }}>✕</button
            >
          </div>
        {/each}
      </div>
    {/if}
    <div class="text-sm font-light text-gray-500 mt-6">
      {$_("dataset.available_tags")}
    </div>
    {#if Object.keys(removeable_tags).length == 0 && remove_tags.size == 0}
      <div class="text-xs font-medium">{$_("dataset.no_tags_on_runs")}</div>
    {:else if Object.keys(removeable_tags).length == 0}
      <div class="text-xs font-medium">
        {$_("dataset.all_tags_selected")}
      </div>
    {:else}
      <div class="flex flex-row flex-wrap gap-2 mt-2">
        {#each Object.entries(removeable_tags).sort((a, b) => b[1] - a[1]) as [tag, count]}
          {#if !remove_tags.has(tag)}
            <div class="badge bg-gray-200 text-gray-500 py-3 px-3 max-w-full">
              <button
                class="truncate"
                on:click={() => {
                  remove_tags.add(tag)
                  remove_tags = remove_tags
                  update_removeable_tags()
                }}
              >
                {tag} ({count})
              </button>
            </div>
          {/if}
        {/each}
      </div>
    {/if}
  </div>
</Dialog>
