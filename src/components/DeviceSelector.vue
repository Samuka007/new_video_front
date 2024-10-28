<template>
  <div class="space-y-4">
    <div class="flex items-center space-x-4">
      <select
        v-model="selectedSource"
        class="flex-1 rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
      >
        <option value="screen">共享屏幕</option>
        <option value="camera">摄像头</option>
      </select>
      
      <select
        v-if="selectedSource === 'camera'"
        v-model="selectedDevice"
        class="flex-1 rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
      >
        <option v-for="device in videoDevices" :key="device.deviceId" :value="device.deviceId">
          {{ device.label || `设备 ${device.deviceId.slice(0, 8)}...` }}
        </option>
      </select>

      <button 
        @click="startCapture"
        class="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors disabled:opacity-50"
        :disabled="isCapturing"
      >
        {{ isCapturing ? '录制中...' : '开始录制' }}
      </button>
    </div>

    <div v-if="error" class="text-red-500 text-sm">
      {{ error }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';

const props = defineProps<{
  isCapturing: boolean;
}>();

const emit = defineEmits<{
  (e: 'streamReady', stream: MediaStream): void;
  (e: 'error', error: string): void;
}>();

const selectedSource = ref('screen');
const selectedDevice = ref('');
const videoDevices = ref<MediaDeviceInfo[]>([]);
const error = ref('');

const getVideoDevices = async () => {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    videoDevices.value = devices.filter(device => device.kind === 'videoinput');
    
    if (videoDevices.value.length > 0) {
      selectedDevice.value = videoDevices.value[0].deviceId;
    }
  } catch (err) {
    error.value = '无法获取视频设备列表';
    console.error(err);
  }
};

const startCapture = async () => {
  try {
    error.value = '';
    let stream: MediaStream;

    if (selectedSource.value === 'screen') {
      stream = await navigator.mediaDevices.getDisplayMedia({ video: true });
    } else {
      stream = await navigator.mediaDevices.getUserMedia({
        video: {
          deviceId: selectedDevice.value ? { exact: selectedDevice.value } : undefined
        }
      });
    }

    emit('streamReady', stream);
  } catch (err) {
    error.value = `无法启动${selectedSource.value === 'screen' ? '屏幕共享' : '摄像头'}: ${err}`;
    emit('error', error.value);
  }
};

onMounted(async () => {
  await getVideoDevices();
});
</script>