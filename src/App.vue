<template>
  <main class="chanel-app">
    <div class="header">
      <h1 class="brand-title">CHANEL</h1>
      <p class="subtitle">Document Recognition System</p>
    </div>

    <!-- Camera Section -->
    <section class="camera-section">
      <div class="camera-container">
        <video ref="video" playsinline autoplay muted class="video-preview"></video>
        <div class="camera-overlay" v-if="!stream">
          <div class="camera-icon">📷</div>
        </div>
      </div>
      
      <div class="controls">
        <button @click="initCamera" v-if="!stream" class="chanel-button primary">
          <span>Start Camera</span>
        </button>
        <button @click="shoot" :disabled="!stream || loading" class="chanel-button secondary">
          <span v-if="!loading">Capture</span>
          <span v-else>Processing...</span>
        </button>
      </div>
      
      <div v-if="loading" class="loading-indicator">
        <div class="loader"></div>
        <p>Analyzing document...</p>
      </div>
    </section>

    <!-- Image Preview -->
    <section v-if="imageUrl" class="image-section">
      <h3 class="section-title">Captured Image</h3>
      <div class="image-container">
        <img :src="imageUrl" class="captured-image" />
      </div>
    </section>

<!-- Data Results -->
<section v-if="parsedData && Object.keys(parsedData).length > 0" class="result-section">
  <h3 class="section-title">Extracted Information</h3>
<div
  v-for="(value, key) in parsedData"
  :key="key"
  class="data-field row-layout"
  v-if="value !== ''"

>
  <label class="field-label">{{ getFieldLabel(key) }}</label>
  <div class="field-value">{{ value }}</div>
</div>

</section>






  </main>
</template>

<script>
import { ref, onUnmounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const video = ref(null)
    const stream = ref(null)
    const loading = ref(false)
    const json = ref('')
    const parsedData = ref(null)
    const imageUrl = ref('')
    const docType = 'idcard'  // ใช้ตายตัว
    const apiKey = import.meta.env.VITE_AZURE_OPENAI_KEY;

    // เริ่มกล้อง
    async function initCamera() {
      try {
        stream.value = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: { ideal: 'environment' } }
        })
        video.value.srcObject = stream.value
      } catch (err) {
        alert('ไม่สามารถเปิดกล้องได้: ' + err.message)
      }
    }

    function stopCamera() {
      if (stream.value) {
        stream.value.getTracks().forEach(t => t.stop())
        stream.value = null
      }
    }

    onUnmounted(stopCamera)

    // เมื่อถ่ายรูป
    async function shoot() {
      const canvas = document.createElement('canvas')
      canvas.width = video.value.videoWidth
      canvas.height = video.value.videoHeight
      canvas.getContext('2d').drawImage(video.value, 0, 0)
      
      const blob = await new Promise(res => canvas.toBlob(res, 'image/jpeg', 0.85))

      loading.value = true
      json.value = ''
      parsedData.value = null
      imageUrl.value = ''

      try {
        const blobUrl = await uploadBlob(blob)
        imageUrl.value = blobUrl

        const raw = await ocrRead(blobUrl)
        const fields = await gptParse(raw)
        json.value = JSON.stringify(fields, null, 2)
        parsedData.value = fields
      } catch (e) {
        alert(e.message || e)
      } finally {
        loading.value = false
      }
    }

    // ✅ Upload ไป API แล้วคืน URL
    async function uploadBlob(blob) {
      const formData = new FormData()
      formData.append('file', blob, `${Date.now()}.jpg`)

      const response = await fetch(
        'https://demo-bot-api-h9g2dfdfavambsex.canadacentral-01.azurewebsites.net/UploadImage/upload',
        {
          method: 'POST',
          body: formData
        }
      )

      const data = await response.json()
      
      if (data.success && data.url) {
        return data.url
      } else {
        throw new Error('Upload API failed: ' + JSON.stringify(data))
      }
    }

    // ✅ OCR ด้วย API ที่คุณสร้างไว้
    async function ocrRead(url) {
      const response = await fetch(
        'https://demo-bot-api-h9g2dfdfavambsex.canadacentral-01.azurewebsites.net/UploadImage/ocr',
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ urlSource: url })
        }
      )

      const data = await response.json()
      
      if (data && data.content) {
        return data.content
      } else {
        throw new Error('OCR API failed: ' + JSON.stringify(data))
      }
    }

    // 💬 ส่งข้อความไป GPT
// ✅ เรียก API parse
async function gptParse(rawText) {
  const res = await fetch(
    'https://demo-bot-api-h9g2dfdfavambsex.canadacentral-01.azurewebsites.net/UploadImage/gpt',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userText: rawText })
    }
  )
  if (!res.ok) throw new Error('Parse API error ' + res.status)
  const data = await res.json()
  return data.data      // JsonElement ที่ API ส่งกลับ
}




    // แปลง field key เป็นชื่อที่อ่านง่าย
    function getFieldLabel(key) {
      const labels = {
        type: 'ประเภทเอกสาร',
        id: 'เลขประจำตัวประชาชน',
        ttlTH: 'คำนำหน้า (ไทย)',
        fTH: 'ชื่อ (ไทย)',
        mTH: 'ชื่อกลาง (ไทย)',
        lTH: 'นามสกุล (ไทย)',
        ttlEN: 'คำนำหน้า (อังกฤษ)',
        fEN: 'ชื่อ (อังกฤษ)',
        mEN: 'ชื่อกลาง (อังกฤษ)',
        lEN: 'นามสกุล (อังกฤษ)',
        dobTH: 'วันเกิด (ไทย)',
        dobISO: 'วันเกิด',
        hn: 'บ้านเลขที่',
        vn: 'หมู่ที่',
        aly: 'ซอย',
        rd: 'ถนน',
        subTH: 'แขวง/ตำบล',
        distTH: 'เขต/อำเภอ',
        provTH: 'จังหวัด',
        zip: 'รหัสไปรษณีย์',
        doiTH: 'วันออกบัตร (ไทย)',
        doiISO: 'วันออกบัตร',
        doeTH: 'วันหมดอายุ (ไทย)',
        doeISO: 'วันหมดอายุ',
        laser: 'รหัสเลเซอร์',
        office: 'สำนักงานออกบัตร',
        pn: 'เลขหนังสือเดินทาง',
        sn: 'นามสกุล',
        gn: 'ชื่อ',
        nTH: 'ชื่อ-สกุล (ไทย)',
        nat: 'สัญชาติ',
        sex: 'เพศ',
        pob: 'สถานที่เกิด',
        ia: 'หน่วยงานที่ออก'
      }
      return labels[key] || key
    }

    return {
      video,
      stream,
      loading,
      json,
      parsedData,
      imageUrl,
      initCamera,
      shoot,
      getFieldLabel
    }
  }
}
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.chanel-app {
  min-height: 100vh;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  font-family: 'Helvetica Neue', Arial, sans-serif;
  color: #2c3e50;
}

.header {
  text-align: center;
  padding: 3rem 2rem 2rem;
  background: linear-gradient(135deg, #2c3e50 0%, #34495e 100%);
  color: white;
  margin-bottom: 2rem;
}

.brand-title {
  font-size: 3.5rem;
  font-weight: 300;
  letter-spacing: 0.5rem;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

.subtitle {
  font-size: 1.1rem;
  margin: 0.5rem 0 0 0;
  opacity: 0.9;
  font-weight: 300;
  letter-spacing: 0.1rem;
}

.camera-section {
  max-width: 600px;
  margin: 0 auto 3rem;
  padding: 0 2rem;
}

.camera-container {
  position: relative;
  background: #000;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  margin-bottom: 2rem;
}

.video-preview {
  width: 100%;
  height: 400px;
  object-fit: cover;
  display: block;
}

.camera-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
}

.camera-icon {
  font-size: 4rem;
  opacity: 0.7;
}

.controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-bottom: 2rem;
}

.chanel-button {
  padding: 1rem 2.5rem;
  border: none;
  border-radius: 50px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.1rem;
  min-width: 150px;
  position: relative;
  overflow: hidden;
}

.chanel-button.primary {
  background: linear-gradient(135deg, #2c3e50 0%, #34495e 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(44, 62, 80, 0.4);
}

.chanel-button.primary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(44, 62, 80, 0.6);
}

.chanel-button.secondary {
  background: linear-gradient(135deg, #ecf0f1 0%, #bdc3c7 100%);
  color: #2c3e50;
  box-shadow: 0 4px 15px rgba(189, 195, 199, 0.4);
}

.chanel-button.secondary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(189, 195, 199, 0.6);
}

.chanel-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none !important;
}

.loading-indicator {
  text-align: center;
  padding: 2rem;
}

.loader {
  width: 40px;
  height: 40px;
  border: 3px solid #ecf0f1;
  border-top: 3px solid #2c3e50;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.image-section, .result-section {
  max-width: 800px;
  margin: 0 auto 3rem;
  padding: 0 2rem;
}

.section-title {
  font-size: 1.8rem;
  color: #2c3e50;
  margin-bottom: 1.5rem;
  text-align: center;
  font-weight: 300;
  letter-spacing: 0.1rem;
}

.image-container {
  background: white;
  padding: 2rem;
  border-radius: 20px;
  box-shadow: 0 8px 25px rgba(0,0,0,0.1);
}

.captured-image {
  width: 100%;
  border-radius: 15px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
}

.data-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
}

.data-field {
  background: white;
  padding: 1.5rem;
  border-radius: 15px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.08);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.data-field:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.12);
}

.field-label {
  display: block;
  font-size: 0.9rem;
  color: #7f8c8d;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.05rem;
  font-weight: 500;
}

.field-value {
  font-size: 1.1rem;
  color: #2c3e50;
  font-weight: 400;
  word-break: break-word;
  line-height: 1.4;
}

@media (max-width: 768px) {
  .brand-title {
    font-size: 2.5rem;
    letter-spacing: 0.3rem;
  }
  
  .header {
    padding: 2rem 1rem 1.5rem;
  }
  
  .camera-section, .image-section, .result-section {
    padding: 0 1rem;
  }
  
  .controls {
    flex-direction: column;
    align-items: center;
  }
  
  .chanel-button {
    width: 100%;
    max-width: 300px;
  }
  
  .data-grid {
    grid-template-columns: 1fr;
  }
}

.field-label {
  font-size: 1rem;
  font-weight: 600;
  color: #34495e;
  margin-bottom: 0.3rem;
  display: block;
  text-align: left;
}

.field-value {
  font-size: 1.2rem;
  font-weight: 400;
  color: #2c3e50;
  word-break: break-word;
  line-height: 1.6;
  background: #f4f6f7;
  padding: 0.75rem 1rem;
  border-radius: 10px;
  box-shadow: inset 0 1px 2px rgba(0,0,0,0.05);
}

.row-layout {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.row-layout .field-label {
  width: 40%;
  margin: 0;
}

.row-layout .field-value {
  width: 58%;
}



</style>