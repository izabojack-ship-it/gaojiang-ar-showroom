---
name: gaojiang-ptgui-enhance
description: >-
  Re-stitches 高將實境 AR 展間 panoramas with licensed PTGui (no trial
  watermark), enhances the large stitch with Real-ESRGAN using the
  machine-ai-upscale pipeline, then updates exhibition background
  images and thumbs. Use when the user mentions PTGui, 重拼, 浮水印,
  大圖, 畫質優化, Real-ESRGAN, machine-ai-upscale, 更新背景圖, 環景,
  or all exhibition stations.
---

# 高將展間 · PTGui 重拼 + AI 畫質優化

把「細部圖 → PTGui 大圖 → Real-ESRGAN 優化 → 展間背景」做成可重跑流程。

參考實作：`C:\程式開發\高將機械\machine-ai-upscale`（`upscale.py` 的 unicode 讀寫、RRDBNet x4plus、tile、記憶體降級）。

展間專案：`D:\高將實境AR展間`。

## 何時用

- 已買 PTGui 標準版、要去掉試用浮水印
- 要把**所有展間**用 PTGui 重拼大圖
- 拼完後要 AI 優化再更新 `media/panoramas`

## 一鍵執行

```powershell
cd "D:\高將實境AR展間"
$env:PYTHONUNBUFFERED = "1"
python -X utf8 -u scripts/ptgui_enhance_pipeline.py
```

或雙擊 `PTGui重拼並優化背景.bat`。

單站 / 從第 N 站 / 略過某步：

```powershell
python -X utf8 -u scripts/ptgui_enhance_pipeline.py --only station-1f-qc
python -X utf8 -u scripts/ptgui_enhance_pipeline.py --from 5
python -X utf8 -u scripts/ptgui_enhance_pipeline.py --skip-stitch
python -X utf8 -u scripts/ptgui_enhance_pipeline.py --recreate
```

日誌：`環景成品圖/_ptgui_enhance.log`

## 流程（不可跳步）

```
Task Progress:
- [ ] 1. 確認 PTGui.exe 與授權（標準版無浮水印）
- [ ] 2. 將 環景細部圖 分鏡 stage 到 C:\ptgui_test\jobs\<sid>\src
- [ ] 3. PTGui -createproject → .pts（已有專案則重輸出）
- [ ] 4. PTGui -stitchnogui → 大圖 JPG
- [ ] 5. 複製到 環景成品圖/ptgui/<標題>_ptgui.jpg
- [ ] 6. Real-ESRGAN 優化 → 環景成品圖/ptgui_enhanced/<sid>_enhanced.jpg
- [ ] 7. finish_gj 寫入 media/panoramas 與 thumbs（10240×5120、高將天底）
- [ ] 8. 更新 stations.json 尺寸，MEDIA_VERSION +1
- [ ] 9. Ctrl+F5 開 http://localhost:8666 確認無浮水印、接縫與畫質
```

**createproject 與 stitchnogui 必須分開跑。** 合併成一步在本機常失敗。

## 八站對照

| # | 細部圖資料夾 | station id | 標題 |
|---|--------------|------------|------|
| 1 | 一樓走道空間 | station-1f-corridor | 一樓走道空間 |
| 2 | 一樓生產線_左新機區 | station-1f-line-left-new | 一樓生產線（左新機區） |
| 3 | 一樓生產線_右前半部 | station-1f-line-right-front | 一樓生產線（右前半部） |
| 4 | 一樓生產線_右後半部 | station-1f-line-right-back | 一樓生產線（右後半部） |
| 5 | 一樓週邊零件製造區 | station-1f-parts-mfg | 一樓週邊零件製造區 |
| 6 | 一樓品管室 | station-1f-qc | 一樓品管室 |
| 7 | 一樓半成品零件庫存區 | station-1f-parts-stock | 一樓半成品零件庫存區 |
| 8 | 二樓 | station-2f | 二樓置物區 |

照片路徑：`環景細部圖/<細部圖資料夾>/`

## 授權

- 試用版功能完整，成品有浮水印；標準版輸出無浮水印。
- 本機設定：`%APPDATA%\PTGui\Configuration.xml`
- 企業佈署金鑰：`C:\ProgramData\PTGui\licensekey.json`（見 PTGui FAQ Q2.13）
- 若設定仍像 Trial：請先開 PTGui 貼上訂單授權碼，再重跑 `--skip-enhance` 前的拼圖，或整段重跑。
- **不要**把授權碼寫進 git、skill 或 log。

## AI 優化規則（對齊 machine-ai-upscale）

腳本：`scripts/realesrgan_pano.py`

- 模型：RealESRGAN_x4plus / RRDBNet scale=4
- unicode 讀寫：`np.fromfile` + `cv2.imdecode` / `imencode` + `tofile`
- 大圖先縮到工作尺寸（上限約 10240×5120、50MP），禁止對 24K 原圖做 4 倍
- 已接近目標用 `outscale=1`（模型 4 倍再縮回，等於畫質優化）
- 較小來源才 `outscale=2` 朝 10240 放大
- tile：邊長 ≥10000→400、≥5000→300，否則 200；OOM 依序降到 300/200/128
- `half=True`, `gpu_id=0`；套件與 `machine-ai-upscale/requirements.txt` 相同

套用背景時 `finish_gj(..., enhance=False)`；僅在來源約 2:1 時 `fill_full=True`。正方形或大面積全黑成品視為失敗，改用走道站 `.pts` 當 `-template` 重拼。

## 路徑

| 用途 | 路徑 |
|------|------|
| PTGui | `C:\Program Files\PTGui\PTGui.exe` |
| 工作區 | `C:\ptgui_test\jobs\<sid>\` |
| 拼圖成品 | `環景成品圖/ptgui/` |
| AI 成品 | `環景成品圖/ptgui_enhanced/` |
| 展間背景 | `media/panoramas/<sid>.jpg` |
| 縮圖 | `media/thumbs/<sid>.jpg` |
| 舊底備份 | `media/panoramas/_before_ptgui/` |
| 參考放大專案 | `C:\程式開發\高將機械\machine-ai-upscale` |

## Do not

- 不要用 OpenCV `stitch_panorama.py` 取代 PTGui（這條 skill 的主路徑是 PTGui）
- 不要對 2 萬像素寬的拼圖直接 4 倍放大
- 不要把授權碼、licensekey.json 提交到 git
- 不要略過 `MEDIA_VERSION`；瀏覽器會吃到舊背景
- 不要在 AI 優化後再跑一輪 `enhance_content` 重銳化

## 驗證

1. 啟動 `python -m http.server 8666 --bind 127.0.0.1`
2. Ctrl+F5 開 `http://localhost:8666`
3. 逐站旋轉：無 PTGui 字樣、天車／貨架接縫、縮放後細節
4. 對照 `stations.json` 的 width/height 應為 10240×5120

## 公開部署（GitHub Pages）

遠端：`https://github.com/izabojack-ship-it/gaojiang-ar-showroom`  
公開網址：`https://izabojack-ship-it.github.io/gaojiang-ar-showroom/`

推 `master` 即部署。只提交展間會用到的檔：

```powershell
git add index.html js/tour.js js/guide.js css/tour.css media/stations.json media/panoramas/station-*.jpg media/thumbs/station-*.jpg .cursor/skills/gaojiang-ptgui-enhance/SKILL.md
git commit -m "更新 PTGui 授權版環景與 AI 優化背景，公開部署。"
git push origin HEAD
```

不要提交：`環景成品圖/`、`環景細部圖/`、`models/`、`scripts/`、授權碼、`_tools/`。

部署後用 `gh api repos/izabojack-ship-it/gaojiang-ar-showroom/pages` 確認 status，再開公開網址 Ctrl+F5。
