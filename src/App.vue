<template>
  <div class="demo-app">
    <div class="demo-container">
      <header class="demo-header">
        <div class="serial-controls">
          <button class="serial-button" :class="{ connected: isConnected }"
            @click="toggleConnection" :disabled="isConnecting">
            {{ isConnected ? '断开' : '连接设备' }}
          </button>
          <span class="serial-status" :class="{ connected: isConnected }">
            {{ isConnected ? '● 已连接' : '○ 未连接' }}
          </span>

          <template v-if="isConnected">
            <select v-model="deviceType" class="serial-select option-select">
              <option v-for="v in [3,4,5]" :key="v" :value="v">D{{ v }}</option>
            </select>
            <button class="serial-button" @click="writeData" :disabled="!fileData.length || writing">
              {{ writing ? '写入中...' : '写入数据' }}
            </button>
            <button class="serial-button" @click="readData" :disabled="reading">
              {{ reading ? '读取中...' : '读取数据' }}
            </button>
            <button class="serial-button" @click="clearData">清空</button>
          </template>
        </div>

        <div v-if="reading || writing" class="progress-container">
          <div class="progress-bar"><div class="progress-fill" :style="{ width: progress + '%' }"></div></div>
          <span class="progress-text">{{ progress.toFixed(1) }}%</span>
        </div>

        <label class="file-input-wrapper">
          <input type="file" @change="loadFile" class="file-input" />
          <span class="file-button">选择文件</span>
        </label>
        <button class="download-button" :disabled="!fileData.length" @click="downloadFile">下载文件</button>
        <span v-if="fileName" class="file-name">{{ fileName }}</span>
        <label class="switch">
          <input type="checkbox" v-model="editable" />
          <span class="switch-ui"></span>
          <span class="switch-text">编辑模式</span>
        </label>
      </header>

      <main class="demo-main">
        <VueHex ref="viewer" v-model="fileData" v-model:window-offset="offset"
          statusbar="top" :statusbar-layout="statusbarLayout" :theme="theme"
          :cell-class-for-byte="highlight" :is-printable="preset?.isPrintable ?? true"
          :render-ascii="preset?.renderAscii ?? true" :cursor="true" search
          :editable="editable" :bytes-per-row="16" :show-chunk-navigator="true"
          chunk-navigator-placement="left" />
      </main>

      <footer class="demo-footer">
        <div class="demo-controls">
          <label class="control-label">
            <span>ASCII Preset:</span>
            <select v-model="presetKey" class="demo-select">
              <option value="standard">Standard ASCII (0x20-0x7E)</option>
              <option value="latin1">Latin-1 Supplement</option>
              <option value="visibleWhitespace">Visible Whitespace</option>
            </select>
          </label>
          <label class="control-label">
            <span>Highlighting:</span>
            <select v-model="highlightKey" class="demo-select">
              <option value="default">ASCII Categories</option>
              <option value="none">None</option>
              <option value="null-bytes">Null Bytes</option>
              <option value="printable">Printable Characters</option>
            </select>
          </label>
          <label class="control-label">
            <span>Theme:</span>
            <select v-model="theme" class="demo-select">
              <option value="auto">Auto</option>
              <option value="light">Light</option>
              <option value="dark">Dark</option>
              <option value="terminal">Terminal</option>
              <option value="sunset">Sunset</option>
            </select>
          </label>
        </div>
        <a class="github-link" href="https://github.com/vvollers/vuehex" target="_blank" rel="noopener noreferrer">
          <svg viewBox="0 0 24 24"><path fill="currentColor" d="M12 0C5.37 0 0 5.37 0 12c0 5.3 3.44 9.8 8.2 11.39.6.11.82-.26.82-.58 0-.29-.01-1.05-.02-2.06-3.34.73-4.04-1.61-4.04-1.61-.55-1.38-1.34-1.75-1.34-1.75-1.09-.75.08-.74.08-.74 1.2.08 1.84 1.23 1.84 1.23 1.07 1.83 2.8 1.3 3.48.99.11-.77.42-1.3.76-1.6-2.66-.3-5.47-1.33-5.47-5.93 0-1.31.47-2.38 1.23-3.22-.12-.3-.53-1.53.12-3.18 0 0 1.01-.32 3.3 1.23A11.5 11.5 0 0 1 12 6.8c1.02 0 2.05.14 3.01.4 2.29-1.55 3.29-1.23 3.29-1.23.66 1.65.25 2.88.13 3.18.77.84 1.23 1.91 1.23 3.22 0 4.61-2.81 5.62-5.48 5.92.43.37.81 1.1.81 2.22 0 1.61-.02 2.91-.02 3.31 0 .32.22.69.83.57A12.01 12.01 0 0 0 24 12C24 5.37 18.63 0 12 0Z"/></svg>
          <span>github.com/vvollers/vuehex</span>
        </a>
      </footer>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, onUnmounted, watch } from "vue";
import VueHex from "./components/VueHex.vue";
import { VUE_HEX_ASCII_PRESETS, type VueHexStatusBarLayout, type VueHexCellClassPayload } from "./components/vuehex-api";
import "./assets/vuehex.css";
import { AbstractSerialDevice, delimiter } from "webserial-core";

class SPDDevice extends AbstractSerialDevice<string> {
  constructor(baudRate: number = 115200) {
    super({
      baudRate,
      dataBits: 8,
      stopBits: 1,
      parity: "none",
      flowControl: "none",
      bufferSize: 255,
      commandTimeout: 3000,
      parser: delimiter("\n"),
      autoReconnect: false,
      autoReconnectInterval: 1500,
      handshakeTimeout: 2000,
      filters: [],
    });
  }

  protected async handshake(): Promise<boolean> {
    return true;
  }

  async sendCommand(cmd: Uint8Array): Promise<void> {
    await this.send(cmd);
  }

  async sendText(data: string): Promise<void> {
    await this.send(data);
  }
}

const DATA_SIZES: Record<number, number> = { 3: 256, 4: 512, 5: 1024 };

const fileData = ref(new Uint8Array(0));
const offset = ref(0);
const editable = ref(false);
const fileName = ref("");
const theme = ref("auto");
const presetKey = ref("standard");
const highlightKey = ref("default");

const isConnected = ref(false);
const isConnecting = ref(false);
const deviceType = ref(3);
const reading = ref(false);
const writing = ref(false);
const progress = ref(0);

let device: SPDDevice | null = null;
let savedData = new Uint8Array(0);
let isReading = false;

const preset = computed(() => {
  const presets = {
    standard: VUE_HEX_ASCII_PRESETS.standard,
    latin1: VUE_HEX_ASCII_PRESETS.latin1,
    visibleWhitespace: VUE_HEX_ASCII_PRESETS.visibleWhitespace,
  };
  return presets[presetKey.value as keyof typeof presets];
});

const statusbarLayout = computed<VueHexStatusBarLayout>(() => ({
  middle: ["selection"],
  left: ["offset", "hex", "ascii"],
}));

const highlight = computed(() => {
  const map: Record<string, ((payload: VueHexCellClassPayload) => string | undefined) | undefined> = {
    none: undefined,
    "null-bytes": (payload) => payload.byte === 0x00 ? "vuehex-highlight-null" : undefined,
    "printable": (payload) => payload.byte >= 0x20 && payload.byte <= 0x7e ? "vuehex-highlight-printable" : undefined,
    default: undefined,
  };
  return map[highlightKey.value] ?? undefined;
});

const crc8 = (data: Uint8Array): number => {
  let crc = 0;
  for (const b of data) {
    crc ^= b;
    for (let i = 0; i < 8; i++) {
      crc = (crc & 1) ? ((crc >> 1) ^ 0x8C) : (crc >> 1);
    }
  }
  return crc;
};

const buildCmd = (type: number, addr: number, data = 0): Uint8Array => {
  const f = new Uint8Array(9);
  f[0] = 0xAA;
  f[1] = 0x55;
  f[2] = type;
  f[3] = deviceType.value;
  f[4] = (addr >> 8) & 0xFF;
  f[5] = addr & 0xFF;
  f[6] = 0;
  f[7] = data;
  f[8] = crc8(f.slice(2, 8));
  return f;
};

const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

const toggleConnection = async () => {
  if (isConnected.value) {
    await disconnectDevice();
    return;
  }
  await connectDevice();
};

const connectDevice = async () => {
  if (isConnecting.value) return;
  isConnecting.value = true;

  try {
    device = new SPDDevice(115200);

    device.on("serial:connected", () => {
      isConnected.value = true;
      isConnecting.value = false;
    });

    device.on("serial:disconnected", () => {
      isConnected.value = false;
      isConnecting.value = false;
      reading.value = false;
      writing.value = false;
      progress.value = 0;
    });

    device.on("serial:data", (line: string) => {
      handleSerialData(line);
    });

    device.on("serial:error", (err: Error) => {
      isConnected.value = false;
      isConnecting.value = false;
      alert(`串口错误: ${err.message}`);
    });

    await device.connect();

  } catch (e) {
    isConnecting.value = false;
    isConnected.value = false;
    if (e instanceof Error && e.name !== "AbortError") {
      alert(`连接失败: ${e.message}`);
    }
  }
};

const disconnectDevice = async () => {
  if (device) {
    try {
      await device.disconnect();
    } catch (e) {
      //
    }
    device = null;
  }
  isConnected.value = false;
  isConnecting.value = false;
  reading.value = false;
  writing.value = false;
  progress.value = 0;
};

const handleSerialData = (line: string) => {
  const trimmed = line.trim();
  
  const bytes: number[] = [];
  for (let i = 0; i < trimmed.length; i += 2) {
    const byte = parseInt(trimmed.substring(i, i + 2), 16);
    if (!isNaN(byte)) {
      bytes.push(byte);
    }
  }
  
  if (bytes.length === 0) return;
  
  const size = DATA_SIZES[deviceType.value];
  const dataBytes = bytes.slice(0, size);
  
  fileData.value = new Uint8Array(dataBytes);
  offset.value = 0;
  progress.value = 100;
};

const readData = async () => {
  if (!isConnected.value || !device || reading.value || isReading) return;

  editable.value = false;
  isReading = true;
  reading.value = true;
  progress.value = 0;
  fileData.value = new Uint8Array(0);
  offset.value = 0;

  try {
    const cmd = buildCmd(0x05, 0);
    await device.sendCommand(cmd);

    let timeout = 0;
    const maxTimeout = 5000;
    
    while (fileData.value.length === 0 && timeout < maxTimeout) {
      await sleep(100);
      timeout += 100;
      progress.value = Math.min((timeout / maxTimeout) * 100, 90);
    }
    
    progress.value = 100;
    savedData = new Uint8Array(fileData.value);

  } catch (e) {
    alert(`读取失败: ${e instanceof Error ? e.message : String(e)}`);
  } finally {
    reading.value = false;
    isReading = false;
    setTimeout(() => progress.value = 0, 1000);
  }
};

const writeData = async () => {
  if (!isConnected.value || !device || !fileData.value.length || writing.value) return;

  writing.value = true;
  reading.value = false;
  progress.value = 0;

  try {
    const current = fileData.value;
    const size = DATA_SIZES[deviceType.value];

    const changes = savedData.length === 0
      ? Array.from({ length: Math.min(current.length, size) }, (_, i) => ({ addr: i, val: current[i] }))
      : Array.from({ length: Math.min(Math.max(current.length, savedData.length), size) }, (_, i) => {
          const cur = i < current.length ? current[i] : 0;
          const prev = i < savedData.length ? savedData[i] : 0;
          return cur !== prev ? { addr: i, val: cur } : null;
        }).filter(Boolean) as Array<{ addr: number; val: number }>;

    if (!changes.length) {
      writing.value = false;
      return;
    }

    for (let i = 0; i < changes.length && isConnected.value && device; i++) {
      const { addr, val } = changes[i];
      progress.value = ((i + 1) / changes.length) * 100;
      const cmd = buildCmd(0x02, addr, val);
      await device.sendCommand(cmd);
      await sleep(10);
    }

    savedData = new Uint8Array(current);

  } catch (e) {
    alert(`写入失败: ${e instanceof Error ? e.message : String(e)}`);
  } finally {
    writing.value = false;
    setTimeout(() => progress.value = 0, 1000);
  }
};

const clearData = () => {
  if (!fileData.value.length || !confirm("确定清空数据？")) return;
  fileData.value = new Uint8Array(0);
  savedData = new Uint8Array(0);
  offset.value = 0;
};

const loadFile = async (e: Event) => {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;

  fileName.value = file.name;
  try {
    fileData.value = new Uint8Array(await file.arrayBuffer());
    savedData = new Uint8Array(0);
    offset.value = 0;
  } catch (error) {
    fileName.value = "";
    fileData.value = new Uint8Array(0);
    savedData = new Uint8Array(0);
  }
};

const downloadFile = () => {
  if (!fileData.value.length) return;

  const blob = new Blob([fileData.value.buffer], { type: "application/octet-stream" });
  const url = URL.createObjectURL(blob);
  const link = Object.assign(document.createElement("a"), {
    href: url,
    download: fileName.value || "data.bin"
  });
  link.click();
  URL.revokeObjectURL(url);
};

watch(deviceType, () => {
  fileData.value = new Uint8Array(0);
  savedData = new Uint8Array(0);
  offset.value = 0;
  progress.value = 0;
});

onUnmounted(() => {
  if (device) {
    device.disconnect().catch(() => {});
    device = null;
  }
});
</script>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
html, body, #app { width: 100%; height: 100%; overflow: hidden; }
.vuehex-highlight-null { background: rgba(255,0,0,0.6); }
.vuehex-highlight-printable { background: rgba(0,255,0,0.6); }
</style>

<style scoped>
.demo-app {
  --bg: #0d1117; --panel: #161b22; --border: #30363d; --text: #e6edf3; --muted: #7d8590;
  --ctrl-bg: #0d1117; --ctrl-border: #30363d; --hover: #161b22; --focus: #1f6feb;
  display: flex; align-items: center; justify-content: center;
  width: 100%; height: 100vh; background: var(--bg); color: var(--text);
  color-scheme: dark; font-family: "Segoe UI", "Noto Sans", Helvetica, Arial, sans-serif;
}
.demo-container {
  display: flex; flex-direction: column; width: 95vw; max-width: 2000px;
  height: calc(100vh - 2rem); background: var(--panel); border-radius: 8px;
  overflow: hidden; box-shadow: 0 8px 24px rgba(0,0,0,0.4);
}
.demo-header {
  display: flex; align-items: center; gap: 0.75rem; padding: 0.6rem 1.25rem;
  background: var(--panel); border-bottom: 1px solid var(--border); flex-shrink: 0; flex-wrap: wrap;
}
.serial-controls {
  display: flex; align-items: center; gap: 0.4rem; padding: 0.2rem 0.6rem;
  background: var(--ctrl-bg); border: 1px solid var(--border); border-radius: 6px; flex-wrap: wrap;
}
.serial-select {
  min-width: 60px; max-width: 80px; padding: 0.2rem 0.4rem; border: 1px solid transparent;
  border-radius: 4px; font-size: 0.75rem; background: transparent; color: var(--text); cursor: pointer;
}
.option-select { min-width: 50px; max-width: 60px; }
.serial-select:disabled { opacity: 0.5; cursor: not-allowed; }
.serial-select option { background: var(--ctrl-bg); color: var(--text); }
.serial-button {
  padding: 0.2rem 0.6rem; border: 1px solid var(--border); border-radius: 4px;
  font-size: 0.75rem; font-weight: 500; background: var(--ctrl-bg); color: var(--text);
  cursor: pointer; transition: all 0.2s; white-space: nowrap;
}
.serial-button:hover:not(:disabled) { background: var(--hover); border-color: #484f58; }
.serial-button.connected { background: #da3633; border-color: #f85149; color: #fff; }
.serial-button.connected:hover:not(:disabled) { background: #f85149; }
.serial-button:disabled { opacity: 0.5; cursor: not-allowed; }
.serial-status { font-size: 0.7rem; color: var(--muted); white-space: nowrap; }
.serial-status.connected { color: #3fb950; }
.progress-container {
  display: flex; align-items: center; gap: 0.4rem; padding: 0.2rem 0.4rem;
  background: var(--ctrl-bg); border: 1px solid var(--border); border-radius: 4px; min-width: 160px;
}
.progress-bar { flex: 1; height: 6px; background: #30363d; border-radius: 4px; overflow: hidden; }
.progress-fill {
  height: 100%; background: linear-gradient(90deg, #238636, #2ea043);
  border-radius: 4px; transition: width 0.1s ease;
}
.progress-text { font-size: 0.7rem; color: var(--text); min-width: 40px; text-align: right; }
.file-input-wrapper { position: relative; cursor: pointer; }
.file-input { position: absolute; opacity: 0; width: 0; height: 0; }
.file-button, .download-button {
  display: inline-block; padding: 0.3rem 0.7rem; border-radius: 6px; font-size: 0.75rem;
  font-weight: 500; cursor: pointer; border: 1px solid rgba(240,246,252,0.1);
  transition: background 0.2s; white-space: nowrap;
}
.file-button { background: #238636; color: #fff; }
.file-button:hover { background: #2ea043; }
.download-button { background: #1f6feb; color: #fff; }
.download-button:hover:not(:disabled) { background: #388bfd; }
.download-button:disabled { opacity: 0.5; cursor: not-allowed; }
.file-name { color: var(--muted); font-size: 0.75rem; max-width: 100px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.switch { display: flex; align-items: center; gap: 0.3rem; font-size: 0.75rem; user-select: none; cursor: pointer; white-space: nowrap; }
.switch input { position: absolute; opacity: 0; width: 1px; height: 1px; }
.switch-ui {
  width: 28px; height: 16px; border-radius: 999px; background: #30363d;
  border: 1px solid #484f58; position: relative; transition: background 0.2s; flex-shrink: 0;
}
.switch-ui::after {
  content: ""; position: absolute; top: 2px; left: 2px; width: 12px; height: 12px;
  border-radius: 999px; background: #7d8590; transition: transform 0.2s, background 0.2s;
}
.switch input:checked + .switch-ui { background: #238636; border-color: #2ea043; }
.switch input:checked + .switch-ui::after { transform: translateX(12px); background: #fff; }
.switch input:focus-visible + .switch-ui { outline: 2px solid var(--focus); outline-offset: 2px; }
.demo-main { flex: 1; padding: 0.6rem; min-height: 0; }
.demo-footer { position: relative; padding: 0.6rem 1.25rem; background: var(--panel); border-top: 1px solid var(--border); flex-shrink: 0; }
.demo-controls { display: flex; gap: 1rem; align-items: center; justify-content: center; flex-wrap: wrap; }
.control-label { display: flex; align-items: center; gap: 0.3rem; font-size: 0.75rem; }
.control-label span { font-weight: 500; color: var(--muted); }
.demo-select {
  padding: 0.2rem 0.5rem; border: 1px solid var(--ctrl-border); border-radius: 6px;
  font-size: 0.75rem; background: var(--ctrl-bg); color: var(--text); cursor: pointer;
}
.demo-select:hover { border-color: #484f58; background: var(--hover); }
.demo-select:focus { outline: none; border-color: var(--focus); box-shadow: 0 0 0 3px rgba(31,111,235,0.15); }
.demo-select option { background: var(--ctrl-bg); color: var(--text); }
.github-link {
  position: absolute; right: 1.25rem; top: 50%; display: inline-flex; align-items: center;
  gap: 0.3rem; color: var(--muted); font-size: 0.65rem; text-decoration: none;
  transform: translateY(-50%); transition: color 0.2s;
}
.github-link:hover { color: var(--text); }
.github-link svg { width: 0.8rem; height: 0.8rem; }

@media (max-width: 1100px) {
  .demo-header { padding: 0.4rem 0.8rem; gap: 0.4rem; }
  .serial-controls { padding: 0.1rem 0.4rem; }
  .serial-select { min-width: 80px; max-width: 120px; font-size: 0.7rem; }
  .file-button, .download-button { padding: 0.2rem 0.5rem; font-size: 0.7rem; }
  .file-name { max-width: 60px; }
}
@media (max-width: 700px) { .serial-controls { width: 100%; justify-content: center; } }
@media (max-width: 900px) {
  .demo-footer { padding-bottom: 2rem; }
  .github-link { top: auto; bottom: 0.4rem; transform: none; }
}
@media (prefers-color-scheme: light) {
  .demo-app {
    --bg: #f6f8fa; --panel: #fff; --border: #d0d7de; --text: #24292f; --muted: #57606a;
    --ctrl-bg: #fff; --ctrl-border: #d0d7de; --hover: #f6f8fa; --focus: #0969da;
  }
  .demo-container { border: 1px solid var(--border); }
  .serial-controls { background: var(--ctrl-bg); border-color: var(--border); }
  .serial-button { background: var(--ctrl-bg); border-color: var(--border); }
  .serial-button.connected { background: #cf222e; border-color: #cf222e; }
  .serial-button.connected:hover:not(:disabled) { background: #a0111f; }
  .serial-status.connected { color: #1a7f37; }
  .file-button { background: #2da44e; }
  .file-button:hover { background: #2c974b; }
  .download-button { background: #0969da; }
  .download-button:hover:not(:disabled) { background: #0860ca; }
  .switch-ui { background: #d0d7de; border-color: #d0d7de; }
  .switch-ui::after { background: #57606a; }
  .progress-bar { background: #d0d7de; }
  .progress-fill { background: linear-gradient(90deg, #2da44e, #2c974b); }
}
</style>