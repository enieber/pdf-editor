<template>
  <PdfEditor pdf="./sample.pdf" />
</template>

<script>
import PdfEditor from './components/PdfEditor.vue'

export default {
  name: 'App',
  components: {
    PdfEditor
  }
}
</script>
