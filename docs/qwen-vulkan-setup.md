# QwenASR Vulkan 設定

若要在 Windows 上使用 `QwenASR` 的 Vulkan 後端（適用 AMD / Intel / NVIDIA GPU），需要另外準備 `chatllm.cpp` 的 Windows 二進位與 `qwen3-asr-1.7b.bin` 模型。

## 必要檔案

將下列檔案放到專案根目錄的 `chatllm/`：

- `main.exe`
- `libchatllm.dll`（可選，未使用於目前版本，但可一起保留）
- `ggml-vulkan.dll`
- 其他 `chatllm.cpp` release 內附帶的 `ggml-*.dll`

模型檔可放在任一位置：

- `GPUModel/qwen3-asr-1.7b.bin`
- `models/qwen3-asr-1.7b.bin`

也可透過 `config.json` 或環境變數指定。

## config.json 設定

```json
{
  "qwen_vulkan": {
    "chatllm_dir": "D:\\Projects\\Private\\jt-live-whisper\\chatllm",
    "model_path": "D:\\Projects\\Private\\jt-live-whisper\\GPUModel\\qwen3-asr-1.7b.bin",
    "device_id": 0
  }
}
```

## 環境變數

- `JT_QWEN_VULKAN_DIR`
- `JT_QWEN_VULKAN_MODEL`
- `JT_QWEN_VULKAN_DEVICE`

環境變數優先於 `config.json`。

## 啟用方式

- CLI：`--asr qwen --qwen-backend vulkan`
- WebUI：`ASR 引擎 = QwenASR`，`Qwen 後端 = Vulkan`

`--qwen-backend auto` 在本機沒有 CUDA、且偵測到可用 Vulkan 後端時，會自動改走 Vulkan。
