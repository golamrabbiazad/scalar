<script setup lang="ts">
import { nanoid } from 'nanoid'
import { type Slot, computed, ref, watch } from 'vue'

import { ScalarIcon } from '../ScalarIcon'
import ComboboxOption from './ScalarComboboxOption.vue'
import { type Option, type OptionGroup, isGroups } from './types'

const props = defineProps<{
  options: Option[] | OptionGroup[]
  modelValue?: Option[]
  placeholder?: string
  open?: boolean
  multiselect?: boolean
  isDeletable?: boolean
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', v: Option[]): void
  (e: 'delete', option: Option): void
}>()

defineSlots<{
  /** A slot for contents before the combobox options */
  before(): Slot
  /** A slot for contents after the combobox options */
  after(): Slot
}>()

defineOptions({ inheritAttrs: false })

const id = nanoid()

/** A flat list of all options */
const options = computed<Option[]>(() =>
  isGroups(props.options)
    ? props.options.flatMap((group) => group.options)
    : props.options,
)

/** An list of all groups */
const groups = computed<OptionGroup[]>(() =>
  isGroups(props.options)
    ? props.options
    : [{ label: '', options: props.options }],
)

const query = ref<string>('')
const active = ref<Option>(props.modelValue?.[0] ?? options.value[0])

// Clear the query on open and close
watch(
  () => props.open,
  () => (query.value = ''),
)

// Set the top option as active when searching
watch(
  () => query.value,
  () => (active.value = filtered.value[0]),
)

const filtered = computed<Option[]>(() =>
  query.value === ''
    ? options.value
    : options.value.filter((option) => {
        return option.label.toLowerCase().includes(query.value.toLowerCase())
      }),
)

const selected = computed<Option[]>({
  get: () => props.modelValue ?? [],
  set: (o: Option[]) => o && emit('update:modelValue', o),
})

function toggleSelected(option: Option) {
  if (props.multiselect) {
    // Remove from selection list
    if (selected.value.some((o) => o.id === option.id))
      selected.value = selected.value.filter((o) => o.id !== option.id)
    // Add to selection list
    else selected.value = [...selected.value, option]
  } else {
    // Set selection for single select mode
    selected.value = [option]
  }
}

function moveActive(dir: 1 | -1) {
  const list = filtered.value

  // Find active index
  const activeIdx = list.findIndex((option) => option.id === active.value.id)

  // Calculate next index and exit if it's out of bounds
  const nextIdx = activeIdx + dir
  if (nextIdx < 0 || nextIdx > list.length - 1) return

  active.value = list[nextIdx]
}
</script>
<template>
  <div class="relative flex">
    <ScalarIcon
      class="pointer-events-none absolute left-3 top-1/2 -translate-y-1/2 text-c-3"
      icon="Search"
      size="sm" />
    <input
      v-model="query"
      aria-autocomplete="list"
      :aria-controls="id"
      class="min-w-0 flex-1 rounded-none border-0 py-2.5 pl-8 pr-3 leading-none text-c-1 outline-none"
      data-1p-ignore
      :placeholder="placeholder"
      role="combobox"
      tabindex="0"
      type="text"
      @keydown.down.prevent="moveActive(1)"
      @keydown.enter.prevent="toggleSelected(active)"
      @keydown.up.prevent="moveActive(-1)" />
  </div>
  <ul
    v-show="filtered.length || $slots.before || $slots.after"
    :id="id"
    class="border-t p-0.75">
    <slot name="before" />
    <template
      v-for="(group, i) in groups"
      :key="i">
      <div
        v-if="
          group.label &&
          // Only show the group label if there are some results
          group.options.some((o) => filtered.some((f) => f.id === o.id))
        "
        class="min-w-0 truncate px-2 py-1.5 text-left text-c-2">
        {{ group.label }}
      </div>
      <template
        v-for="option in filtered"
        :key="option.id">
        <ComboboxOption
          v-if="group.options.some((o) => o.id === option.id)"
          :active="active?.id === option.id"
          :isDeletable="option.isDeletable ?? isDeletable"
          :selected="selected.some((o) => o.id === option.id)"
          :style="multiselect ? 'checkbox' : 'radio'"
          @click="toggleSelected(option)"
          @delete="$emit('delete', option)"
          @mousedown.prevent
          @mouseenter="active = option">
          {{ option.label }}
        </ComboboxOption>
      </template>
    </template>
    <slot name="after" />
  </ul>
</template>
