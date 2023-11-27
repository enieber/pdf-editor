<template>
  <div
    :class="['absolute w-full h-full cursor-grab',
      { 'cursor-grabbing operation': moveOperation }]"
    :style="{ width: `${width + data.dw}px`, height: `${height + data.dh}px`, transform: `translate(${x + data.dx}px, ${y + data.dy}px)` }"
  >
    <ObjectImage v-if="type == 'image'"
      @panstart="handlePanStart"
      @panmove="handlePanMove"
      @panend="handlePanEnd" />

    <ObjectSignature v-else-if="type == 'signature'"
      @panstart="handlePanStart"
      @panmove="handlePanMove"
      @panend="handlePanEnd" />
    <div
      @click="$emit('delete')"
      class="absolute left-0 top-0 right-0 w-12 h-12 m-auto rounded-full bg-white cursor-pointer transform -translate-y-1/2 md:scale-25"
    >
      <img class="w-full h-full" src="delete.svg" alt="delete object" />
    </div>
    <canvas v-if="type == 'image'" class="w-full h-full" ref="canvasImage" />
    <svg v-else-if="type == 'signature'" ref="signature" :viewBox="`0 0 ${width} ${height}`" width="100%" height="100%">
      <path stroke-width="5" stroke-linejoin="round" stroke-linecap="round" stroke="black" fill="none" :d="object.path" />
    </svg>
  </div>
</template>

<script>
import { reactive, ref, computed } from 'vue'
import ObjectImage from './objects/ObjectImage.vue'
import ObjectSignature from './objects/ObjectSignature.vue'

export default {
  name: 'ObjectContainer',
  components: {
    ObjectImage,
    ObjectSignature,
  },
  props: {
    payload: { required: true },
    x: { required: true },
    y: { required: true },
    file: { required: true },
    width: { required: true },
    height: { required: true },
    pageScale: { required: true },
    opacity: { required: true },
    type: { required: true },
    path: { required: false, default: null },

    object: {
      required: true,
      type: Object,
    },
  },
  data() {
    return {
      canvasImage: ref(),
      signature: ref(null),
      operation: ref(null),
      data: reactive({
        startX: null,
        startY: null,
        directions: [],
        dx: 0,
        dy: 0,
        dw: 0,
        dh: 0,
        pannableFunction: null,
      }),
      moveOperation: computed(() => {
        return this.operation === 'move'
      })
    }
  },
  mounted() {
    this.setCanvas()
  },
  methods: {
    setCanvas() {
      if (this.type == 'image') {
        // use canvas to prevent img tag's auto resize
        this.$refs.canvasImage.width = this.width
        this.$refs.canvasImage.height = this.height
        this.$refs.canvasImage.getContext('2d').drawImage(this.payload, 0, 0)

        let scale = 1
        const limit = 500
        if (this.width > limit) {
          scale = limit / this.width
        }
        if (this.height > limit) {
          scale = Math.min(scale, limit / this.height)
        }
        this.$emit('update', {
          width: this.width * scale,
          height: this.height * scale,
        })

        if (!['image/jpeg', 'image/png'].includes(this.file.type)) {
          this.$refs.canvasImage.toBlob((blob) => {
            this.$emit('update', {
              file: blob,
            })
          })
        }
      } else if (this.type == 'signature') {
        // signature.value.setAttribute("viewBox", `0 0 ${width} ${height}`);
      }
    },
    handlePanMove(event) {
      const _dx = (event.x - this.data.startX) / this.pageScale
      const _dy = (event.y - this.data.startY) / this.pageScale
      if (this.operation === 'move') {
        this.data.dx = _dx
        this.data.dy = _dy
      } else if (this.operation === 'scale') {
        if (this.data.directions.includes('left')) {
          this.data.dx = _dx
          this.data.dw = -_dx
        }
        if (this.data.directions.includes('top')) {
          this.data.dy = _dy
          this.data.dh = -_dy
        }
        if (this.data.directions.includes('right')) {
          this.data.dw = _dx
        }
        if (this.data.directions.includes('bottom')) {
          this.data.dh = _dy
        }
      }
    },
    handlePanEnd() {
      if (this.operation === 'move') {
        this.$emit('update', {
          x: this.x + this.data.dx,
          y: this.y + this.data.dy,
        })
        this.data.dx = 0
        this.data.dy = 0
      } else if (this.operation === 'scale') {
        this.$emit('update', {
          x: this.x + this.data.dx,
          y: this.y + this.data.dy,
          width: this.width + this.data.dw,
          height: this.height + this.data.dh,
        })

        this.data.dx = 0
        this.data.dy = 0
        this.data.dw = 0
        this.data.dh = 0
        this.data.directions = []
      }
      this.operation = ''
    },
    handlePanStart(event) {
      this.data.startX = event.x
      this.data.startY = event.y
      if (event.target === event.currentTarget) {
        return (this.operation = 'move')
      }
      this.operation = 'scale'
      this.data.directions = event.target.dataset.direction.split('-')
    },
  },
}

</script>

<style scoped>
  .operation {
    background-color: rgba(0, 0, 0, 0.3)
  }
</style>
