<script setup lang="ts">
const voiceTxt = ref('')
const inputTxt = ref('')
const isRecording = ref(false)
let times: any = null
let voice: any = null
onMounted(() => {
  voice = new Voice({
    // 服务接口认证信息 注：apiKey 和 apiSecret 的长度都差不多，请要填错哦，！
    appId: '5ec244d5',
    apiSecret: '37912e3e3f205e2a6201ec290452470a',
    apiKey: '78b6c006f1f3df5e24d315e3dff09212',
    // 注：要获取以上3个参数，请到迅飞开放平台：https://www.xfyun.cn/services/voicedictation 【注：这是我的迅飞语音听写（流式版）每天服务量500（也就是调500次），如果你需求里大请购买服务量：https://www.xfyun.cn/services/voicedictation?target=price】

    onWillStatusChange(oldStatus, newStatus) {
    // 可以在这里进行页面中一些交互逻辑处理：注：倒计时（语音听写只有60s）,录音的动画，按钮交互等！
      // fixedBox.style.display = 'block'
    },
    onTextChange(text) {
      console.log(text)
      // 监听识别结果的变化
      voiceTxt.value = text

      // 3秒钟内没有说话，就自动关闭
      if (text) {
        clearTimeout(times)
        times = setTimeout(() => {
          this.stop() // voice.stop();
          // fixedBox.style.display = 'none'
        }, 3000)
      };
    },
  })
})
function handleStart() {
  voice.start()
  isRecording.value = true
}
function handleEnd() {
  voice.stop()
  isRecording.value = false
  // fixedBox.style.display = 'none';
}
</script>

<template>
  <div flex flex-col items-center>
    <a-typography :style="{ marginTop: '-40px' }">
      <a-typography-title>
        科大讯飞语音听写（流式版）WebAPI
      </a-typography-title>
      <div flex justify-center>
        <a-card :style="{ width: '360px' }" title="Arco Card">
          <template #extra>
            <a-link v-if="!isRecording" @click="handleStart">
              开始识别
            </a-link>
            <a-link v-else @click="handleEnd">
              停止识别
            </a-link>
          </template>
          {{ voiceTxt }}
          <a-input v-model:model-value="inputTxt" />
        </a-card>
      </div>
    </a-typography>
  </div>
</template>
