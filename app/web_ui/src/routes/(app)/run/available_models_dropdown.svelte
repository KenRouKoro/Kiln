<script lang="ts">
  import {
    available_models,
    load_available_models,
    available_model_details,
    ui_state,
  } from "$lib/stores"
  import type { AvailableModels } from "$lib/types"
  import { onMount } from "svelte"
  import FormElement from "$lib/utils/form_element.svelte"
  import Warning from "$lib/ui/warning.svelte"
  import { _ } from "svelte-i18n"

  export let model: string = $ui_state.selected_model
  export let requires_structured_output: boolean = false
  export let requires_data_gen: boolean = false
  export let requires_logprobs: boolean = false
  export let error_message: string | null = null
  export let suggested_mode: "data_gen" | "evals" | null = null
  $: $ui_state.selected_model = model
  $: model_options = format_model_options(
    $available_models || {},
    requires_structured_output,
    requires_data_gen,
    requires_logprobs,
    $ui_state.current_task_id,
  )

  // Export the parsed model name and provider name
  export let model_name: string | null = null
  export let provider_name: string | null = null
  $: get_model_provider(model)
  function get_model_provider(model_provider: string) {
    model_name = model_provider
      ? model_provider.split("/").slice(1).join("/")
      : null
    provider_name = model_provider ? model_provider.split("/")[0] : null
  }

  onMount(async () => {
    await load_available_models()
  })

  let unsupported_models: [string, string][] = []
  let untested_models: [string, string][] = []
  function format_model_options(
    providers: AvailableModels[],
    structured_output: boolean,
    requires_data_gen: boolean,
    requires_logprobs: boolean,
    current_task_id: string | null,
  ): [string, [unknown, string][]][] {
    let options = []
    unsupported_models = []
    untested_models = []
    for (const provider of providers) {
      let model_list = []
      for (const model of provider.models) {
        // Exclude models that are not available for the current task
        if (
          model &&
          model.task_filter &&
          current_task_id &&
          !model.task_filter.includes(current_task_id)
        ) {
          continue
        }

        let id = provider.provider_id + "/" + model.id
        let long_label = provider.provider_name + " / " + model.name
        if (model.untested_model) {
          untested_models.push([id, long_label])
          continue
        }
        if (requires_data_gen && !model.supports_data_gen) {
          unsupported_models.push([id, long_label])
          continue
        }
        if (structured_output && !model.supports_structured_output) {
          unsupported_models.push([id, long_label])
          continue
        }
        if (requires_logprobs && !model.supports_logprobs) {
          unsupported_models.push([id, long_label])
          continue
        }
        let model_name = model.name
        if (suggested_mode === "data_gen" && model.suggested_for_data_gen) {
          model_name = model.name + "  —  " + $_("models.recommended")
        } else if (suggested_mode === "evals" && model.suggested_for_evals) {
          model_name = model.name + "  —  " + $_("models.recommended")
        }
        model_list.push([id, model_name])
      }
      if (model_list.length > 0) {
        options.push([provider.provider_name, model_list])
      }
    }

    if (untested_models.length > 0) {
      options.push([$_("models.untested_models"), untested_models])
    }

    if (unsupported_models.length > 0) {
      let not_recommended_label = $_("models.not_recommended")
      if (requires_data_gen) {
        not_recommended_label = $_("models.not_recommended_data_gen_label")
      } else if (requires_structured_output) {
        not_recommended_label = $_("models.not_recommended_structured_label")
      } else if (requires_logprobs) {
        not_recommended_label = $_("models.not_recommended_logprobs_label")
      }
      options.push([not_recommended_label, unsupported_models])
    }

    // @ts-expect-error this is the correct format, but TS isn't finding it
    return options
  }

  // Extra check to make sure the model is available to use
  export function get_selected_model(): string | null {
    for (const provider of model_options) {
      if (provider[1].find((m) => m[0] === model)) {
        return model
      }
    }
    return null
  }

  $: selected_model_untested = untested_models.find((m) => m[0] === model)
  $: selected_model_unsupported = unsupported_models.find((m) => m[0] === model)

  $: selected_model_suggested_data_gen =
    available_model_details(model_name, provider_name, $available_models)
      ?.suggested_for_data_gen || false

  $: selected_model_suggested_evals =
    available_model_details(model_name, provider_name, $available_models)
      ?.suggested_for_evals || false
</script>

<div>
  <FormElement
    label={$_("models.model_label")}
    bind:value={model}
    id="model"
    inputType="select"
    bind:error_message
    select_options_grouped={model_options}
  />

  {#if selected_model_untested}
    <Warning warning_message={$_("models.not_tested")} />
  {:else if selected_model_unsupported}
    {#if requires_data_gen}
      <Warning warning_message={$_("models.not_recommended_data_gen")} />
    {:else if requires_logprobs}
      <Warning warning_message={$_("models.no_logprobs")} />
    {:else if requires_structured_output}
      <Warning warning_message={$_("models.not_recommended_structured")} />
    {/if}
  {:else if suggested_mode === "data_gen"}
    <Warning
      warning_icon={!model
        ? "info"
        : selected_model_suggested_data_gen
          ? "check"
          : "exclaim"}
      warning_color={!model
        ? "gray"
        : selected_model_suggested_data_gen
          ? "success"
          : "warning"}
      warning_message={$_("models.suggest_data_gen")}
    />
  {:else if suggested_mode === "evals"}
    <Warning
      warning_icon={!model
        ? "info"
        : selected_model_suggested_evals
          ? "check"
          : "exclaim"}
      warning_color={!model
        ? "gray"
        : selected_model_suggested_evals
          ? "success"
          : "warning"}
      warning_message={$_("models.suggest_evals")}
    />
  {/if}
</div>
