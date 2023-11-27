<template>
  <div ref="signatureContainer"
    @mousedown.passive="handleMousedown"
    @touchstart.passive="handleTouchStart"
    :class="['absolute w-full h-full cursor-grab',
      { 'cursor-grabbing operation': moveOperation }]">
    <div data-direction="left"
      class="resize-border h-full w-1 left-0 top-0 border-l cursor-ew-resize" />
    <div data-direction="top"
      class="resize-border w-full h-1 left-0 top-0 border-t cursor-ns-resize" />
    <div data-direction="bottom"
      class="resize-border w-full h-1 left-0 bottom-0 border-b cursor-ns-resize" />
    <div data-direction="right"
      class="resize-border h-full w-1 right-0 top-0 border-r cursor-ew-resize" />
    <div data-direction="left-top"
      class="resize-corner left-0 top-0 cursor-nwse-resize transform
        -translate-x-1/2 -translate-y-1/2 md:scale-25" />
    <div data-direction="right-top"
      class="resize-corner right-0 top-0 cursor-nesw-resize transform
        translate-x-1/2 -translate-y-1/2 md:scale-25" />
    <div data-direction="left-bottom"
      class="resize-corner left-0 bottom-0 cursor-nesw-resize transform
        -translate-x-1/2 translate-y-1/2 md:scale-25" />
    <div data-direction="right-bottom"
      class="resize-corner right-0 bottom-0 cursor-nwse-resize transform
      translate-x-1/2 translate-y-1/2 md:scale-25" />
  </div>
</template>

<script>
import { reactive, computed } from "vue"

export default {
  name: 'ObjectImage',
  props: {
    operation: { required: true }
  },
  data() {
    return {
      data: reactive({
        x: 0,
        y: 0,
      }),
      moveOperation: computed(() => this.operation === 'move')
    }
  },
  mounted() {
    this.setupEvent()
  },
  beforeUnmount() {
    this.$refs.signatureContainer.removeEventListener('mousedown', this.handleMousedown)
    this.$refs.signatureContainer.removeEventListener('touchstart', this.handleTouchStart)
  },
  methods: {
    setupEvent() {
      this.$refs.signatureContainer.addEventListener('mousedown', this.handleMousedown)
      this.$refs.signatureContainer.addEventListener('touchstart', this.handleTouchStart)
    },
    handleMousedown(event) {
      this.data.x = event.clientX
      this.data.y = event.clientY
      const target = event.target
      this.$emit('panstart', {
        x: this.data.x,
        y: this.data.y,
        target,
        currentTarget: this.$refs.signatureContainer
      })
      this.$refs.signatureContainer.addEventListener('mousemove', this.handleMousemove)
      this.$refs.signatureContainer.addEventListener('mouseup', this.handleMouseup)
    },
    handleMousemove(event) {
      const dx = event.clientX - this.data.x
      const dy = event.clientY - this.data.y
      this.data.x = event.clientX
      this.data.y = event.clientY

      this.$emit('panmove', {
        x: this.data.x,
        y: this.data.y,
        dx,
        dy
      })
    },
    handleMouseup(event) {
      this.data.x = event.clientX
      this.data.y = event.clientY

      this.$emit('panend', { x: this.data.x, y: this.data.y })

      this.$refs.signatureContainer.removeEventListener('mousemove', this.handleMousemove)
      this.$refs.signatureContainer.removeEventListener('mouseup', this.handleMouseup)
    },
    handleTouchStart(event) {
      if (event.touches.length > 1) return
      const touch = event.touches[0]
      this.data.x = touch.clientX
      this.data.y = touch.clientY
      const target = touch.target

      this.$emit('panstart', { x: this.data.x, y: this.data.y, target })
      this.$refs.signatureContainer.addEventListener('touchmove', this.handleTouchmove)
      this.$refs.signatureContainer.addEventListener('touchend', this.handleTouchend)
    },
    handleTouchmove(event) {
      event.preventDefault()
      if (event.touches.length > 1) return
      const touch = event.touches[0]
      const dx = touch.clientX - this.data.x
      const dy = touch.clientY - this.data.y
      this.data.x = touch.clientX
      this.data.y = touch.clientY

      this.$emit('panmove', { x: this.data.x, y: this.data.y, dx, dy })
    },
    handleTouchend(event) {
      const touch = event.changedTouches[0]
      this.data.x = touch.clientX
      this.data.y = touch.clientY

      this.$emit('panend', { x: this.data.x, y: this.data.y })
      this.$refs.signatureContainer.removeEventListener('touchmove', this.handleTouchmove)
      this.$refs.signatureContainer.removeEventListener('touchend', this.handleTouchend)
    },
  },
}
</script>

<style scoped>
  .operation {
    background-color: rgba(0, 0, 0, 0.3)
  }
  .resize-border {
    position: absolute;
    border-style: dashed;
    --border-opacity: 1;
    border-color: rgba(113, 128, 150, var(--border-opacity));
  }
  .resize-corner {
    position: absolute;
    width: 2.5rem;
    height: 2.5rem;
    --bg-opacity: 1;
    background-color: rgba(144, 205, 244, var(--bg-opacity));
    border-radius: 9999px;
  }
</style>
