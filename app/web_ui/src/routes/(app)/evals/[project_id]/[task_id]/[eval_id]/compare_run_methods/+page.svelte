<script lang="ts">
  import AppPage from "../../../../../app_page.svelte"
  import type { Eval } from "$lib/types"
  import { client, base_url } from "$lib/api_client"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount, tick } from "svelte"
  import { page } from "$app/stores"
  import FormElement from "$lib/utils/form_element.svelte"
  import type {
    EvalConfig,
    ProviderModels,
    TaskRunConfig,
    EvalResultSummary,
  } from "$lib/types"
  import { goto } from "$app/navigation"
  import {
    model_info,
    load_model_info,
    model_name,
    provider_name_from_id,
    prompt_name_from_id,
    current_task_prompts,
    load_available_prompts,
    load_available_models,
  } from "$lib/stores"
  import Dialog from "$lib/ui/dialog.svelte"
  import AvailableModelsDropdown from "../../../../../run/available_models_dropdown.svelte"
  import PromptTypeSelector from "../../../../../run/prompt_type_selector.svelte"
  import Warning from "$lib/ui/warning.svelte"
  import { string_to_json_key } from "$lib/utils/json_schema_editor/json_schema_templates"
  import RunEval from "../run_eval.svelte"
  import { eval_config_to_ui_name } from "$lib/utils/formatters"
  import OutputTypeTablePreview from "../output_type_table_preview.svelte"
  import InfoTooltip from "$lib/ui/info_tooltip.svelte"
  import { _ } from "svelte-i18n"

  $: project_id = $page.params.project_id
  $: task_id = $page.params.task_id
  $: eval_id = $page.params.eval_id

  let evaluator: Eval | null = null
  let eval_error: KilnError | null = null
  let eval_loading = true

  let eval_configs: EvalConfig[] | null = null
  let eval_configs_error: KilnError | null = null
  let eval_configs_loading = true
  let current_eval_config_id: string | null = null

  let task_run_configs: TaskRunConfig[] | null = null
  let task_run_configs_error: KilnError | null = null
  let task_run_configs_loading = true

  let score_summary: EvalResultSummary | null = null
  let score_summary_error: KilnError | null = null

  // Note: not including score_summary_error, because it's not a critical error we should block the UI for
  $: loading = eval_loading || eval_configs_loading || task_run_configs_loading
  $: error = eval_error || eval_configs_error || task_run_configs_error

  $: should_select_eval_config =
    task_run_configs?.length && !evaluator?.current_run_config_id
  $: focus_select_eval_config = !!(
    should_select_eval_config && eval_state?.includes("complete")
  )

  onMount(async () => {
    // Wait for page params to load
    await tick()
    // Wait for these 3 to load, as they are needed for better labels. Usually already cached and instant.
    await Promise.all([
      load_model_info(),
      load_available_prompts(),
      load_available_models(),
    ])
    // Get the eval first (want it to set the current config id before the other two load)
    await get_eval()
    // These two can be parallel
    await Promise.all([get_eval_configs(), get_task_run_configs()])
    // This needs the selected eval config id, set from above requests
    get_score_summary()
  })

  async function get_eval() {
    try {
      eval_loading = true
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}",
        {
          params: {
            path: {
              project_id,
              task_id,
              eval_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      evaluator = data
      // Set the selected eval config: prefer query params, then eval's default, then first eval config (set below in load_eval_configs)
      current_eval_config_id =
        $page.url.searchParams.get("selected_eval_config") ||
        evaluator.current_config_id ||
        null
    } catch (error) {
      eval_error = createKilnError(error)
    } finally {
      eval_loading = false
    }
  }

  async function get_eval_configs() {
    try {
      eval_configs_loading = true
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}/eval_configs",
        {
          params: {
            path: {
              project_id,
              task_id,
              eval_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      eval_configs = data
      // Fallback to first eval config if no current eval config id is set from load_eval()
      if (
        !current_eval_config_id &&
        eval_configs.length > 0 &&
        eval_configs[0].id
      ) {
        current_eval_config_id = eval_configs[0].id
      }
    } catch (error) {
      eval_configs_error = createKilnError(error)
    } finally {
      eval_configs_loading = false
    }
  }

  async function get_task_run_configs() {
    try {
      task_run_configs_loading = true
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/task_run_configs",
        {
          params: {
            path: {
              project_id,
              task_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      task_run_configs = data
    } catch (error) {
      task_run_configs_error = createKilnError(error)
    } finally {
      task_run_configs_loading = false
    }
  }

  async function get_score_summary() {
    score_summary = null
    if (!current_eval_config_id) {
      score_summary_error = new KilnError(
        $_("errors.no_evaluation_method_selected"),
        null,
      )
      return
    }
    try {
      score_summary = null
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}/eval_config/{eval_config_id}/score_summary",
        {
          params: {
            path: {
              project_id,
              task_id,
              eval_id,
              eval_config_id: current_eval_config_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      score_summary = data
    } catch (error) {
      score_summary_error = createKilnError(error)
    }
  }

  // Watches the current eval config id, performing actions based on it
  $: watch_selected_eval_config(current_eval_config_id)
  function watch_selected_eval_config(selected_id: string | null) {
    if (selected_id === "add_config") {
      // if it's the "add_config" special value, navigate to the create eval config page
      goto(`/evals/${project_id}/${task_id}/${eval_id}/create_eval_config`)
      return
    }
    // If the selected id is not null, then get the score summary
    score_summary = null
    if (selected_id) {
      get_score_summary()
    }
  }

  type UiProperty = {
    name: string
    value: string
  }

  // A dropdown name for the eval config that is human readable and helpful
  // Combine's it's name with it's properties
  function get_eval_config_name(
    eval_config: EvalConfig,
    model_info: ProviderModels | null,
  ): string {
    let parts = []
    parts.push(eval_config_to_ui_name(eval_config.config_type))
    parts.push(model_name(eval_config.model_name, model_info))
    return eval_config.name + " — " + parts.join(", ")
  }

  $: current_eval_config = eval_configs?.find(
    (config) => config.id === current_eval_config_id,
  )

  // Sort task run configs - default first, then by last output score
  $: sorted_task_run_configs = task_run_configs
    ? sortTaskRunConfigs(task_run_configs, evaluator, score_summary)
    : []

  function sortTaskRunConfigs(
    configs: TaskRunConfig[] | null,
    evaluator: Eval | null,
    score_summary: EvalResultSummary | null,
  ): TaskRunConfig[] {
    if (!configs || !configs.length) return []

    return [...configs].sort((a, b) => {
      // Default run config always comes first
      if (a.id === evaluator?.current_run_config_id) return -1
      if (b.id === evaluator?.current_run_config_id) return 1

      // If we have evaluator and score summary, sort by the last output score
      if (evaluator?.output_scores?.length && score_summary?.results) {
        const lastScoreKey = string_to_json_key(
          evaluator.output_scores[evaluator.output_scores.length - 1].name,
        )

        const scoreA =
          score_summary.results["" + a.id]?.[lastScoreKey]?.mean_score
        const scoreB =
          score_summary.results["" + b.id]?.[lastScoreKey]?.mean_score

        // If both have scores, sort by score (higher first)
        if (
          scoreA !== null &&
          scoreA !== undefined &&
          scoreB !== null &&
          scoreB !== undefined
        ) {
          return scoreB - scoreA
        }

        // If only one has a score, it comes first
        if (scoreA !== null && scoreA !== undefined) return -1
        if (scoreB !== null && scoreB !== undefined) return 1
      }

      // Fallback to sort by name
      return a.name.localeCompare(b.name)
    })
  }

  function get_eval_config_properties(
    eval_config_id: string | null,
    model_info: ProviderModels | null,
  ): UiProperty[] {
    const eval_config = eval_configs?.find(
      (config) => config.id === eval_config_id,
    )
    if (!eval_config) {
      return [
        {
          name: $_("errors.no_evaluation_method_selected"),
          value: $_(
            "evaluation.compare_run_methods.select_config_from_dropdown",
          ),
        },
      ]
    }

    const properties: UiProperty[] = []

    properties.push({
      name: $_("evaluation.results.properties.algorithm"),
      value: eval_config_to_ui_name(eval_config.config_type),
    })
    properties.push({
      name: $_("evaluation.compare_run_methods.eval_model"),
      value: model_name(eval_config.model_name, model_info),
    })
    properties.push({
      name: $_("evaluation.results.properties.model_provider"),
      value: provider_name_from_id(eval_config.model_provider),
    })
    return properties
  }

  function get_eval_config_select_options(
    configs: EvalConfig[] | null,
  ): [string, [unknown, string][]][] {
    const configs_options: [string, string][] = []
    for (const c of configs || []) {
      if (c.id) {
        configs_options.push([c.id, get_eval_config_name(c, $model_info)])
      }
    }

    const results: [string, [unknown, string][]][] = []
    if (configs_options.length > 0) {
      results.push([
        $_("evaluation.compare_run_methods.select_eval_method"),
        configs_options,
      ])
    }
    results.push([
      $_("evaluation.compare_run_methods.manage_eval_methods"),
      [["add_config", $_("evaluation.add_eval_method")]],
    ])
    return results
  }

  let eval_state:
    | "not_started"
    | "running"
    | "complete"
    | "complete_with_errors" = "not_started"
  $: run_eval_url = `${base_url}/api/projects/${project_id}/tasks/${task_id}/eval/${eval_id}/eval_config/${current_eval_config_id}/run_task_run_eval?all_run_configs=true`

  let task_run_config_model_name = ""
  let task_run_config_provider_name = ""
  let task_run_config_prompt_method = "simple_prompt_builder"
  let task_run_config_long_prompt_name_provider = ""

  let add_task_config_dialog: Dialog | null = null
  let add_task_config_error: KilnError | null = null
  async function add_task_config(): Promise<boolean> {
    if (
      !task_run_config_model_name ||
      !task_run_config_provider_name ||
      !task_run_config_prompt_method
    ) {
      add_task_config_error = new KilnError(
        $_("errors.missing_required_fields"),
        null,
      )
      return false
    }

    try {
      const { error } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/task_run_config",
        {
          params: {
            path: {
              project_id,
              task_id,
            },
          },
          body: {
            model_name: task_run_config_model_name,
            // @ts-expect-error not checking types here, server will check them
            model_provider_name: task_run_config_provider_name,
            prompt_id: task_run_config_prompt_method,
          },
        },
      )
      if (error) {
        throw error
      }
      // Load the updated list of task run configs after success
      get_task_run_configs()
    } catch (error) {
      add_task_config_error = createKilnError(error)
      return false
    }
    return true
  }

  function show_incomplete_warning(
    score_summary: EvalResultSummary | null,
  ): boolean {
    if (!score_summary?.run_config_percent_complete) {
      return false
    }

    const values = Object.values(score_summary.run_config_percent_complete)
    const minComplete =
      values.length > 0
        ? values.reduce((min, val) => Math.min(min, val), 1.0)
        : 1.0
    return minComplete < 1.0
  }

  $: has_default_eval_config = evaluator && evaluator.current_config_id

  async function set_current_run_config(
    run_config_id: string | null | undefined,
  ) {
    if (!run_config_id) {
      return
    }
    try {
      const { data, error } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}/set_current_run_config/{run_config_id}",
        {
          params: {
            path: {
              project_id: $page.params.project_id,
              task_id: $page.params.task_id,
              eval_id: $page.params.eval_id,
              run_config_id: run_config_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      // Update the evaluator with the latest
      evaluator = data
    } catch (error) {
      eval_error = createKilnError(error)
    }
  }
</script>

<AppPage
  title={$_("evaluation.compare_run_methods.title")}
  subtitle={$_("evaluation.compare_run_methods.subtitle")}
  sub_subtitle={$_("evaluation.compare_run_methods.sub_subtitle")}
  sub_subtitle_link="https://docs.getkiln.ai/docs/evaluations#finding-the-ideal-run-method"
  action_buttons={[
    {
      label: $_("evaluation.compare_run_methods.compare_eval_methods_button"),
      href: `/evals/${project_id}/${task_id}/${eval_id}/eval_configs`,
      primary: !has_default_eval_config,
    },
  ]}
>
  {#if loading}
    <div class="w-full min-h-[50vh] flex justify-center items-center">
      <div class="loading loading-spinner loading-lg"></div>
    </div>
  {:else if error}
    <div
      class="w-full min-h-[50vh] flex flex-col justify-center items-center gap-2"
    >
      <div class="font-medium">{$_("errors.loading_task")}</div>
      <div class="text-error text-sm">
        {error.getMessage() || $_("errors.unknown_error")}
      </div>
    </div>
  {:else if evaluator}
    <div class="flex flex-col xl:flex-row gap-8 xl:gap-16 mb-8">
      <div class="grow flex flex-col gap-4">
        <div>
          <div class="text-xl font-bold">
            {$_("evaluation.compare_run_methods.evaluation_method_section")}
          </div>
          <div class="text-sm text-gray-500 mb-2">
            {$_("evaluation.compare_run_methods.evaluation_method_desc")}
          </div>

          <FormElement
            hide_label={true}
            id="eval_config_select"
            label={$_("evaluation.add_eval_method")}
            inputType="select"
            bind:value={current_eval_config_id}
            select_options_grouped={get_eval_config_select_options(
              eval_configs,
            )}
          />
          {#if !has_default_eval_config}
            <Warning
              warning_message={$_(
                "evaluation.compare_run_methods.no_default_warning",
              )}
              warning_color="warning"
              tight={true}
            />
          {:else if has_default_eval_config && evaluator.current_config_id != current_eval_config_id}
            <Warning
              warning_message={$_(
                "evaluation.compare_run_methods.not_default_warning",
              )}
              warning_color="warning"
              tight={true}
            />
          {/if}
        </div>
        <div
          class="grid grid-cols-[auto,1fr] gap-y-2 gap-x-4 text-sm 2xl:text-base"
        >
          {#each get_eval_config_properties(current_eval_config_id, $model_info) as property}
            <div class="flex items-center">{property.name}</div>
            <div class="flex items-center text-gray-500 overflow-x-hidden">
              {property.value}
            </div>
          {/each}
          <div class="flex items-center">
            {$_("evaluation.compare_run_methods.eval_method_quality")}
          </div>
          <div class="flex items-center text-gray-500 overflow-x-hidden">
            <a
              href={`/evals/${project_id}/${task_id}/${eval_id}/eval_configs`}
              class="link"
            >
              {$_("evaluation.compare_run_methods.compare_and_optimize")}
            </a>
          </div>
        </div>
      </div>
    </div>
    <div class="mt-16">
      {#if task_run_configs?.length}
        <div class="flex flex-col lg:flex-row gap-4 lg:gap-8 mb-6">
          <div class="grow">
            <div class="text-xl font-bold">
              {$_("evaluation.compare_run_methods.run_methods_section")}
            </div>
            <div class="text-xs text-gray-500">
              {$_("evaluation.compare_run_methods.run_methods_desc")}
              <InfoTooltip
                tooltip_text={$_(
                  "evaluation.compare_run_methods.run_methods_tooltip",
                  {
                    values: {
                      eval_name:
                        current_eval_config?.name ||
                        $_("evaluation.compare_run_methods.select_above"),
                    },
                  },
                )}
                position="left"
                no_pad={true}
              />
            </div>
            {#if score_summary_error}
              <div class="text-error text-sm">
                {score_summary_error.getMessage() || $_("errors.unknown_error")}
              </div>
            {/if}
          </div>
          <div class="shrink-0">
            <button
              class="btn btn-mid mr-2"
              on:click={() => {
                add_task_config_dialog?.show()
              }}>{$_("evaluation.compare_run_methods.add_run_method")}</button
            >
            <RunEval
              bind:eval_state
              bind:run_url={run_eval_url}
              on_run_complete={() => {
                get_score_summary()
              }}
            />
          </div>
        </div>

        <!-- Warn the user if some evals are incomplete -->
        {#if show_incomplete_warning(score_summary)}
          <div class="mt-6 mb-4">
            <button
              class="tooltip tooltip-top cursor-pointer"
              data-tip={$_("evaluation.compare_run_methods.incomplete_tooltip")}
            >
              <Warning
                warning_message={$_(
                  "evaluation.compare_run_methods.incomplete_warning",
                )}
                tight={true}
              />
            </button>
          </div>
        {:else if should_select_eval_config}
          <div class="mb-4">
            <Warning
              warning_message={$_(
                "evaluation.compare_run_methods.select_winner_warning",
              )}
              warning_color={focus_select_eval_config ? "primary" : "gray"}
              warning_icon="info"
              large_icon={focus_select_eval_config}
              tight={true}
            />
          </div>
        {/if}

        <div class="overflow-x-auto rounded-lg border">
          <table class="table">
            <thead>
              <tr>
                <th>
                  <div>
                    {$_("evaluation.compare_run_methods.run_method_column")}
                  </div>
                  <div class="font-normal">
                    {$_(
                      "evaluation.compare_run_methods.run_method_column_desc",
                    )}
                  </div>
                </th>
                {#each evaluator.output_scores as output_score}
                  <th class="text-center">
                    {output_score.name}
                    <OutputTypeTablePreview
                      output_score_type={output_score.type}
                    />
                  </th>
                {/each}
              </tr>
            </thead>
            <tbody>
              {#each sorted_task_run_configs as task_run_config}
                {@const percent_complete =
                  score_summary?.run_config_percent_complete?.[
                    "" + task_run_config.id
                  ]}{@const prompt_name =
                  task_run_config.prompt?.name ||
                  prompt_name_from_id(
                    task_run_config?.run_config_properties?.prompt_id,
                    $current_task_prompts,
                  )}
                <tr
                  class="hover cursor-pointer"
                  on:click={() => {
                    goto(
                      `/evals/${project_id}/${task_id}/${eval_id}/${current_eval_config_id}/${task_run_config.id}/run_result`,
                    )
                  }}
                >
                  <td>
                    <div class="font-medium">
                      {model_name(
                        task_run_config?.run_config_properties?.model_name,
                        $model_info,
                      )}
                    </div>

                    <div class="text-sm text-gray-500">
                      {$_("evaluation.compare_run_methods.prompt_label")}
                      {#if task_run_config?.prompt?.generator_id && task_run_config?.run_config_properties?.prompt_id?.startsWith("task_run_config::")}
                        <!-- Special description for prompts frozen to the task run config. The name alone isn't that helpful, so we say where it comes from (eg "Basic (Zero Shot")) -->
                        {prompt_name_from_id(
                          task_run_config?.prompt?.generator_id,
                          $current_task_prompts,
                        )}
                        <InfoTooltip
                          tooltip_text={$_(
                            "evaluation.compare_run_methods.prompt_frozen_tooltip",
                            { values: { prompt_name } },
                          )}
                          position="right"
                          no_pad={true}
                        />
                      {:else}
                        {prompt_name}
                      {/if}
                    </div>
                    <div class="text-sm text-gray-500">
                      {$_("evaluation.compare_run_methods.provider_label")}
                      {provider_name_from_id(
                        task_run_config?.run_config_properties
                          ?.model_provider_name,
                      )}
                    </div>
                    <div class="text-sm text-gray-500">
                      {$_(
                        "evaluation.compare_run_methods.run_method_name_label",
                      )}
                      {task_run_config.name}
                    </div>
                    {#if percent_complete}
                      {#if percent_complete < 1.0}
                        <div class="text-sm 'text-error'">
                          {$_("evaluation.compare_run_methods.progress_label")}
                          {(percent_complete * 100.0).toFixed(1)}%
                        </div>
                      {/if}
                    {:else if score_summary}
                      <!-- We have results, but not for this run config -->
                      <div class="text-sm text-error">
                        {$_("evaluation.compare_run_methods.progress_label")} 0%
                      </div>
                    {/if}
                    {#if task_run_config.id == evaluator.current_run_config_id}
                      <button
                        class="badge badge-primary mt-2"
                        on:click={(event) => {
                          event.stopPropagation()
                          set_current_run_config("None")
                        }}
                      >
                        {$_("evaluation.compare_run_methods.default_badge")}
                        <span class="pl-2">&#x2715;</span>
                      </button>
                    {:else}
                      <button
                        class="badge mt-1 {focus_select_eval_config
                          ? 'badge-primary'
                          : 'badge-secondary badge-outline'}"
                        on:click={(event) => {
                          event.stopPropagation()
                          set_current_run_config(task_run_config.id)
                        }}
                      >
                        {$_("evaluation.compare_run_methods.set_as_default")}
                      </button>
                    {/if}
                  </td>
                  {#each evaluator.output_scores as output_score}
                    {@const score =
                      score_summary?.results?.["" + task_run_config.id]?.[
                        string_to_json_key(output_score.name)
                      ]?.mean_score}
                    <td class="text-center">
                      {score != null
                        ? score.toFixed(2)
                        : $_("formatters.unknown").toLowerCase()}
                    </td>
                  {/each}
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {:else}
        <div class="text-xl font-bold">
          {$_("evaluation.compare_run_methods.no_run_methods_title")}
        </div>
        <div class="text-sm text-gray-500">
          {$_("evaluation.compare_run_methods.no_run_methods_desc")}
        </div>

        <button
          class="btn min-w-[200px] mt-4 {has_default_eval_config
            ? 'btn-primary'
            : ''}"
          on:click={() => {
            add_task_config_dialog?.show()
          }}
        >
          {$_("evaluation.compare_run_methods.add_run_method")}
        </button>
      {/if}
    </div>
  {/if}
</AppPage>

<Dialog
  bind:this={add_task_config_dialog}
  title={$_("evaluation.compare_run_methods.add_task_config_dialog.title")}
  action_buttons={[
    {
      label: $_("common.cancel"),
      isCancel: true,
    },
    {
      label: $_("common.create"),
      isPrimary: true,
      asyncAction: add_task_config,
    },
  ]}
>
  <h4 class="text-sm text-gray-500">
    {$_("evaluation.compare_run_methods.add_task_config_dialog.description")}
  </h4>
  <h4 class="text-sm text-gray-500 mt-1">
    {$_("evaluation.compare_run_methods.add_task_config_dialog.description2")}
  </h4>
  <div class="flex flex-col gap-2 pt-6">
    <AvailableModelsDropdown
      bind:model_name={task_run_config_model_name}
      bind:provider_name={task_run_config_provider_name}
      bind:model={task_run_config_long_prompt_name_provider}
    />
    <PromptTypeSelector
      bind:prompt_method={task_run_config_prompt_method}
      bind:linked_model_selection={task_run_config_long_prompt_name_provider}
    />
    {#if add_task_config_error}
      <div class="text-error text-sm">
        {add_task_config_error.getMessage() || $_("errors.unknown_error")}
      </div>
    {/if}
  </div>
</Dialog>
