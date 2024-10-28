<template>
  <div class="min-h-screen bg-gray-100 p-8">
    <div class="max-w-4xl mx-auto">
      <header class="mb-8">
        <h1 class="text-3xl font-bold text-gray-800 mb-2">实时视频流和缺陷检测</h1>
        <p class="text-gray-600">实时监测和分析视频流中的缺陷</p>
      </header>

      <div class="bg-white rounded-lg shadow-lg p-6">
        <div class="mb-4">
          <div class="flex items-center space-x-4 mb-4">
            <div class="flex items-center">
              <div class="w-3 h-3 rounded-full" :class="isConnected ? 'bg-green-500' : 'bg-red-500'"></div>
              <span class="ml-2 text-sm text-gray-600">{{ isConnected ? '已连接' : '未连接' }}</span>
            </div>
            <span class="text-sm text-gray-600">检测状态: {{ status }}</span>
          </div>

          <DeviceSelector
            :is-capturing="isCapturing"
            @stream-ready="handleStream"
            @error="handleError"
          />
        </div>

        <VideoDisplay
          :stream="currentStream"
          :defects="defects"
        />

        <div class="mt-4">
          <h3 class="text-lg font-semibold mb-2">检测日志</h3>
          <div class="bg-gray-50 rounded-md p-4 h-32 overflow-y-auto">
            <div v-for="(log, index) in detectionLogs" :key="index" class="text-sm text-gray-600 mb-1">
              {{ log }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { io, Socket } from 'socket.io-client';
import { useTimeoutFn } from '@vueuse/core';
import DeviceSelector from './DeviceSelector.vue';
import VideoDisplay from './VideoDisplay.vue';

interface Defect {
  x: number;
  y: number;
  width: number;
  height: number;
}

const host = 'http://localhost:5000';
const socket = ref<Socket>();
const isConnected = ref(false);
const isCapturing = ref(false);
const status = ref('待机');
const defects = ref<Defect[]>([]);
const detectionLogs = ref<string[]>([]);
const currentStream = ref<MediaStream | null>(null);

const addLog = (message: string) => {
  detectionLogs.value.unshift(`[${new Date().toLocaleTimeString()}] ${message}`);
  if (detectionLogs.value.length > 100) detectionLogs.value.pop();
};

const handleError = (error: string) => {
  addLog(error);
  status.value = '错误';
  isCapturing.value = false;
  currentStream.value = null;
};

const handleStream = (stream: MediaStream) => {
  currentStream.value = stream;
  isCapturing.value = true;
  status.value = '检测中';
  addLog('开始视频录制');
  sendStreamToServer(stream);

  stream.getTracks()[0].onended = () => {
    isCapturing.value = false;
    status.value = '待机';
    currentStream.value = null;
    addLog('视频录制已停止');
  };
};

const sendStreamToServer = (stream: MediaStream) => {
  const mediaRecorder = new MediaRecorder(stream);
  mediaRecorder.ondataavailable = (event) => {
    if (event.data.size > 0) {
      const reader = new FileReader();
      reader.onloadend = () => {
        socket.value?.emit('video_stream', { frame: reader.result });
      };
      reader.readAsDataURL(event.data);
    }
  };
  mediaRecorder.start(1000);
};

const setupSocket = () => {
  socket.value = io(host);

  socket.value.on('connect', () => {
    isConnected.value = true;
    addLog('已连接到服务器');
  });

  socket.value.on('disconnect', () => {
    isConnected.value = false;
    addLog('与服务器断开连接');
  });

  socket.value.on('defect_result', (data) => {
    if (data.defect) {
      defects.value = [data.defect];
      addLog(`检测到新缺陷: (${data.defect.x}, ${data.defect.y})`);
      
      useTimeoutFn(() => {
        defects.value = [];
      }, 3000);
    }
  });
};

onMounted(() => {
  setupSocket();
});

onUnmounted(() => {
  socket.value?.disconnect();
  if (currentStream.value) {
    currentStream.value.getTracks().forEach(track => track.stop());
  }
});
</script>