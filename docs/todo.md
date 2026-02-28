# 🀄 台灣麻將 AI (Mortal + SimCat) 開發實作 To-Do List

這份文件記錄了將日麻 AI (Mortal) 架構改造為 16 張台灣麻將，並結合 C# (SimCat) 與 Rust 引擎進行「監督式學習 + PPO 強化學習」的完整實作路徑。

---

## 階段一：特徵工程與 Python 模型改造 (Feature & Model Definition)
> **目標：** 確立 16 張台麻的資料結構，讓 C# 產生的資料能完美對接 Python 神經網路。

- [ ] **定義狀態特徵矩陣 (State Tensor Shape)**
  - [ ] 盤點台麻所需特徵：16 張手牌、吃碰槓副露、河牌、台數狀態、花牌（若有實作）。
  - [ ] 決定 Tensor 維度：例如從日麻的 `[N, 34]` 擴充為適合台麻的維度（如考慮花牌需擴充為 `[N, 42]`）。
- [ ] **定義動作空間 (Action Space)**
  - [ ] 列出台麻所有合法動作（打牌、吃、碰、明槓、暗槓、加槓、胡、過）。
  - [ ] 確立神經網路輸出的維度（Output Layer Size），確保涵蓋所有可能動作。
- [ ] **修改 Mortal Python 端的 PyTorch 模型**
  - [ ] 調整 Input Layer 維度以接收新的台麻 State Tensor。
  - [ ] 調整 Output Layer (Policy Head & Value Head) 維度以符合台麻 Action Space。

---

## 階段二：監督式學習 (Supervised Learning / Behavioral Cloning)
> **目標：** 賦予 AI 「基礎打牌能力」，避免 PPO 初期都在亂打。

- [ ] **實作 C# 端的特徵萃取器 (Feature Extractor)**
  - [ ] 在 SimCat 中寫一段邏輯，將遊戲盤面狀態即時轉換為定義好的 Tensor 矩陣格式。
- [ ] **使用 C# SimCat 產出訓練 Log (約 10 萬局)**
  - [ ] 讓 SimCat 隨機對弈或使用啟發式規則（Rule-based）產生牌局。
  - [ ] 將 `狀態 (State)` 與 `動作 (Action)` 序列化，儲存為 Python 易讀的高效格式（推薦 `.npz` Numpy 格式或 `HDF5`）。
- [ ] **訓練第一版模型 (V1 Baby AI)**
  - [ ] 撰寫 Python `DataLoader` 讀取 C# 產生的 Log。
  - [ ] 使用 PyTorch 進行監督式學習訓練，產出第一版 `.pt` 權重檔。

---

## 階段三：核心引擎 Rust 翻寫 (High-Performance Engine Rewrite)
> **目標：** 為了應付 PPO 海量自我對弈，將 C# 邏輯翻譯為極致效能的 Rust，並取代 Mortal 原有的日麻底層。

- [ ] **使用 Antigravity 進行 TDD 逐步翻譯 (C# -> Rust)**
  - [ ] **資料結構：** 將 C# 的牌、面子、手牌結構丟給 AI 轉寫為 Rust `struct` 與 `enum`。
  - [ ] **向聽數 (Shanten) 演算法：** 讓 AI 將台麻的向聽數計算（17 張胡牌規則）翻譯為 Rust。
  - [ ] **合法動作過濾器 (Action Masking)：** 實作判斷吃、碰、槓、胡的合法性邏輯。
  - [ ] **計分與獎勵 (Reward Function)：** 實作台麻的基礎算台邏輯，並轉換為 PPO 需要的 Float Reward。
- [ ] **建立嚴格的單元測試 (Unit Tests)**
  - [ ] 在 Rust 中寫入 10~20 組邊界牌型，確保 Rust 算出來的向聽數與 C# SimCat 產生的結果 **100% 一致**。
- [ ] **實作 Python 綁定 (PyO3 / Maturin)**
  - [ ] 讓 Antigravity 幫忙寫 PyO3 Wrapper，將寫好的 Rust 台麻環境封裝成 Python 可以 `import` 並呼叫 `step()` 的模組。

---

## 階段四：PPO 自我對弈與強化學習 (Self-Play & Reinforcement Learning)
> **目標：** AI 透過與自己對戰，突破人類邏輯上限。


- [ ] **串接新版 Rust 環境與 PPO 訓練腳本**
  - [ ] 將 Mortal 原本呼叫日麻 Rust 底層的程式碼，替換成剛剛寫好的台麻 Rust 模組。
  - [ ] 載入階段二訓練好的 V1 模型作為初始大腦。
- [ ] **建立評估與監控機制 (Metrics & Arena)**
  - [ ] 接入 TensorBoard 或 Weights & Biases (WandB) 監看 `Mean Reward` 與 `Policy Entropy`。
  - [ ] **定期存檔 (Checkpointing)：** 設定每 N 局自動儲存一次 `.pt` 模型檔。
  - [ ] **AI 競技場 (Arena)：** 寫腳本讓最新版模型自動與「前一版模型」對戰 1,000 局，勝率大於 55% 才更新最佳模型。
  - [ ] **基準對手 (Baseline Bot)：** 在 Rust 中實作一個簡單的貪婪算法機器人（例如只打字牌/無腦進向聽），用來測試 AI 的絕對勝率。
- [ ] **執行訓練並設定停止條件**
  - [ ] 放著讓它跑！當平均 Reward 趨於平緩、且對戰基準機器人勝率穩定不變時，手動停止訓練。

---

## 階段五：模型導出與 C# 應用整合 (Deployment)
> **目標：** 將訓練好的最終大腦，裝回你熟悉的 C# .NET 8 生態系中。

- [ ] **導出為 ONNX 格式**
  - [ ] 寫一段 Python 腳本，將勝率最高的 `best_model.pt` 轉換為 `mortal_taiwan.onnx`。
- [ ] **在 C# SimCat 中載入模型**
  - [ ] 透過 NuGet 安裝 `Microsoft.ML.OnnxRuntime`。
  - [ ] 實作 `IAgent` 介面，讓 C# 遊戲邏輯在輪到 AI 時，能夠將盤面組裝成 Tensor，餵給 ONNX 模型並取得最佳動作。
- [ ] **開發應用服務**
  - [ ] 結合你的開發經驗，將這個 AI 封裝成 Web API 或整合進你的系統中！
