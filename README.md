# course-backup（線上課程備份）

A Claude Code skill that backs up an online course **you have paid for** before your access expires — videos, PDF handouts, captions, plain-text transcripts and a table of contents — so you can keep studying offline. Units without captions get a local Whisper transcript, proofread by agents and double-checked by re-listening.

買了線上課，卻只有一段時間能看？這個 skill 讓 Claude 在你已登入的 Chrome 裡讀懂課程結構，把影片、講義、字幕整份搬回自己的電腦，沒有字幕的單元用本機語音辨識補上逐字稿並校對。課程到期之後，你仍然可以慢慢研讀。

## 它會做什麼

說一句「備份這門課程」並附上課程網址，Claude 會：

1. **確認你已登入**：沿用你自己在 Chrome 的登入狀態，不會向你要帳號密碼。
2. **偵察課程結構**：找出單元清單、影片來源、講義下載方式，並檢查影片有沒有 DRM。
3. **整理下載清單**：在頁面內產生一份 `manifest.json`，經剪貼簿交給本機。
4. **依時效順序下載**：講義先抓（簽名網址常只有 10 分鐘），再抓字幕與影片；影片 4 路並行，最高畫質、不轉碼。
5. **逐項驗證**：每支影片比對時長、每份講義比對檔案大小，並產生 `目錄.md`。
6. **補逐字稿**：沒有平台字幕的單元，用本機 faster-whisper 轉寫，音檔不離開你的電腦。
7. **校對與重聽**：每份逐字稿由一個校對 agent 修正同音錯字與術語拼寫；拿不準的字切出那幾秒，用三種辨識設定重聽，一致才改。

落地結構：

```
<你選的資料夾>/<課程名>/
├── 目錄.md
├── CH1 <章名>/
│   ├── 1-1 <單元名>.mp4
│   ├── 1-1 <單元名>.zh.vtt
│   ├── 1-1 <單元名>.逐字稿.txt
│   └── 講義/<原檔名>.pdf
└── ...
```

## 三條界線

這個 skill 刻意只做「個人備份」，以下三件事不會做：

- **不碰帳密**：你自己在 Chrome 登入，Claude 只在頁面內沿用你的登入狀態，token 不會被複製或存檔。
- **不繞 DRM**：偵測到 Widevine、FairPlay、PlayReady 等加密就停下，只備份講義與字幕。
- **不做分享用的打包**：課程內容是講師與平台的著作，備份只供你自己研讀。想推薦給朋友，請分享課程連結或你自己寫的筆記。

各平台的使用條款不同，請先確認你的課程允許下載到個人裝置。

## 需要什麼

| 項目 | 用途 |
|---|---|
| [Claude Code](https://claude.com/claude-code)（桌面 app 或終端機） | 執行 skill；claude.ai 網頁版沒有本機檔案系統，無法使用 |
| Claude in Chrome 擴充套件 | 在你已登入的 Chrome 裡讀取課程頁 |
| macOS | 剪貼簿交接用 `pbpaste`；其他系統要自行替換 |
| `ffmpeg`、`ffprobe`、`curl`、Python 3 | 下載、合併、驗證 |
| `pip3 install faster-whisper opencc-python-reimplemented` | 補逐字稿、簡繁正規化 |
| `yt-dlp`（選用） | 課程影片放在 YouTube 時 |
| `mlx-whisper`（選用，Apple Silicon） | 重聽時多一個獨立模型交叉比對 |

磁碟空間：1080p 影片每分鐘約 4–5 MB，十小時的課約 3–4 GB。

## 安裝

```
git clone https://github.com/goingli0324/course-backup.git ~/.claude/skills/course-backup
```

或透過 marketplace：`/plugin marketplace add goingli0324/muzi-going-skills`

## 使用

在 Chrome 登入課程平台後，對 Claude 說：

```
備份這門課程 https://<課程網址>
```

Claude 會先問你要存在哪個資料夾，其餘自動進行，最後回報驗證結果、哪些逐字稿是模型補的，以及仍需你親耳確認的地方（附時間點）。

## 檔案說明

- `SKILL.md`：工作流與界線
- `references/recon.md`：遇到新平台時的通用偵察法、為什麼只能用剪貼簿把資料帶出瀏覽器
- `references/youtube.md`：YouTube 播放清單與內嵌影片
- `references/manifest-schema.md`：下載清單格式，所有平台的偵察結果都整理成這個格式
- `references/proofread.md`：逐字稿校對流程、校對 agent 的 prompt 模板、常見辨識錯字表、重聽結果的判準
- `scripts/download_manifest.py`：依清單下載、驗證、產生目錄
- `scripts/transcribe.py`：本機 faster-whisper 轉逐字稿
- `scripts/clean_transcript.py`：逐字稿正規化與落地，行數一變就擋下
- `scripts/relisten.py`：把拿不準的句子切出來，用三種辨識設定重聽比對

## 限制

- 目前只內建 YouTube 的適配；其他平台由 Claude 依通用偵察法現場摸索，第一次會比較慢。摸熟後可以把該平台的做法寫成 `references/<平台>.md`。
- 逐字稿校對沒有逐句對照音檔，專有名詞與真同音字（如「章／張」）仍可能有錯，skill 會把不確定的地方列出來。
- 平台改版會讓偵察結果失效，屆時需要重新偵察。

---

## English

**What it does.** Say "back up this course" with the course URL. Claude reuses your logged-in Chrome session (never asks for your password), maps the course structure, checks for DRM, builds a download manifest inside the page, hands it to your machine through the clipboard, then downloads handouts first (their signed URLs expire fastest), captions, and videos at the highest quality without re-encoding. Every video is checked for duration and every PDF for size. Units without captions are transcribed locally with faster-whisper, proofread by one agent per file, and uncertain words are settled by re-listening to the clip with three recognizer settings.

**What it won't do.** It never handles credentials, never bypasses DRM (Widevine, FairPlay, PlayReady), and never packages content for sharing. Course materials belong to their creators; the backup is for your own study. Check your platform's terms before using it.

**Requirements.** Claude Code (desktop or CLI, not claude.ai web), the Claude in Chrome extension, macOS, ffmpeg, Python 3, `faster-whisper`, `opencc-python-reimplemented`; optionally `yt-dlp` and `mlx-whisper`.

**Install.** `git clone https://github.com/goingli0324/course-backup.git ~/.claude/skills/course-backup`, or add the marketplace with `/plugin marketplace add goingli0324/muzi-going-skills`.

## License

MIT — see [LICENSE](LICENSE).
