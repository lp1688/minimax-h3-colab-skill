# MiniMax H3 Colab 技能

這是一個完整、可獨立使用的 Codex 技能儲存庫，能將本機參照圖片交給 Google Colab，製作短篇 MiniMax H3 Ref2VA 影片。使用者可以直接 clone、安裝技能、完成 Colab CLI 登入，再使用內附的 runner 或 Shell 啟動器。儲存庫包含：

- `SKILL.md`：Codex 選用此技能時讀取的指示；
- `scripts/runner.py`：管理 session、查詢用量、上傳檔案、批次執行、下載與清理的 runner；
- `assets/MiniMax_H3_Turbo_Colab.ipynb`：在遠端 Colab runtime 執行的推論 Notebook；
- `run_colab_inference.sh`：單支影片的便利啟動器；
- `install.sh`：可攜、可重複執行且預設不覆蓋舊檔的技能安裝程式；
- `tests/test_runner.py`：使用假的 Colab CLI 執行、不會消耗額度的 runner 測試。

本機電腦只負責準備與上傳輸入檔案；模型推論會在 Colab GPU 執行，完成的 MP4 再下載回指定位置。

## 需求

- 已安裝 Codex。設定 `CODEX_HOME` 時，技能會安裝到 `$CODEX_HOME/skills`；未設定時會安裝到 `~/.codex/skills`。
- Python 3.11 以上版本，用於執行內附 runner。runner 只使用 Python 標準函式庫。
- [`uv`](https://docs.astral.sh/uv/)，用來安裝 Colab CLI；也可以用其他方式讓 `colab` 出現在 `PATH`。
- [`google-colab-cli`](https://pypi.org/project/google-colab-cli/)。本 runner 已用 Colab CLI 0.7.4 驗證，使用文件所列的 `version`、`usage`、`new`、`upload`、`exec`、`download`、`stop` 指令。目前的 CLI 版本需要 Python 3.12 以上；`uv` 可以另外管理 CLI 使用的 Python，不影響 runner 的 Python 3.11 以上需求。
- 可使用 Colab compute units 且能配置 GPU 的 Google 帳號。A100 或其他高記憶體 runtime 可能需要對應的 Colab 方案與足夠餘額。

如果系統有安裝選用的 `ffprobe`，runner 會檢查下載的 MP4 是否同時包含視訊與音訊串流。沒有 `ffprobe` 時，仍會檢查檔案存在且大小不為零。

## Clone 與安裝技能

```bash
git clone <repository-url> minimax-h3-colab-skill
cd minimax-h3-colab-skill
./install.sh
```

安裝程式會把必要檔案複製到：

```text
$CODEX_HOME/skills/minimax-h3-colab
```

如果沒有設定 `CODEX_HOME`，目的地是 `~/.codex/skills/minimax-h3-colab`。

需要指定其他技能目錄時（例如測試或使用不同的 Codex 設定），可以明確傳入：

```bash
./install.sh --dest /absolute/path/to/codex/skills
```

預設行為是可重複執行且不會破壞既有安裝。如果目的地已存在，安裝程式會保留原目錄並正常結束。要明確替換既有版本時才使用 `--force`；舊目錄會先移到帶時間戳記的 `.backup.*` 路徑，方便復原：

```bash
./install.sh --force
```

安裝新的技能時，程式會先寫入暫存目錄，再以原子重新命名完成安裝。安裝到 Codex 的內容只有 `SKILL.md`、`scripts/` 與 `assets/`，不會把 Git 資料、測試、README 或輸出檔複製進去。

安裝完成後，請在 Codex 開啟新的回合，或重新整理技能清單（若 Codex 用戶端會快取可用技能）。技能名稱是 `minimax-h3-colab`。

## 安裝並授權 Colab CLI

以使用者工具安裝 CLI：

```bash
uv python install 3.12
uv tool install --python 3.12 google-colab-cli
```

確認 CLI 可以使用：

```bash
colab version
```

runner 預設使用 OAuth2。第一次授權並查詢帳戶餘額：

```bash
colab --auth=oauth2 usage
```

請依照 CLI 印出的網址與驗證碼說明完成流程。Token 會由 CLI 存放在它自己的本機設定中；不要把憑證、Token 或瀏覽器狀態放進本儲存庫、prompt 或工作 manifest。如果環境已經設定 Google Application Default Credentials，可以明確指定另一種認證方式：

```bash
COLAB_AUTH=adc colab --auth=adc usage
```

runner 會把 `--auth="$COLAB_AUTH"` 傳給每一個 Colab 指令；預設值是 `oauth2`。

## 單支影片推論

先建立 UTF-8 文字檔，例如 `prompt.txt`，再執行：

```bash
./run_colab_inference.sh \
  --image /absolute/path/reference_1.png \
  --image /absolute/path/reference_2.jpg \
  --prompt /absolute/path/prompt.txt \
  --output /absolute/path/intro.mp4
```

第一張圖片會在 Notebook 中對應 `<Picture 1>`，第二張對應 `<Picture 2>`，依此類推。每次傳入 1–9 張非空白圖片。prompt 檔案必須是非空白 UTF-8 文字，並會原封不動上傳。如果省略 `--output`，MP4 會存到第一張參照圖片旁邊，檔名加上 `_minimax_h3.mp4`。

預設影片長度是 12 秒；可用 `H3_DURATION_SECONDS` 設為 4–15 秒：

```bash
H3_DURATION_SECONDS=8 ./run_colab_inference.sh \
  --image /absolute/path/reference.png \
  --prompt /absolute/path/prompt.txt
```

Shell 啟動器預設要求 A100 高記憶體 runtime。以下環境變數可以調整設定：

| 變數 | 預設值 | 說明 |
| --- | --- | --- |
| `COLAB_AUTH` | `oauth2` | Colab CLI 認證策略：`oauth2` 或 `adc` |
| `COLAB_GPU` | `A100` | 傳給 `colab new` 的 GPU 名稱 |
| `COLAB_HIGH_MEM` | `1` | 設為 `0` 以不傳入 `--high-mem` |
| `COLAB_EXEC_TIMEOUT` | `3600` | 每個 Notebook 的執行逾時秒數 |
| `COLAB_SESSION_NAME` | 自動產生 | runner 建立的可重用 session 名稱 |
| `H3_DURATION_SECONDS` | `12` | 影片長度，範圍 4–15 秒 |

## 批次推論

要製作多支影片，請把所有工作放在同一份 manifest，讓它們共用一個 Colab session 與已載入的模型。`jobs.json` 範例：

```json
{
  "jobs": [
    {
      "id": "intro",
      "title": "Presenter introduction",
      "reference_images": [
        "/absolute/path/girl-front.png",
        "/absolute/path/girl-side.png"
      ],
      "prompt_file": "/absolute/path/intro-prompt.txt",
      "duration_seconds": 8,
      "output_name": "intro"
    },
    {
      "id": "demo",
      "title": "CLI demo",
      "reference_images": ["/absolute/path/girl-front.png"],
      "prompt": "A concise Ref2VA prompt referring to <Picture 1>.",
      "duration_seconds": 12,
      "output_name": "demo"
    }
  ]
}
```

在 clone 下來的儲存庫中執行：

```bash
python3 scripts/runner.py batch \
  --manifest /absolute/path/jobs.json \
  --gpu A100 \
  --timeout 10800 \
  --output-dir /absolute/path/outputs \
  --progress /absolute/path/outputs/progress.json
```

runner 會在建立 session 前驗證所有本機輸入，逐工作上傳圖片與 prompt、執行 Notebook、下載並驗證 MP4，再處理下一項。後續工作失敗時，已完成的輸出仍會保留。由批次建立的 session 在清理階段會停止，即使中途發生錯誤或逾時也會嘗試停止。若用 `--session NAME` 重用既有 session，想在佇列完成後停止它時，請加上 `--stop-on-complete`。

也可以直接執行已安裝技能中的 runner，不需要留在 clone 下來的儲存庫：

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/minimax-h3-colab/scripts/runner.py" \
  batch --manifest /absolute/path/jobs.json --output-dir /absolute/path/outputs
```

## Prompt 與圖片規則

- 每個工作需要 1–9 張非空白本機參照圖片。
- 圖片順序固定；`<Picture N>` 只依照 1–9 張上傳圖片的順序對應。
- prompt 檔案會以 UTF-8 讀取並作為完整 prompt 傳送；runner 不會翻譯、摘要或改寫內容。
- prompt 不可引用超過該工作的圖片數量的 `<Picture N>`。
- 每支影片長度必須是 4–15 秒。
- 其他程式需要組合導引式 prompt 時，可以使用 runner 的 `compose_ref2va_prompt` helper，產生 `subject_definitions`、`summary`、`retention_analysis`、有順序的 shot 區塊、`overall_soundscape` 與 `non_diegetic_music`。
- Prompt 寫作可參考 [MiniMax H3 Ref2VA prompt guide](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_ref_en.md)。

參照圖片用於描述人物與畫面的視覺身份，不會自動產生 shot 時間；請在 prompt 裡明確描述時間與鏡頭變化。

## 直接使用 Notebook

`assets/MiniMax_H3_Turbo_Colab.ipynb` 就是 runner 上傳並執行的 Notebook，也可以手動在 Colab 開啟來檢查或除錯。runner 透過環境變數選擇 reference mode、遠端圖片路徑、prompt 路徑、影片長度、seed、輸出路徑與執行逾時。請讓本儲存庫中的 Notebook 與 runner 一起維護，以保持相容。

## 疑難排解

| 現象 | 處理方式 |
| --- | --- |
| 找不到 `colab` | 執行 `uv tool install google-colab-cli`，並確認 uv 工具的 bin 目錄在 `PATH`。 |
| OAuth 或 usage 失敗 | 互動式執行 `colab --auth=oauth2 usage`，依照印出的 Google 授權流程完成設定。 |
| GPU 配置失敗 | 檢查 Colab 方案、compute-unit 餘額、要求的 GPU 與高記憶體可用性；可嘗試 `--no-high-mem` 或其他支援的 GPU。 |
| 批次逾時 | 遠端 kernel 可能仍在執行，不要直接重試。runner 會嘗試停止自己建立的 session，並在 progress state 記錄已完成工作。 |
| Prompt 圖片驗證失敗 | 讓 `<Picture N>` 對應該工作 1–9 個 `reference_images` 的 1 起始順序。 |
| 輸出存在但驗證失敗 | 安裝 `ffprobe`，檢查下載檔案是否同時包含視訊與音訊串流。 |

## 離線驗證

不需要 GPU 或 Colab session 即可執行儲存庫測試：

```bash
python3 -m unittest discover -s tests -v
python3 scripts/runner.py --help
./run_colab_inference.sh --help
```

測試會以本機 fake `colab` 與 `ffprobe` 取代真實程式，因此不會消耗 compute units，也不會存取認證資料。

## 範圍與安全

本儲存庫不包含 Google 憑證、Token、模型權重或產生的影片。Colab session 會消耗帳戶的 compute units。請不要把秘密資料放在 prompt、manifest、log 或上傳檔案中；開始真實批次前，先確認 GPU 與 timeout 設定。

## 本 fork 的修改（lp1688）

本 fork 在上游基礎上新增兩類修改，皆已在 Windows 11 + Git Bash 環境、透過 `google-colab-cli` 0.7.4 實際驗證。此技能同樣適用 Kimi Code（以 `./install.sh --dest ~/.kimi-code/skills` 安裝）；技能內容沒有任何 Codex 專屬之處。

### Windows 支援

上游流程以 macOS/Linux 為目標，以下是在 Windows 上驗證過的調整：

- **`python3` 指令**：Windows 的 Python 只有 `python`。在 `~/.local/bin/python3` 放一個 shim（`#!/usr/bin/env bash` 加 `exec python "$@"`），`install.sh` 與 runner 就能原樣運作。
- **`google-colab-cli` 0.7.4 需要兩處修補**（位於 `%APPDATA%\uv\tools\google-colab-cli\Lib\site-packages\colab_cli\`）：
  1. `console.py`：`import termios` / `import tty` 是 Unix 專屬。包上 `try/except ImportError`，失敗時兩個名稱設為 `None`；只有互動式 console/ssh 功能會用到。
  2. `commands/automation.py`：Drive 授權流程會等待 `open("/dev/tty")`，Windows 沒有這個裝置。在 `OSError` 時改為 `sys.stdin.readline()`（EOF 會立即繼續）。
  重新執行 `uv tool install --force` 或 `uv tool upgrade` 會覆蓋這些修補，之後需重新套用。
- **MSYS 路徑轉換**：Git Bash 會把長得像 Unix 路徑的參數改寫，`colab drivemount ... /content/drive` 會變成 `C:/Program Files/Git/content/drive`。直接呼叫 `colab` 且用到遠端路徑時，請先 `export MSYS_NO_PATHCONV=1`。`runner.py` 內部的呼叫不受影響，因為 Python 的 `subprocess` 不做轉換。
- **主控台編碼**：Windows 主控台預設字碼頁（cp950/cp936）會在 runner 印出含其他字元的 Notebook 輸出時讓 Python 崩潰。請以 `PYTHONIOENCODING=utf-8`（或 `PYTHONUTF8=1`）執行 runner。沒設的話，批次可能已全部成功，卻在最後印出時回報誤導性的編碼錯誤。
- **`os.killpg` 修正（已包含在本 fork 的 `runner.py`）**：Windows 沒有 `os.killpg`，會讓逾時終止流程崩潰並掩蓋原始錯誤。runner 現在會退回 `child.terminate()` / `child.kill()`。
- **離線測試**：8 項儲存庫測試中有 4 項在 Windows 失敗，原因是 fake `colab`/`ffprobe` 是無副檔名的 shell script，Windows 無法執行（`WinError 193`），`shutil.which("colab")` 也找不到。這是測試框架的限制；runner 本體已用真實 CLI 驗證過。

### Google Drive 持久模型快取

新 session 正常需要從 Hugging Face 重新下載整套模型（約 38 GiB）。本 fork 新增可選的 Google Drive 持久快取：

```bash
python3 scripts/runner.py batch \
  --manifest /absolute/path/jobs.json \
  --drive-cache /content/drive/MyDrive/minimax-h3-models \
  --output-dir /absolute/path/outputs
```

`single` 與 `H3_DRIVE_CACHE` 環境變數的用法相同。行為如下：

1. runner 以 `colab drivemount` 在 session 上掛載 Google Drive。因為 `colab exec` 即使遠端程式報錯也回傳 exit 0，掛載是否成功改用執行探針印出標記（`H3_DRIVE_MOUNT_OK`）來驗證；失敗最多重試 3 次，仍不成功就放棄批次。
2. Notebook 使用快取前會 assert `os.path.ismount('/content/drive')`，掛載失敗時絕不會把 38 GiB 權重默默寫進 VM 的暫時本機碟。
3. 每個模型檔：快取命中就從 Drive 複製到 VM 本機碟（`shutil.copy2`）；未命中就從 Hugging Face 下載，再經 `.partial` 暫存檔與原子改名推回快取。快取目錄結構與 `ComfyUI/models/` 一致（`diffusion_models/`、`text_encoders/`、`vae/`、`loras/`）。

**每台 VM 都要互動授權一次（重要）。** Drive 授權綁定單一 VM endpoint，不是整個 Google 帳號。每個新的 Colab session 都需要一次瀏覽器授權：

```bash
colab drivemount --session SESSION /content/drive   # 印出授權網址
# 在瀏覽器開啟網址並同意（約 5 秒）
colab drivemount --session SESSION /content/drive   # 第二次執行就會傳遞憑證並完成掛載
```

在 Colab 網頁版掛載過 Drive 並不能免除這個要求，已實測確認新 VM 仍會要求授權。由於 CLI 印出網址後會在 stdin 等待 Enter，非互動的 runner 必須把第一次 `drivemount` 當成取得網址的探針，待使用者授權後再執行第二次。

**實測效能（A100 高記憶體，整套模型約 38 GiB）：**

| 模型來源 | 單批次總耗時（一支 4 秒影片） |
| --- | --- |
| 新 session 從 Hugging Face 下載 | 約 7.5–9.5 分鐘 |
| Drive 快取命中（Drive → VM 複製） | 約 20.5 分鐘 |

Colab 上 Drive FUSE 的讀取遠慢於 Hugging Face CDN，所以快取**不會省時間**。建議用法：

1. **預設**：不加 `--drive-cache`；從 Hugging Face 重抓更快也完全可靠（每個新 session 多花約 0.6 compute units）。
2. **同一工作時段**：用 `--session 名字` 跨批次重用 session。模型只載入一次，這才是真正省時間的做法。用完記得停止 session；閒置的 A100 每小時約扣 6.77 compute units。
3. **`--drive-cache` 留作備援**：當 Hugging Face 限流或連不上時使用，接受較慢的載入時間。

整套模型約需 38 GiB Drive 空間；第一次填充快取前請先確認 Drive 配額。
