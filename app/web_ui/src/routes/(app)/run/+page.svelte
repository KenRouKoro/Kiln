<script lang="ts">
  import AppPage from "../app_page.svelte"
  import { current_task, current_project, ui_state } from "$lib/stores"
  import { createKilnError } from "$lib/utils/error_handlers"
  import FormContainer from "$lib/utils/form_container.svelte"
  import PromptTypeSelector from "./prompt_type_selector.svelte"
  import { KilnError } from "$lib/utils/error_handlers"
  import Run from "./run.svelte"
  import { client } from "$lib/api_client"
  import type { TaskRun } from "$lib/types"
  import AvailableModelsDropdown from "./available_models_dropdown.svelte"
  import RunInputForm from "./run_input_form.svelte"
  import { _ } from "svelte-i18n"

  // TODO: implement checking input content
  // let warn_before_unload
  // TODO UI for errors
  let error: KilnError | null = null
  let submitting = false
  let run_complete = false

  let input_form: RunInputForm

  let prompt_method = "simple_prompt_builder"
  let model: string = $ui_state.selected_model

  $: model_name = model ? model.split("/").slice(1).join("/") : ""
  $: provider = model ? model.split("/")[0] : ""
  let model_dropdown: AvailableModelsDropdown
  let model_dropdown_error_message: string | null = null

  let response: TaskRun | null = null
  $: run_focus = !response

  $: subtitle = $current_task ? $_("run.task_prefix") + $current_task.name : ""
  $: input_schema = $current_task?.input_json_schema
  $: requires_structured_output = !!$current_task?.output_json_schema

  async function run_task() {
    try {
      submitting = true
      error = null
      response = null
      run_complete = false
      model_dropdown_error_message = null
      let selected_model = model_dropdown.get_selected_model()
      if (!selected_model || selected_model != model) {
        model_dropdown_error_message = $_("run.required")
        throw new Error($_("run.model_selection_error"))
      }
      const {
        data, // only present if 2XX response
        error: fetch_error, // only present if 4XX or 5XX response
      } = await client.POST("/api/projects/{project_id}/tasks/{task_id}/run", {
        params: {
          path: {
            project_id: $current_project?.id || "",
            task_id: $current_task?.id || "",
          },
        },
        body: {
          model_name: model_name,
          provider: provider,
          plaintext_input: input_form.get_plaintext_input_data(),
          // @ts-expect-error openapi-fetch generates the wrong type for this: Record<string, never>
          structured_input: input_form.get_structured_input_data(),
          ui_prompt_method: prompt_method,
          tags: ["manual_run"],
        },
      })
      if (fetch_error) {
        throw fetch_error
      }
      response = data
    } catch (e) {
      error = createKilnError(e)
    } finally {
      submitting = false
    }
  }

  function clear_all() {
    input_form.clear_input()
    response = null
    run_complete = false
  }

  function next_task_run() {
    // Keep the input, but clear the response
    response = null
    run_complete = false
    clear_all()
  }
</script>

<div class="max-w-[1400px]">
  <AppPage
    title={$_("run.title")}
    bind:subtitle
    action_buttons={[{ label: $_("run.clear_all"), handler: clear_all }]}
  >
    <div class="flex flex-col xl:flex-row gap-8 xl:gap-16">
      <div class="grow">
        <div class="text-xl font-bold mb-4">{$_("run.input")}</div>
        <FormContainer
          submit_label={$_("run.title")}
          on:submit={run_task}
          bind:error
          bind:submitting
          bind:primary={run_focus}
          bind:keyboard_submit={run_focus}
        >
          <RunInputForm bind:input_schema bind:this={input_form} />
        </FormContainer>
      </div>
      <div class="w-72 2xl:w-96 flex-none flex flex-col gap-4">
        <div class="text-xl font-bold">{$_("run.options")}</div>
        <div>
          <PromptTypeSelector
            bind:prompt_method
            info_description={$_("run.choose_prompt_description")}
            bind:linked_model_selection={model}
          />
        </div>
        <AvailableModelsDropdown
          bind:model
          bind:requires_structured_output
          bind:error_message={model_dropdown_error_message}
          bind:this={model_dropdown}
        />
      </div>
    </div>
    {#if $current_task && !submitting && response != null && $current_project?.id}
      <div class="mt-10 xl:mt-24">
        <Run
          initial_run={response}
          task={$current_task}
          project_id={$current_project.id}
          bind:model_name
          bind:provider
          bind:run_complete
          focus_repair_on_appear={true}
        />
      </div>
    {/if}
    {#if run_complete}
      <div class="flex flex-col md:flex-row gap-6 place-content-center mt-10">
        <button
          class="btn btn-primary mt-2 min-w-48"
          on:click={() => next_task_run()}
        >
          {$_("run.next_run")}
        </button>
      </div>
    {/if}
  </AppPage>
</div>
