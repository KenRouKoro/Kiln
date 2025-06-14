<script lang="ts">
  import { _ } from "svelte-i18n"
  import JsonSchemaFormElement from "$lib/utils/json_schema_editor/json_schema_form_element.svelte"
  import {
    type SchemaModel,
    example_schema_model,
    model_from_schema,
  } from "$lib/utils/json_schema_editor/json_schema_templates"

  let id = Math.random().toString(36)

  // Not the best svelte bindings here.
  // We're reactive for setting the schema_string, which allows the caller to set a well known schema (like the demo/example)
  // It's not reactive when the user starts editing (the string stays static), as we need a more complex in memory view-model. Access the new string via the accessor below.
  export let schema_string: string | null = null
  let schema_model: SchemaModel = schema_model_from_string(schema_string)
  $: schema_model = schema_model_from_string(schema_string)
  let plaintext: boolean = !schema_string
  $: plaintext = !schema_string

  // Keep the copy the raw state here, so it persists even if they toggle the radio back to plaintext, and the component is removed.
  let raw = false
  let raw_schema_string: string = ""

  // Update our live VM from the schema string
  function schema_model_from_string(
    new_schema_string: string | null,
  ): SchemaModel {
    if (new_schema_string) {
      return model_from_schema(JSON.parse(new_schema_string))
    } else {
      return example_schema_model()
    }
  }

  let schema_form_element: JsonSchemaFormElement | null = null

  export function get_schema_string(): string | null {
    if (plaintext) {
      return null
    }
    return schema_form_element?.get_schema_string() || null
  }
</script>

<div>
  <div class="form-control">
    <label class="label cursor-pointer flex flex-row gap-3">
      <input
        type="radio"
        name="radio-input-schema-{id}"
        class="radio"
        value={true}
        bind:group={plaintext}
      />
      <span class="label-text text-left grow"
        >{$_("task.schema_section.plain_text")}</span
      >
    </label>
  </div>
  <div class="form-control">
    <label class="label cursor-pointer flex flex-row gap-3">
      <input
        type="radio"
        name="radio-input-schema-{id}"
        class="radio"
        value={false}
        bind:group={plaintext}
      />
      <span class="label-text text-left grow"
        >{$_("task.schema_section.structured_json")}</span
      >
    </label>
  </div>

  {#if !plaintext}
    <JsonSchemaFormElement
      bind:this={schema_form_element}
      {schema_model}
      bind:raw_schema={raw_schema_string}
      bind:raw
    />
  {/if}
</div>
