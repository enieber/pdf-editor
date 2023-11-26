<template>
  <div
    class="absolute left-0 top-0 select-none"
    :style="{ width: `${width + data.dw}px`, height: `${height + data.dh}px`, transform: `translate(${x + data.dx}px, ${y + data.dy}px)` }"
  >
    <ObjectImage v-if="type == 'image'"
      :operation="operation"
      @panstart="handlePanStart"
      @panmove="handlePanMove"
      @panend="handlePanEnd" />

    <ObjectSignature v-if="type == 'signature'"
      :operation="operation"
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

<script setup>
import { reactive, toRefs, ref, computed, onMounted } from 'vue'
import ObjectImage from './objects/ObjectImage.vue'
import ObjectSignature from './objects/ObjectSignature.vue'

const props = defineProps({
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
})

const emit = defineEmits(['update', 'delete'])

const canvasImage = ref()
const signature = ref(null)
const operation = ref(null)

const data = reactive({
  startX: null,
  startY: null,
  directions: [],
  dx: 0,
  dy: 0,
  dw: 0,
  dh: 0,
  pannableFunction: null,
})

const moveOperation = computed(() => {
  return operation.value === 'move'
})

onMounted(() => {
  setCanvas(props)
})

function setCanvas(props) {
  let { width, height } = props
  if (props.type == 'image') {
    // use canvas to prevent img tag's auto resize
    canvasImage.value.width = width
    canvasImage.value.height = height
    canvasImage.value.getContext('2d').drawImage(props.payload, 0, 0)

    let scale = 1
    const limit = 500
    if (width > limit) {
      scale = limit / width
    }
    if (height > limit) {
      scale = Math.min(scale, limit / height)
    }
    emit('update', {
      width: width * scale,
      height: height * scale,
    })

    if (!['image/jpeg', 'image/png'].includes(props.file.type)) {
      canvasImage.value.toBlob((blob) => {
        emit('update', {
          file: blob,
        })
      })
    }
  } else if (props.type == 'signature') {
    // signature.value.setAttribute("viewBox", `0 0 ${width} ${height}`);
  }
}

function handlePanMove(event) {
  const _dx = (event.x - data.startX) / props.pageScale
  const _dy = (event.y - data.startY) / props.pageScale
  if (operation.value === 'move') {
    data.dx = _dx
    data.dy = _dy
  } else if (operation.value === 'scale') {
    if (data.directions.includes('left')) {
      data.dx = _dx
      data.dw = -_dx
    }
    if (data.directions.includes('top')) {
      data.dy = _dy
      data.dh = -_dy
    }
    if (data.directions.includes('right')) {
      data.dw = _dx
    }
    if (data.directions.includes('bottom')) {
      data.dh = _dy
    }
  }
}

function handlePanEnd() {
  if (operation.value === 'move') {
    emit('update', {
      x: props.x + data.dx,
      y: props.y + data.dy,
    })
    data.dx = 0
    data.dy = 0
  } else if (operation.value === 'scale') {
    emit('update', {
      x: props.x + data.dx,
      y: props.y + data.dy,
      width: props.width + data.dw,
      height: props.height + data.dh,
    })

    data.dx = 0
    data.dy = 0
    data.dw = 0
    data.dh = 0
    data.directions = []
  }
  operation.value = ''
}

function handlePanStart(event) {
  data.startX = event.x
  data.startY = event.y
  if (event.target === event.currentTarget) {
    return (operation.value = 'move')
  }
  operation.value = 'scale'
  data.directions = event.target.dataset.direction.split('-')
}
</script>
