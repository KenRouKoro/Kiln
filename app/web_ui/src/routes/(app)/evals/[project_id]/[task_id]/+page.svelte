<script lang="ts">
  import AppPage from "../../../app_page.svelte"
  import EmptyEvaluator from "./empty_eval.svelte"
  import type { Eval, TaskRunConfig } from "$lib/types"
  import { client } from "$lib/api_client"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount, tick } from "svelte"
  import { goto } from "$app/navigation"
  import { page } from "$app/stores"
  import {
    model_info,
    load_model_info,
    model_name,
    prompt_name_from_id,
    current_task_prompts,
  } from "$lib/stores"
  import { prompt_link } from "$lib/utils/link_builder"
  import { _ } from "svelte-i18n"

  $: project_id = $page.params.project_id
  $: task_id = $page.params.task_id

  let evals: Eval[] | null = null
  let evals_error: KilnError | null = null
  let evals_loading = true

  $: is_empty = !evals || evals.length == 0
  $: sorted_evals = (
    evals
      ? [...evals].sort((a, b) => {
          // First sort by favorite status
          const favDiff = Number(b.favourite) - Number(a.favourite)
          if (favDiff !== 0) return favDiff
          // If favorite status is the same, sort by ID
          return (a.id || "").localeCompare(b.id || "")
        })
      : []
  ) as Eval[]

  onMount(async () => {
    // Wait for params to load
    await tick()
    // Usually cached and fast
    load_model_info()
    // Load the evals and run configs in parallel
    Promise.all([get_evals(), get_run_configs()])
  })

  async function get_evals() {
    try {
      evals_loading = true
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/evals",
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
      evals = data
    } catch (error) {
      evals_error = createKilnError(error)
    } finally {
      evals_loading = false
    }
  }

  let run_configs: TaskRunConfig[] | null = null
  let run_configs_error: KilnError | null = null
  let run_configs_loading = true

  async function get_run_configs() {
    // "/api/projects/{project_id}/tasks/{task_id}/task_run_configs"
    try {
      run_configs_loading = true
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
      run_configs = data
    } catch (error) {
      run_configs_error = createKilnError(error)
    } finally {
      run_configs_loading = false
    }
  }

  let toggle_eval_favourite_error: KilnError | null = null

  $: loading = evals_loading || run_configs_loading
  $: error = evals_error || run_configs_error || toggle_eval_favourite_error

  async function toggle_eval_favourite(evaluator: Eval) {
    try {
      if (!evaluator.id) {
        throw new Error($_("errors.eval_id_required"))
      }
      const new_fav_state = !evaluator.favourite
      const { data, error } = await client.PATCH(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}/fav",
        {
          params: {
            path: { project_id, task_id, eval_id: evaluator.id },
          },
          body: { favourite: new_fav_state },
        },
      )
      if (error) {
        throw error
      }
      evaluator.favourite = data.favourite
      // Trigger reactivity
      evals = evals
    } catch (error) {
      toggle_eval_favourite_error = createKilnError(error)
    }
  }
</script>

<AppPage
  title={$_("evaluation.page_title")}
  subtitle={$_("evaluation.page_subtitle")}
  sub_subtitle={is_empty ? undefined : $_("evaluation.read_docs")}
  sub_subtitle_link="https://docs.getkiln.ai/docs/evaluations"
  action_buttons={is_empty
    ? []
    : [
        {
          label: $_("evaluation.new_evaluator"),
          href: `/evals/${project_id}/${task_id}/create_evaluator`,
          primary: true,
        },
      ]}
>
  {#if loading}
    <div class="w-full min-h-[50vh] flex justify-center items-center">
      <div class="loading loading-spinner loading-lg"></div>
    </div>
  {:else if is_empty}
    <div class="flex flex-col items-center justify-center min-h-[60vh]">
      <EmptyEvaluator {project_id} {task_id} />
    </div>
  {:else if error}
    <div
      class="w-full min-h-[50vh] flex flex-col justify-center items-center gap-2"
    >
      <div class="font-medium">{$_("common.error")}</div>
      <div class="text-error text-sm">
        {error.getMessage() || $_("evaluation.unknown_error_occurred")}
      </div>
    </div>
  {:else if evals}
    <div class="overflow-x-auto rounded-lg border">
      <table class="table">
        <thead>
          <tr>
            <th></th>
            <th>{$_("evaluation.eval_name")}</th>
            <th>{$_("evaluation.description")}</th>
            <th>{$_("evaluation.selected_run_method")}</th>
          </tr>
        </thead>
        <tbody>
          {#each sorted_evals as evaluator}
            {@const run_config = run_configs?.find(
              (run_config) => run_config.id === evaluator.current_run_config_id,
            )}
            {@const prompt_href = run_config?.run_config_properties.prompt_id
              ? prompt_link(
                  project_id,
                  task_id,
                  run_config.run_config_properties.prompt_id,
                )
              : undefined}
            {@const prompt_name =
              run_config?.prompt?.name ||
              prompt_name_from_id(
                run_config?.run_config_properties.prompt_id || "",
                $current_task_prompts,
              )}
            <tr
              class="hover cursor-pointer"
              on:click={() => {
                goto(`/evals/${project_id}/${task_id}/${evaluator.id}`)
              }}
            >
              <td class="pr-0">
                <button
                  class="mask mask-star-2 h-5 w-5 pt-1 {evaluator.favourite
                    ? 'bg-amber-300 hover:bg-amber-400'
                    : 'bg-gray-300 hover:bg-gray-400'}"
                  on:click={(event) => {
                    event.stopPropagation()
                    toggle_eval_favourite(evaluator)
                  }}
                ></button>
              </td>
              <td> {evaluator.name} </td>
              <td> {evaluator.description} </td>
              <td>
                {#if run_config}
                  <div
                    class="grid grid-cols-[auto_1fr] gap-y-1 gap-x-4 lg:min-w-[260px]"
                  >
                    <div>{$_("evaluation.model")}:</div>
                    <div class="text-gray-500">
                      {model_name(
                        run_config.run_config_properties.model_name,
                        $model_info,
                      )}
                    </div>
                    <div>{$_("evaluation.prompt")}:</div>
                    <div class="text-gray-500">
                      {#if prompt_href}
                        <a href={prompt_href} class="link">{prompt_name}</a>
                      {:else}
                        {prompt_name}
                      {/if}
                    </div>
                  </div>
                {:else}
                  <div class="text-gray-500">{$_("evaluation.na")}</div>
                {/if}
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}
</AppPage>
