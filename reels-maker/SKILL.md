---
name: reels-maker
description: 把使用者上傳的影片片段或照片剪成 9:16 直式短影音成片（IG／FB Reels、YouTube Shorts、TikTok）：依腳本排段落、裁成直式、燒繁體中文字幕與開頭字卡、音量標準化，輸出可直接上傳的 MP4、封面和字幕檔。使用者說「做 Reels」「剪成直式短影音」「幫我出片」「上字幕」「做 Shorts」時使用；只要腳本、不要成片時改用 viral-reel-generator。
---

# Reels Maker

交付物：一支可直接上傳的直式 MP4、一張封面 JPG、一份 SRT 字幕檔，以及一段貼文說明。

## 0. 動手前先確認

- **素材**：只用使用者上傳的影片或照片。沒有素材就停下來請對方上傳，不用網路圖片或 AI 生成的人物冒充實拍。
- **腳本**：已有腳本（例如 viral-reel-generator 的輸出）就照用；沒有就先寫逐段腳本（每段：秒數、畫面、字幕或口白），給使用者確認後再剪。
- **平台與長度**：預設 IG／FB Reels，15–45 秒。需要時再出 YouTube Shorts 或 TikTok 版本。各平台長度上限常改，以上傳當下為準。
- **音樂**：只放使用者提供、確定可以商用的音樂。來源不明的流行歌不放，請使用者上傳時從平台音樂庫加。

## 1. 檢查環境

```bash
ffmpeg -version | head -1
fc-list :lang=zh-tw family | head -5
```

- 沒有 ffmpeg：請使用者確認 Settings → Capabilities 的程式碼執行（Code execution and file creation）有打開。
- 沒有繁中字型：請使用者上傳一個 .otf 或 .ttf（例如 Google Fonts 的 Noto Sans TC），放進一個資料夾，後面的 `fontsdir=` 指到那裡。沒有中文字的字型會讓字幕變成方塊。

## 2. 看素材

```bash
ffprobe -v error -show_entries stream=codec_type,width,height,r_frame_rate:format=duration -of compact=p=0 INPUT
```

記下每個檔的長度、直式或橫式、有沒有聲音。手機直拍常帶旋轉標記，ffmpeg 會自動轉正，不用另外處理。

## 3. 先排剪輯表，給使用者確認

把腳本每一段對到素材和時間點：

| 段 | 素材 | 入點 | 出點 | 畫面上的字 |
|---|---|---|---|---|

各段長度加總要等於目標長度。使用者說可以再開始算圖；算圖慢，先對齊比重做省時間。

## 4. 每段統一成 1080×1920、30fps、有聲音軌

所有段落都要同樣的解析度、幀率、像素格式和音訊格式，後面才能直接串接。

**直式影片**（填滿並裁掉多出的部分）：

```bash
ffmpeg -y -ss IN -to OUT -i clip.mp4 \
  -vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,setsar=1,fps=30" \
  -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p \
  -c:a aac -b:a 160k -ar 48000 -ac 2 seg01.mp4
```

**橫式影片**（不想裁掉兩邊時，用同一段畫面的模糊版當背景）：

```bash
ffmpeg -y -ss IN -to OUT -i wide.mp4 -filter_complex \
  "[0:v]split[a][b];[a]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=20:2[bg];[b]scale=1080:-2[fg];[bg][fg]overlay=(W-w)/2:(H-h)/2,setsar=1,fps=30[v]" \
  -map "[v]" -map 0:a -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p \
  -c:a aac -b:a 160k -ar 48000 -ac 2 seg02.mp4
```

**沒有聲音的影片**：補一條靜音軌，否則串接會失敗。

```bash
ffmpeg -y -ss IN -to OUT -i silent.mp4 -f lavfi -i anullsrc=r=48000:cl=stereo \
  -vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,setsar=1,fps=30" \
  -map 0:v -map 1:a -shortest -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p \
  -c:a aac -b:a 160k -ar 48000 -ac 2 seg03.mp4
```

**照片**（每張 2–3 秒，加一點慢速推近）：

```bash
ffmpeg -y -loop 1 -t 3 -i photo.jpg -f lavfi -i anullsrc=r=48000:cl=stereo \
  -vf "scale=1188:2112:force_original_aspect_ratio=increase,crop=1188:2112,zoompan=z='min(zoom+0.0008,1.08)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=90:s=1080x1920:fps=30,setsar=1" \
  -map 0:v -map 1:a -t 3 -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p \
  -c:a aac -b:a 160k -ar 48000 -ac 2 seg04.mp4
```

## 5. 串接

```bash
for f in seg*.mp4; do echo "file '$f'"; done > list.txt
ffmpeg -y -f concat -safe 0 -i list.txt -c copy joined.mp4
```

## 6. 繁中字幕與開頭字卡

- 一句 8–14 個字，畫面上最多兩行；口白長就拆成多句，不要縮小字體硬塞。
- 字幕放中下方，避開最底部約兩成（平台的說明文字和按鈕）和右側按鈕區。
- 開頭字卡（鉤子）放上方三分之一，0–2.5 秒出現，字數 10 個以內。
- 不用 emoji 當字幕裝飾，不在每個字加特效。

用下面的 Python 從剪輯表產生 ASS（燒進畫面用）和 SRT（上傳平台用）。`FONT` 填 `fc-list` 看到的字型名稱：

```python
FONT = "Noto Sans CJK TC"
hook = (0.0, 2.5, "避震器軟硬怎麼調")          # 開頭字卡，沒有就設成 None
lines = [                                       # (開始秒, 結束秒, 字幕)
    (0.0, 2.5, "很多人一裝完就不再調"),
    (2.5, 6.0, "其實阻尼只要轉三格"),
]

def ts_ass(t):
    h, rem = divmod(t, 3600); m, s = divmod(rem, 60)
    return f"{int(h)}:{int(m):02d}:{s:05.2f}"

def ts_srt(t):
    ms = round(t * 1000); h, ms = divmod(ms, 3600000); m, ms = divmod(ms, 60000); s, ms = divmod(ms, 1000)
    return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"

ass = f"""[Script Info]
ScriptType: v4.00+
PlayResX: 1080
PlayResY: 1920
WrapStyle: 0

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Caption,{FONT},66,&H00FFFFFF,&H00FFFFFF,&H00000000,&H64000000,1,0,0,0,100,100,0,0,1,5,0,2,90,150,420,1
Style: Hook,{FONT},92,&H0000E1FF,&H0000E1FF,&H00000000,&H64000000,1,0,0,0,100,100,0,0,1,7,0,8,80,80,300,1

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
"""
if hook:
    ass += f"Dialogue: 1,{ts_ass(hook[0])},{ts_ass(hook[1])},Hook,,0,0,0,,{{\\fad(120,120)}}{hook[2]}\n"
for a, b, text in lines:
    ass += f"Dialogue: 0,{ts_ass(a)},{ts_ass(b)},Caption,,0,0,0,,{text}\n"
open("captions.ass", "w", encoding="utf-8").write(ass)
open("captions.srt", "w", encoding="utf-8").write(
    "\n".join(f"{i}\n{ts_srt(a)} --> {ts_srt(b)}\n{t}\n" for i, (a, b, t) in enumerate(lines, 1)))
```

字幕顏色是 ASS 的 `&HAABBGGRR` 格式：上面 Caption 是白字黑框，Hook 是黃字黑框。換品牌色時記得 BGR 順序是反過來的。

## 7. 燒字幕、統一音量、輸出

```bash
ffmpeg -y -i joined.mp4 \
  -vf "ass=captions.ass:fontsdir=/usr/share/fonts" \
  -af "loudnorm=I=-14:TP=-1.5:LRA=11" \
  -c:v libx264 -preset medium -crf 21 -maxrate 6M -bufsize 12M -profile:v high -pix_fmt yuv420p \
  -c:a aac -b:a 160k -ar 48000 -movflags +faststart reel.mp4
```

- 使用者自己上傳的字型放在別的資料夾時，把 `fontsdir=` 改成那個路徑。
- 成品要小於 30MB（claude.ai 單檔上傳、下載的上限）。超過就把 `-crf` 加 2、或把 `-maxrate` 調低後重出。60 秒以內通常在 15–25MB。

## 8. 封面

開頭字卡那一格通常就是好封面：

```bash
ffmpeg -y -ss 1.2 -i reel.mp4 -frames:v 1 -q:v 2 cover.jpg
```

## 9. 交付前一定要檢查

1. `ffprobe reel.mp4` 確認 1080×1920、30fps、H.264、AAC、長度正確，`ls -lh` 確認小於 30MB。
2. 抽開頭、中間、結尾三張畫面存成 PNG，實際看過：字有沒有變方塊、有沒有被裁到、有沒有壓到人臉或產品重點、字幕和畫面時間對不對。

   ```bash
   for t in 1 DUR_HALF DUR_END; do ffmpeg -y -ss $t -i reel.mp4 -frames:v 1 check_$t.png; done
   ```

3. 有問題修一次再交，不要把沒看過的成品交出去。

## 10. 交付

- 傳出 `reel.mp4`、`cover.jpg`、`captions.srt`。
- 附一段貼文說明：第一句是第二個鉤子（不要只是摘要），內文一兩句重點，3–5 個相關 hashtag。有 socialpost 或 humanizer-zh-tw 時，用它們潤稿。
- 提醒使用者：音樂在平台上傳時從平台音樂庫加；發布前看一次手機預覽，確認字幕沒有被平台介面擋住。
