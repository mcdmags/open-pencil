<script setup lang="ts">
import { ref } from 'vue'

import { PropertyListRoot, useColorVariableBinding, useStrokeControls, useI18n } from '@open-pencil/vue'

import ColorStyleRow from '@/components/properties/ColorStyleRow.vue'
import AppSelect from '@/components/ui/AppSelect.vue'
import ColorInput from '@/components/ColorInput.vue'
import ScrubInput from '@/components/ScrubInput.vue'
import Tip from '@/components/ui/Tip.vue'
import { iconButton } from '@/components/ui/icon-button'
import { sectionLabel, sectionWrapper } from '@/components/ui/section'

import type { SceneNode, Stroke } from '@open-pencil/core'

const strokeCtx = useStrokeControls()
const strokeVarCtx = useColorVariableBinding('strokes')
const { panels } = useI18n()

const { store } = useNodeProps()
const { nodes, isMulti, active, activeNode, targetNodes, isArrayMixed, updateArrayItem, removeArrayItem, toggleArrayVisibility } = useMultiProps()

const strokesAreMixed = computed(() => isArrayMixed('strokes'))

const ALIGN_OPTIONS: { value: Stroke['align']; label: string }[] = [
  { value: 'INSIDE', label: 'Inside' },
  { value: 'CENTER', label: 'Center' },
  { value: 'OUTSIDE', label: 'Outside' }
]

const currentAlign = computed<Stroke['align']>(() => {
  const n = activeNode.value
  if (!n || n.strokes.length === 0) return 'CENTER'
  return n.strokes[0].align
})

const currentSides = computed<StrokeSides>(() => {
  const n = activeNode.value
  if (!n || !n.independentStrokeWeights) return 'ALL'
  const { borderTopWeight: t, borderRightWeight: r, borderBottomWeight: b, borderLeftWeight: l } = n
  const active = [t > 0, r > 0, b > 0, l > 0]
  const count = active.filter(Boolean).length
  if (count === 4 && t === r && r === b && b === l) return 'ALL'
  if (count === 1) {
    if (t > 0) return 'TOP'
    if (b > 0) return 'BOTTOM'
    if (l > 0) return 'LEFT'
    if (r > 0) return 'RIGHT'
  }
  return 'CUSTOM'
})

const hasStrokes = computed(
  () => !strokesAreMixed.value && (activeNode.value?.strokes?.length ?? 0) > 0
)

const sideMenuOpen = ref(false)

function updateColor(index: number, color: Color) {
  updateArrayItem('strokes', index, { color }, 'Change stroke')
}

function updateWeight(index: number, weight: number) {
  updateArrayItem('strokes', index, { weight }, 'Change stroke')
}

function updateOpacity(index: number, opacity: number) {
  updateArrayItem('strokes', index, { opacity: Math.max(0, Math.min(1, opacity / 100)) }, 'Change stroke')
}

function updateAlign(align: Stroke['align']) {
  for (const n of targetNodes()) {
    const strokes = n.strokes.map((s) => ({ ...s, align }))
    store.updateNodeWithUndo(n.id, { strokes }, 'Change stroke align')
  }
}

function toggleVisibility(index: number) {
  toggleArrayVisibility('strokes', index)
}

function add() {
  const stroke: Stroke = {
    color: { r: 0, g: 0, b: 0, a: 1 },
    weight: 1,
    opacity: 1,
    visible: true,
    align: 'CENTER'
  }
  for (const n of targetNodes()) {
    const strokes = isMulti.value ? [stroke] : [...n.strokes, stroke]
    store.updateNodeWithUndo(n.id, { strokes }, isMulti.value ? 'Set stroke' : 'Add stroke')
  }
}

function remove(index: number) {
  removeArrayItem('strokes', index, 'Remove stroke')
}

function selectSide(side: StrokeSides) {
  for (const n of targetNodes()) {
    const weight = n.strokes.length > 0 ? n.strokes[0].weight : 1
    if (side === 'ALL') {
      store.updateNodeWithUndo(
        n.id,
        {
          independentStrokeWeights: false,
          borderTopWeight: 0,
          borderRightWeight: 0,
          borderBottomWeight: 0,
          borderLeftWeight: 0
        } as Partial<SceneNode>,
        'Stroke all sides'
      )
    } else if (side === 'CUSTOM') {
      const current = n.independentStrokeWeights
        ? {
            top: n.borderTopWeight,
            right: n.borderRightWeight,
            bottom: n.borderBottomWeight,
            left: n.borderLeftWeight
          }
        : { top: weight, right: weight, bottom: weight, left: weight }
      store.updateNodeWithUndo(
        n.id,
        {
          independentStrokeWeights: true,
          borderTopWeight: current.top,
          borderRightWeight: current.right,
          borderBottomWeight: current.bottom,
          borderLeftWeight: current.left
        } as Partial<SceneNode>,
        'Custom stroke sides'
      )
    } else {
      store.updateNodeWithUndo(
        n.id,
        {
          independentStrokeWeights: true,
          borderTopWeight: side === 'TOP' ? weight : 0,
          borderRightWeight: side === 'RIGHT' ? weight : 0,
          borderBottomWeight: side === 'BOTTOM' ? weight : 0,
          borderLeftWeight: side === 'LEFT' ? weight : 0
        } as Partial<SceneNode>,
        `Stroke ${side.toLowerCase()} only`
      )
    }
  }
  sideMenuOpen.value = false
}

function updateBorderWeight(side: 'top' | 'right' | 'bottom' | 'left', value: number) {
  const fieldMap = {
    top: 'borderTopWeight',
    right: 'borderRightWeight',
    bottom: 'borderBottomWeight',
    left: 'borderLeftWeight'
  } as const
  for (const n of targetNodes()) {
    store.updateNodeWithUndo(
      n.id,
      { [fieldMap[side]]: value } as Partial<SceneNode>,
      'Change stroke weight'
    )
  }
}

const SIDE_OPTIONS: { value: StrokeSides; label: string }[] = [
  { value: 'ALL', label: 'All' },
  { value: 'TOP', label: 'Top' },
  { value: 'BOTTOM', label: 'Bottom' },
  { value: 'LEFT', label: 'Left' },
  { value: 'RIGHT', label: 'Right' },
  { value: 'CUSTOM', label: 'Custom' }
]

const BORDER_SIDES = ['top', 'right', 'bottom', 'left'] as const

const borderWeights = computed(() => {
  const n = activeNode.value
  return {
    top: n?.borderTopWeight ?? 0,
    right: n?.borderRightWeight ?? 0,
    bottom: n?.borderBottomWeight ?? 0,
    left: n?.borderLeftWeight ?? 0
  }
})
</script>

<template>
  <PropertyListRoot
    v-slot="{ items, isMixed, activeNode, add, remove, patch, toggleVisibility }"
    prop-key="strokes"
    :label="panels.stroke"
  >
    <div data-test-id="stroke-section" :class="sectionWrapper()">
      <div class="flex items-center justify-between">
        <label :class="sectionLabel()">{{ panels.stroke }}</label>
        <button
          data-test-id="stroke-section-add"
          :class="iconButton()"
          @click="add(strokeCtx.defaultStroke)"
        >
          +
        </button>
      </div>

      <p v-if="isMixed" class="text-[11px] text-muted">{{ panels.mixedStrokesHelp }}</p>

      <ColorStyleRow
        v-for="(stroke, i) in items as Stroke[]"
        :key="`${i}:${stroke.visible ? 'visible' : 'hidden'}`"
        :item="stroke"
        :index="i"
        :active-node-id="activeNode?.id ?? null"
        :binding-api="strokeVarCtx"
        :visibility-test-id="`stroke-visibility-${i}`"
        unbind-test-id="stroke-unbind-variable"
        data-test-id="stroke-item"
        :data-test-index="i"
        @patch="patch(i, $event)"
        @toggle-visibility="toggleVisibility(i)"
        @remove="remove(i)"
      >
        <ColorInput class="min-w-0 flex-1" :color="stroke.color" editable @update="patch(i, { color: $event })" />
      </ColorStyleRow>

      <div
        v-if="!isMixed && (items as unknown[]).length > 0"
        class="mt-1 flex items-center gap-1.5"
      >
        <AppSelect
          class="w-[72px]"
          :model-value="strokeCtx.currentAlign(activeNode)"
          :options="strokeCtx.alignOptions"
          @update:model-value="strokeCtx.updateAlign($event as Stroke['align'], activeNode!)"
        />
        <ScrubInput
          v-if="!expandedSides"
          class="flex-1"
          :model-value="activeNode!.strokes[0]?.weight ?? 1"
          :min="0"
          @update:model-value="patch(0, { weight: $event })"
        >
          <template #icon>
            <svg
              class="size-3"
              viewBox="0 0 12 12"
              fill="none"
              stroke="currentColor"
              stroke-width="1.5"
            >
              <line x1="1" y1="3" x2="11" y2="3" />
              <line x1="1" y1="6" x2="11" y2="6" />
              <line x1="1" y1="9" x2="11" y2="9" />
            </svg>
          </template>
        </ScrubInput>
        <Tip :label="panels.strokeSides">
          <button
            data-test-id="stroke-sides-toggle"
            class="flex size-[26px] shrink-0 cursor-pointer items-center justify-center rounded border border-border bg-input text-muted hover:bg-hover hover:text-surface"
            :class="{ '!border-accent !text-accent': expandedSides }"
            @click="onToggleSides(activeNode!)"
          >
            <svg class="size-3.5" viewBox="0 0 14 14" fill="currentColor">
              <rect x="1" y="1" width="5" height="5" rx="1" />
              <rect x="8" y="1" width="5" height="5" rx="1" />
              <rect x="1" y="8" width="5" height="5" rx="1" />
              <rect x="8" y="8" width="5" height="5" rx="1" />
            </svg>
          </button>
        </Tip>
      </div>

      <div
        v-if="!isMixed && (items as unknown[]).length > 0 && expandedSides"
        class="mt-1.5 grid grid-cols-2 gap-1.5"
      >
        <ScrubInput
          v-for="side in strokeCtx.borderSides"
          :key="side"
          :model-value="
            activeNode![
              `border${side[0].toUpperCase()}${side.slice(1)}Weight` as keyof SceneNode
            ] as number
          "
          :min="0"
          @update:model-value="strokeCtx.updateBorderWeight(side, $event, activeNode!)"
        >
          <template #icon>
            <svg class="size-3" viewBox="0 0 12 12" fill="none" stroke-width="1.5">
              <rect
                x="1"
                y="1"
                width="10"
                height="10"
                rx="1"
                stroke="currentColor"
                stroke-opacity="0.3"
                stroke-dasharray="2 2"
              />
              <line v-if="side === 'top'" x1="1" y1="1" x2="11" y2="1" stroke="currentColor" />
              <line
                v-else-if="side === 'right'"
                x1="11"
                y1="1"
                x2="11"
                y2="11"
                stroke="currentColor"
              />
              <line
                v-else-if="side === 'bottom'"
                x1="1"
                y1="11"
                x2="11"
                y2="11"
                stroke="currentColor"
              />
              <line v-else x1="1" y1="1" x2="1" y2="11" stroke="currentColor" />
            </svg>
          </template>
        </ScrubInput>
      </div>
    </div>
  </PropertyListRoot>
</template>
