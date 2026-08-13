# Table of Contents
- [HEVC](#HEVC)
- [Dolby Vision](#dolbyVision)
- [opencode](#opencode)
- [potplayer](#potplayer)
- [MPC video renderer](#MPC)
- [LAV filter(video, audio, splitter)](#lav)
- [K-Lite Codec Pack(DTS audio decoder)](#kLiteCodec)

---

<a name="HEVC"></a>
# HEVC
https://apps.microsoft.com/detail/9N4WGH0Z6VHQ?hl=en-us&gl=HK&ocid=pdpshare

<a id="dolbyVision"></a>
# Dolby Vision
https://apps.microsoft.com/detail/9PLTG1LWPHLF?hl=en-us&gl=HK&ocid=pdpshare

<a id="opencode"></a>
# opencode
https://github.com/sst/opencode

<a id="potplayer"></a>
# potplayer
https://potplayer.tv

<a id="MPC"></a>
# MPC video renderer
https://github.com/Aleksoid1978/VideoRenderer
<img width="1091" height="954" alt="image" src="https://github.com/user-attachments/assets/6561f8b3-3cd9-40e7-8093-7cf50c530f4e" />

##Deinterlacing
交錯影片（Interlaced，常見於舊電視節目、DVD、1080i）一幀其實是由兩個半幀（Field）組成的
| 選項 / 狀態 | 結果說明 |
| :--- | :--- |
| **關閉** | 輸出原本的幀率（例如 25fps / 30fps）。把兩個半幀合成一幀，畫面比較「靜態」。 |
| **開啟** | 幀率加倍（變成 50fps / 60fps）。把每一個半幀都當成完整一幀來輸出，動作會更流暢。 |

## Shader Video Processor（著色器視訊處理器）
統方式用較舊的硬體加速路徑處理影像, Shader Video Processor用 GPU 著色器（Pixel Shader） 來處理影像

### Chroma Upsampling
大多數影片（尤其是 1080p、4K）都是用 4:2:0 格式儲存：
亮度（Luma）：完整解析度
色彩（Chroma）：只有亮度的 1/4 解析度（寬高各一半）
好的 Chroma Upsampling 能明顯減少：
- 顏色模糊
- 色塊邊緣不乾淨
- 文字或細線周圍的色邊

播放時必須把低解析度的色彩資訊「放大」回完整解析度，這個過程就叫 Chroma Upsampling。
| 演算法品質 | 效果 | 效能消耗 |
| :--- | :--- | :--- |
| **低品質**（Bilinear 等） | 顏色容易模糊、滲色 | 很低 |
| **中高品質**（Lanczos、Jinc 等） | 顏色較清晰、邊緣較乾淨 | 中等 |
| **高品質**（NGU、SuperRes 等） | 顏色最銳利、細節保留最好 | 較高 |

| 選項 | 畫質評價 | 說明 | 建議 |
| :--- | :--- | :--- | :--- |
| **Nearest-neighbor** | 最差 | 直接複製最近的像素，容易出現明顯色塊 | 不建議 |
| **Bilinear** | 普通 | 簡單雙線性插值，顏色比較柔和、偏模糊 | 可接受 |
| **Catmull-Rom** | 最好 | 屬於高品質的 Bicubic 演算法，邊緣較清晰 | 推薦選這個 |


## Use the "Upscaling" method to reducing the frame to 50%
正常情況下：

放大畫面 → 用 Upscaling 演算法
縮小畫面 → 用 Downscaling 演算法

但有些渲染器（包含 MPC Video Renderer）會提供這個選項：
當縮小幅度很大（例如縮小到原尺寸的 50% 或以下）時，不要用 Downscaling 演算法，而是改用你設定的 Upscaling 演算法來處理。
為什麼會有這個選項？
某些高品質的 Upscaling 演算法（例如 Lanczos、Jinc）在「先放大再縮小」或特定比例縮小時，效果有時會比專門的 Downscaling 演算法更好，細節保留更多、邊緣更乾淨。
| 情況 | 建議 |
| :--- | :--- |
| **追求最高畫質** | 開啟 |
| **顯卡效能較低、想省資源** | 關閉 |
| **不確定** | 先開啟試試看 |

## dithering
當影片的色深（例如 10-bit）要輸出到你的螢幕（通常是 8-bit）時，會發生位元深度降低。如果直接截斷，容易出現色帶（Banding），也就是原本平滑的漸層變成一條一條的色塊，看起來很不自然（尤其在天空、暗部、陰影處很明顯）。Dithering（抖動） 就是在降低色深時，加入細微的雜訊來「打散」這些色帶，讓漸層看起來更平滑自然。
在 4K 螢幕 + 10-bit 影片的情況下，開啟 Dithering 幾乎都是有幫助的。
效能消耗非常小，幾乎感覺不到。
大部分高階渲染器（MPC Video Renderer、madVR）預設都會建議開啟。

# Swap Effect (Flip 或 Discard)
控制畫面用什麼方式送到螢幕（Flip 或 Discard），主要影響穩定性和效能，不影響畫質。
1. Use exclusive fullscreen（使用獨占全螢幕）

作用：進入全螢幕時，使用「獨占模式」（Exclusive Fullscreen），而不是無邊框視窗模式。
好處：
通常延遲更低
效能較好
比較容易正確輸出 10-bit / HDR

缺點：切換全螢幕時可能會閃一下，或與某些桌面效果衝突。
建議：一般建議開啟。


2. Wait for VBlank before Present（呈現前等待垂直空白區間）

作用：在把畫面送到螢幕之前，先等到螢幕的垂直同步訊號（VBlank）再送出。
好處：
減少畫面撕裂（Tearing）
畫面時間更精準，流暢度通常更好

缺點：可能會稍微增加一點延遲。
建議：建議開啟，尤其是你有開 VSync 或想追求穩定流暢時。


3. Adjust the frame presentation time（調整幀呈現時間）

作用：微調每一幀送到螢幕的時間，讓它更接近理想的顯示時間點。
好處：減少抖動、卡頓，讓播放更順。
建議：建議開啟。


4. Reinitialize device when changing display（更換顯示器時重新初始化裝置）

作用：當你把視窗拖到另一個螢幕，或切換主要顯示器時，會重新初始化渲染裝置。
好處：
避免換螢幕後顏色、解析度、HDR 狀態出錯
減少黑屏或畫面異常

缺點：切換螢幕時可能會有短暫黑屏或重新載入。
建議：有多螢幕的話建議開啟；單螢幕可開可關。



<a id="lav"></a>
# LAV filter(video, audio, splitter)
https://github.com/nevcairiel/lavfilters/releases


<a id="kLiteCodec"></a>
# K-Lite Codec Pack(DTS audio decoder)
https://codecguide.com/
<img width="1952" height="932" alt="image" src="https://github.com/user-attachments/assets/c385245d-5da9-41b4-9c26-25f76a4bb889" />













