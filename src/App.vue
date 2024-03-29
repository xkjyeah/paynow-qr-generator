<template>
  <header class="no-print">
    <h1>PayNow QR Code Generator</h1>
  </header>

  <main>
    <div class="no-print">
      This page generates a QR for your phone number, or for your UEN.

      Print it out and paste it wherever you run a hawker stall, pasar malam stall,
      or a garage sale!
    </div>

    <div class="field no-print">
      <h4>PayNow destination type</h4>
      <div>
        <label>
          <input type="radio" :checked="mode === 'phone'" @click="mode = 'phone'" />
          Phone number
        </label>
      </div>

      <div>
        <label>
          <input type="radio" :checked="mode === 'uen'" @click="mode = 'uen'" />
          UEN
        </label>
      </div>

    </div>

    <div class="field  no-print">
      <label for="phone">
        <h4>{{ mode === 'phone' ? 'Phone' : 'UEN' }}</h4>
      </label>
      <input type="tel" v-if="mode === 'phone'" id="phone" :value="target"
        @input="event => target = (event.target as any).value" />
      <input type="text" v-else id="phone" :value="target" @input="event => target = (event.target as any).value" />
    </div>

    <div class="field no-print" v-if="mode === 'uen'">
      <label for="reference">
        <h4>Transaction Reference</h4>
      </label>
      <input type="text" id="reference" :value="reference" @input="event => reference = (event.target as any).value" />
    </div>
    <img src="./assets/paynow-logo-2-01.png" ref="logoRef" class="logo" />

    <img :src="qrcodeDataURL" class="generated-QR" />

    <canvas ref="canvasRef" width="600" height="600" style="display: none"></canvas>
    <div class="caption">{{ qrcodeDataTarget.replace(/^\+65/, '') }}</div>
    <div style="text-align: center">
      <br />
      <button class="no-print" @click="print">Print</button>
    </div>
  </main>

  <footer class="no-print">
    <h2>References</h2>
    <ol>
      <li>
        <a
          href="https://www.emvco.com/specifications/emv-qr-code-specification-for-payment-systems-emv-qrcps-merchant-presented-mode/">EMV®
          QR Code Specification for Payment Systems (EMV® QRCPS) Merchant-Presented Mode</a>
      </li>
      <li>
        <a href="https://www.fullstacksys.com/paynow-qr-code-generator">
          Fullstacksys.com Paynow QR Code Generator
        </a> - this website generates QR codes with suggested amounts, but not static codes.
      </li>
      <li>
        Singapore QR Codes for Payments ("SGQR") Specifications v1.7
      </li>
    </ol>
  </footer>
</template>

<style scoped>
@media print {
  .no-print {
    display: none;
  }
}

@media screen {
  .generated-QR {
    max-width: 500px;
    max-height:
      500px;
  }

  header,
  main,
  footer {
    max-width: 520px;
    display: block;
    margin:
      auto;
  }
}

h4 {
  margin-top: 1em;
  margin-bottom: 0;
}

input[type="radio"] {
  margin: 0.8em 0;
}


input[type="text"],
input[type="tel"] {
  padding: 0.3em;
  line-height: 1.5;
  width: 100%;
  box-sizing: border-box;
}

.logo {
  display: hidden;
  height: 1px;
  width: 1px;
}

.generated-QR {
  width: 100%;
  height: auto;
  margin: auto;
  display: block;
}

.caption {
  text-align: center;
}

footer ol li {
  margin-top: 1em;
}

button {
  padding: 1em;
}
</style>

<script setup lang="ts">
import { ref, watch } from 'vue'
import { sortBy, throttle } from 'lodash'
import * as qrcode from 'qrcode'
import crc16ccitt from 'crc/calculators/crc16ccitt'

const mode = ref('phone')
const target = ref('')
const reference = ref('')
const canvasRef = ref(null as null | HTMLCanvasElement)
const logoRef = ref(null as null | HTMLImageElement)

type QRDataComponentValue = null | string | QRData

class QRData {
  components: Record<string, QRDataComponentValue>

  constructor(components: Record<string, QRDataComponentValue>) {
    this.components = components
  }

  toString(): string {
    return sortBy(Object.entries(this.components), ([k, v]: [string, any]) => k)
      .map(([key, comp]: [key: string, comp: QRDataComponentValue]) => {
        if (!key.match(/^[0-9]{2}$/)) {
          throw new Error("Key should be a 2-digit numeric key code")
        }

        if (comp === null) {
          return ''
        }

        const encodedValue = (typeof comp === 'string') ? comp : comp.toString()

        const length = encodedValue.length

        if (length > 99) {
          throw new Error("Encoded length should not exceed 99!")
        }

        return [
          key,
          length.toString().padStart(2, '0'),
          encodedValue,
        ].join('')
      })
      .join('')
  }

  toStringWithCRC(): string {
    const result = this.toString() + '6304'
    const rcode = crc16ccitt(new TextEncoder().encode(result)).toString(16).padStart(4, '0').toUpperCase();

    return result + rcode
  }
}

const print = () => window.print()

const getLocalStorage = (key: string) => {
  try {
    return window.localStorage[key]
  } catch {
    return ''
  }
}

const updateLocalStorage = (key: string, value: string) => {
  try {
    window.localStorage[key] = value
  } catch {
    return ''
  }
}

const qrcodeDataURL = ref(getLocalStorage('phone'))
const qrcodeDataTarget = ref('')

const standardizePhone = (mode: string, s: string) => {
  if (mode !== 'phone') {
    return s;
  } else {
    if (s.match(/^[0-9]{8}$/)) {
      return '+65' + s
    } else {
      return s
    }
  }
}

watch([mode, target, reference, canvasRef], throttle(async () => {
  const finalTarget = standardizePhone(mode.value, target.value)
  const data = new QRData({
    // EMV-Co specs: https://www.emvco.com/specifications/emv-qr-code-specification-for-payment-systems-emv-qrcps-merchant-presented-mode/
    '00': '01', // Payload format indicator: Version we're using
    '01': '11', // Point of initiation method: 11 - static, 12 - dynamic
    '26': new QRData({
      '00': 'SG.PAYNOW',
      '01': mode.value === 'phone' ? '0' : '2',
      '02': finalTarget,
      '03': '1', // Amount is not editabe
      '04': mode.value === 'uen' ? reference.value || null : null,
    }),
    '52': '0000', // Merchant category code
    '53': '702', // ISO 4217 code for SGD
    '58': 'SG',
    '59': 'NA', // Merchant name -- doesn't matter because PayNow has its own lookup
    '60': 'Singapore', // City
  }).toStringWithCRC()

  updateLocalStorage('phone', target.value)

  console.log(data)

  // const dataURL = await qrcode.toDataURL(data, { errorCorrectionLevel: 'H' })

  // qrcodeDataURL.value = dataURL;
  if (canvasRef.value) {
    await qrcode.toCanvas(canvasRef.value, data, { errorCorrectionLevel: 'M', width: 600, color: { dark: '#941c80' } })
    const context = canvasRef.value?.getContext('2d')

    const logoElem = logoRef.value
    if (logoElem) {
      const width = 1323, height = 894;
      const targetWidth = 150, targetHeight = height / width * targetWidth
      context!.fillStyle = 'white'
      context!.fillRect(300 - targetWidth / 2, 300 - targetHeight / 2, targetWidth, targetHeight)
      context!.drawImage(logoElem, 300 - targetWidth / 2, 300 - targetHeight / 2, targetWidth, targetHeight)

      qrcodeDataURL.value = canvasRef.value.toDataURL()
      qrcodeDataTarget.value = finalTarget
    }
  }
}, 500, { leading: true, trailing: true }), { immediate: true })


</script>
