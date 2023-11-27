<template>
  <div>
    <div use:pannable
      @panstart="handlePanStart"
      @panmove="handlePanMove"
      @panend="handlePanEnd" >
      <div data-direction="left-top"
        class="absolute left-0 top-0 w-10 h-10 bg-green-400 rounded-full
        cursor-nwse-resize transform -translate-x-1/2 -translate-y-1/2 md:scale-25" />
      <div data-direction="right-bottom"
        class="absolute right-0 bottom-0 w-10 h-10 bg-green-400 rounded-full
        cursor-nwse-resize transform translate-x-1/2 translate-y-1/2 md:scale-25" />
    </div>
    <svg v-bind="svg" width="100%" height="100%">
      <path
        stroke-width="5"
        stroke-linejoin="round"
        stroke-linecap="round"
        stroke="black"
        fill="none"
        :d="path" />
    </svg>
  </div>
</template>

<script>
import { ref, computed } from 'vue'
// import { onMount, createEventDispatcher } from 'svelte'
import { pannable } from '../utils/pannable.js'
// import { readAsArrayBuffer } from './utils/asyncReader.js'

export default {
  name: 'ObjectSignature',
  components: {
    pannable,
  },
  props: {
    operation: { required: true }
  },
  data() {
    return {
      x: ref(0),
      y: ref(0),
      path: ref(''),
      svg: {},
      ratio: ref(1),
    }
  },
  mount() {
    this.render()
  },
  methods: {
    async render() {
      svg.setAttribute('viewBox', `0 0 ${originWidth} ${originHeight}`)
    },
    onDelete() {
      dispatch('delete')
    },
  }
}
</script>
