<template>
  <div style="height: 50%" class="fixed z-10 top-0 left-0 right-0 border-b border-gray-300 bg-white shadow-lg">
    <div
      ref="signatureCanvas"
      @panstart="handlePanStart"
      @panmove="handlePanMove"
      @panend="handlePanEnd"
      class="relative w-full h-full select-none"
    >
      <div class="absolute right-0 bottom-0 mr-4 mb-4 flex">
        <button @click="cancel" class="w-24 bg-red-500 hover:bg-red-700 text-white font-bold py-1 px-4 rounded mr-4">Cancel</button>
        <button @click="finish" class="w-24 bg-blue-600 hover:bg-blue-700 text-white font-bold py-1 px-4 rounded">Done</button>
      </div>
      <svg class="w-full h-full pointer-events-none">
        <path stroke-width="5" stroke-linejoin="round" stroke-linecap="round" :d="path" stroke="black" fill="none" />
      </svg>
    </div>
  </div>
</template>

<script>
import { reactive, ref } from 'vue'

export default {
  name: 'SignatureCanvas',
  data() {
    return {
      paths: ref([]),
      path: ref(''),
      data: reactive({
        drawing: false,
        x: 0,
        y: 0,
        minX: Infinity,
        minY: Infinity,
        maxX: 0,
        maxY: 0,
      }),
    }
  },
  mounted() {
    this.setupEvent()
  },
  beforeUnmount() {
    this.$refs.signatureCanvas.removeEventListener('mousedown', this.handleMousedown)
    this.$refs.signatureCanvas.removeEventListener('touchstart', this.handleTouchStart)
  },
  methods: {
    setupEvent() {
      this.$refs.signatureCanvas.addEventListener('mousedown', this.handleMousedown)
      this.$refs.signatureCanvas.addEventListener('touchstart', this.handleTouchStart)
    },
    handleMousedown(event) {
      this.data.x = event.clientX
      this.data.y = event.clientY
      const target = event.target
      // emit('panstart', {
      //     x: this.data.x,
      //     y: this.data.y,
      //     target,
      //     currentTarget: this.$refs.signatureCanvas
      // })
      this.handlePanStart({
        x: this.data.x,
        y: this.data.y,
        target,
        currentTarget: this.$refs.signatureCanvas,
      })
      this.$refs.signatureCanvas.addEventListener('mousemove', this.handleMousemove)
      this.$refs.signatureCanvas.addEventListener('mouseup', this.handleMouseup)
    },
    handleMousemove(event) {
      const dx = event.clientX - this.data.x
      const dy = event.clientY - this.data.y
      this.data.x = event.clientX
      this.data.y = event.clientY

      // emit('panmove', {
      //     x: this.data.x,
      //     y: this.data.y,
      //     dx,
      //     dy
      // })
      this.handlePanMove({
        x: this.data.x,
        y: this.data.y,
        dx,
        dy,
      })
    },
    handleMouseup(event) {
      this.data.x = event.clientX
      this.data.y = event.clientY

      // emit('panend', { x: this.data.x, y: this.data.y })
      this.handlePanEnd({ x: this.data.x, y: this.data.y })
      this.$refs.signatureCanvas.removeEventListener('mousemove', this.handleMousemove)
      this.$refs.signatureCanvas.removeEventListener('mouseup', this.handleMouseup)
    },
    handleTouchStart(event) {
      if (event.touches.length > 1) return
      const touch = event.touches[0]
      this.data.x = touch.clientX
      this.data.y = touch.clientY
      const target = touch.target

      // emit('panstart', { x: this.data.x, y: this.data.y, target })
      this.handlePanStart({ x: this.data.x, y: this.data.y, target })

      this.$refs.signatureCanvas.addEventListener('touchmove', this.handleTouchmove) // { passive: false }
      this.$refs.signatureCanvas.addEventListener('touchend', this.handleTouchend)
    },
    handleTouchmove(event) {
      event.preventDefault()
      if (event.touches.length > 1) return
      const touch = event.touches[0]
      const dx = touch.clientX - this.data.x
      const dy = touch.clientY - this.data.y
      this.data.x = touch.clientX
      this.data.y = touch.clientY

      // emit('panmove', { x: this.data.x, y: this.data.y, dx, dy })
      this.handlePanMove({ x: this.data.x, y: this.data.y, dx, dy })
    },
    handleTouchend(event) {
      const touch = event.changedTouches[0]
      this.data.x = touch.clientX
      this.data.y = touch.clientY

      // emit('panend', { x: this.data.x, y: this.data.y })
      this.handlePanEnd({ x: this.data.x, y: this.data.y })
      this.$refs.signatureCanvas.removeEventListener('touchmove', this.handleTouchmove)
      this.$refs.signatureCanvas.removeEventListener('touchend', this.handleTouchend)
    },
    handlePanStart(event) {
      if (event.target !== event.currentTarget) {
        this.data.drawing = false
        return
      }

      this.data.drawing = true
      this.data.x = event.x
      this.data.y = event.y
      this.data.minX = Math.min(this.data.minX, this.data.x)
      this.data.maxX = Math.max(this.data.maxX, this.data.x)
      this.data.minY = Math.min(this.data.minY, this.data.y)
      this.data.maxY = Math.max(this.data.maxY, this.data.y)
      this.paths.push(['M', this.data.x, this.data.y])
      this.path += `M${this.data.x},${this.data.y}`
    },
    handlePanMove(event) {
      if (!this.data.drawing) return

      this.data.x = event.x
      this.data.y = event.y
      this.data.minX = Math.min(this.data.minX, this.data.x)
      this.data.maxX = Math.max(this.data.maxX, this.data.x)
      this.data.minY = Math.min(this.data.minY, this.data.y)
      this.data.maxY = Math.max(this.data.maxY, this.data.y)
      this.path += `L${this.data.x},${this.data.y}`

      this.paths.push(['L', this.data.x, this.data.y])
    },
    handlePanEnd() {
      this.data.drawing = false
    },
    finish() {
      if (!this.paths.length) return
      const dx = -(this.data.minX - 10)
      const dy = -(this.data.minY - 10)
      const originWidth = this.data.maxX - this.data.minX + 20
      const originHeight = this.data.maxY - this.data.minY + 20

      let scale = 1
      if (originWidth > 500) {
        scale = 500 / originWidth
      }

      this.$emit('finish', {
        width: originWidth,
        height: originHeight,
        path: this.paths.reduce((acc, cur) => {
          return acc + cur[0] + (cur[1] + dx) + ',' + (cur[2] + dy)
        }, ''),
        scale
      })
    },
    cancel() {
      this.$emit('cancel')
    }
  }
}
</script>
