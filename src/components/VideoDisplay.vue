<template>
  <div class="relative w-full aspect-video bg-gray-900 rounded-lg overflow-hidden">
    <video 
      ref="videoRef"
      class="w-full h-full object-contain"
      autoplay
      :class="{ 'opacity-0': !stream }"
    ></video>
    
    <div id="overlay" class="absolute inset-0 pointer-events-none">
      <transition-group name="fade">
        <div
          v-for="(defect, index) in defects"
          :key="index"
          class="absolute border-2 border-red-500 bg-red-500/20"
          :style="{
            left: `${defect.x}px`,
            top: `${defect.y}px`,
            width: `${defect.width}px`,
            height: `${defect.height}px`
          }"
        >
          <span class="absolute -top-6 left-0 bg-red-500 text-white text-xs px-2 py-1 rounded">
            缺陷 #{{ index + 1 }}
          </span>
        </div>
      </transition-group>
    </div>
    
    <div 
      v-if="!stream" 
      class="absolute inset-0 flex items-center justify-center bg-gray-800/50"
    >
      <p class="text-white text-lg">请选择视频源并开始录制</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';

interface Defect {
  x: number;
  y: number;
  width: number;
  height: number;
}

const props = defineProps<{
  stream: MediaStream | null;
  defects: Defect[];
}>();

const videoRef = ref<HTMLVideoElement | null>(null);

watch(() => props.stream, (newStream) => {
  if (videoRef.value && newStream) {
    videoRef.value.srcObject = newStream;
  }
}, { immediate: true });
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>