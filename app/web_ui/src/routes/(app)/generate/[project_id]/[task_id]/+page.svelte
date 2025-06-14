<script lang="ts">
  import AppPage from "../../../app_page.svelte"
  import { client } from "$lib/api_client"
  import { current_task } from "$lib/stores"
  import type { Task } from "$lib/types"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount } from "svelte"
  import { page } from "$app/stores"
  import type { SampleDataNode } from "./gen_model"
  import GeneratedDataNode from "./generated_data_node.svelte"
  import AvailableModelsDropdown from "../../../run/available_models_dropdown.svelte"
  import { beforeNavigate } from "$app/navigation"
  import { ui_state } from "$lib/stores"
  import PromptTypeSelector from "../../../run/prompt_type_selector.svelte"
  import FormContainer from "$lib/utils/form_container.svelte"
  import { type SampleData } from "./gen_model"
  import FormElement from "$lib/utils/form_element.svelte"
  import Warning from "$lib/ui/warning.svelte"
  import Dialog from "$lib/ui/dialog.svelte"
  import Splits from "$lib/ui/splits.svelte"
  import { _ } from "svelte-i18n"

  let session_id = Math.floor(Math.random() * 1000000000000).toString()

  let human_guidance = ""
  let splits: Record<string, number> = {}
  let splits_subtitle: string | undefined = undefined
  let split_object: Splits | null = null

  let task: Task | null = null
  let task_error: KilnError | null = null
  let task_loading = true

  $: project_id = $page.params.project_id
  $: task_id = $page.params.task_id

  let prompt_method = "simple_prompt_builder"
  let model: string = $ui_state.selected_model

  // Shared vars for all nodes, so UI saves last used value
  let num_subtopics_to_generate: number = 8
  let num_samples_to_generate: number = 8

  function show_human_guidance_dialog() {
    human_guidance_dialog?.show()
    const text_area = document.getElementById(
      "human_guidance",
    ) as HTMLTextAreaElement
    if (text_area) {
      text_area.focus()
    }
  }

  let human_guidance_dialog: Dialog | null = null
  $: action_buttons = [
    {
      label:
        human_guidance.length > 0
          ? $_("generate.edit_guidance")
          : $_("generate.add_guidance"),
      notice: human_guidance.length > 0,
      handler: show_human_guidance_dialog,
    },
    {
      label: $_("generate.save_all"),
      handler: show_save_all_modal,
    },
  ]

  let root_node: SampleDataNode = {
    topic: "",
    samples: [],
    sub_topics: [],
  }

  onMount(() => {
    get_task()

    // Handle browser reload/close: warn if there are unsaved changes
    window.addEventListener("beforeunload", handleBeforeUnload)
    return () => {
      window.removeEventListener("beforeunload", handleBeforeUnload)
    }
  })

  async function get_task() {
    try {
      task_loading = true
      if (!project_id || !task_id) {
        throw new Error($_("errors.project_task_id_not_set"))
      }
      if ($current_task?.id === task_id) {
        task = $current_task
        return
      }
      const { data: task_response, error: get_error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}",
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
      task = task_response
    } catch (e) {
      if (e instanceof Error && e.message.includes("Load failed")) {
        task_error = new KilnError($_("errors.could_not_load_task"), null)
      } else {
        task_error = createKilnError(e)
      }
    } finally {
      task_loading = false
    }
  }

  function has_unsaved_changes(): boolean {
    if (save_all_completed && save_all_sub_errors.length > 0) {
      return true
    }
    update_data_for_save()
    if (samples_to_save.length > 0) {
      return true
    }
    return false
  }

  // Handle browser reload/close: warn if there are unsaved changes
  function handleBeforeUnload(event: BeforeUnloadEvent) {
    if (has_unsaved_changes()) {
      event.preventDefault()
    }
  }

  // Handle Svelte navigation: warn if there are unsaved changes
  beforeNavigate((navigation) => {
    if (save_all_running) {
      if (!confirm($_("generate.content_generation_running"))) {
        navigation.cancel()
      }
    } else if (has_unsaved_changes()) {
      if (!confirm($_("generate.unsaved_changes_warning"))) {
        navigation.cancel()
      }
    } else if (root_node.sub_topics.length > 0) {
      if (!confirm($_("generate.topic_tree_warning"))) {
        navigation.cancel()
      }
    }
  })

  function show_save_all_modal() {
    // Reset the modal state unless it was already running
    if (!save_all_running) {
      save_all_completed = false
      save_all_error = null
      update_data_for_save()
    }

    // @ts-expect-error showModal is not a method on HTMLElement
    document.getElementById("save_all_dialog")?.showModal()
  }

  // Two functions for recursive collection of data to save.
  let already_saved_count = 0
  let samples_to_save: SampleData[] = []
  let saved_count = 0
  function visit_node_for_collection(node: SampleDataNode, path: string[]) {
    const topic_path = node.topic ? [...path, node.topic] : path
    node.samples.forEach((sample) => {
      if (sample.saved_id) {
        already_saved_count++
      } else {
        // Path may not have been set yet
        sample.topic_path = topic_path
        samples_to_save.push(sample)
      }
    })
    node.sub_topics.forEach((sub_topic) => {
      visit_node_for_collection(sub_topic, topic_path)
    })
  }

  function update_data_for_save() {
    saved_count = 0
    already_saved_count = 0
    samples_to_save = []
    visit_node_for_collection(root_node, [])
  }

  let save_all_running = false
  let save_all_error: KilnError | null = null
  let save_all_sub_errors: KilnError[] = []
  let save_all_completed = false
  let save_all_samples_mode: "parallel" | "sequential" = "parallel"
  let ui_show_errors = false

  // Worker function that processes items until queue is empty
  async function worker(
    queue: SampleData[],
    model_name: string,
    provider: string,
    prompt_method: string,
  ) {
    while (queue.length > 0) {
      const sample = queue.shift()!
      const result = await save_sample(
        sample,
        model_name,
        provider,
        prompt_method,
        sample.topic_path,
      )

      if (result.error) {
        save_all_sub_errors.push(result.error)
        // Trigger reactivity
        save_all_sub_errors = save_all_sub_errors
      } else if (!result.saved_id) {
        save_all_sub_errors.push(new KilnError($_("generate.no_id_returned")))
        // Trigger reactivity
        save_all_sub_errors = save_all_sub_errors
      } else {
        sample.saved_id = result.saved_id
        saved_count++
      }
    }
  }

  async function save_all_samples() {
    try {
      save_all_running = true
      save_all_error = null
      save_all_completed = false
      save_all_sub_errors = []
      const provider = model.split("/")[0]
      const model_name = model.split("/").slice(1).join("/")

      const queue = [...samples_to_save]
      let parallelism = save_all_samples_mode === "parallel" ? 25 : 1

      // Create and start N workers
      const workers = Array(parallelism)
        .fill(null)
        .map(() => worker(queue, model_name, provider, prompt_method))

      // Wait for all workers to complete
      await Promise.all(workers)
    } catch (e) {
      save_all_error = createKilnError(e)
    } finally {
      save_all_running = false
      save_all_completed = true
    }
  }

  type SaveSampleResponse = {
    saved_id: string | null
    error: KilnError | null
  }

  async function save_sample(
    sample: SampleData,
    model_name: string,
    provider: string,
    prompt_method: string,
    topic_path: string[] | undefined,
  ): Promise<SaveSampleResponse> {
    try {
      const formatted_input = task?.input_json_schema
        ? JSON.parse(sample.input)
        : sample.input
      const save_sample_guidance =
        human_guidance.length > 0 ? human_guidance : undefined
      // Get a random split tag, if splits are defined
      const split_tag = split_object?.get_random_split_tag()
      const tags = split_tag ? [split_tag] : []
      const {
        error: post_error,
        data,
        response,
      } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/save_sample",
        {
          params: {
            path: {
              project_id,
              task_id,
            },
            query: {
              session_id,
            },
          },
          body: {
            input: formatted_input,
            input_model_name: sample.model_name,
            input_provider: sample.model_provider,
            output_model_name: model_name,
            output_provider: provider,
            prompt_method,
            topic_path: topic_path || [],
            human_guidance: save_sample_guidance,
            tags,
          },
        },
      )
      if (post_error) {
        throw post_error
      }
      if (response.status !== 200 || !data.id) {
        throw new KilnError($_("generate.failed_to_save_sample"))
      }

      return { saved_id: data.id, error: null }
    } catch (e) {
      const error = createKilnError(e)
      return { saved_id: null, error }
    }
  }

  function clear_human_guidance() {
    human_guidance = ""
    return true
  }
</script>

<Splits bind:splits bind:subtitle={splits_subtitle} bind:this={split_object} />
<div class="max-w-[1400px]">
  <AppPage
    title={$_("generate.synthetic_data_generation")}
    subtitle={splits_subtitle}
    sub_subtitle_link="https://docs.getkiln.ai/docs/synthetic-data-generation"
    sub_subtitle={$_("generate.read_the_docs")}
    {action_buttons}
  >
    {#if task_loading}
      <div class="w-full min-h-[50vh] flex justify-center items-center">
        <div class="loading loading-spinner loading-lg"></div>
      </div>
    {:else if task}
      <div class="flex flex-col">
        <GeneratedDataNode
          data={root_node}
          path={[]}
          {project_id}
          {task_id}
          {human_guidance}
          bind:num_subtopics_to_generate
          bind:num_samples_to_generate
        />
      </div>
    {:else if task_error}
      <div
        class="w-full min-h-[50vh] flex flex-col justify-center items-center gap-2"
      >
        <div class="font-medium">{$_("errors.loading_task")}</div>
        <div class="text-error text-sm">
          {task_error.getMessage() || $_("errors.unknown_error")}
        </div>
      </div>
    {/if}
  </AppPage>
</div>

<dialog id="save_all_dialog" class="modal">
  <div class="modal-box">
    <form method="dialog">
      <button
        class="btn btn-sm text-xl btn-circle btn-ghost absolute right-2 top-2 focus:outline-none"
        >✕</button
      >
    </form>

    {#if save_all_running}
      <div class="min-h-[200px] flex flex-col justify-center items-center">
        <div class="loading loading-spinner loading-lg mb-6 text-success"></div>
        <progress
          class="progress w-56 progress-success"
          value={saved_count}
          max={samples_to_save.length}
        ></progress>
        <div class="font-light text-xs text-center mt-1">
          {saved_count} of {samples_to_save.length}
          {#if save_all_sub_errors && save_all_sub_errors.length > 0}
            {$_("generate.complete")} — {save_all_sub_errors.length}
            {$_("generate.failed")}
          {/if}
        </div>
      </div>
    {:else if save_all_completed}
      <div
        class="text-center flex flex-col items-center justify-center min-h-[150px] p-12"
      >
        {#if saved_count > 0}
          <!-- Uploaded to: SVG Repo, www.svgrepo.com, Generator: SVG Repo Mixer Tools -->
          <svg
            fill="currentColor"
            class="size-10 text-success mb-2"
            viewBox="0 0 56 56"
            xmlns="http://www.w3.org/2000/svg"
            ><path
              d="M 27.9999 51.9063 C 41.0546 51.9063 51.9063 41.0781 51.9063 28 C 51.9063 14.9453 41.0312 4.0937 27.9765 4.0937 C 14.8983 4.0937 4.0937 14.9453 4.0937 28 C 4.0937 41.0781 14.9218 51.9063 27.9999 51.9063 Z M 27.9999 47.9219 C 16.9374 47.9219 8.1014 39.0625 8.1014 28 C 8.1014 16.9609 16.9140 8.0781 27.9765 8.0781 C 39.0155 8.0781 47.8983 16.9609 47.9219 28 C 47.9454 39.0625 39.0390 47.9219 27.9999 47.9219 Z M 25.0468 39.7188 C 25.8202 39.7188 26.4530 39.3437 26.9452 38.6172 L 38.5234 20.4063 C 38.8046 19.9375 39.0858 19.3984 39.0858 18.8828 C 39.0858 17.8047 38.1483 17.1484 37.1640 17.1484 C 36.5312 17.1484 35.9452 17.5 35.5234 18.2031 L 24.9296 35.1484 L 19.4921 28.1172 C 18.9765 27.4141 18.4140 27.1563 17.7812 27.1563 C 16.7499 27.1563 15.9296 28 15.9296 29.0547 C 15.9296 29.5703 16.1405 30.0625 16.4687 30.5078 L 23.0312 38.6172 C 23.6640 39.3906 24.2733 39.7188 25.0468 39.7188 Z"
            /></svg
          >
        {/if}
        <div class="font-medium">
          {$_("generate.saved_new_items", { values: { count: saved_count } })}
        </div>
        <div class="font-light text-sm">
          {$_("generate.use_dataset_tab")}
          <a href={`/dataset/${project_id}/${task_id}`} class="link"
            >{$_("generate.dataset_tab")}</a
          >
          {$_("generate.to_review_manage")}
        </div>
        <div class="font-light text-xs mt-4 text-gray-500">
          {$_("generate.set_tagged_with", {
            values: { tag: `synthetic_session_${session_id}` },
          })}
        </div>
        {#if save_all_sub_errors.length > 0}
          <div class="text-error font-light text-sm mt-4">
            {save_all_sub_errors.length}
            {$_("generate.samples_failed_to_save")}
            <button
              class="link"
              on:click={() => (ui_show_errors = !ui_show_errors)}
            >
              {ui_show_errors
                ? $_("generate.hide_errors")
                : $_("generate.show_errors")}
            </button>
          </div>
          <div
            class="flex flex-col gap-2 mt-4 text-xs text-error {ui_show_errors
              ? ''
              : 'hidden'}"
          >
            {#each save_all_sub_errors as error}
              <div>{error.getMessage()}</div>
            {/each}
          </div>
        {/if}
        {#if save_all_error}
          <div class="text-error font-light text-sm mt-4">
            {$_("generate.error_message")}
            {save_all_error.getMessage() || $_("errors.unknown_error")}
          </div>
        {/if}
      </div>
    {:else if samples_to_save.length == 0}
      <div
        class="flex flex-col items-center justify-center min-h-[150px] gap-2"
      >
        <div class="font-medium">{$_("generate.no_items_to_save")}</div>
        <div class="font-light">{$_("generate.generate_data_to_start")}</div>
        {#if already_saved_count > 0}
          <div class="font-light text-sm">
            {already_saved_count}
            {$_("generate.existing_items_saved")}
          </div>
        {/if}
      </div>
    {:else}
      <h3 class="text-lg font-bold">{$_("generate.save_all_items")}</h3>
      <p class="text-sm font-light mb-8">
        {$_("generate.run_generation")}
      </p>
      <FormContainer
        submit_label={$_("generate.run_and_save")}
        bind:submitting={save_all_running}
        bind:error={save_all_error}
        on:submit={save_all_samples}
      >
        <div>
          <div class="font-medium text-sm">{$_("generate.status")}</div>
          <div class="font-light">
            {samples_to_save.length}
            {$_("generate.items_pending")}
            {#if already_saved_count > 0}
              / {already_saved_count} {$_("generate.already_saved")}
            {/if}
          </div>
        </div>
        {#if human_guidance.length > 0}
          {#if prompt_method.includes("::")}
            <Warning warning_message={$_("generate.human_guidance_enabled")} />
          {:else}
            <Warning
              warning_message={$_("generate.human_guidance_warning")}
              warning_color="warning"
            />
          {/if}
        {/if}
        <AvailableModelsDropdown
          requires_structured_output={task?.output_json_schema ? true : false}
          bind:model
        />
        <PromptTypeSelector bind:prompt_method />
        <FormElement
          id="save_all_samples_mode_element"
          inputType="select"
          info_description={$_("generate.run_mode_description")}
          select_options={[
            ["parallel", $_("generate.parallel_mode")],
            ["sequential", $_("generate.sequential_mode")],
          ]}
          bind:value={save_all_samples_mode}
          label={$_("generate.run_mode")}
        />
      </FormContainer>
    {/if}
  </div>
  <form method="dialog" class="modal-backdrop">
    <button>{$_("generate.close")}</button>
  </form>
</dialog>

<Dialog
  bind:this={human_guidance_dialog}
  title={$_("data_generation.human_guidance")}
  action_buttons={[
    {
      label: $_("data_generation.clear"),
      action: clear_human_guidance,
      disabled: human_guidance.length == 0,
    },
    {
      label: $_("data_generation.done"),
      isPrimary: true,
    },
  ]}
>
  <div>
    <div class="text-sm text-gray-500">
      {$_("data_generation.guidance_description")}
      <a
        href="https://docs.getkiln.ai/docs/synthetic-data-generation#human-guidance"
        target="_blank"
        class="link">{$_("data_generation.in_the_docs")}</a
      >.
    </div>

    <div class="flex flex-col gap-2 w-full mt-4">
      <label for="human_guidance" class="label font-medium p-0 text-sm"
        >{$_("data_generation.guidance_label")}</label
      >
      <textarea
        id="human_guidance"
        bind:value={human_guidance}
        class="input input-bordered h-[200px] py-2"
      />
    </div>
  </div>
</Dialog>
