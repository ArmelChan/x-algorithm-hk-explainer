# X 演算法深度解析：四大核心機制完全指南

> 適合初學者入門、中階用戶進階、高階用戶技術參考

---

## 一、候選篩選（Candidate Pipeline）

### 是什麼？

X 每天有約 5 億則貼文。每次你下拉刷新，系統需要在 1.5 秒內，從這 5 億則貼文中挑出大約 1,500 則「候選內容」，再交給後續排名引擎處理。這個篩選過程叫做 Candidate Pipeline。

它不是隨機的。它是一套多層漏斗式架構，每一層都在縮窄範圍。

---

### 初學者理解

想像你去超市，貨架上有 50 萬件產品，但你的購物車只能放 1,500 件。店員不會讓你自己逐一翻，而是事先按你的購物歷史、鄰居的喜好、熱賣榜，預先準備一批「你大概會想要的」。

Candidate Pipeline 就是那個「預先準備」的過程。

---

### 中階理解：兩大來源

候選內容分為兩條主要管道，各佔約 50%：

**In-Network（追蹤圈內）**
- 來源：你追蹤的帳號的近期貼文
- 使用工具：Earlybird（即 Search Index），一個全文搜尋引擎
- 篩選方式：Light Ranker 先做初步評分，取分數最高的幾百則
- 核心邏輯：Real Graph 模型——根據你過去與某作者的互動頻率，預測你有多大可能對他的新貼文感興趣

**Out-of-Network（追蹤圈外）**
- 來源：你沒有追蹤的帳號
- 使用工具：
  - SimClusters：將所有用戶分成約 145,000 個興趣群組（cluster），你所屬的群組喜歡什麼，系統就推什麼
  - UTEG（User Tweet Entity Graph）：一個記憶體內的互動關係圖，追蹤「誰 like 了誰的文」，再做圖遍歷找出你可能感興趣的陌生人貼文
  - TwHIN Embedding：一個訓練在整個 X 互動歷史上的 embedding 模型，用向量距離衡量貼文與你的相關性

---

### 高階理解：系統架構

整個 Candidate Pipeline 是模組化設計（composable framework），由四類組件組成：

| 組件類型 | 功能 |
|---|---|
| Sources | 從 Thunder（追蹤圈）和 Phoenix（排名引擎）取候選 |
| Hydrators | 為候選貼文補充 metadata（作者資訊、媒體類型、語言等） |
| Filters | 在評分前剔除不合格的貼文（見第三節） |
| Scorers | 呼叫 Phoenix 對每則候選貼文預測互動機率，輸出最終分數 |

**Thunder** 是一個吃 Kafka 串流的記憶體貼文儲存，實現追蹤圈內容的亞毫秒級查詢。

**Phoenix** 是以 Grok 為基礎的 Two-Tower Transformer：分別為用戶和貼文各建一個 embedding，計算點積（dot product），輸出互動分數。它取代了原本 4800 萬參數的 MaskNet 模型。

信號使用上，系統同時消費以下數據：
- 你的 like、reply、retweet、bookmark 歷史
- 你點進過的帳號 profile
- 你在某則貼文上停留的時間（dwell time）
- 你使用的裝置、語言設定、地理位置

開源代碼路徑：`candidate-pipeline/`（Rust 寫成）

---

## 二、主頁混合器（Home Mixer）

### 它是什麼？它不是 Explore。

**Home Mixer 對應的是「For You」分頁，即你的主頁推薦流。**

- **For You 分頁** = Home Mixer 的輸出。混合了追蹤圈 + 演算法推薦的內容。
- **Following 分頁** = 純時序排列，只顯示你追蹤的帳號，不經演算法篩選。
- **Explore / 搜尋頁** = 完全獨立的系統，用不同的 trending 和推薦邏輯，與 Home Mixer 無關。

所以「主頁混合器」不是 Explore，也不是你的 Following 分頁，它是 For You 那個由演算法自行組裝的頁面背後的大腦。

---

### 初學者理解

Candidate Pipeline 送來約 1,500 則貼文。Home Mixer 的工作是決定：最終你實際看到的那幾百則，順序是什麼？

它不儲存任何數據。它的角色是「協調者」，負責把所有服務的輸出整合成一條有序的 feed。

---

### 中階理解：六個處理步驟

Home Mixer 的管道流程如下：
用戶請求 → 候選來源 → 貼文 Hydration → 過濾 → 評分排名 → 最終選取


1. **Query Hydration**：讀取你的用戶狀態（你的興趣、最近互動、語言設定等）
2. **Candidate Sourcing**：向 Candidate Pipeline 要求候選貼文
3. **Post Hydration**：補充每則貼文的完整資訊（作者驗證狀態、媒體內容、轉發來源等）
4. **Filtering**：移除你已封鎖的帳號、已看過的貼文、不符合安全標準的內容
5. **Scoring**：Phoenix 為每則貼文打最終分
6. **Injection**：在 feed 中插入廣告、「你可能認識」模組、話題標籤等非貼文元素

---

### 高階理解：排名信號與權重

排名分數的計算公式（來自開源代碼）：Final Score = Σ (P(action_i) × weight_i)


確認的互動權重如下：

| 互動行為 | 權重 | 說明 |
|---|---|---|
| 作者回覆你的回覆 | +75 | 最強信號，雙向互動 |
| Reply（你回覆） | +13.5 | 高成本互動，代表真實興趣 |
| Profile 點入後互動 | +12.0 | 代表你對作者有持續興趣 |
| 點入串文後互動 | +11.0 | 深度閱讀信號 |
| Dwell time 超過 2 分鐘 | +10.0 | 停留時間 |
| Bookmark | +10.0 | 儲存代表長期價值（有 5x 乘數） |
| Retweet | +1.0 | 較低，因為可以無意識轉發 |
| Like | +0.5 | 最低，門檻最低的互動 |
| Mute / Block / Report | 強負分 | 直接懲罰作者觸及率 |

對普通用戶的實操意義：
- 不要只期待 Like，追求 Reply 和 Bookmark 才能真正拉高觸及
- 留言後如果原作者有回應，那串文的分數會大幅提升
- 發文後的頭一小時互動率決定該貼文後續能否進入更廣泛的 Out-of-Network 推薦

---

## 三、濫用防治（Abuse Enforcement）

### 它的作用

Abuse Enforcement 是 Candidate Pipeline 和 Home Mixer 之間的守門員。它的工作：過濾掉不應該出現在 feed 中的內容，包括垃圾貼文、機器人帳號，以及違反規定的行為。

---

### 初學者理解：什麼會被過濾？

以下情況會讓你的貼文被降低分數或直接過濾：

- 貼文含有已知的垃圾詞彙或重複性句型
- 連結指向被標記的外部網站
- 發文頻率異常（每分鐘多則、24 小時不停）
- 帳號創立時間極短但互動量極大
- 全大寫字母或過多特殊符號

---

### 中階理解：機器人帳號的判定方式

「機器人帳號」不是指有 bot 字樣的帳號，而是指行為模式類似自動化程序的帳號。

**X 的 Botometer 類系統使用以下信號判斷：**

| 信號類別 | 具體特徵 |
|---|---|
| 帳號元數據 | 帳號年齡、有否頭像、bio 完整度、是否驗證 |
| 發文行為 | 發文時間間隔的規律性、每日發文量、內容重複率 |
| 互動模式 | Follow/Unfollow 的速率、回覆的回應率、是否只轉發不原創 |
| 網絡關係 | 追蹤者的帳號年齡分布、互相追蹤比例 |
| 內容特徵 | 是否大量使用相同 hashtag、連結是否多樣或集中 |

機器人帳號有幾種類型：

- **垃圾型 Bot**：大量發送相同廣告或釣魚連結
- **放大型 Bot（Amplification Bot）**：專門 like 和 retweet 某類內容以人工推高觸及
- **觸發型 Bot（Trigger Bot）**：監聽特定關鍵字後自動回覆，製造虛假互動
- **休眠型帳號**：長期沉默，但在需要時集體啟動刷分

根據 2026 年的學術研究，最活躍的 500 個可疑 Bot 帳號佔所有新聞相關連結分享的 22%。

---

### 高階理解：如何避免誤判成垃圾帳號？

X 的 Spam Filter 是機器學習模型，並非純規則系統，存在誤判空間。以下行為會提高你被誤標的風險：

**高風險行為：**
- 短時間內 Follow 大量新帳號（尤其每日超過 400 個）
- 使用第三方 API 自動發文（即使內容是正常的）
- 同一 IP 下多個帳號同時互動
- 多個帳號同時發送相同文字

**保護自身的方式：**
- 維持發文的時間不規律性（人類不會每小時準時發文）
- 確保每則貼文有獨立的原創文字，避免大量複製貼上
- 讓帳號資料完整：頭像、bio、已驗證手機
- 若使用自動化工具（如排程發文），確保工具使用官方 API 且有合理的速率限制

對開發者而言，X 的 Abuse Enforcement 系統有一個 Safety Layer，在 Candidate Pipeline 的 Filter 階段運行，使用 BERTweet 類模型做內容分類，同時維護一個動態更新的黑名單（涵蓋 IP、裝置指紋、帳號集群）。

---

## 四、內容分級系統（Content Labeling System）

### 它是什麼？

內容分級系統是 X 用來自動為貼文打標籤的機制，決定：
- 這則貼文能推薦給哪些用戶群體
- 是否需要附加警告標籤
- 是否僅限年齡驗證用戶可見
- 是否完全從推薦系統中移除（但保留在帳號頁面）

它不是人工審核，而是自動化 ML 分類系統，再配合人工審核隊伍處理邊界案例。

---

### 初學者理解：五個分級層次

想像一個電視節目評級系統：

| 分級 | 對應情況 | 觸及影響 |
|---|---|---|
| 無標籤 | 普通貼文 | 完全推薦 |
| Sensitive Content | 暴力畫面、輕微裸露 | 附警告，可設定是否顯示 |
| Adult Content (NSFW) | 成人內容，需標記 | 僅向開啟成人內容的帳號推薦 |
| Misinformation Label | 被第三方核查機構標記為不實資訊 | 觸及大幅降低 |
| Suspended / Removed | 嚴重違規 | 完全移除出推薦系統 |

---

### 中階理解：Community Notes 的角色

Community Notes（前身 Birdwatch）是一個群眾協作的事實核查機制：

- 任何人都可以為貼文撰寫補充說明（Note）
- Note 需要來自不同政治傾向的用戶共同評為「有幫助」才會顯示
- 一旦某則貼文有被顯示的 Note，其推薦觸及通常顯著下降

這個設計的目的是防止單一陣營主導事實核查，但批評者認為它反應速度太慢。

對於創作者而言，Community Notes 是無法直接申訴的——它是去中心化的，X 平台本身不控制哪些 Note 最終顯示。

---

### 高階理解：Safety Level 分類架構

X 的內容分類器在技術上分兩層：

**Layer 1：媒體分類（Media Classifier）**
- 分析圖片和影片的視覺內容
- 輸出：Safe / Sensitive / Adult 三個類別
- 模型：CNN 架構，在大量人工標記數據上訓練

**Layer 2：文字分類（Text Classifier）**
- 分析貼文文字
- 識別仇恨言論、騷擾、垃圾信息、不實健康資訊等
- 模型：BERTweet（針對社交媒體語言微調的 BERT 變體）

帳號層面的分級：
- **Safety Level**：每個帳號有一個隱性安全評分，影響其貼文被推薦的基礎機率
- 帳號過去被 Report、被 Block、Content Label 的歷史都會累積影響這個分數
- 新帳號缺乏歷史，Safety Level 預設較低，需要時間建立信任

**對高階用戶的意義：**
如果你的帳號在某段時期內因違規被降低 Safety Level，換 IP 或創新帳號並不能完全解決問題——X 使用裝置指紋和行為聚類，能夠關聯你的新帳號與舊帳號。

---

## 給三類讀者的學習路線建議

**初學者——讀懂這份文件後可做的事：**
- 改善你的個人 bio 和頭像，提高帳號基礎可信度
- 發文後積極回覆留言，爭取雙向互動分數
- 避免使用批量追蹤工具

**中階用戶——可以進一步研究：**
- 了解 SimClusters 的工作原理，調整你的發文主題以鞏固在特定 cluster 的標籤
- 研究 Real Graph 的運作，策略性地與高互動作者互動以提升自己的信號
- 測試不同互動行為（Bookmark vs Like）對觸及的實際影響

**高階用戶——技術層面的參考資源：**
- GitHub 原始碼：[xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)（Rust + Python）
- 官方開源文檔：[twitter-the-algorithm.mintlify.app](https://twitter-the-algorithm.mintlify.app/how-it-works)
- 學術論文：Ferrara et al. 2022，"Twitter spam and false accounts prevalence, detection, and mitigation"
- 系統設計分析：[singhajit.com/system-design/x-twitter-for-you-algorithm](https://singhajit.com/system-design/x-twitter-for-you-algorithm)

---
關鍵更正補充說明:
- Home Mixer 不是 Explore，Explore 是完全獨立的系統。Home Mixer 是 For You 分頁的後端協調層，它本身不儲存數據，只負責協調各服務輸出。
- 機器人帳號判斷是基於行為模式（時間規律性、Follow 速率、內容重複率、網絡關係），而非帳號名稱，且最新研究顯示最活躍的 500 個可疑 Bot 帳號佔新聞相關貼文的 22%
- 內容分級是兩層 ML 系統（視覺 CNN + BERTweet 文字分類），帳號本身有隱性 Safety Level 評分，歷史違規紀錄會累積影響。


*本文件基於 xai-org/x-algorithm 開源代碼、學術研究及可信第三方技術分析整理，屬非官方科普性質。*

