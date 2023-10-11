<script setup lang="ts">
import { Message } from '@arco-design/web-vue'
import CryptoJS from 'crypto-js'

// 这是别人的KEY
const APPID = '5ec244d5'
const API_SECRET = '37912e3e3f205e2a6201ec290452470a'
const API_KEY = '78b6c006f1f3df5e24d315e3dff09212'
const btnStatus = ref('UNDEFINED') // "UNDEFINED" "CONNECTING" "OPEN" "CLOSING" "CLOSED"
const result = ref('')
const resultText = ref('')
const resultTextTemp = ref('')
const btnText = ref('开始录音')
let iatWS: any
let countdownInterval: any
// eslint-disable-next-line ts/ban-ts-comment
// @ts-expect-error
const recorder: any = new RecorderManager('./')

function toBase64(buffer: any) {
  let binary = ''
  const bytes = new Uint8Array(buffer)
  const len = bytes.byteLength
  for (let i = 0; i < len; i++)
    binary += String.fromCharCode(bytes[i])

  return window.btoa(binary)
}

recorder.onStart = () => {
  changeBtnStatus('OPEN')
}

recorder.onFrameRecorded = ({ isLastFrame, frameBuffer }: any) => {
  if (iatWS.readyState === iatWS.OPEN) {
    iatWS.send(
      JSON.stringify({
        data: {
          status: isLastFrame ? 2 : 1,
          format: 'audio/L16;rate=16000',
          encoding: 'raw',
          audio: toBase64(frameBuffer),
        },
      }),
    )
    if (isLastFrame)
      changeBtnStatus('CLOSING')
  }
}
recorder.onStop = () => {
  clearInterval(countdownInterval)
}

/**
 * 获取websocket url
 * 该接口需要后端提供，这里为了方便前端处理
 */
function getWebSocketUrl() {
  // 请求地址根据语种不同变化
  let url = 'wss://iat-api.xfyun.cn/v2/iat'
  const host = 'iat-api.xfyun.cn'
  const apiKey = API_KEY
  const apiSecret = API_SECRET
  const date = new Date().toUTCString()
  const algorithm = 'hmac-sha256'
  const headers = 'host date request-line'
  const signatureOrigin = `host: ${host}\ndate: ${date}\nGET /v2/iat HTTP/1.1`
  const signatureSha = CryptoJS.HmacSHA256(signatureOrigin, apiSecret)
  const signature = CryptoJS.enc.Base64.stringify(signatureSha)
  const authorizationOrigin = `api_key="${apiKey}", algorithm="${algorithm}", headers="${headers}", signature="${signature}"`
  const authorization = btoa(authorizationOrigin)
  url = `${url}?authorization=${authorization}&date=${date}&host=${host}`
  return url
}
function countdown() {
  let seconds = 60
  btnText.value = `录音中（${seconds}s）`
  countdownInterval = setInterval(() => {
    console.warn('countdown', seconds)
    seconds = seconds - 1
    if (seconds <= 0) {
      clearInterval(countdownInterval)
      recorder.stop()
    }
    else {
      btnText.value = `录音中（${seconds}s）`
    }
  }, 1000)
}
function renderResult(resultData: any) {
// 识别结束
  const jsonData = JSON.parse(resultData)
  if (jsonData.data && jsonData.data.result) {
    const data = jsonData.data.result
    let str = ''
    const ws = data.ws
    for (let i = 0; i < ws.length; i++)
      str = str + ws[i].cw[0].w

    // 开启wpgs会有此字段(前提：在控制台开通动态修正功能)
    // 取值为 "apd"时表示该片结果是追加到前面的最终结果；取值为"rpl" 时表示替换前面的部分结果，替换范围为rg字段
    if (data.pgs) {
      if (data.pgs === 'apd') {
        // 将resultTextTemp同步给resultText
        resultText.value = resultTextTemp.value
      }
      // 将结果存储在resultTextTemp中
      resultTextTemp.value = resultText.value + str
    }
    else {
      resultText.value = resultText.value + str
    }
    result.value = resultTextTemp.value || resultText.value || ''
  }
  if (jsonData.code === 0 && jsonData.data.status === 2)
    iatWS.close()

  if (jsonData.code !== 0) {
    iatWS.close()
    console.error(jsonData)
  }
}
function changeBtnStatus(status: string) {
  // eslint-disable-next-line no-console
  console.log(status)
  btnStatus.value = status
  if (status === 'CONNECTING') {
    btnText.value = '建立连接中'
    result.value = ''
    resultText.value = ''
    resultTextTemp.value = ''
  }
  else if (status === 'OPEN') {
    countdown()
  }
  else if (status === 'CLOSING') {
    btnText.value = '关闭连接中'
  }
  else if (status === 'CLOSED') {
    btnText.value = '开始录音'
  }
}
function connectWebSocket() {
  const websocketUrl = getWebSocketUrl()
  console.warn(websocketUrl)
  if ('WebSocket' in window) {
    iatWS = new WebSocket(websocketUrl)
  }
  else {
    Message.warning('浏览器不支持WebSocket')
    return
  }
  changeBtnStatus('CONNECTING')
  iatWS.onopen = () => {
    // 开始录音
    recorder.start({
      sampleRate: 16000,
      frameSize: 1280,
    })
    const params = {
      common: {
        app_id: APPID,
      },
      business: {
        language: 'zh_cn',
        domain: 'iat',
        accent: 'mandarin',
        vad_eos: 5000,
        dwa: 'wpgs',
      },
      data: {
        status: 0,
        format: 'audio/L16;rate=16000',
        encoding: 'raw',
      },
    }
    console.warn(params)
    iatWS.send(JSON.stringify(params))
  }
  iatWS.onmessage = (e: any) => {
    console.warn(e.data)
    renderResult(e.data)
  }
  iatWS.onerror = (e: any) => {
    console.error(e)
    recorder.stop()
    changeBtnStatus('CLOSED')
  }
  iatWS.onclose = () => {
    recorder.stop()
    changeBtnStatus('CLOSED')
  }
}
function handleBtnClick() {
  if (btnStatus.value === 'UNDEFINED' || btnStatus.value === 'CLOSED') {
    console.warn('start!')
    connectWebSocket()
  }
  //

  else if (btnStatus.value === 'OPEN' || btnStatus.value === 'CONNECTING') {
    //
    console.warn('stop!')
    recorder.stop()
  }
}
</script>

<template>
  <div flex flex-col items-start>
    <div>
      浏览器录音听写：
      <a-button type="primary" @click="handleBtnClick">
        {{ btnText }}
      </a-button>
    </div>
    <div>
      录音内容：
      <span text-red-5>{{ result }}</span>
    </div>
  </div>
</template>
