---
name: "video-highlight-clipper"
description: "當 使用者提供一支影片、加上一份文字腳本（腳本裡通常會自己標記重點），要求剪出精華短片、或要幫影片上字幕/字卡時使用。流程：語音轉文字→比對腳本重點的時間碼→依目標長度篩選→去贅字→單一濾鏡圖剪輯串接成短片（避免音畫不同步）→視需要製作字卡與字幕燒錄；大檔案時改在 使用者電腦端執行抽音軌與剪輯，網路受限時改請 使用者在自己的 Terminal 執行轉錄。"
---

# 影片精華剪輯 / 字卡 / 字幕 Skill

## 使用時機

使用者給一支影片＋一份文字腳本，要求「剪精華」「剪短影片」「抓重點」之類的請求時套用第一段主流程（剪輯）。同一支素材後續常會接著要求「上字卡」「上字幕」，這兩個是獨立但常常串接在剪輯之後的步驟，也都屬於本 skill 範圍。

腳本通常是**純文字、沒有時間碼**，使用者會在腳本裡自己標記出他覺得是重點的段落（常見標記方式：`**粗體**`、【】、《》、或整行用「★」「重點：」「金句：」開頭）。如果完全沒有標記，就把整份腳本文字都當作候選，由 Claude 自己判斷金句／口播重點。

目標短片長度預設 **30-60 秒**，除非 使用者這次另外指定長度。

## 大檔案原則（很重要，先判斷再動手）

聊天上傳官方上限是每檔 500MB；就算改用電腦裝置橋接來傳檔案，單檔也大約是 400MB、單次總量 500MB 左右的量級。一支完整的直播錄影或長版原始素材常常是好幾 GB，這兩條路都走不通，**不要預設請 使用者把整支原始影片傳上來**。

正確原則：**原始影片留在它原本的地方，不整支搬動**。實際的判斷邏輯：

1. 先問清楚（或推斷）影片目前放在哪裡：使用者自己的電腦／已同步到電腦的雲端硬碟資料夾／純雲端連結（沒同步到任何本機硬碟）。
2. 如果影片在 使用者電腦上（或已同步成一個本機資料夾），**請 使用者連結那個資料夾**，接下來的操作都改成用 device 工具在他的電腦上執行，完全不上傳、不搬移那支大檔案：
   - 用 `extract_audio.py`（見下方）在他電腦上只抽出純音軌（通常只有原始檔案的 2-5% 大小），這個小音訊檔才是唯一需要跨到雲端工作環境的東西。
   - **⚠️ 音軌（包含後續每一次重新轉錄，例如上字幕前的重新轉錄）一定要從「目前這一版影片檔案本身」用 `extract_audio.py` 抽，不能拿現場另外用別的裝置錄的音檔（外接收音機、分軌錄音機的 Mic 1/Mic 2/Stereo Mix 之類）來代替**——即使那個音檔看起來更小、更方便，只要它不是從同一支影片檔案切出來的，時間軸幾乎一定跟影片對不上，拿去轉錄定位出來的時間碼套回影片會對錯位置或造成字幕/字卡跑掉。
   - 轉錄、定位重點時間碼、去贅字、（新增）重新對齊字幕時間碼這幾步都只處理文字和時間碼，資料量很小，一樣在雲端工作環境做，效率比較好。
   - 剪輯/燒字幕字卡這種需要處理影片本體的步驟，一律用 device 工具在「原始影片實際所在的那台電腦」執行，影片本體從頭到尾都不離開 使用者的電腦。
3. 如果影片純粹在雲端（Google 雲端硬碟、公司 NAS 等）且沒有同步成本機資料夾：優先建議 使用者把該資料夾同步到電腦上，同步完就回到情境 2 的做法。有 Google Drive 之類的連接工具時，只拿來確認檔案清單/大小（`get_file_metadata`／`search_files`），**絕對不要**用它的下載/讀取內容功能去搬音訊或影片本體——那類工具是把檔案內容轉成文字塞進對話裡，幾百 MB 的媒體檔案會直接爆量。
4. 只有在影片本身明顯小於 500MB（例如已經是剪好的短版素材）時，才維持最簡單的做法：直接請 使用者上傳到對話中，全部在雲端工作環境處理。

## 整體流程

以下以「影片已經在雲端工作環境可以直接讀取」為預設寫法；套用大檔案原則情境 2 時，把涉及影片本體（抽音軌、剪輯、燒字幕/字卡）的步驟改成用 device 工具在 使用者電腦上執行，其餘（轉錄、定位、清理、對齊）維持在雲端工作環境做。

### A. 剪精華短片

1. **確認素材**：拿到影片檔案與腳本文字，先看檔案大小套用大檔案原則。若腳本內容或影片位置不清楚，先問 使用者一次，不要用假設的檔案硬跑。
2. **環境準備**：檢查 `ffmpeg`、`python3`；安裝 `faster-whisper`、`opencc-python-reimplemented`（`pip3 install --break-system-packages faster-whisper opencc-python-reimplemented`）。把下方腳本寫到工作目錄的 `scripts/`。
3. **（大檔案模式）抽出純音軌**：用 `extract_audio.py` 在 使用者電腦上抽出音軌，傳進雲端工作環境。
4. **轉錄**：`transcribe.py` 產生帶時間碼的逐字稿 JSON（`--lang zh --model small`，精準度要求高時升級 `medium`）。
5. **抓出重點候選清單**：依標記慣例抓 使用者自己標記的段落（`priority: "user"`），視覆蓋長度需要自己補充候選（`priority: "ai"`），寫成 `highlights.json`。
6. **定位時間碼並套用長度預算**：`build_segments.py` 定位時間碼、依 user 優先累加時長，輸出 `segments.json`。
7. **去除贅字與過長靜音**：`apply_cleaning.py` 逐段比對贅字/口吃/過長停頓，輸出 `final_segments.json`（全程規則比對，不吃模型 token）。
8. **人工複核清單**：把 `final_segments.json` 簡短列給 使用者看一眼，尤其標示 Claude 自行判斷的段落。大檔案模式下把這個小 JSON 傳回 使用者電腦。
9. **執行剪輯**：`cut_clip.py`（見下方，已改為單一濾鏡圖架構，取代會造成音畫不同步的舊版）用 `final_segments.json` 剪出並串接成最終短片。大檔案模式在 使用者電腦上用 device 工具執行，成品直接寫進他的資料夾；小檔案模式在雲端工作環境執行後用 SendUserFile 送出。
10. **交付**：附上片段清單（時間碼＋理由）方便複核。若 使用者對某段不滿意，回到步驟 5-6 調整 `highlights.json` 重跑即可。

### B. 上字卡（KeyCard）— 剪輯定案後才做

11. **抓字卡候選**：從逐字稿裡挑「情緒轉折點、關鍵數字、結論句」等適合做強調花字的片刻（不是逐字上字卡，是稀疏地在關鍵節點出現），先列出候選連同時間碼與建議文字，給 使用者看。
12. **文字精簡到底**：**預設文字風格是極簡關鍵詞，不是完整句子**——只抓最核心的詞或短語，把前導語、連接詞、修飾語全部去掉（例如「這個蠻慘的完全沒有發芽」只留「完全沒有發芽」；「很不幸的第二年又崩潰一崩」只留「崩潰」）。這是 使用者明確要求過的標準做法，之後所有專案的字卡預設都照這個精簡度做，不用每次重新確認，除非 使用者特別要求更完整的句子。
13. **視覺風格**：預設走「綜藝字卡」風格——大字（fontsize ~150，依畫布解析度調整）、粗外框＋陰影、高對比亮色（例如黃底黑框）、置中（Alignment 5，MarginV 0），並加上 pop-in 彈跳進場動畫。細節與 ASS 範例見下方「字卡與字幕技術細節」。想換風格再依 使用者回饋調整（例如之前調整過「不夠大、不夠綜藝」「置中」）。
14. **燒錄確認**：燒成小檔案版本（見下方 30MB 限制）送給 使用者看，依回饋迭代 fontsize/顏色/位置/動畫，不用整個流程重跑，只要改 `.ass` 檔案裡的 Style 行或 Dialogue 前綴的 `{...}` override tag 再重新燒錄即可。

### C. 上字幕（Caption）— 通常是最後一步

15. **一律對「目前最終版影片」重新轉錄**：即使先前已經有逐字稿，只要影片後來又調整過（哪怕只是修過音畫同步），都要用 `extract_audio.py` + `transcribe.py` 對目前這一版重新跑一次，不能沿用舊逐字稿——duration 差 0.3 秒看起來很小，但分攤到十幾二十行字幕上會有能感覺出來的斷句/對齊偏移。
16. **新轉錄有辨識誤差時，用 `retime_captions.py` 合併新舊逐字稿**：如果手上還留著一份文字正確（例如更早一次已校對過、或就是 使用者提供的原始腳本文字整理成的逐字稿）但時間碼可能過時的逐字稿，用 `retime_captions.py` 把新逐字稿的準確時間碼套用到舊逐字稿的正確文字上（字元級 diff 對齊＋未對齊處線性內插），避免畫面上出現辨識錯字。沒有舊逐字稿可用時，才直接用新逐字稿的文字（就要接受偶爾的辨識誤差，可人工抽查修正明顯的錯字）。
17. **斷行參數**：預設 `--char-limit 8 --gap-threshold 0.25`（依語感停頓斷行，單行不超過 8 字）。曾經試過 `--char-limit 10`，在雙字詞正中間斷行（因為 whisper 中文是逐字辨識、沒有詞界概念），效果較差；8 是目前驗證過比較順的預設值，仍建議燒錄後人工看一次，覺得斷得不順可微調重跑。
18. **字幕與字卡合併在同一份 `.ass`**：兩個是不同 Style（`Caption` 貼底部密集字幕、`KeyCard` 置中大字強調），用 `Layer` 區分（Caption 用 Layer 0，KeyCard 用 Layer 1）就能同時存在畫面上不互相干擾。組裝方式：寫一份共用的 `[Script Info]`+`[V4+ Styles]`（兩個 Style 都列進去）+`[Events]` 表頭，把 `retime_captions.py` 產生的 Dialogue 行接上，再把 `KeyCard` 的 Dialogue 行接上。
19. **燒錄**：`ffmpeg -i <input> -vf "ass=<path.ass>" -c:v libx264 -preset veryfast -crf 18 -c:a copy <output>`（音訊只是 stream copy，因為只有畫面被疊字幕/字卡覆蓋，音軌本身沒變）。
20. **交付**：全品質版本留在 使用者的資料夾，另外燒一份縮小版（見下方 30MB 限制）送進對話讓他確認。

## 標記慣例辨識細節

解析腳本文字時，依序嘗試以下標記（找到哪種就用哪種，也可能混用）：
- `**文字**`（Markdown 粗體）
- `【文字】`、《文字》
- 整行以 `★`、`重點：`、`金句：`、`Highlight:` 開頭的內容
- 都沒有的話，跟 使用者確認一次他是怎麼標記的，或直接把整份文字交給 Claude 自己判斷金句/重點段落

## 輔助腳本

以下腳本要先寫到工作目錄（例如 `./scripts/`）才能執行；大檔案模式下，`extract_audio.py` 和 `cut_clip.py` 這兩支還要另外寫一份到 使用者電腦那端（用 device 工具執行）。`clean_segment.py`／`apply_cleaning.py`／`retime_captions.py` 全程是規則/演算法比對，不呼叫 Claude，不吃額外模型 token。

### scripts/extract_audio.py

```python
#!/usr/bin/env python3
"""
extract_audio.py — 只抽出影片的純音軌（不含畫面），用來在「大檔案模式」下
只把一個很小的音訊檔案送去做語音轉文字，原始大檔案完全不用搬動。

⚠️ 一定要對「目前這一版影片檔案本身」執行這支腳本（包括每一次重新轉錄，
例如上字幕前的重新轉錄），不要拿現場另外錄的音檔（外接收音機、分軌 Mic 檔、
Mixdown 等）取代——那些檔案的時間軸通常跟影片對不上。

用法:
    python3 extract_audio.py <來源影片> <輸出mp3> [--bitrate 96k]
"""
import argparse
import subprocess
import sys


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("source", help="來源影片路徑")
    ap.add_argument("output", help="輸出音訊檔路徑（建議 .mp3）")
    ap.add_argument("--bitrate", default="96k", help="音訊位元率，人聲轉錄不需要太高")
    args = ap.parse_args()

    cmd = [
        "ffmpeg", "-y", "-i", args.source,
        "-vn", "-acodec", "libmp3lame", "-b:a", args.bitrate,
        args.output,
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print("抽取音軌失敗:", result.stderr[-2000:], file=sys.stderr)
        raise SystemExit(1)

    import os
    src_size = os.path.getsize(args.source)
    out_size = os.path.getsize(args.output)
    print(
        f"完成！原始檔案 {src_size/1e6:.1f}MB -> 音軌 {out_size/1e6:.1f}MB "
        f"（約為原檔 {out_size/src_size*100:.1f}%）",
        file=sys.stderr,
    )


if __name__ == "__main__":
    main()
```

### scripts/transcribe.py

```python
#!/usr/bin/env python3
"""
transcribe.py — 用 faster-whisper 把影片/音訊轉成「帶時間碼的逐字稿」。

用法:
    python3 transcribe.py <影片或音訊檔> <輸出json> [--model small] [--lang zh]

輸出 JSON 格式:
{
  "text": "完整逐字稿文字",
  "words": [{"word": "大", "start": 0.12, "end": 0.34}, ...]
}

words 是字元/詞級時間碼列表，供 locate_highlight.py／retime_captions.py 做文字比對定位用。
"""
import argparse
import json
import sys


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("source", help="影片或音訊檔路徑")
    ap.add_argument("output", help="輸出 JSON 路徑")
    ap.add_argument("--model", default="small", help="whisper 模型大小 (tiny/base/small/medium/large-v3)")
    ap.add_argument("--lang", default=None, help="語言代碼，例如 zh；不指定則自動偵測")
    args = ap.parse_args()

    from faster_whisper import WhisperModel
    from opencc import OpenCC

    cc = OpenCC("s2twp")

    print(f"載入模型 {args.model} ...", file=sys.stderr)
    model = WhisperModel(args.model, device="cpu", compute_type="int8")

    print(f"開始轉錄 {args.source} ...", file=sys.stderr)
    segments, info = model.transcribe(
        args.source,
        language=args.lang,
        word_timestamps=True,
        vad_filter=True,
    )

    words = []
    full_text_parts = []
    for seg in segments:
        full_text_parts.append(seg.text)
        if seg.words:
            for w in seg.words:
                words.append({
                    "word": cc.convert(w.word),
                    "start": round(w.start, 3),
                    "end": round(w.end, 3),
                })
        print(f"  [{seg.start:.1f}s - {seg.end:.1f}s] {cc.convert(seg.text)}", file=sys.stderr)

    result = {
        "text": cc.convert("".join(full_text_parts).strip()),
        "language": info.language,
        "duration": info.duration,
        "words": words,
    }

    with open(args.output, "w", encoding="utf-8") as f:
        json.dump(result, f, ensure_ascii=False, indent=2)

    print(f"\n完成！逐字稿已存到 {args.output}（共 {len(words)} 個詞/字）", file=sys.stderr)


if __name__ == "__main__":
    main()
```

### scripts/locate_highlight.py

```python
#!/usr/bin/env python3
"""
locate_highlight.py — 把「使用者腳本裡標記的一段文字」定位回影片的時間碼。
用字元級 difflib 模糊比對，允許 ASR 辨識誤差（同音字、漏字），只要大部分字元對得上就能定位。

用法（當作 library 被 import，也可單獨測試）:
    python3 locate_highlight.py <transcript.json> "腳本裡的一段重點文字"
"""
import json
import re
import sys
import difflib


def normalize(s: str) -> str:
    return re.sub(r"[\s，。、！？「」『』,.!?\"'()（）\-—…:：;；]", "", s)


def build_char_index(words):
    chars = []
    timestamps = []
    for w in words:
        token = normalize(w["word"])
        if not token:
            continue
        n = len(token)
        for i, ch in enumerate(token):
            chars.append(ch)
            frac_start = w["start"] + (w["end"] - w["start"]) * i / max(n, 1)
            frac_end = w["start"] + (w["end"] - w["start"]) * (i + 1) / max(n, 1)
            timestamps.append((frac_start, frac_end))
    return "".join(chars), timestamps


def locate(transcript_words, query_text, min_ratio=0.45):
    full_chars, timestamps = build_char_index(transcript_words)
    query = normalize(query_text)
    if not query or not full_chars:
        return None

    sm = difflib.SequenceMatcher(None, full_chars, query, autojunk=False)
    blocks = [b for b in sm.get_matching_blocks() if b.size > 0]
    if not blocks:
        return None

    min_block_size = 1 if len(query) <= 4 else 2
    good_blocks = [b for b in blocks if b.size >= min_block_size]
    if not good_blocks:
        good_blocks = blocks

    start_idx = min(b.a for b in good_blocks)
    end_idx = max(b.a + b.size for b in good_blocks) - 1
    end_idx = min(end_idx, len(timestamps) - 1)

    matched_chars = sum(b.size for b in good_blocks)
    ratio = matched_chars / max(len(query), 1)

    if ratio < min_ratio:
        return None

    start_time = timestamps[start_idx][0]
    end_time = timestamps[end_idx][1]
    return (round(start_time, 2), round(end_time, 2), round(ratio, 2))


def main():
    if len(sys.argv) < 3:
        print("用法: python3 locate_highlight.py <transcript.json> \"查詢文字\"", file=sys.stderr)
        raise SystemExit(1)
    with open(sys.argv[1], "r", encoding="utf-8") as f:
        data = json.load(f)
    result = locate(data["words"], sys.argv[2])
    if result is None:
        print("找不到匹配（可能腳本文字與逐字稿差異太大）")
    else:
        start, end, ratio = result
        print(f"定位結果: {start}s - {end}s（匹配度 {ratio:.0%}）")


if __name__ == "__main__":
    main()
```

### scripts/build_segments.py

```python
#!/usr/bin/env python3
"""
build_segments.py — 把「Claude 判斷出的重點文字清單」轉成 cut_clip.py 可用的 segments.json，
並依目標總長度自動取捨。

用法:
    python3 build_segments.py <transcript.json> <highlights.json> <輸出segments.json> \
        --min-sec 30 --max-sec 60

highlights.json 格式（由 Claude 依腳本標記 + 自己的判斷產生）:
[
  {"text": "節省百分之三十的成本，這是一個非常重要的突破", "priority": "user", "label": "腳本標記重點"},
  {"text": "接下來我們來看示範", "priority": "ai", "label": "Claude 判斷：口播重點"}
]

priority: "user"（使用者腳本裡自己標記的，優先保留）或 "ai"（Claude 自行判斷補上的）
"""
import argparse
import json
import sys

sys.path.insert(0, __file__.rsplit("/", 1)[0])
from locate_highlight import locate


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("transcript", help="transcribe.py 產生的逐字稿 JSON")
    ap.add_argument("highlights", help="重點文字清單 JSON")
    ap.add_argument("output", help="輸出 segments.json 路徑")
    ap.add_argument("--min-sec", type=float, default=30.0)
    ap.add_argument("--max-sec", type=float, default=60.0)
    ap.add_argument("--min-ratio", type=float, default=0.45, help="低於此匹配度視為定位失敗")
    args = ap.parse_args()

    with open(args.transcript, "r", encoding="utf-8") as f:
        transcript = json.load(f)
    with open(args.highlights, "r", encoding="utf-8") as f:
        highlights = json.load(f)

    located = []
    for h in highlights:
        result = locate(transcript["words"], h["text"], min_ratio=args.min_ratio)
        if result is None:
            print(f"[跳過，定位失敗] {h.get('priority','?')}: {h['text'][:20]}...", file=sys.stderr)
            continue
        start, end, ratio = result
        located.append({
            "start": start,
            "end": end,
            "priority": h.get("priority", "ai"),
            "reason": h.get("label", h["text"][:20]),
            "match_ratio": ratio,
        })

    order_key = {"user": 0, "ai": 1}
    located_sorted = sorted(
        enumerate(located),
        key=lambda x: (order_key.get(x[1]["priority"], 1), x[0]),
    )

    kept = []
    total = 0.0
    dropped = []
    for _, seg in located_sorted:
        dur = seg["end"] - seg["start"]
        if total + dur > args.max_sec and kept:
            dropped.append(seg)
            continue
        kept.append(seg)
        total += dur

    if dropped:
        print(f"\n[因超過 {args.max_sec} 秒上限而捨棄 {len(dropped)} 段]:", file=sys.stderr)
        for d in dropped:
            print(f"  - ({d['priority']}) {d['reason']}  {d['start']}-{d['end']}", file=sys.stderr)

    if total < args.min_sec:
        print(
            f"\n[警告] 目前選出的片段總長只有 {total:.1f} 秒，低於目標下限 {args.min_sec} 秒，"
            f"建議再從逐字稿裡挑幾段補上。",
            file=sys.stderr,
        )

    kept_chrono = sorted(kept, key=lambda s: s["start"])

    with open(args.output, "w", encoding="utf-8") as f:
        json.dump(kept_chrono, f, ensure_ascii=False, indent=2)

    print(f"\n完成！選出 {len(kept_chrono)} 段，總長 {total:.1f} 秒，已寫入 {args.output}")
    for s in kept_chrono:
        tag = "★使用者標記" if s["priority"] == "user" else "○Claude判斷"
        print(f"  {tag} [{s['start']}s-{s['end']}s] {s['reason']} (匹配度 {s['match_ratio']:.0%})")


if __name__ == "__main__":
    main()
```

### scripts/clean_segment.py

```python
#!/usr/bin/env python3
"""
clean_segment.py — 在一段已經選定的精華區間裡，用「規則比對」（不經過模型）
自動抓出贅字/口頭禪/過長靜音，並把它們從區間裡挖掉，回傳挖掉後剩下的子片段清單。

用法:
    python3 clean_segment.py <transcript.json> <start> <end> [--silence-th 0.5] [--pad 0.12]
"""
import argparse
import json
import re
import sys

FILLER_WORDS = [
    "呃", "嗯", "唉", "啊這", "那個", "這個", "就是說", "就是",
    "你知道", "你知道嗎", "這樣子", "那麼", "然後呢", "欸",
]

FILLER_PATTERN = re.compile("|".join(re.escape(w) for w in FILLER_WORDS))


def is_filler(word: str) -> bool:
    w = word.strip()
    if not w:
        return False
    return bool(FILLER_PATTERN.fullmatch(w)) or bool(FILLER_PATTERN.match(w))


def find_cut_spans(words, start, end, silence_th=0.5, pad=0.12):
    in_range = [w for w in words if w["start"] >= start and w["end"] <= end]
    in_range.sort(key=lambda w: w["start"])

    cuts = []

    for w in in_range:
        if is_filler(w["word"]):
            cuts.append((w["start"], w["end"]))

    for i in range(1, len(in_range)):
        prev, cur = in_range[i - 1], in_range[i]
        if prev["word"].strip() == cur["word"].strip() and cur["word"].strip():
            gap = cur["start"] - prev["end"]
            if gap < 0.3:
                cuts.append((cur["start"], cur["end"]))

    for i in range(1, len(in_range)):
        prev, cur = in_range[i - 1], in_range[i]
        gap = cur["start"] - prev["end"]
        if gap > silence_th:
            cuts.append((prev["end"] + pad, cur["start"] - pad))

    cuts = [c for c in cuts if c[1] > c[0]]
    cuts.sort()
    merged = []
    for c in cuts:
        if merged and c[0] <= merged[-1][1] + 0.05:
            merged[-1] = (merged[-1][0], max(merged[-1][1], c[1]))
        else:
            merged.append(list(c))
    return [tuple(m) for m in merged]


def subtract_cuts(start, end, cuts):
    keep = []
    cursor = start
    for c_start, c_end in cuts:
        if c_start > cursor:
            keep.append((cursor, min(c_start, end)))
        cursor = max(cursor, c_end)
    if cursor < end:
        keep.append((cursor, end))
    return [(round(s, 2), round(e, 2)) for s, e in keep if e - s > 0.15]


def clean_segment(words, start, end, silence_th=0.5, pad=0.12):
    cuts = find_cut_spans(words, start, end, silence_th=silence_th, pad=pad)
    return subtract_cuts(start, end, cuts), cuts


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("transcript")
    ap.add_argument("start", type=float)
    ap.add_argument("end", type=float)
    ap.add_argument("--silence-th", type=float, default=0.5)
    ap.add_argument("--pad", type=float, default=0.12)
    args = ap.parse_args()

    with open(args.transcript, "r", encoding="utf-8") as f:
        data = json.load(f)

    keep, cuts = clean_segment(data["words"], args.start, args.end, args.silence_th, args.pad)

    print(f"原始區間: {args.start}s - {args.end}s ({args.end-args.start:.2f}s)", file=sys.stderr)
    print(f"剪掉 {len(cuts)} 段贅字/靜音:", file=sys.stderr)
    for c in cuts:
        print(f"  - {c[0]:.2f}s - {c[1]:.2f}s", file=sys.stderr)
    kept_total = sum(e - s for s, e in keep)
    print(f"剩下 {len(keep)} 段，總長 {kept_total:.2f}s\n", file=sys.stderr)

    print(json.dumps([{"start": s, "end": e} for s, e in keep], ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

### scripts/apply_cleaning.py

```python
#!/usr/bin/env python3
"""
apply_cleaning.py — 把 build_segments.py 產生的 segments.json，逐段套用
clean_segment.py 的贅字/口吃/過長靜音偵測，展開成更細的子片段清單。

用法:
    python3 apply_cleaning.py <transcript.json> <segments.json> <輸出final_segments.json> \
        [--silence-th 0.5] [--pad 0.12]
"""
import argparse
import json
import sys

sys.path.insert(0, __file__.rsplit("/", 1)[0])
from clean_segment import clean_segment


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("transcript")
    ap.add_argument("segments")
    ap.add_argument("output")
    ap.add_argument("--silence-th", type=float, default=0.5)
    ap.add_argument("--pad", type=float, default=0.12)
    args = ap.parse_args()

    with open(args.transcript, "r", encoding="utf-8") as f:
        transcript = json.load(f)
    with open(args.segments, "r", encoding="utf-8") as f:
        segments = json.load(f)

    final = []
    total_before = 0.0
    total_after = 0.0
    for seg in segments:
        dur_before = seg["end"] - seg["start"]
        total_before += dur_before
        keep, cuts = clean_segment(
            transcript["words"], seg["start"], seg["end"],
            silence_th=args.silence_th, pad=args.pad,
        )
        if not keep:
            keep = [(seg["start"], seg["end"])]
        for s, e in keep:
            total_after += (e - s)
            final.append({
                "start": s,
                "end": e,
                "reason": seg.get("reason", ""),
                "priority": seg.get("priority", ""),
            })
        if cuts:
            print(
                f"[{seg['start']}s-{seg['end']}s] {seg.get('reason','')}: "
                f"剪掉 {len(cuts)} 處贅字/靜音，{dur_before:.1f}s -> {sum(e-s for s,e in keep):.1f}s",
                file=sys.stderr,
            )

    with open(args.output, "w", encoding="utf-8") as f:
        json.dump(final, f, ensure_ascii=False, indent=2)

    print(
        f"\n完成！{len(segments)} 個原始片段 -> {len(final)} 個淨化後子片段，"
        f"總長 {total_before:.1f}s -> {total_after:.1f}s，已寫入 {args.output}"
    )


if __name__ == "__main__":
    main()
```

### scripts/cut_clip.py（★已改版：單一濾鏡圖架構，取代舊版易造成音畫不同步的做法）

**舊版問題**：舊版做法是把每個片段各自用 `-ss/-to` 重新編碼裁切（每段各自的時間戳記歸零），再用 concat demuxer 把獨立編碼的片段串接起來。這在片段數一多、或來源是可變影格率（VFR）素材時，會讓每段各自獨立的時間軸捨入誤差逐段疊加，剪到後段就能感覺出音畫偏移。**新版**改成在單一 ffmpeg 濾鏡圖裡，對同一份輸入的 `[0:v]`/`[0:a]` 直接套用多個 `trim`/`atrim` + `setpts`/`asetpts` 歸零，最後用一個 `concat` 濾鏡一次輸出，全程只有一條輸出時間軸，不會有多段拼接誤差；片段數較多時分批處理（每批先各自渲染成一支乾淨的中繼影片），批次之間才用 concat demuxer（copy，不重新編碼）做一次串接，讓濾鏡圖不會無限膨脹，同時把「可能累積誤差的拼接次數」壓到最低。

```python
#!/usr/bin/env python3
"""
cut_clip.py（v2） — 依 segments.json 指定的時間區間，用單一濾鏡圖 trim+atrim+concat
剪出片段並串接成一支短片，避免舊版「各自重新編碼再用 concat demuxer 串接」造成的
音畫不同步隨片段數累積的問題。

用法:
    python3 cut_clip.py <來源影片> <segments.json> <輸出檔案> [--pad 0.15] [--batch-size 10]

segments.json 格式:
[
  {"start": 12.3, "end": 18.7, "reason": "開場金句"},
  {"start": 45.0, "end": 52.4, "reason": "數據亮點"}
]
"""
import argparse
import json
import os
import subprocess
import sys
import tempfile


def run(cmd):
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print("指令失敗:", " ".join(cmd), file=sys.stderr)
        print(result.stderr[-2000:], file=sys.stderr)
        raise SystemExit(1)
    return result


def get_duration(path):
    r = run([
        "ffprobe", "-v", "error", "-show_entries", "format=duration",
        "-of", "default=noprint_wrappers=1:nokey=1", path
    ])
    return float(r.stdout.strip())


def render_batch(source, clips, out_path):
    """用單一濾鏡圖把 clips（[(start,end),...]）從 source 剪出並串接成 out_path。"""
    filter_parts = []
    concat_inputs = []
    for i, (start, end) in enumerate(clips):
        filter_parts.append(
            f"[0:v]trim=start={start:.3f}:end={end:.3f},setpts=PTS-STARTPTS[v{i}];"
        )
        filter_parts.append(
            f"[0:a]atrim=start={start:.3f}:end={end:.3f},asetpts=PTS-STARTPTS[a{i}];"
        )
        concat_inputs.append(f"[v{i}][a{i}]")
    filter_complex = "".join(filter_parts) + "".join(concat_inputs) + \
        f"concat=n={len(clips)}:v=1:a=1[outv][outa]"

    run([
        "ffmpeg", "-y", "-i", source,
        "-filter_complex", filter_complex,
        "-map", "[outv]", "-map", "[outa]",
        "-c:v", "libx264", "-preset", "veryfast", "-crf", "18",
        "-c:a", "aac",
        out_path,
    ])


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("source", help="來源影片路徑")
    ap.add_argument("segments", help="segments.json 路徑")
    ap.add_argument("output", help="輸出影片路徑")
    ap.add_argument("--pad", type=float, default=0.15, help="每段前後 padding 秒數")
    ap.add_argument("--batch-size", type=int, default=10, help="單一濾鏡圖最多容納幾段，避免圖過大")
    args = ap.parse_args()

    with open(args.segments, "r", encoding="utf-8") as f:
        segments = json.load(f)

    if not segments:
        print("segments.json 是空的，沒有東西可以剪。", file=sys.stderr)
        raise SystemExit(1)

    src_duration = get_duration(args.source)

    clips = []
    for seg in segments:
        start = max(0.0, seg["start"] - args.pad)
        end = min(src_duration, seg["end"] + args.pad)
        if end > start:
            clips.append((start, end))
            print(f"  片段: {start:.2f}s - {end:.2f}s ({end-start:.2f}s)  {seg.get('reason','')}")

    if not clips:
        print("沒有有效的片段可以剪。", file=sys.stderr)
        raise SystemExit(1)

    with tempfile.TemporaryDirectory() as tmpdir:
        batches = [clips[i:i + args.batch_size] for i in range(0, len(clips), args.batch_size)]

        if len(batches) == 1:
            render_batch(args.source, batches[0], args.output)
        else:
            batch_paths = []
            for bi, batch in enumerate(batches):
                bp = os.path.join(tmpdir, f"batch_{bi:03d}.mp4")
                render_batch(args.source, batch, bp)
                batch_paths.append(bp)

            concat_list = os.path.join(tmpdir, "concat.txt")
            with open(concat_list, "w", encoding="utf-8") as f:
                for p in batch_paths:
                    f.write(f"file '{p}'\n")

            # 每個批次都是同樣參數重新編碼過的乾淨檔案，這裡用 copy 模式串接，
            # 批次數遠少於片段數，拼接次數大幅減少，不會有可感知的偏移。
            run([
                "ffmpeg", "-y", "-f", "concat", "-safe", "0",
                "-i", concat_list,
                "-c", "copy",
                args.output,
            ])

    out_duration = get_duration(args.output)
    total = sum(e - s for s, e in clips)
    print(f"\n完成！輸出影片: {args.output}")
    print(f"總長度: {out_duration:.2f} 秒（{len(clips)} 個片段，原始總長 {total:.2f} 秒 + padding）")


if __name__ == "__main__":
    main()
```

### scripts/retime_captions.py（新增）

```python
#!/usr/bin/env python3
"""
retime_captions.py — 把「文字正確但時間碼過時」的舊逐字稿，跟「時間碼準確但文字有
辨識錯誤」的新逐字稿合併，產出「文字正確 + 時間碼正確」的字幕分行清單（ASS Dialogue 格式）。

用法:
    python3 retime_captions.py <舊逐字稿(文字正確).json> <新逐字稿(時間正確).json> \
        <輸出captions_body.txt> --char-limit 8 --gap-threshold 0.25 [--style Caption] [--layer 0]

原理：
  1. 舊逐字稿正規化成一串字元（正確文字）。
  2. 新逐字稿也正規化成一串字元＋每個字元的時間碼（正確時間）。
  3. 用 difflib.SequenceMatcher 在字元層級對齊兩串文字，把舊文字每個字元對應到
     新逐字稿裡最相近的時間碼；對不上的字元（代表新轉錄那段辨識錯了）用前後最近的
     對齊點線性內插。
  4. 依 gap-threshold（停頓超過此秒數斷行）和 char-limit（單行超過此字數斷行）分組成
     字幕行，輸出成可直接貼進 .ass [Events] 區塊的 Dialogue 文字。

char-limit 預設 8：whisper 中文是逐字辨識、沒有詞界概念，用 10 曾經在雙字詞正中間斷行
（例如「…完全沒有發」／「芽沒有…」），8 是目前驗證過比較不會切斷雙字詞的數值，仍建議
燒錄後人工看一次，覺得斷得不順可以微調重跑。
"""
import argparse
import difflib
import json
import re
import sys


def normalize(s: str) -> str:
    return re.sub(r"[\s，。、！?？「」『』,.!?\"'()（）\-—…:：;；]", "", s)


def old_chars_from_text(old_transcript_path):
    with open(old_transcript_path, "r", encoding="utf-8") as f:
        data = json.load(f)
    return list(normalize(data["text"]))


def new_words_char_index(new_transcript_path):
    with open(new_transcript_path, "r", encoding="utf-8") as f:
        data = json.load(f)
    chars, timestamps = [], []
    for w in data["words"]:
        token = normalize(w["word"])
        if not token:
            continue
        n = len(token)
        for i, ch in enumerate(token):
            chars.append(ch)
            frac_start = w["start"] + (w["end"] - w["start"]) * i / max(n, 1)
            frac_end = w["start"] + (w["end"] - w["start"]) * (i + 1) / max(n, 1)
            timestamps.append((frac_start, frac_end))
    return chars, timestamps


def align(old_chars, new_chars, new_timestamps):
    """回傳跟 old_chars 等長的 (start,end) 時間碼列表，未對上的字用線性內插補齊。"""
    sm = difflib.SequenceMatcher(a=old_chars, b=new_chars, autojunk=False)
    blocks = sm.get_matching_blocks()

    result = [None] * len(old_chars)
    for b in blocks:
        for k in range(b.size):
            result[b.a + k] = new_timestamps[b.b + k]

    n = len(result)
    i = 0
    while i < n:
        if result[i] is not None:
            i += 1
            continue
        j = i
        while j < n and result[j] is None:
            j += 1
        left = result[i - 1][1] if i > 0 else (result[j][0] if j < n else 0.0)
        right = result[j][0] if j < n else (result[i - 1][1] if i > 0 else left)
        span = max(right - left, 0.001)
        count = j - i
        for k in range(count):
            frac_s = left + span * k / count
            frac_e = left + span * (k + 1) / count
            result[i + k] = (frac_s, frac_e)
        i = j
    return result


def group_lines(chars, timestamps, gap_threshold=0.25, char_limit=8):
    lines = []
    cur_chars = []
    cur_start = None
    prev_end = None
    for ch, (s, e) in zip(chars, timestamps):
        if cur_start is None:
            cur_start = s
        elif (s - prev_end > gap_threshold) or (len(cur_chars) >= char_limit):
            lines.append((cur_start, prev_end, "".join(cur_chars)))
            cur_chars = []
            cur_start = s
        cur_chars.append(ch)
        prev_end = e
    if cur_chars:
        lines.append((cur_start, prev_end, "".join(cur_chars)))
    return lines


def fmt_time(t):
    h = int(t // 3600)
    m = int((t % 3600) // 60)
    s = t % 60
    return f"{h:d}:{m:02d}:{s:05.2f}"


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("old_transcript", help="文字正確、時間碼可能過時的逐字稿 json")
    ap.add_argument("new_transcript", help="時間碼正確（對目前最終版影片重新轉錄）的逐字稿 json")
    ap.add_argument("output", help="輸出 ASS Dialogue 文字檔路徑")
    ap.add_argument("--char-limit", type=int, default=8)
    ap.add_argument("--gap-threshold", type=float, default=0.25)
    ap.add_argument("--style", default="Caption")
    ap.add_argument("--layer", type=int, default=0)
    args = ap.parse_args()

    old_chars = old_chars_from_text(args.old_transcript)
    new_chars, new_ts = new_words_char_index(args.new_transcript)
    aligned_ts = align(old_chars, new_chars, new_ts)
    lines = group_lines(old_chars, aligned_ts, args.gap_threshold, args.char_limit)

    with open(args.output, "w", encoding="utf-8") as f:
        for start, end, text in lines:
            f.write(
                f"Dialogue: {args.layer},{fmt_time(start)},{fmt_time(end)},{args.style},,0,0,0,,{text}\n"
            )

    print(f"完成！產生 {len(lines)} 行字幕，已寫入 {args.output}", file=sys.stderr)


if __name__ == "__main__":
    main()
```

## 字卡與字幕技術細節（ASS / Advanced SubStation Alpha）

- `PlayResX`/`PlayResY` 定義渲染畫布大小，直式短影片用來源解析度（例如 1080x1920）。
- `[V4+ Styles]` 裡每個 Style 的關鍵欄位：`Fontsize`；`PrimaryColour`/`OutlineColour`/`BackColour` 是 `&HAABBGGRR`（BGR 順序＋透明度，00=不透明）；`BorderStyle`（1=外框＋陰影，3=不透明色塊）；`Outline`/`Shadow` 粗細；`Alignment`（小鍵盤方位：1-3 底部/4-6 中間/7-9 頂部，×1/4/7=靠左、2/5/8=置中、3/6/9=靠右，密集字幕常用 2，強調字卡置中常用 5）；`MarginL/R/V`。
- 兩個 Style 建議寫法：
  ```
  [V4+ Styles]
  Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
  Style: Caption,Noto Sans CJK TC,58,&H00FFFFFF,&H000000FF,&H00101010,&H00000000,-1,0,0,0,100,100,0,0,1,4,1,2,60,60,300,1
  Style: KeyCard,Noto Sans CJK TC,150,&H0000FFFF,&H000000FF,&H00000000,&H00101010,-1,0,0,0,100,100,0,0,1,10,5,5,40,40,0,1
  ```
  （`Caption`＝密集字幕，貼底部、白字黑框；`KeyCard`＝強調花字，置中、黃底黑框、大字。）
- `[Events]` 用 `Layer` 區分字幕與字卡（例如 Caption 用 0、KeyCard 用 1），兩者可同時出現在畫面上不互相干擾。
- **字卡 pop-in 彈跳進場動畫**：在 Dialogue 的 `Text` 欄位開頭加 override tag：
  ```
  {\fscx60\fscy60\t(0,150,\fscx115\fscy115)\t(150,260,\fscx100\fscy100)}文字內容
  ```
  意思是從 60% 縮放開始，0-150ms 放大到 115%（衝過頭，製造彈跳感），150-260ms 收回 100%——典型的綜藝字卡進場效果。
- **燒錄指令**：`ffmpeg -i <input> -vf "ass=<path.ass>" -c:v libx264 -preset veryfast -crf 18 -c:a copy <output>`（音訊只是 stream copy，因為只有畫面被疊字幕/字卡覆蓋）。

## 字卡設計原則（使用者已確認的預設風格，之後專案直接套用）

1. **候選來源**：從逐字稿裡挑「情緒轉折、關鍵數字、結論句」等值得強調的稀疏節點，不是每句話都做字卡。
2. **文字要精簡到只剩關鍵詞**：把完整句子裡的前導語/連接詞/修飾語全部去掉，只留 2-6 個字的核心詞或短語（例如「連續2年顆粒無收」這種完整句改成「完全沒有發芽」，「很不幸的第二年又崩潰一崩」改成「崩潰」）。這是明確驗證過的標準，預設就照這個做，不用每次都先問。
3. **視覺風格走「綜藝」路線**（已用 Google 圖片搜尋「短影片字卡」的常見範例驗證過方向）：大字（fontsize 遠大於一般字幕，約為畫布寬度的 1/7~1/8）、高對比亮色（例如黃字黑框）、粗外框＋陰影、置中顯示、進場帶彈跳動畫。這是目前的預設，除非 使用者要求換成低調的色塊/企業風格（`BorderStyle 3` 的實色底＋白字組合，之前也有實作過，可以在 使用者要求時切回去）。
4. **迭代方式**：字卡風格通常需要 2-3 輪來回（例如先做出來、使用者說「不夠大不夠綜藝」、再放大加動畫、使用者再說「置中」），每輪只要改 `.ass` 的 Style 行或 override tag 重新燒錄，不用整個流程重跑。

## 動態直式裁切 + 分割畫面（進階，视素材需要才用）

遇到原始素材是橫式、但成品要直式 9:16，且畫面中有多個說話者需要跟拍時，兩種常見處理方式：
- **動態跟隨裁切**：依時間軸分段，每段估一個 crop 視窗（x/y 依主要說話者位置），用 `crop=w:h:x:y` 對各段分別套用，再用 `trim`/`atrim`＋`concat` 濾鏡（跟 `cut_clip.py` 同一套單一濾鏡圖手法）接起來，避免逐段輸出再用 demuxer 串接造成的同步問題。段與段之間的 x/y 若要平滑過渡，可以在同一段內用 `crop` 的 x/y 搭配表達式（例如 `x='min(max(0,t*k+offset),max_x)'`）做線性移動，而不是整段跳切。
- **分割畫面**：用 `crop` 各自切出每個說話者的區域，再用 `hstack`（左右）或 `vstack`（上下）拼成一個畫面，注意兩路畫面的解析度/長寬比要先用 `scale` 對齊，音訊維持主要說話者那一路或用 `amix` 混合，依需求決定。

這兩種都屬於「素材特性決定做法」，沒有固定腳本，實作時依當次素材的鏡位、人數現場調整濾鏡參數，核心原則跟 `cut_clip.py` 一致：所有片段在同一個濾鏡圖裡處理完再輸出，不要先各自輸出再用 concat demuxer 拼接，除非片段數多到需要分批（分批時比照 `cut_clip.py` 的批次做法）。

## 執行環境備註

- **雲端工作環境**：已內建 ffmpeg；`faster-whisper`、`opencc-python-reimplemented` 需要用 pip 安裝一次（`pip3 install --break-system-packages faster-whisper opencc-python-reimplemented`），之後同一個 session 內都可以重複使用。
- **`pip3 install` 遇到多依賴套件（例如 faster-whisper 會拉進 ctranslate2、av、tokenizers、huggingface-hub 等）可能因為單次工具呼叫的逾時（約 170-180 秒）沒裝完就中斷**——這不代表失敗，pip 的本地快取/部分下載進度會留著（跨 device_bash 呼叫仍保留在檔案系統上，即使背景 process 不會存活到下一次呼叫），直接原樣重跑同一條指令通常會在下一個時間窗口內接著完成，不需要改用背景執行（`&`/`nohup`）——背景 process 不會存活到下一次 device_bash 呼叫，反而更難追蹤進度。
- **使用者的電腦（device_bash 是橋接到 Mac 的 Linux VM，不是原生 macOS shell）**：`sed -i` 要用 GNU 語法 `sed -i 's/.../.../ file'`，**不能**用 macOS 慣用的 `sed -i '' 's/.../.../ file'`（空字串當成品備份副檔名的寫法在 GNU sed 下會把空字串當成檔案路徑，報 "No such file or directory"）。其餘 `ffmpeg`、`python3`、`pip3` 都可直接用，`pip3 download --no-deps` 測試個別套件通常幾秒內完成，只有多依賴的整批安裝容易逾時（見上一點）。
- **huggingface.co 在雲端工作環境與 使用者的 device_bash 環境都被機構網路政策擋掉（403 Forbidden）**——已分別用 `curl -sS "$HTTPS_PROXY/__agentproxy/status"`（雲端）和直接對 huggingface.co 發 curl（device_bash）驗證過，兩邊都是機構刻意的網路政策拒絕，不是暫時性錯誤。**依標準規則絕不繞過機構的網路政策拒絕**（不找鏡像、不手動上傳模型檔、不用 VPN）。**已驗證的替代做法**：请 使用者在他自己原生的 Terminal.app（不是透過 device_bash）執行 `transcribe.py`——那裡的網路不受此限制，使用者完成後把產生的逐字稿 json 提供回來即可繼續流程。
- **每次要為字幕/字卡定位時間碼時，一律對「目前最終確認版本」的影片重新跑 `extract_audio.py` + `transcribe.py`，絕對不要沿用更早一版剪輯留下的逐字稿**——即使影片只調整過音畫同步、時長只差零點幾秒，套用到十幾二十行密集字幕上會有能感覺出來的錯位（已有實測案例：舊逐字稿對應的影片時長 49.728s，現行版本是 49.439~49.493s，約 0.6% 的累積偏移）。
- **VFR（可變影格率）素材要提高警覺**：不要僅憑「影片時長看起來差不多」就判斷同步沒問題，`cut_clip.py` 的單一濾鏡圖架構已經降低了逐段疊加誤差的風險，但來源本身是 VFR 時仍建議剪完抽查頭尾幾個片段的實際音畫對齊，不要單靠 ffprobe 的 duration 數字下結論。
- 影片較長（10 分鐘以上）時，`transcribe.py` 用 `small` 模型在 CPU 上跑可能要幾分鐘，屬正常現象，不用中途取消重跑。
- 如果同一段文字在影片裡出現不只一次，`locate_highlight.py` 目前會取第一個夠好的匹配，若定位錯誤可以提供更長、更獨特的上下文文字重新比對。
- `clean_segment.py` 的贅字清單、靜音門檻（預設 0.5 秒）都是寫死在腳本裡的合理預設值；如果 使用者常講的口頭禪不在清單裡，或覺得跳接太密／太鬆，可以直接調整 `FILLER_WORDS` 清單或 `--silence-th` 參數。
- **成品傳進對話有約 30MB 的實務上限**：全品質燒錄（例如 `-crf 18`）的成品常常會超過，這種情況下額外燒一份縮小版（`-crf 24 -c:a aac -b:a 128k`）專門用來傳進對話讓 使用者快速看過，全品質版本另外留在他自己的資料夾。
- **大檔案模式的實測結果**：整套「本機抽音軌 → 音軌送雲端轉錄/判斷 → 結果送回本機 → 本機直接剪輯/燒字幕字卡」流程已跑通——使用者電腦的 device 環境本來就有 ffmpeg、python3、pip3 可用，原始影片檔案全程沒有被上傳或搬移。套用到實際情境時，先用 `device_list_dir`／`ls -lh` 確認原始檔案大小，超過上傳上限就直接走大檔案模式，不要嘗試先上傳看看。
- 如果有 Google Drive 之類的連接工具可以讀到 使用者雲端硬碟裡的檔案，只適合拿來確認資料夾裡有哪些檔案、檔案多大，**不要**用它的下載功能去搬音訊或影片本體。
- **切記：即使現場另外有外接收音設備錄出來的音檔（分軌／Mixdown），也不能拿來取代 `extract_audio.py` 從影片本身抽出的音軌**——兩者時間軸不保證一致，拿不同來源的音檔做轉錄定位，時間碼套回影片會剪錯位置或讓字幕/字卡跑掉。