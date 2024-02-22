<script setup>
import { mapProps } from '@/mixins'
import { computed, nextTick, onBeforeMount, reactive, ref, watch } from 'vue'
import debounce from 'lodash/debounce'
import { useEventListener } from '@vueuse/core'
import get from 'lodash/get'
import findIndex from 'lodash/findIndex'
import { createPopper } from '@popperjs/core'

const container = ref(null)
const dropdown = ref(null)
const input = ref(null)
const search = ref(null)
const selectedOption = ref(null)

const emit = defineEmits(['clear', 'input', 'shown', 'closed', 'selected'])

defineExpose({ open, close, clear })

defineOptions({ inheritAttrs: false })

const props = defineProps({
  dusk: {},
  disabled: { type: Boolean, default: false },
  readOnly: { type: Boolean, default: false },
  value: {},
  data: {},
  trackBy: {},
  error: { type: Boolean, default: false },
  boundary: {},
  debounce: { type: Number, default: 500 },
  clearable: { type: Boolean, default: true },
  ...mapProps(['mode']),
})

const data = reactive({
  debouncer: null,
  show: false,
  search: '',
  selectedOptionIndex: 0,
  popper: null,
  inputWidth: null,
})

onBeforeMount(() => {
  data.debouncer = debounce(callback => callback(), props.debounce)
})

useEventListener(document, 'keydown', e => {
  // 'tab' or 'escape'
  if (data.show && (e.keyCode === 9 || e.keyCode === 27)) {
    setTimeout(() => close(), 50)
  }
})

watch(
  () => data.search,
  search => {
    data.selectedOptionIndex = 0

    if (container.value) {
      container.value.scrollTop = 0
    } else {
      nextTick(() => (container.value.scrollTop = 0))
    }

    data.debouncer(() => emit('input', search))
  }
)

watch(
  () => data.show,
  show => {
    if (show) {
      let selected = findIndex(props.data, [
        props.trackBy,
        get(props.value, props.trackBy),
      ])
      if (selected !== -1) props.selected = selected
      data.inputWidth = input.value.offsetWidth

      nextTick(() => {
        data.popper = createPopper(input.value, dropdown.value, {
          placement: 'bottom-start',
          onFirstUpdate: state => {
            container.value.scrollTop = container.value.scrollHeight
            updateScrollPosition()
            search.value.focus()
          },
        })
      })
    } else {
      if (data.popper) data.popper.destroy()
    }
  }
)

function getTrackedByKey(option) {
  return get(option, props.trackBy)
}

function open() {
  if (!props.disabled && !props.readOnly) {
    data.show = true
    data.search = ''
    emit('shown')
  }
}

function close() {
  data.show = false
  emit('closed')
}

function clear() {
  if (!props.disabled) {
    data.selectedOptionIndex = null
    emit('clear')
  }
}

function move(offset) {
  let newIndex = data.selectedOptionIndex + offset

  if (newIndex >= 0 && newIndex < props.data.length) {
    data.selectedOptionIndex = newIndex
    updateScrollPosition()
  }
}

function updateScrollPosition() {
  nextTick(() => {
    if (selectedOption.value) {
      if (
        selectedOption.value.offsetTop >
        container.value.scrollTop +
          container.value.clientHeight -
          selectedOption.value.clientHeight
      ) {
        container.value.scrollTop =
          selectedOption.value.offsetTop +
          selectedOption.value.clientHeight -
          container.value.clientHeight
      }

      if (selectedOption.value.offsetTop < container.value.scrollTop) {
        container.value.scrollTop = selectedOption.value.offsetTop
      }
    }
  })
}

function chooseSelected(event) {
  if (event.isComposing || event.keyCode === 229) return

  if (props.data[data.selectedOptionIndex] !== undefined) {
    emit('selected', props.data[data.selectedOptionIndex])
    input.value.focus()
    nextTick(() => close())
  }
}

function choose(option) {
  data.selectedOptionIndex = findIndex(props.data, [
    props.trackBy,
    get(option, props.trackBy),
  ])
  emit('selected', option)
  input.value.focus()
  nextTick(() => close())
}

function setSelectedRef(index, el) {
  if (data.selectedOptionIndex === index) {
    selectedOption.value = el
  }
}

const shouldShowDropdownArrow = computed(() => {
  return props.value === '' || props.value == null || !props.clearable
})
</script>

<template>
  <div v-bind="$attrs" class="relative" :dusk="dusk" ref="searchInputContainer">
    <div
      ref="input"
      @click.stop="open"
      @focus="open"
      @keydown.down.prevent="open"
      @keydown.up.prevent="open"
      :class="{
        'ring dark:border-gray-500 dark:ring-gray-700': data.show,
        'form-input-border-error': error,
        'bg-gray-50 dark:bg-gray-700': disabled || readOnly,
      }"
      class="relative flex items-center form-control form-input-bordered form-select pr-6"
      :tabindex="data.show ? -1 : 0"
      :aria-expanded="data.show === true ? 'true' : 'false'"
      :dusk="`${dusk}-selected`"
    >
      <IconArrow
        v-if="shouldShowDropdownArrow && !disabled"
        class="pointer-events-none form-select-arrow"
      />

      <slot name="default">
        <div class="text-gray-400 dark:text-gray-400">
          {{ __('Click to choose') }}
        </div>
      </slot>
    </div>

    <button
      type="button"
      @click="clear"
      v-if="!shouldShowDropdownArrow && !disabled"
      tabindex="-1"
      class="absolute p-2 inline-block right-[4px]"
      style="top: 6px"
      :dusk="`${dusk}-clear-button`"
    >
      <svg
        class="block fill-current icon h-2 w-2"
        xmlns="http://www.w3.org/2000/svg"
        viewBox="278.046 126.846 235.908 235.908"
      >
        <path
          d="M506.784 134.017c-9.56-9.56-25.06-9.56-34.62 0L396 210.18l-76.164-76.164c-9.56-9.56-25.06-9.56-34.62 0-9.56 9.56-9.56 25.06 0 34.62L361.38 244.8l-76.164 76.165c-9.56 9.56-9.56 25.06 0 34.62 9.56 9.56 25.06 9.56 34.62 0L396 279.42l76.164 76.165c9.56 9.56 25.06 9.56 34.62 0 9.56-9.56 9.56-25.06 0-34.62L430.62 244.8l76.164-76.163c9.56-9.56 9.56-25.06 0-34.62z"
        />
      </svg>
    </button>
  </div>

  <teleport to="body">
    <div
      v-if="data.show"
      ref="dropdown"
      class="rounded-lg px-0 bg-white dark:bg-gray-900 shadow border border-gray-200 dark:border-gray-700 absolute top-0 left-0 my-1 overflow-hidden"
      :style="{ width: data.inputWidth + 'px', zIndex: 2000 }"
      :dusk="`${dusk}-dropdown`"
    >
      <!-- Search Input -->
      <input
        :disabled="disabled || readOnly"
        v-model="data.search"
        ref="search"
        @keydown.enter.prevent="chooseSelected"
        @keydown.down.prevent="move(1)"
        @keydown.up.prevent="move(-1)"
        class="h-10 outline-none w-full px-3 text-sm leading-normal bg-white dark:bg-gray-700 rounded-t border-b border-gray-200 dark:border-gray-800"
        tabindex="-1"
        type="search"
        :placeholder="__('Search')"
        spellcheck="false"
      />

      <!-- Search Results -->
      <div
        ref="container"
        class="relative overflow-y-scroll text-sm"
        tabindex="-1"
        style="max-height: 155px"
        :dusk="`${dusk}-results`"
      >
        <div
          v-for="(option, index) in props.data"
          :dusk="`${dusk}-result-${index}`"
          :key="getTrackedByKey(option)"
          :ref="el => setSelectedRef(index, el)"
          @click.stop="choose(option)"
          class="px-3 py-1.5 cursor-pointer z-[50]"
          :class="{
            'border-t border-gray-100 dark:border-gray-700': index !== 0,
            [`search-input-item-${index}`]: true,
            'hover:bg-gray-100 dark:hover:bg-gray-800':
              index !== data.selectedOptionIndex,
            'bg-primary-500 text-white dark:text-gray-900':
              index === data.selectedOptionIndex,
          }"
        >
          <slot
            name="option"
            :option="option"
            :selected="index === data.selectedOptionIndex"
          />
        </div>
      </div>
    </div>

    <Backdrop @click="close" :show="data.show" class="z-[30]" />
  </teleport>
</template>
