# Sky UI 圖示擷取工具

從 Sky: Children of the Light 的 UI 圖集資料 (`UIPackedAtlas*.ktx` +
`UIPackedAtlas.lua` + `OutfitDefs.json`) 擷取出每個服裝/道具的 icon PNG。

## 這次重寫解決的問題

1. **不再需要 Visual Studio / DLL / `texture2ddecoder`。**
   實測這批 `.ktx` 檔案的 `glInternalFormat` 全部是 `0x8DBB`
   (`GL_COMPRESSED_RED_RGTC1`，也就是 **BC4**，單通道灰階)。
   BC4 的區塊解碼演算法規格公開、非常單純，
   `core/texture/bc4.py` 直接用 **NumPy 向量化**實作，
   不需要編譯任何 C/Rust 擴充套件，也不需要 `ktx.dll`。
   已經用實際的 `UIPackedAtlas31~40.ktx` 驗證過，解碼結果與
   `texture2ddecoder` 的舊寫法逐 pixel 比對完全一致。

2. **修好一個真的存在的 bug。**
   舊版 `atlas_parser.py` 假設「圖片名稱」和「uv 座標」在不同行，
   但實際檔案格式兩者在同一行，導致 uv 永遠沒被解析到。
   新版用單一 regex 一次抓完，已用真實的 `UIPackedAtlas.lua` 驗證。

3. **相依套件只剩 `numpy` + `Pillow`。**
   拿掉了 `opencv-python-headless` (改用標準函式庫 `colorsys` 重寫染色邏輯)、
   `texture2ddecoder`、以及 `ktx.dll`/`ctypes` 呼叫。
   這兩個套件都是最主流、有完整 wheel 支援的套件，之後不管是包成 exe
   (PyInstaller) 還是搬到 Android (python-for-android)，都不會卡在
   「需要編譯原生模組」這件事上。

4. **統一了三種輸入方式。**
   `core/source/open_source(path)` 會自動判斷輸入是資料夾、`.zip`、
   還是 `.apk` (本質也是 zip)，呼叫端完全不用分支處理：

   ```bash
   # 模式 1：遊戲安裝包 (資料夾，或直接 .apk 檔案) — 路徑請換成實際位置
   python app/main.py --source "<GAME_INSTALL_PATH_OR_APK>" --output ./UI

   # 模式 2：任意 .zip 檔案，用法跟模式 1 一樣
   python app/main.py --source "<PATH_TO_ZIP>" --output ./UI

   # 模式 3：已經整理好的資料夾 (例如把上傳的 zip 解壓縮到這裡)
   #         裡面放 OutfitDefs.json / UIPackedAtlas.lua / UIPackedAtlas*.ktx
   python app/main.py --source "<INPUT_DIR>" --output ./UI
   ```

   三種模式已經各自用真實資料測過 (資料夾 / 產生的測試 zip /
   nested `assets/` 路徑的假 apk)，輸出結果逐檔案 byte-for-byte 一致。

## BC 資料夾裡「不在圖集裡」的獨立貼圖

BC 資料夾除了 `UIPackedAtlas*.ktx` (圖集本體) 之外，通常還有大量「不需要
`UIPackedAtlas.lua` / `OutfitDefs.json` 就能直接用」的獨立 `.ktx` 檔案
(單張 icon、材質貼圖等)。實測這些檔案大多是 **BC7** (`0x8E8C`/`0x8E8D`)
全彩壓縮格式，跟圖集用的 BC4 (單通道灰階) 不一樣，所以另外寫了
`core/texture/bc7.py`，演算法逐行對應公開領域的 `bcdec.h`
(https://github.com/iOrange/bcdec) 移植成 Python，一樣不需要編譯器。
兩個查表用的 partition 表格已經逐一比對過原始 C 檔案數值，確保沒有手動轉錄
錯誤；另外實測用純色材質 (`Yellow.ktx` / `Blue.ktx` / `White.ktx` /
`Black.ktx`) 反解出來的顏色跟檔名完全對應，驗證解碼正確。

用法完全獨立、不需要 lua/json:

```bash
# 只轉換 BC 資料夾下的獨立貼圖，完全跳過圖集 + 服裝定義流程
python app/main.py --source "<INPUT_DIR>" --output ./UI \
    --skip-atlas --loose-ktx-dir BC

# 也可以跟圖集流程一起跑 (會自動排除 UIPackedAtlasN 本體，不會重複輸出)
python app/main.py --source "<INPUT_DIR>" --output ./UI --loose-ktx-dir BC
```

輸出會放在 `./UI/Standalone/`，檔名跟原本的 `.ktx` 同名。已經用實際的
596 個檔案 (材質 + 獨立 icon) 完整跑過一次，全部轉換成功、無錯誤。

## 目錄結構

```
app/main.py             CLI 入口，三種輸入模式都從這裡進
core/source/            資源讀取抽象層 (資料夾 / zip / apk 三選一, 自動判斷)
core/atlas/             解析 UIPackedAtlas.lua
core/outfit/            解析 OutfitDefs.json
core/texture/           KTX 檔頭解析 + BC4 純 Python/NumPy 解碼器
core/pipeline/          串接以上模組的主流程
image/                  裁切 (crop) + HSV 染色 (color) + 輸出 PNG (export)
game_data/              測試用的樣本資料 (OutfitDefs.json / lua / 10 張 ktx)
```

## 關於未來做成 APK 這件事

先說一個要注意的重點：在 Android 上，如果目標是讀「其他已安裝遊戲的
安裝包」，非 root 手機基本上**讀不到**別的 App 的 APK 檔案
(Android 的 scoped storage 限制)。實務上通常是使用者要自己先用檔案
管理員把 APK 匯出到「下載」資料夾，再用系統的檔案選擇器選取。

這也是為什麼這次把三種輸入方式的底層邏輯統一成同一套
(`open_source()` 自動判斷)：不管檔案是怎麼被拿到手的 (檔案選擇器回傳
的路徑、還是使用者手動指定的資料夾)，處理邏輯完全不用改，之後真的要
包成 Android App 時，只需要換掉「怎麼拿到檔案路徑/bytes」這一層。

另外，由於整個解碼流程現在只依賴 `numpy` + `Pillow`
(不需要編譯任何原生模組)，用 python-for-android / Buildozer 之類的
工具打包時，會比之前依賴 `texture2ddecoder` 的版本單純很多。

## 安裝

```bash
pip install -r requirements.txt
```
