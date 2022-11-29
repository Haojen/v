<template>
  <div class="w-screen h-screen flex flex-col justify-center items-center">
      <img class="mb-20" src="/img/v.png" alt="" style="width: 80px; opacity: 0.68">
      <video style="width: 640px; height: 360px" :src="previewMedialURL" controls></video>
      <article class="flex flex-col justify-center items-center text-center text-sm mb-10">
          <section class="flex flex-col justify-center items-center">
              <label for="uploadMaskImage">
                  <span role="button" class="inline-block bg-blue-500 p-2 rounded-lg font-semibold">1. Upload Mask Image</span>
              </label>

              <div class="flex justify-center items-center mt-2">
                  <span>Mask Image Opacity</span>
                  <input class="mx-2" type="range" min="0.1" max="1" step="0.01" v-model="maskOpacity">
                  <span v-text="maskOpacity" style="width: 30px"></span>
              </div>
          </section>
          <section class="mt-6">
              <label for="uploadVideo">
                  <span role="button" class="inline-block bg-blue-500 p-2 rounded-lg font-semibold">2. Upload Video</span>
              </label>
          </section>
          <section class="flex justify-center items-center mt-2" v-show="isProcessingVideo">
              <span class="mr-2">Processing</span>
              <svg class="animate-spin" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path fill="none" d="M0 0h24v24H0z"/><path d="M12 3a9 9 0 0 1 9 9h-2a7 7 0 0 0-7-7V3z" fill="rgba(255,255,255,1)"/></svg>
          </section>
      </article>
      <input class="hidden" id="uploadMaskImage" type="file" accept="image/png, image/gif, image/jpeg, image/webp, image/jpg" @change="onUpdateMaskImg">
      <input class="hidden" id="uploadVideo" type="file" @change="onUpdateMediaFiles">
  </div>
</template>

<script lang="ts" setup>
import { ref } from 'vue';
import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";

const maskOpacity = ref(0.3)

const ffmpeg = createFFmpeg({ log: true })

const previewMedialURL = ref()

const isUploadedMask = ref(false)
async function onUpdateMaskImg(event: Event) {
    const files = (event?.target as HTMLInputElement).files

    if (!files || files.length == 0) return

    ffmpeg.FS('writeFile', 'mask.png', await fetchFile(files[0]))
    isUploadedMask.value = true
}

function onUpdateMediaFiles(event: Event) {
    const files = (event?.target as HTMLInputElement).files

    if (!files || files.length == 0) return

    videoProcess(files!)
}

interface IVideoGenerateOptions {
    outputFileName?: string // 日期 + 随机事件戳
    framesFilter?: []
    audiosFilter?: Blob,
    insertFrame?: [],
}

const isProcessingVideo = ref(false)
async function videoProcess(files: FileList, generateOptions?: IVideoGenerateOptions) {
    isProcessingVideo.value = true
    const { name } = files[0];
    ffmpeg.FS('writeFile', name, await fetchFile(files[0]));

    let maskFilter:string[] = []
    if (isUploadedMask.value) {
        maskFilter = [
            '-i',
            'mask.png',
            '-filter_complex',
            `[1:v]format=rgba,colorchannelmixer=aa=${maskOpacity.value}[fg];[0][fg]overlay=(W-w)/2:(H-h)/2`,
        ]
    }

    console.log(isUploadedMask.value, 'isUploadedMask.value', maskFilter)

    const ffmpegRunOptions = [
        '-i',
        name,
        ...maskFilter,
        '-pix_fmt',
        'yuv420p',
        '-c:a',
        'copy',
        'result.mp4',
    ]

    await ffmpeg.run(...ffmpegRunOptions)
    const data = ffmpeg.FS('readFile', 'result.mp4');
    previewMedialURL.value = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));

    isProcessingVideo.value = false
}

ffmpeg.load();
</script>
