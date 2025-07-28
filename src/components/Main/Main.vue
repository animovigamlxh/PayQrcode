<template>
  <div v-if="isPaymentPage">
    <t-space direction="vertical" class="payment-page">
      <h2>请选择支付方式</h2>
      <t-space>
        <div class="payment-qrcode">
          <img :src="alipayPaymentUrl" alt="Alipay" />
          <p>支付宝</p>
        </div>
        <div class="payment-qrcode">
          <img :src="wechatPaymentUrl" alt="Wechat" />
          <p>微信支付</p>
        </div>
      </t-space>
    </t-space>
  </div>
  <div v-else>
    <header>
      <section class="upload-list">
        <div class="item">
          <t-input-adornment>
            <template #append>
              <t-upload
                accept="image/*"
                :showUploadProgress="false"
                :useMockProgress="false"
                :autoUpload="false"
                theme="custom"
                :onChange="alipayChange"
                :trigger-button-props="{ theme: 'primary', variant: 'base' }"
              />
            </template>
            <t-input
              :status="alipayVal ? 'success' : ''"
              readonly
              v-model="alipayVal"
              placeholder="请上传 支付宝 收款码"
            />
          </t-input-adornment>
        </div>
        <div class="item">
          <t-input-adornment>
            <template #append>
              <t-upload
                accept="image/*"
                :showUploadProgress="false"
                :useMockProgress="false"
                :autoUpload="false"
                theme="custom"
                :onChange="wechatChange"
                :trigger-button-props="{ theme: 'success', variant: 'base' }"
              />
            </template>
            <t-input
              :status="wechatVal ? 'success' : ''"
              readonly
              v-model="wechatVal"
              placeholder="请上传 微信 收款码"
            />
          </t-input-adornment>
        </div>
        <!--        <div class="item flex">-->
        <!--          <t-radio-group size="small" v-model="themeStatus">-->
        <!--            <t-radio-button value="none">默认</t-radio-button>-->
        <!--            <t-radio-button value="theme_a">主题A</t-radio-button>-->
        <!--            <t-radio-button value="theme_b">主题B</t-radio-button>-->
        <!--            <t-radio-button value="theme_c">主题C</t-radio-button>-->
        <!--            <t-radio-button value="theme_d">主题D</t-radio-button>-->
        <!--          </t-radio-group>-->
        <!--        </div>-->
      </section>
      <!--      <t-card class="alipay-hide">-->
      <!--        <span class="title">缺省比例</span>-->
      <!--        <t-slider v-model="qrHideSize" :onChange="alipayHideChange" />-->
      <!--      </t-card>-->
    </header>
    <main :class="{ show: finalQrcodeImg }">
      <div class="qr-main" ref="qrcodeDom">
        <img class="final-qr" :src="finalQrcodeImg" crossorigin="anonymous" />
      </div>
      <t-button theme="success" size="large" class="download-qrcode" :onClick="downloadQrcode">
        <template #icon>
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path stroke="none" d="M0 0h24v24H0z" fill="none" />
            <path d="M19 18a3.5 3.5 0 0 0 0 -7h-1a5 4.5 0 0 0 -11 -2a4.6 4.4 0 0 0 -2.1 8.4" />
            <path d="M12 13l0 9" />
            <path d="M9 19l3 3l3 -3" />
          </svg>
        </template>
        下载二维码
      </t-button>
    </main>
  </div>
</template>
<script lang="ts" setup>
import { ref, onMounted, watch } from 'vue'
import html2canvas from 'html2canvas'
import jsQR from 'jsqr'
import QrCode from 'qrcode'
import { NotifyPlugin } from 'tdesign-vue-next'

//  是否为支付页面
const isPaymentPage = ref<boolean>(false)
const alipayPaymentUrl = ref<string>('')
const wechatPaymentUrl = ref<string>('')

//  是否美化
const themeStatus = ref<string>('none')

//  上传支付宝收款码
const alipayFileList = ref<any>()
const alipayVal = ref<any>()
const alipayQrcodeImg = ref<string>()
// 支付宝收款码白名单
const alipayWhiteList = ['alipay.com']
const alipayChange = async (v: any) => {
  alipayFileList.value = v && v.length ? v[0].raw : alipayFileList.value
  if (!alipayFileList.value) return
  const res = await loadImg(alipayFileList.value)
  if (!res || !alipayWhiteList.some((i: string) => String(res).toLowerCase().includes(i)))
    return NotifyPlugin.error({
      title: 'Error',
      content: '请上传正确的 支付宝 收款码',
    })
  alipayVal.value = String(res)
  alipayQrcodeImg.value = await drawQR(alipayVal.value, true)
  showQrcode()
}

// 上传微信
const wechatFileList = ref<any>()
const wechatVal = ref<any>()
const wechatQrcodeImg = ref<string>()
// 最终生成的合并二维码
const finalQrcodeImg = ref<string>()

watch([alipayVal, wechatVal], ([newAlipay, newWechat]) => {
  if (newAlipay && newWechat) {
    const baseUrl = window.location.origin + window.location.pathname
    const search = new URLSearchParams({
      alipay_url: newAlipay,
      wechat_url: newWechat,
    })
    const finalUrl = `${baseUrl}?${search.toString()}`
    drawQR(finalUrl).then((url) => {
      finalQrcodeImg.value = url
    })
  }
})
// 微信收款码白名单
const wechatWhiteList = ['wxp://', 'weixin.qq.com', 'wechatpay.cn']
const wechatChange = async (v: any) => {
  wechatFileList.value = v && v.length ? v[0].raw : wechatFileList.value
  if (!wechatFileList.value) return
  const res = await loadImg(wechatFileList.value)
  if (!res || !wechatWhiteList.some((i: string) => String(res).toLowerCase().includes(i)))
    return NotifyPlugin.error({
      title: 'Error',
      content: '请上传正确的 微信 收款码',
    })
  wechatVal.value = String(res)
  // wechatQrcodeImg.value = await drawQR(wechatVal.value)
  // showQrcode()
}

// 支付宝隐藏比例
const qrHideSize = ref<number>(50)
const alipayHideChange = (v: any) => {
  qrHideSize.value = v
  alipayChange(null)
}

// 生成二维码
const showQrcode = async () => {
  if (!alipayQrcodeImg.value || !wechatQrcodeImg.value) return
}

// 加载图片
const loadImg = (file: any) => {
  if (!file) return
  const reader = new FileReader()
  reader.readAsDataURL(file)
  return new Promise((resolve) => {
    reader.onload = (e) => {
      const image = new Image()
      image.src = e.target!.result as any
      image.onload = () => {
        const code = qrcodeResult(image)
        resolve(code)
      }
    }
  })
}

//  识别二维码
const qrcodeResult = (image: any) => {
  const canvas = document.createElement('canvas')
  const context: any = canvas.getContext('2d')
  canvas.width = image.width
  canvas.height = image.height
  context.drawImage(image, 0, 0, canvas.width, canvas.height)
  const imageData = context.getImageData(0, 0, canvas.width, canvas.height)
  const decodedResult = jsQR(imageData.data, imageData.width, imageData.height, {
    inversionAttempts: 'dontInvert',
  })
  return decodedResult?.data
}

// 添加清除右上角的函数
const qrcodeDom = ref<any>()
const qrSize = ref<number>(0)
const drawQR = async (qrValue: string, hide: boolean = false) => {
  const qrDom = document.createElement('canvas')
  await QrCode.toCanvas(qrDom, qrValue, {
    errorCorrectionLevel: 'H',
    width: qrSize.value,
    margin: 0,
    color: {
      dark: '#000000',
      light: '#ffffff',
    },
  })
  if (hide) {
    const ctx: any = qrDom.getContext('2d')
    const clearWidth = (qrSize.value / 2) * (qrHideSize.value / 100)
    ctx.clearRect(qrSize.value / 2, qrSize.value - clearWidth, qrSize.value / 2, clearWidth)
  }
  return qrDom.toDataURL('image/png')
}

// 下载二维码
const downloadQrcode = async () => {
  if (!finalQrcodeImg.value) {
    NotifyPlugin.error({
      title: 'Error',
      content: '请先上传支付宝和微信的收款码',
    })
    return
  }
  const link = document.createElement('a')
  link.href = finalQrcodeImg.value
  link.download = 'PayQrcode.png'
  link.click()
}

onMounted(async () => {
  const params = new URLSearchParams(window.location.search)
  const alipayUrl = params.get('alipay_url')
  const wechatUrl = params.get('wechat_url')
  const ua = navigator.userAgent.toLowerCase()

  if (alipayUrl && wechatUrl) {
    isPaymentPage.value = true
    if (ua.includes('alipay')) {
      window.location.href = alipayUrl
      return
    }
    if (ua.includes('micromessenger')) {
      window.location.href = wechatUrl
      return
    }
    // 在普通浏览器中，生成两个二维码以供选择
    alipayPaymentUrl.value = await drawQR(alipayUrl)
    wechatPaymentUrl.value = await drawQR(wechatUrl)
    return
  }

  qrSize.value = qrcodeDom.value!.clientWidth
})
</script>
<style lang="less" scoped>
@import url('./Main.less');
</style>
