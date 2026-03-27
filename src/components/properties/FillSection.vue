<script setup lang="ts">
import { colorToHexRaw } from '@open-pencil/core'
import { PropertyListRoot, useFillControls, useI18n } from '@open-pencil/vue'

import FillPicker from '@/components/FillPicker.vue'
import ColorStyleRow from '@/components/properties/ColorStyleRow.vue'
import { iconButton } from '@/components/ui/icon-button'
import { sectionLabel, sectionWrapper } from '@/components/ui/section'

const { store } = useNodeProps()
const { nodes, isMulti, active, activeNode, targetNodes, isArrayMixed, updateArrayItem, removeArrayItem, toggleArrayVisibility } = useMultiProps()

const fillsAreMixed = computed(() => isArrayMixed('fills'))

const colorVariables = computed(() => store.graph.getVariablesByType('COLOR'))

function getBoundVariable(index: number): Variable | undefined {
  const n = activeNode.value
  if (!n) return undefined
  const varId = n.boundVariables[`fills/${index}/color`]
  return varId ? store.graph.variables.get(varId) : undefined
}

function bindVariable(index: number, variableId: string) {
  const n = activeNode.value
  if (!n) return
  store.graph.bindVariable(n.id, `fills/${index}/color`, variableId)
  store.requestRender()
}

function unbindVariable(index: number) {
  const n = activeNode.value
  if (!n) return
  store.graph.unbindVariable(n.id, `fills/${index}/color`)
  store.requestRender()
}

function resolvedSwatchStyle(variable: Variable): string {
  const color = store.graph.resolveColorVariable(variable.id)
  if (!color) return 'background: #000'
  return `background: ${colorToCSS(color)}`
}

function updateFill(index: number, fill: Fill) {
  updateArrayItem('fills', index, fill, 'Change fill')
}

function updateOpacity(index: number, opacity: number) {
  updateArrayItem('fills', index, { opacity: Math.max(0, Math.min(1, opacity / 100)) }, 'Change fill')
}

function toggleVisibility(index: number) {
  toggleArrayVisibility('fills', index)
}

function add() {
  for (const n of targetNodes()) {
    const fills = isMulti.value ? [{ ...DEFAULT_SHAPE_FILL }] : [...n.fills, { ...DEFAULT_SHAPE_FILL }]
    store.updateNodeWithUndo(n.id, { fills }, isMulti.value ? 'Set fill' : 'Add fill')
  }
}

function remove(index: number) {
  removeArrayItem('fills', index, 'Remove fill')
}

const searchTerm = ref('')
const { contains } = useFilter({ sensitivity: 'base' })
const filteredVariables = computed(() => {
  if (!searchTerm.value) return colorVariables.value
  return colorVariables.value.filter((v) => contains(v.name, searchTerm.value))
})
</script>

<template>
  <PropertyListRoot
    v-slot="{ items, isMixed, activeNode, add, remove, update, patch, toggleVisibility }"
    prop-key="fills"
    :label="panels.fill"
  >
    <div data-test-id="fill-section" :class="sectionWrapper()">
      <div class="flex items-center justify-between">
        <label :class="sectionLabel()">{{ panels.fill }}</label>
        <button
          data-test-id="fill-section-add"
          :class="iconButton()"
          @click="add({ ...fillCtx.defaultFill })"
        >
          +
        </button>
      </div>
      <p v-if="isMixed" class="text-[11px] text-muted">{{ panels.mixedFillsHelp }}</p>
      <ColorStyleRow
        v-for="(fill, i) in items as Fill[]"
        :key="`${i}:${fill.visible ? 'visible' : 'hidden'}`"
        :item="fill"
        :index="i"
        :active-node-id="activeNode?.id ?? null"
        :binding-api="fillCtx"
        :visibility-test-id="`fill-visibility-${i}`"
        unbind-test-id="fill-unbind-variable"
        data-test-id="fill-item"
        :data-test-index="i"
        @patch="patch(i, $event)"
        @toggle-visibility="toggleVisibility(i)"
        @remove="remove(i)"
      >
        <FillPicker :fill="fill" @update="update(i, $event)" />

        <template v-if="activeNode && fillCtx.getBoundVariable(activeNode.id, i)">
          <span
            class="min-w-0 flex-1 truncate rounded bg-violet-500/10 px-1 font-mono text-xs text-violet-400"
          >
            {{ fillCtx.getBoundVariable(activeNode.id, i)!.name }}
          </span>
        </template>
        <template v-else>
          <span class="min-w-0 flex-1 font-mono text-xs text-surface">
            <template v-if="fill.type === 'SOLID'">{{ colorToHexRaw(fill.color) }}</template>
            <template v-else-if="fill.type.startsWith('GRADIENT')">{{
              fill.type.replace('GRADIENT_', '')
            }}</template>
            <template v-else>{{ fill.type }}</template>
          </span>
        </template>
      </ColorStyleRow>
    </div>
  </PropertyListRoot>
</template>
