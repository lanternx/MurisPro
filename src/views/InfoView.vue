<template>
  <div class="main-content bg-image">
    <!-- 主内容区 -->
    <main class="promotion-content">
      <div class="buttons-container">
        <!-- 四个主要按钮 -->
        <a 
          href="https://www.bilibili.com/video/BV1Q12xB6Ep5/" 
          target="_blank" 
          class="promo-button primary"
        >
          <i class="material-icons">play_circle</i>
          <span>视频教程</span>
        </a>

        <button 
          class="promo-button secondary"
          @click="showVerifyModal = true"
        >
          <i class="material-icons">verified</i>
          <span>PDF验证</span>
        </button>

        <a 
          href="https://github.com/lanternx/MurisPro" 
          target="_blank" 
          class="promo-button accent"
        >
          <i class="material-icons">code</i>
          <span>GitHub仓库</span>
        </a>

        <button 
          class="promo-button qr-button"
          @click="showQrModal = true"
        >
          <i class="material-icons">qr_code_2</i>
          <span>实验动物交流群</span>
        </button>
      </div>
    </main>
    <!-- 二维码弹窗 -->
    <div v-if="showQrModal" class="modal-overlay" @click="showQrModal = false">
      <div class="qr-modal" @click.stop>
        <button class="close-btn" @click="showQrModal = false">
          <i class="material-icons">close</i>
        </button>
        <h3>扫码联系</h3>
        <div class="qr-code-container">
          <img src="@/assets/qrcode.png" alt="qrcode" style="max-height: 450px;">
        </div>
        <p class="qr-tip">使用手机QQ扫描二维码</p>
      </div>
    </div>
    <!-- PDF验证弹窗 -->
    <div v-if="showVerifyModal" class="modal-overlay" @click="showVerifyModal = false">
      <div class="verify-modal" @click.stop>
        <button class="close-btn" @click="showVerifyModal = false">
          <i class="material-icons">close</i>
        </button>
        <h3>PDF时间戳验证</h3>
        <p style="color: #666; margin-bottom: 15px;">上传PDF文件，验证其时间戳签名是否有效</p>

        <div class="file-upload-area" @click="triggerPdfUpload" @dragover.prevent @drop.prevent="onPdfDrop">
          <input type="file" ref="pdfFileInput" accept=".pdf" @change="onPdfFileSelect" style="display: none">
          <i class="material-icons" style="font-size: 48px; color: #999;">upload_file</i>
          <p>点击选择或拖拽PDF文件到此处</p>
          <p v-if="verifyFileName" style="color: #2196F3;">已选择: {{ verifyFileName }}</p>
        </div>
        <div style="margin-top: 15px;">
          <button class="btn btn-primary" @click="verifyPdf" :disabled="!verifyFileData || isVerifying">
            <i class="material-icons">verified</i>
            {{ isVerifying ? '验证中...' : '开始验证' }}
          </button>
        </div>

        <div v-if="verifyResult" style="margin-top: 15px;">
          <div :class="verifyResult.valid ? 'verify-result-success' : 'verify-result-fail'">
            <p style="font-size: 18px; font-weight: bold;">
              {{ verifyResult.valid ? '✅ 验证通过' : '❌ 验证失败' }}
            </p>
            <table class="settings-table" style="margin-top: 10px;">
              <tbody>
                <tr>
                  <td style="width: 150px; font-weight: bold;">审计链匹配</td>
                  <td>{{ verifyResult.chainMatch ? '✅ 是' : '❌ 否' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">DB哈希匹配</td>
                  <td>{{ verifyResult.hashMatch ? '✅ 是' : '❌ 否' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">签名验证</td>
                  <td>{{ verifyResult.signatureValid ? '✅ 是' : '❌ 否' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">认证时间</td>
                  <td>{{ verifyResult.time || '无' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">数据库名</td>
                  <td>{{ verifyResult.db_name || '无' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">认证时DB哈希</td>
                  <td style="font-size: 12px; word-break: break-all;">{{ verifyResult.cert_db_hash || '无' }}</td>
                </tr>
                <tr>
                  <td style="font-weight: bold;">审计链</td>
                  <td style="font-size: 12px; word-break: break-all;">{{ verifyResult.cert_record || '无' }}</td>
                </tr>
                <tr v-if="verifyResult.error">
                  <td style="font-weight: bold;">错误信息</td>
                  <td style="color: red;">{{ verifyResult.error }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import axios from 'axios'
import { toast } from 'vue3-toastify'
import 'vue3-toastify/dist/index.css'

// 控制二维码弹窗显示
const showQrModal = ref(false)

// PDF验证相关状态
const showVerifyModal = ref(false)
const pdfFileInput = ref(null)
const verifyFileName = ref('')
const verifyFileData = ref(null)
const isVerifying = ref(false)
const verifyResult = ref(null)

function triggerPdfUpload() {
    pdfFileInput.value.click()
}

function onPdfFileSelect(event) {
    const file = event.target.files[0]
    if (file) loadPdfFile(file)
}

function onPdfDrop(event) {
    const file = event.dataTransfer.files[0]
    if (file && file.type === 'application/pdf') loadPdfFile(file)
}

function loadPdfFile(file) {
    verifyFileName.value = file.name
    verifyResult.value = null
    const reader = new FileReader()
    reader.onload = (e) => {
        verifyFileData.value = new Uint8Array(e.target.result)
    }
    reader.readAsArrayBuffer(file)
}

async function verifyPdf() {
    if (!verifyFileData.value) {
        toast.warning('请先选择PDF文件')
        return
    }
    
    isVerifying.value = true
    verifyResult.value = null
    
    try {
        const response = await axios.post('/api/verify-pdf', verifyFileData.value, {
            headers: { 'Content-Type': 'application/pdf' }
        })
        verifyResult.value = response.data
        
        if (response.data.valid) {
            toast.success('PDF验证通过')
        } else {
            toast.error('PDF验证失败')
        }
    } catch (error) {
        console.error('PDF验证错误:', error)
        const msg = error.response?.data?.error || error.message
        verifyResult.value = { valid: false, hashMatch: false, chainMatch: false, error: msg }
        toast.error('PDF验证出错: ' + msg)
    } finally {
        isVerifying.value = false
    }
}
</script>

<style scoped>

/* 主内容区 */
.promotion-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 2rem;
  text-align: center;
}

.bg-image {
  background: url('@/assets/background.jpg') no-repeat center center fixed;
  background-size: cover;
}

.buttons-container {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 2rem;
  max-width: 800px;
  margin-bottom: 3rem;
}

/* 按钮样式 */
.promo-button {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 16px;
  text-decoration: none;
  color: #333;
  transition: all 0.3s ease;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  min-height: 120px;
}

.promo-button:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.2);
  background: rgba(255, 255, 255, 1);
}

.promo-button i {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.promo-button span {
  font-size: 1.2rem;
  font-weight: 600;
}

/* 不同按钮颜色 */
.promo-button.primary i { color: #2196F3; }
.promo-button.secondary i { color: #FF9800; }
.promo-button.accent i { color: #9C27B0; }
.promo-button.qr-button i { color: #4CAF50; }

/* 弹窗遮罩 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(5px);
}

/* 二维码弹窗 */
.qr-modal {
  background: white;
  border-radius: 20px;
  padding: 2rem;
  text-align: center;
  position: relative;
  max-height: 700px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: modalAppear 0.3s ease-out;
}

/* PDF验证弹窗 */
.verify-modal {
  background: white;
  border-radius: 20px;
  padding: 2rem;
  position: relative;
  width: 90%;
  max-width: 600px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: modalAppear 0.3s ease-out;
}

@keyframes modalAppear {
  from {
    opacity: 0;
    transform: scale(0.8) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

.close-btn {
  position: absolute;
  top: 1rem;
  right: 1rem;
  background: none;
  border: none;
  font-size: 1.5rem;
  color: #666;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 50%;
  transition: background 0.3s;
}

.close-btn:hover {
  background: #f5f5f5;
}

.qr-modal h3 {
  margin-bottom: 1.5rem;
  color: #333;
  font-size: 1.5rem;
}

.qr-code-container {
  padding: 1rem;
  background: #f8f8f8;
  border-radius: 12px;
  margin-bottom: 1rem;
}

.qr-tip {
  color: #666;
  font-size: 0.9rem;
  margin: 0;
}

/* PDF验证样式 */
.file-upload-area {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 30px;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.3s, background-color 0.3s;
}

.file-upload-area:hover {
  border-color: #2196F3;
  background-color: #f0f7ff;
}

.verify-result-success {
  background-color: #e8f5e9;
  border: 1px solid #4caf50;
  border-radius: 8px;
  padding: 15px;
}

.verify-result-fail {
  background-color: #ffebee;
  border: 1px solid #f44336;
  border-radius: 8px;
  padding: 15px;
}

.settings-table {
  width: 100%;
  border-collapse: collapse;
}

.settings-table td {
  padding: 8px 12px;
  border-bottom: 1px solid #eee;
}

.btn-primary {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
}

.btn-primary:hover {
  background: #1976D2;
}

.btn-primary:disabled {
  background: #ccc;
  cursor: not-allowed;
}

@media (max-width: 767px) {
  .bg-image {
    background-attachment: scroll;
  }
}
</style>
