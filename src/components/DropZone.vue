<!-- eslint-disable vue/require-v-for-key -->
<template>
  <div
    id="dropzone"
    v-if="fileUrl === undefined"
    @drop.prevent="dropHandler"
    @click="triggerFileSelect"
  >
    <div class="dropzone-wrapper">
      <span class="dropzone-text">{{ dropzoneText }}</span>
    </div>
  </div>
  <input
    type="file"
    class="dropzone-input"
    accept="video/*"
    @input="fileSelectHandler"
    ref="fileInput"
  />
  <span class="ffmpeg-status">{{ ffmpegStatus }}</span>
  <div class="video-wrapper" v-if="fileUrl !== undefined">
    <video controls v-bind:src="fileUrl" class="video-result" v-bind:type="file?.type"></video>
    <div class="target-format-wrapper" v-if="outputFile === undefined">
      <label for="target-format" class="target-format-label">convert to: </label>
      <select ref="outputSelect" name="target-format">
        <option v-for="t in Object.keys(fileTypeExtension)" v-bind:value="t">{{ t }}</option>
      </select>
      <button class="target-format-button" @click="transcode">Go</button>
    </div>
    <div class="output" v-if="outputFileUrl">
      <video controls class="output-result" v-bind:src="outputFileUrl"></video>
    </div>
  </div>
</template>

<style scoped>
.target-format-label {
  background-color: var(--color-border-hover);
}

.target-format-wrapper {
  display: inline-grid;
  grid-template-columns: 1fr 1fr 1fr;
}

.video-result {
  max-width: 100%;
  display: flex;
}

#dropzone {
  border: 2px dotted var(--color-border);
  background-color: var(--color-border-hover);
  height: 200px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.video-wrapper {
  border: 2px dotted var(--color-border);
}

.dropzone-input {
  display: none;
}

#dropzone > .dropzone-wrapper {
  display: flex;
  justify-content: center;
  user-select: none;
  padding: 0 2rem;
}

#dropzone:hover {
  border: 2px dotted var(--color-border-hover);
}
</style>

<script setup lang="ts">
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { ref } from 'vue'

interface LogEvent {
  type: string
  message: string
}

type FileInputEventTarget = EventTarget & { files: FileList }

const dropzoneText = ref('select or drag+drop here')
const ffmpegStatus = ref('awaiting file...')
const file = ref<File>()
const fileUrl = ref()

const fileInput = ref<HTMLInputElement>()
const outputSelect = ref<HTMLSelectElement>()

const outputFile = ref()
const outputFileUrl = ref()

function triggerFileSelect(): void {
  fileInput.value?.click()
}

function dropHandler(event: Event): void {
  handleFile((event.target as FileInputEventTarget).files)
}

function fileSelectHandler(event: Event): void {
  handleFile((event.target as FileInputEventTarget).files)
}

async function handleFile(files: FileList) {
  if (files[0] && files[0].type.startsWith('video/')) {
    const reader = new FileReader()
    reader.onload = async () => {
      fileUrl.value = URL.createObjectURL(files[0])
    }
    reader.readAsDataURL(files[0])
    file.value = files[0]
    dropzoneText.value = formatFileInfo(files[0])
    ffmpegStatus.value = `Received file ${files[0].name}`
  }
}

async function transcode() {
  const ffmpeg = new FFmpeg()
  ffmpegStatus.value = `transcoding ${file.value?.name}`
  ffmpeg.on('log', ({ message: msg }: LogEvent) => {
    ffmpegStatus.value = msg
  })
  await ffmpeg.load({
    coreURL: await toBlobURL(
      `${window.location.origin}/ttttranscode/ffmpeg-core.js`,
      'text/javascript'
    ),
    wasmURL: await toBlobURL(
      `${window.location.origin}/ttttranscode/ffmpeg-core.wasm`,
      'application/wasm'
    )
  })
  await ffmpeg.writeFile(file.value!.name, await fetchFile(fileUrl.value))
  const outputFilename = `${file.value!.name.split('.')[0]}.${outputSelect.value?.value}`
  await ffmpeg.exec(['-i', file.value!.name, outputFilename])
  ffmpegStatus.value = `transcoding ${file.value!.name} to ${outputFilename} DONE`
  const data = await ffmpeg.readFile(outputFilename)
  const blob = new Blob([(data as Uint8Array).buffer])
  outputFile.value = new File([blob], outputFilename, {
    type: fileTypeExtension[outputSelect.value!.value as keyof typeof fileTypeExtension]
  })
  outputFileUrl.value = URL.createObjectURL(blob)
}

function formatFileInfo(file: File): string {
  return `Filename: ${file.name} - Size: ${file.size} - Type: ${file.type}`
}

const fileTypeExtension = {
  mp3: 'audio/mpeg',
  mp4: 'video/mp4',
  aac: 'audio/aac',
  weba: 'audio/webm',
  opus: 'audio/ogg',
  wav: 'audio/wav',
  webm: 'video/webm',
  '3gp': 'video/3gpp'
}
</script>
