# 將 QwenASR、停頓觸發辨識與 Windows AMD Vulkan 支援整合到 `jt-live-whisper`

## 摘要
- 在現有 Whisper / Moonshine / faster-whisper 架構旁新增一條 `QwenASR` 管線，支援 `即時` 與 `離線` 兩種模式。
- 把目前「固定 `length_ms + step_ms` 週期辨識」的即時模式，抽象成可切換的 `chunking strategy`；新增預設的 `pause_vad` 停頓觸發模式，保留現有固定週期模式作為相容 fallback。
- Windows 本機新增 `Qwen Vulkan` 後端，作為 AMD / Intel / NVIDIA 共用 GPU 路徑；現有 CUDA / Metal / 遠端 CUDA 不移除。
- `QwenASR` 離線模式必須保留完整時間軸，讓 SRT / VTT / HTML / diarization 仍可用；做法是把「辨識」與「時間軸對齊」拆成兩步。

## 介面與行為變更
- CLI 新增 `--asr qwen`。
- CLI 新增 `--qwen-backend {auto,openvino,vulkan}`。
  - `auto` 預設。
  - Windows 上若偵測到 Vulkan 可用且使用者選 `qwen`，優先走 `vulkan`。
  - 其他平台或 Vulkan 不可用時，走 `openvino`/CPU fallback。
- CLI 新增 `--chunk-mode {fixed,pause_vad}`。
  - 預設改為 `pause_vad`，但 `whisper-stream` 舊路徑若不適合停頓模式，內部自動降回 `fixed`。
- CLI 新增停頓/VAD 參數：
  - `--pause-ms` 預設 `800`
  - `--min-speech-ms` 預設 `250`
  - `--max-segment-ms` 預設 `12000`
  - `--vad-threshold` 預設先沿用現有音量門檻思路，實作上集中到共用設定物件
- WebUI 新增：
  - ASR 引擎選項 `QwenASR`
  - `分段方式` 選項：`停頓自動辨識` / `固定週期`
  - `Qwen 後端` 顯示與選擇：`自動 / OpenVINO / Vulkan`
  - AMD/Intel/NVIDIA 裝置能力提示與 fallback 訊息
- 設定檔新增可持久化欄位：
  - `webui_last.asr`
  - `webui_last.chunk_mode`
  - `webui_last.qwen_backend`
  - `webui_last.pause_ms`
  - `webui_last.min_speech_ms`
  - `webui_last.max_segment_ms`

## 實作重點
- 在 `translate_meeting.py` 建立 `ASRAdapter` 抽象層，至少統一這些能力：
  - `transcribe_file(...) -> segments`
  - `transcribe_live_chunk(...) -> segments/full_text/proc_time`
  - `supports_word_timestamps`
  - `supports_live_pause_chunking`
  - `backend_label`
- 現有 `faster-whisper`、`remote_whisper`、`mlx-whisper` 路徑先包進 adapter，不先大改既有商業邏輯；`QwenASR` 作為新 adapter 接入。
- 即時辨識新增共用 `PauseChunker`：
  - 持續收音並計算 RMS / VAD 狀態。
  - 說話開始後累積音訊。
  - 偵測到連續停頓達 `pause_ms` 才送 ASR。
  - 句中短停不切段。
  - 為避免一直不送出，達 `max_segment_ms` 強制 flush。
  - flush 後清空目前 segment buffer，不再每次重送整個 ring buffer，避免重複辨識與尾句被覆蓋。
- `run_stream_local_whisper()` 與 `run_stream_remote()` 改成共用 chunker/dispatcher；`whisper-stream` 舊路徑保留，但不作為停頓模式主實作。
- `QwenASR` 後端規劃：
  - `OpenVINO`：CPU/通用 fallback，優先支援離線與本機即時。
  - `Vulkan`：Windows GPU 主路徑，針對 AMD/Intel/NVIDIA。
  - 安裝與啟動腳本僅在 Windows 上加入 Vulkan runtime / binary / model 檢查，不擴到 Linux AMD。
- 時間軸策略：
  - 即時模式：以 pause chunk 的累積起訖時間作 segment 邊界，再用 Qwen 結果回填文字。
  - 離線模式：先用 Qwen 取得文字與粗分段，再走對齊步驟產出 `start/end`，確保 SRT/VTT/HTML/diarization 可沿用既有資料結構。
  - 若 Qwen 後端缺少可用對齊器，離線 `qwen` 直接報明確錯誤，不默默輸出無時間軸的假字幕。
- 安裝與偵測：
  - 在 `install.ps1` 新增 Windows `Qwen` 選用安裝邏輯，檢查 Vulkan 可用性與模型檔。
  - 保留既有 NVIDIA CUDA 檢測，不把 AMD 假裝成 CUDA。
  - 啟動時能力探測順序明確化：`Metal`、`CUDA`、`Vulkan`、`CPU`，並將結果回傳給 WebUI。
- WebUI 參數傳遞：
  - 在 `webui.py` 與 `webui.html` 補齊新參數的讀取、顯示、送出與 last-used 記憶。
  - 雙向模式 `en_zh` / `ja_zh` 第一版不開放 `QwenASR`，直接在 UI 與 CLI 提示僅支援現有 faster-whisper/mlx 路徑，避免一次把雙路同步問題也拉進來。

## 測試計畫
- CLI 參數與 fallback
  - `--asr qwen --qwen-backend auto` 在 Windows AMD / Intel / NVIDIA / 無 GPU 四種探測結果下選到正確後端。
  - 不支援平台時會清楚降級或報錯，不會靜默走錯引擎。
  - `--chunk-mode pause_vad`、`--chunk-mode fixed` 互斥正常。
- 即時停頓辨識
  - 連續說話中短停不切段。
  - 停頓超過 `pause_ms` 會觸發一次辨識。
  - 長講話超過 `max_segment_ms` 會強制送出。
  - 靜音時不重複送出空白段。
  - 重複片段過濾仍有效，不因 segment 改制而大量重複。
- 離線 Qwen
  - 音檔可輸出 `segments_data`，並正常產出 SRT/VTT/HTML。
  - `--diarize` 能吃到 Qwen 對齊後的 segment 邊界。
  - 對齊器缺失時，流程在前段即失敗並提示缺少相依，不輸出不完整結果。
- WebUI
  - 新欄位可顯示、送出、重整後保留。
  - 選 `QwenASR` 時正確顯示後端/停頓設定。
  - 雙向模式下 `QwenASR` 被禁用且有原因說明。
- 回歸
  - 原有 Whisper、Moonshine、faster-whisper、遠端 GPU、離線 faster-whisper 路徑仍可啟動。
  - Windows 無 AMD/NVIDIA、macOS Apple Silicon、遠端 GPU 模式至少做 smoke test。

## 假設與預設
- 這一輪的 `QwenASR` 目標平台是 `Windows 本機` 與 `離線音檔`；Linux AMD/ROCm 不納入。
- AMD 支援的定義是 `Windows 上透過 Vulkan 本機推論`，不是 ROCm、不是 CUDA 相容層。
- `QwenASR` 第一版只支援單路即時模式與離線模式，不支援 `en_zh` / `ja_zh` 雙向即時翻譯。
- `pause_vad` 將成為新的預設即時分段策略；若特定引擎不適配，內部自動退回 `fixed`。
- 若參考專案使用的 Vulkan / 對齊 runtime 授權、分發方式或 Python 呼叫方式與本專案不相容，實作時以「保留功能等價」為原則，不強綁同一組封裝，但外部行為維持一致。
