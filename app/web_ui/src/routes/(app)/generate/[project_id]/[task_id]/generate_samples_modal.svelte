<script lang="ts">
  import type { SampleDataNode } from "./gen_model"
  import AvailableModelsDropdown from "../../../run/available_models_dropdown.svelte"
  import IncrementUi from "./increment_ui.svelte"
  import { KilnError } from "../../../../../lib/utils/error_handlers"
  import { client } from "$lib/api_client"
  import { createKilnError } from "$lib/utils/error_handlers"
  import FormElement from "../../../../../lib/utils/form_element.svelte"
  import { _ } from "svelte-i18n"

  // the number of workers to use for parallel generation
  const PARALLEL_WORKER_COUNT = 25

  type GenerateSamplesOutcome = {
    topic: TopicNodeWithPath
    error: KilnError | null
  }

  interface TopicNodeWithPath {
    path: string[]
    node: SampleDataNode
  }

  export let id: string
  export let data: SampleDataNode
  export let path: string[]
  export let project_id: string
  export let task_id: string
  export let human_guidance: string | null = null
  export let model: string
  export let num_samples_to_generate: number = 8
  export let custom_topics_string: string | null = null

  /**
   * If true, generate samples for each topic leaf descendant of the current topic.
   */
  export let cascade_mode: boolean = false

  /**
   * If true, generate samples in parallel.
   */
  export let generate_samples_mode: "parallel" | "sequential" = "parallel"

  let ui_show_errors = false
  let generate_samples_outcomes: Record<string, GenerateSamplesOutcome> = {}

  $: topics_failed_to_generate = Object.values(
    generate_samples_outcomes,
  ).filter((o) => o.error)

  $: topics_failed_to_generate_count = topics_failed_to_generate.length

  export let on_completed: () => void

  function add_synthetic_samples(
    topic: SampleDataNode,
    samples: unknown[],
    model_name: string,
    model_provider: string,
  ) {
    for (const sample of samples) {
      if (!sample) {
        continue
      }
      let input: string | null = null
      if (typeof sample == "string") {
        input = sample
      } else if (typeof sample == "object" || Array.isArray(sample)) {
        input = JSON.stringify(sample)
      }
      if (input) {
        topic.samples.push({
          input: input,
          saved_id: null,
          model_name,
          model_provider,
        })
      }
    }
  }

  type GenerateSampleResponse = {
    error: KilnError | null
  }

  let sample_generating: boolean = false
  async function request_generate_samples(
    topic: TopicNodeWithPath,
  ): Promise<GenerateSampleResponse> {
    try {
      if (!model) {
        throw new KilnError(
          $_("generate.samples_modal.no_model_selected"),
          null,
        )
      }
      const model_provider = model.split("/")[0]
      const model_name = model.split("/").slice(1).join("/")
      if (!model_name || !model_provider) {
        throw new KilnError(
          $_("generate.samples_modal.invalid_model_selected"),
          null,
        )
      }
      const { data: generate_response, error: generate_error } =
        await client.POST(
          "/api/projects/{project_id}/tasks/{task_id}/generate_samples",
          {
            body: {
              topic: topic.path,
              num_samples: num_samples_to_generate,
              model_name: model_name,
              provider: model_provider,
              human_guidance: human_guidance ? human_guidance : null, // clear empty string
            },
            params: {
              path: {
                project_id,
                task_id,
              },
            },
          },
        )
      if (generate_error) {
        throw generate_error
      }
      const response = JSON.parse(generate_response.output.output)
      if (
        !response ||
        !response.generated_samples ||
        !Array.isArray(response.generated_samples)
      ) {
        throw new KilnError(
          $_("generate.samples_modal.no_options_returned"),
          null,
        )
      }
      // Add new samples
      add_synthetic_samples(
        topic.node,
        response.generated_samples,
        model_name,
        model_provider,
      )
    } catch (e) {
      if (e instanceof KilnError) {
        return { error: e }
      }
      return { error: createKilnError(e) }
    }

    return { error: null }
  }

  /**
   * Return all descendant topic nodes in the tree (excluding the node itself).
   */
  function get_descendant_topic_nodes(
    node: TopicNodeWithPath,
  ): TopicNodeWithPath[] {
    const descendants: TopicNodeWithPath[] = []

    const stack: TopicNodeWithPath[] = [node]
    while (stack.length > 0) {
      const curr = stack.pop()!
      descendants.push(curr)

      // add children and construct their paths
      curr.node.sub_topics.forEach((n) => {
        stack.push({
          path: curr.path.concat(n.topic),
          node: n,
        })
      })
    }

    // remove the starting node from the list
    return descendants.slice(1)
  }

  /**
   * Return all leaf topic nodes in the tree (including the node itself if it is a leaf).
   */
  function collect_leaf_topic_nodes(): TopicNodeWithPath[] {
    const node_with_path = { path, node: data }
    return [node_with_path]
      .concat(get_descendant_topic_nodes(node_with_path))
      .filter((t) => t.node.sub_topics.length == 0)
  }

  async function generate_samples() {
    sample_generating = true
    generate_samples_outcomes = {}

    const queue = cascade_mode
      ? collect_leaf_topic_nodes()
      : [{ path, node: data }]

    let parallelism =
      generate_samples_mode === "parallel" ? PARALLEL_WORKER_COUNT : 1

    // Create and start N workers
    const workers = Array(parallelism)
      .fill(null)
      .map(() => worker(queue))

    // Wait for all workers to complete
    await Promise.all(workers)

    sample_generating = false

    // if every topic was generated successfully, move on
    // otherwise we stay here to show the user the errors
    if (topics_failed_to_generate_count === 0) {
      on_completed()
    }
  }

  async function worker(queue: TopicNodeWithPath[]) {
    while (queue.length > 0) {
      const topic = queue.shift()!
      const result = await request_generate_samples(topic)

      const path_serialized = serialize_topic_path(topic.path)
      if (result.error) {
        generate_samples_outcomes[path_serialized] = {
          topic,
          error: result.error,
        }
      } else {
        generate_samples_outcomes[path_serialized] = {
          topic,
          error: null,
        }
      }

      // Trigger reactivity
      generate_samples_outcomes = generate_samples_outcomes
    }
  }

  function serialize_topic_path(path: string[]) {
    return path.join("/")
  }
</script>

<dialog id={`${id}-generate-samples`} class="modal">
  <div class="modal-box">
    <form method="dialog">
      <button
        class="btn btn-sm text-xl btn-circle btn-ghost absolute right-2 top-2 focus:outline-none"
        >✕</button
      >
    </form>
    <h3 class="text-lg font-bold">{$_("generate.samples_modal.title")}</h3>
    <p class="text-sm font-light mb-8">
      {$_("generate.samples_modal.subtitle")}
      {#if path.length > 0}
        {$_("generate.samples_modal.subtitle_to")}
        {cascade_mode
          ? $_("generate.samples_modal.subtitle_each_subtopic") + " "
          : ""}{path.join(" → ")}
      {/if}
    </p>
    {#if sample_generating}
      <div class="flex flex-row justify-center">
        <div class="loading loading-spinner loading-lg my-12"></div>
      </div>
    {:else}
      <div class="flex flex-col gap-2">
        <div class="flex flex-row items-center gap-4 mt-4 mb-2">
          <div class="flex-grow font-medium text-sm">
            {$_("generate.samples_modal.sample_count")}
          </div>
          <IncrementUi bind:value={num_samples_to_generate} />
        </div>
        <AvailableModelsDropdown
          requires_data_gen={true}
          bind:model
          suggested_mode="data_gen"
        />
        {#if cascade_mode}
          <!-- parallelization only makes sense in cascade mode -->
          <FormElement
            id="generate_samples_mode_element"
            inputType="select"
            info_description={$_("generate.run_mode_description")}
            select_options={[
              ["parallel", $_("generate.parallel_mode")],
              ["sequential", $_("generate.sequential_mode")],
            ]}
            bind:value={generate_samples_mode}
            label={$_("generate.run_mode")}
          />
        {/if}

        <!-- display errors after the generation has completed -->
        {#if topics_failed_to_generate_count > 0}
          {#if !cascade_mode}
            <div class="text-error font-light text-sm mt-4">
              {$_("generate.samples_modal.failed_to_generate", {
                values: {
                  topic: topics_failed_to_generate[0].topic.path.join(" → "),
                },
              })}
              <div>
                {topics_failed_to_generate[0].error?.getMessage()}
              </div>
            </div>
          {:else}
            <div class="text-error font-light text-sm mt-4">
              {$_("generate.samples_modal.topics_failed", {
                values: { count: topics_failed_to_generate_count },
              })}
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
              {#each topics_failed_to_generate as outcome}
                <div>
                  {outcome.topic.path.join(" → ")}: {outcome.error?.getMessage()}
                </div>
              {/each}
            </div>
          {/if}
        {/if}

        <button
          class="btn mt-6 {custom_topics_string ? '' : 'btn-primary'}"
          on:click={generate_samples}
        >
          {$_("generate.samples_modal.generate_samples", {
            values: { count: num_samples_to_generate },
          })}
          {#if cascade_mode}
            {$_("generate.samples_modal.for_each_topic")}
          {/if}
        </button>
      </div>
    {/if}
  </div>
  <form method="dialog" class="modal-backdrop">
    <button>{$_("common.close")}</button>
  </form>
</dialog>
