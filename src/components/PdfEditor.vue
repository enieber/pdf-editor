<template>
  <div class="flex flex-col items-center py-16 bg-gray-100 min-h-screen">
    <!-- Signature Canvas -->
    <SignatureCanvas v-if="signatureCanvas.isShow" @finish="pasteSignature" @cancel="signatureCanvas.isShow = false" />
    <!-- END -->
    <div class="fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center bg-gray-200 border-b border-gray-300">
      <div class="relative mr-3 flex h-8 bg-gray-400 rounded-sm overflow-hidden md:mr-4">
        <label for="image" class="flex items-center justify-center h-full w-8 hover:bg-gray-500 cursor-pointer">
          <input type="file" id="image" name="image" class="hidden" @change="uploadImage" />
          <img src="image.svg" alt="An icon for adding images" />
        </label>
        <button
          @click="addSignature"
          class="flex items-center justify-center h-full w-8 hover:bg-gray-500 cursor-pointer"
        >
          <img src="gesture.svg" alt="An icon for adding drawing" />
        </button>
      </div>
      <button
        class="w-20 bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-3 md:px-4 mr-3 md:mr-4 rounded"
        @click="download"
      >
        Save
      </button>
    </div>
    <div class="w-full overflow-auto">
      <div
        v-for="(page, pageIndex) in pages"
        :key="pageIndex"
        class="p-5 w-full flex flex-col items-center overflow-hidden"
        @mousedown="() => selectPage(pageIndex)"
        @touchstart="() => selectPage(pageIndex)"
      >
        <div :class="['relative shadow-lg mb-4', { 'selected-pdf': pageIndex == selectedPageIndex }]">
          <PdfPage :page="pages[pageIndex]" @measure="(payload) => onMeasure(payload, pageIndex)" />
          <div
            class="absolute top-0 left-0 transform origin-top-left"
            :style="{ transform: `scale(${pagesScale[pageIndex]})`, touchAction: 'none' }"
          >
            <div v-for="(object, objectIndex) in allObjects"
              :key="objectIndex">
              <ObjectContainer
                v-if="object.page == pageIndex"
                @update="(payload) => updateObject(object.id, payload)"
                @delete="() => deleteObject(object.id)"
                :file="object.file"
                :payload="object.payload"
                :x="object.x"
                :y="object.y"
                :width="object.width"
                :height="object.height"
                :opacity="opacity"
                :pageScale="pagesScale[pageIndex]"
                :type="object.type"
                :path="object.type"
                :object="object"
              />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { reactive, ref } from 'vue'
import { readAsImage, readAsPDF, readAsDataURL } from './utils/asyncReader'
import prepareAssets, { fetchFont } from './utils/prepareAssets'
import PdfPage from './PdfPage.vue'
import ObjectContainer from './ObjectContainer.vue'
import SignatureCanvas from './SignatureCanvas.vue'
import { save } from './utils/PDF'
import { ggID } from './utils/helper'
// import { dataType64toFile } from './utils/image'

export default {
  name: 'PdfEditor',
  components: {
    PdfPage,
    ObjectContainer,
    SignatureCanvas,
  },
  props: {
    pdf: {
      required: true,
      type: String,
    },
  },
  data() {
    return {
      pages: ref([]),
      allObjects: ref([]),
      pagesScale: ref([]),
      selectedPageIndex: ref(0),
      signatureCanvas: reactive({
        isShow: false,
      }),
      opacity: ref(1),
      pdfFile: ref(null),
      sequence: ref(0),
      genID: ggID(),
    }
  },
  mounted() {
    this.mountPdf()
  },
  methods: {
    async mountPdf() {
      try {
        const res = await fetch(this.pdf)
        const pdfBlob = await res.blob()
        await this.addPDF(pdfBlob)

        this.selectedPageIndex = 0

        setTimeout(() => {
          fetchFont('Times-Roman')
          prepareAssets()
        }, 5000)
      } catch (e) {
        // eslint-disable-next-line no-console
        console.log('Error:', e)
      }
    },
    async addPDF(file) {
      try {
        const pdf = await readAsPDF(file)
        this.pdfFile = file
        const numPages = pdf.numPages

        this.pages.splice(0, this.pages.length)
        this.pages = Array(numPages).fill().map((_, i) => pdf.getPage(i + 1))

        this.allObjects.splice(0, this.allObjects.length)

        this.pagesScale = Array(numPages).fill(1)
      } catch (e) {
        // eslint-disable-next-line no-console
        console.log('Failed to add pdf. Please try again.', e)
      }
    },
    onMeasure(scale, pageIndex) {
      this.pagesScale[pageIndex] = scale
    },
    selectPage(index) {
      this.selectedPageIndex = index
    },
    updateObject(objectId, payload) {
      this.allObjects = this.allObjects.map((object) => {
        if (object.id === objectId) {
          return { ...object, ...payload }
        }
        return object
      })
    },
    deleteObject(objectId) {
      this.allObjects = this.allObjects.filter((object) => object.id !== objectId)
    },
    async addImage(file) {
      try {
        // get dataURL to prevent canvas from tainted
        const url = await readAsDataURL(file)
        const img = await readAsImage(url)
        const id = this.genID()
        const { width, height } = img

        const object = {
          id,
          type: 'image',
          width,
          height,
          x: 0,
          y: 0,
          payload: img,
          file: file,
          page: this.selectedPageIndex,
        }

        this.allObjects.push(object)
      } catch (e) {
        // eslint-disable-next-line no-console
        console.log('Failed to add image.', e)
      }
    },
    async download() {
      try {
        await save(this.pdfFile, this.allObjects, 'PDF Copy', this.pagesScale)
      } catch (e) {
        // eslint-disable-next-line no-console
        console.log('Error on saving, please try again.', e)
      }
    },
    uploadImage(e) {
      const file = e.target.files[0]
      if (file && this.selectedPageIndex >= 0) {
        this.addImage(file)
      }
      e.target.value = null
    },
    addSignature() {
      // @TODO check selectedPageIndex
      this.signatureCanvas.isShow = true
    },
    pasteSignature({ width, height, path, scale }) {
      const id = this.genID()
      const signatureObject = {
        id,
        path,
        type: 'signature',
        x: 0,
        y: 0,
        height,
        width: width * scale,
        scale: 500,
        page: this.selectedPageIndex,
      }

      this.allObjects.push(signatureObject)
      this.signatureCanvas.isShow = false
    },
  }
}
</script>

<style scoped>
.selected-pdf {
  box-shadow: 0 0 0 3px rgba(66, 153, 225, 0.5);
}

.slider-blue {
  --slider-connect-bg: #3475e0;
  --slider-tooltip-bg: #3475e0;
  --slider-handle-ring-color: #3475e0;
}
</style>
