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

套用背景時 `finish_gj(..., enhance=False, fill_full=True, nadir=False, zenith=False)`。

**標準版不會在 CLI 產生控制點。** `-createproject` 只建檔，`-stitchnogui` / `-batch` 只縫圖。必須開 PTGui 視窗選「專案 → 對齊影像」（Shift+F5），等到「請稍候」結束再存檔輸出。

**禁止**把其他展站的 `.pts` 當 `-template`（會複製錯誤相機方位，造成重影／斷樑）。只可用 `scripts/make_equirect_template.py` 產出的鏡頭+360 樣板（`imageparams=false`）。

單站正確重拼：

```powershell
python -X utf8 -u scripts/restitch_one_ptgui.py station-1f-qc
```

## 拼好後的三種缺陷與修法（2026-08 實戰）

成品出現「一團灰霧／模糊遮罩」或「整組錯亂」時，依序診斷：

1. **孤兒照片**（對齊後 0 控制點，被亂放進畫面）：
   `python -X utf8 -u scripts/exclude_orphans.py <sid> ...` — 解析 .pts 控制點數，移除孤兒、重出、套用。
2. **手震模糊照片**（清晰度低於全站中位數一半，被縫進成品變成霧塊）：
   `python -X utf8 -u scripts/exclude_blurry.py [--ratio=0.5] <sid> ...` — Laplacian 清晰度過濾，僅在該方位另有清晰照片時剔除。
3. **錯位群組**（一群照片互相連結但整組貼錯方向，如天花板特寫被優化到水平）：
   - 先試 `python -X utf8 -u scripts/reinforce_ptgui_gui.py <pts>`（GUI 自動化：控制點 → 為所有重疊影像產生控制點 → F5 優化 → 存檔），品管室靠這招修好。
   - 若優化仍收斂到錯位（庫存區案例）：做 contact sheet 對照每張照片的內容與優化後 yaw/pitch，把「內容是天花板/地板特寫、pitch 卻接近 0」的照片用 `exclude_blurry.remove_groups` 移除後重出。

4. **重複紋理牆／近距離視差**（浪板牆、電梯門這種平面近拍，全域優化拉不回，牆面窗框交錯）：
   - 優先找 `C:\ptgui_test\jobs\_good_pts_backup\` 的試用版時代人工整理專案當幾何基底，剔除其中的孤兒與錯位群組後重出（走道案例：備份幾何正確，只要移除 10 張被優化到水平的地板特寫＋2 張孤兒）。
   - 平面近拍鬼影（二樓電梯門、橘柱殘影）：把跨越該平面上下的近拍照片移除做變體比較（`station-2f_varA/varB.pts`），接縫層數變少鬼影即消失；先出變體確認無破洞再套用。
   - 二樓殘餘水平拼縫（v197）：2A 柱上段兩折（y5360 +55px、y5540 +34px 接到已對齊柱身）＋電梯左框 y6600（下段 +57/+36px）。腳本 `scripts/retouch_2f_remain.py`。
   - 二樓 2C 柱列微調（v199）：同一條 y≈5400 水平拼縫，2C＋橘梯整段上柱偏右（-12/−6px）、右側鄰柱上段偏左（+19px），必須分柱位移；上沿拉到 y≈4680 以免標籤附近再折。腳本 `scripts/retouch_2f_col2c.py`。
   - 二樓同列其餘柱（v200）：D 柱 +15、s6 鄰柱 +14、2C 左前方牆柱 +67、左側細柱 +27、遠側左 +11／右 −52。腳本 `scripts/retouch_2f_colrow.py`。
   - 二樓柱列完整接縫（v201）：從 `*_pre_col2c` 重做。y=5400 上帶依高信心 NCC 逐欄位移且縫處不羽化（先前位移在縫上被羽化掉，段差看起來沒修）。2A y≈6600 浪管上下已對齊，塗抹假邊用縫外紋理重建。腳本 `scripts/retouch_2f_fullseam.py`。
   - 二樓柱緣對齊（v202）：NCC 對浪板牆會鎖錯週期。改從 `*_pre_col2c` 依暗柱左右緣分緣平移上帶，接縫處全量不羽化。腳本 `scripts/retouch_2f_pillars.py`。
   - 二樓中央走道灰柱（v206）：v202 只修 x≈19000 以後。預設視角中間灰柱在 x≈12100–13100。右柱 y=5300 上段偏左 +77、中柱 y=5400 上段偏右 −19，接縫全量不羽化。腳本 `scripts/retouch_2f_center.py`。
   - 走道門口棧板（v205 維持原縫）：開口近拍（idx 26/31/32/33/38）控制點只有 9–38，是門口中段唯一覆蓋。整組移除或只拿 32/33 重出都會破洞、牆面崩。SIFT 補點後 F5 方位不變（近距離視差）。整帶垂直位移會把已對齊棧板／黃線拉壞。此帶目前維持 `*_pre_doorretouch` 幾何。

5. **殘餘小接縫（重拼救不回時的像素級修補）**：對「線不齊的小細節」直接改 `環景成品圖/ptgui/` 大圖再 `apply_one`。工具在 `scripts/seam_retouch.py`（`grid` 子命令輸出帶座標格線的放大裁切，先定位再修）＋各站 `retouch_*.py`（自動備份 `*_pre_retouch.jpg`，可反覆調參重跑）：
   - `ramp_shift`：接縫一側整片有固定平移（牆面溝縫、白飾條錯層）→ 從接縫全量位移、往遠端線性衰減（走道 x9280 右移 -30px 案例）。
   - `redraw`（黃線重繪）：接縫處黃線又斷又有殘影 → 整塊先用附近乾淨地板羽化填底（來源要做環境色均值匹配，否則會有可見矩形），再用左右乾淨線剖面線性內插重繪線帶，最後加 σ≈3.5 雜訊避免噴槍感。
   - 細帶 `inpaint`（TELEA）：柔化硬切邊、去除位移的殘影邊條（門板邊、牆底硬邊）。
   - 曝光帶阻平整化：藍牆／浪板牆上的中尺度亮度補丁（拼接曝光差、軟殘影）→ LAB 的 L 通道減去（σ小≈40-60 高斯 − σ大≈250-400 高斯）×0.7，遮罩限定該材質、邊緣羽化。
   - 注意：media 套用版有 fill_full 垂直重framing，media 座標≠大圖座標，一律在大圖上重新定位；PTGui 補產控制點＋重新優化對「近距離視差」的殘餘小錯位無效（會收斂回原樣），別浪費時間。

輔助知識：
- .pts 控制點格式 `{"t":0,"0":[群組idx,張idx,x,y],"1":[...]}`；移除 imagegroups 必須重映射控制點索引與 anchorimagegroup。
- **錯位群組的控制點數可能很高**（整組互聯成孤島後一起漂走），不能只看 cp=0；一定要 contact sheet 對照內容與 yaw/pitch。孤兒的典型特徵是多張停在同一個預設座標（如 y-57.5 p-2.1）。
- 試用版存的 `.pts` 開頭是 `# PTGui Trial Project File`（加密格式，不能直接 JSON 編輯）；用 `scripts/convert_pts_gui.py`（GUI 開啟 → Ctrl+S）轉成授權版 JSON 後即可程式化編輯。
- 樣板 blend `fillholes=True` 會把沒覆蓋的區域抹成灰霧；缺陷大多不是破洞而是上述照片問題。
- PTGui 優化結果視窗是 `#32770` 對話框（只有「關閉」鈕），pywinauto 要用 win32 列舉找 modal，UIAWrapper 沒有 `child_window`。
- 選單「專案 → 優化」有時點了沒觸發（PTGui CPU 不動、無進度視窗）；補救：對主視窗重新 set_focus 後再送一次 `{F5}`（`scripts/optimize_ptgui_gui.py` 已內建 fallback）。
- 大圖輸出後 PTGui 可能還鎖檔（WinError 1224），複製要重試。
- 只重出＋套用單站（不重新對齊）：`python -X utf8 -u scripts/stitch_apply_one.py <sid>`。

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
