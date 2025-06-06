# 信用卡詐欺交易檢測專案

本專案旨在運用機器學習技術，分析信用卡交易數據，以期能有效地識別並預防潛在的金融詐欺行為。專案將分別探討監督式學習與非監督式學習方法在詐欺檢測領域的應用與效能。

---

##  監督式模型：XGBoost

監督式學習選用梯度提升決策樹（Gradient Boosting Decision Trees）的進階實作——XGBoost作為主要的分類模型。XGBoost 以其卓越的效能、高效率以及處理稀疏數據和缺失值的內在能力而聞名，非常適合處理複雜的分類任務，如詐欺檢測。

### 使用技術

* **核心分類器**：採用 `XGBoostClassifier`。此模型透過整合多個弱學習器（決策樹）來建構一個強健的預測模型，並透過梯度提升算法逐步優化損失函數，有效降低偏差並控制方差。
* **處理類別極度不平衡問題**：
    * **權重調整機制**：利用 `scale_pos_weight` 參數，此參數根據訓練數據中多數類別與少數類別（詐欺交易）的比例來自動調整，賦予少數類別樣本更高的錯誤懲罰權重，促使模型更關注對詐欺交易的正確識別。
    * **下採樣策略 (Undersampling)**：為進一步平衡數據分佈，從佔絕大多數的正常交易樣本中，隨機抽取一部分（例如，5000筆）與所有詐欺交易樣本合併，形成一個數據規模更易於管理且類別分佈相對均衡的訓練子集。此舉有助於防止模型過度偏向多數類別，並加快訓練速度。
* **特徵篩選與工程**：
    * **相關性分析與選擇**：優先選用與「Class」（是否詐欺）欄位具有較高統計相關性或業務邏輯上重要的特徵（例如，經過PCA降維後的 `V1` 至 `V28` 中的特定欄位，以及標準化後的 `Amount` 交易金額）。這樣做可以降低模型複雜度，減少噪聲影響，並可能提升模型的解釋性與效能。
    * **數據標準化**：對如 `Amount` 這類數值範圍差異較大的特徵進行標準化處理，使其符合標準常態分佈，這有助於梯度下降等算法的穩定收斂。
* **資料分割與模型驗證**：
    * **分層抽樣分割**：使用 `train_test_split` 函數時，啟用 `stratify` 選項，確保在分割訓練集與測試集時，兩個集合中詐欺樣本與正常樣本的比例與原始平衡後數據集中的比例保持一致。這對於評估模型在不平衡數據上的真實泛化能力至關重要。
    * **效能評估指標**：除了準確率 (Accuracy)，更側重於召回率 (Recall)、精確率 (Precision) 及 F1 分數 (F1-Score)，這些指標能更全面地反映模型在識別少數類別（詐欺交易）方面的能力。同時，也會參考 AUC-PR（Precision-Recall Curve下的面積）作為重要評估標準。

---

##  非監督式模型：Isolation Forest

在非監督式學習方面，採用 Isolation Forest（孤立森林）演算法。此演算法特別適用於異常檢測任務，其核心思想是異常點通常比正常點更容易被孤立。它不需要預先標記的數據，可以直接從數據本身的結構中識別出與大多數樣本行為模式顯著不同的個體。

### 使用技術

* **核心模型**：採用 `IsolationForest`。該模型隨機選擇特徵並在該特徵的隨機分割點上切割數據，重複此過程直到每個數據點都被孤立。異常點由於其稀疏性和獨特性，平均而言會更快地被孤立，即其在樹中的路徑長度較短。
* **訓練策略與數據應用**：
    * **針對性訓練數據**：為了強化模型對「正常」交易模式的學習，並防止詐欺樣本的特徵模式（儘管未知）影響正常模式的定義，`IsolationForest` 模型將僅在確認為正常交易的樣本子集（即 `y == 0` 的數據）上進行訓練。這種策略有助於構建一個更純粹的「正常行為」基線。
    * **避免資訊洩漏**：通過僅使用正常樣本訓練，可以有效避免測試集中的詐欺標籤信息間接影響到模型的異常判斷標準，確保評估的客觀性。
* **異常分數計算與閾值決策**：
    * **異常分數機制**：利用模型的 `decision_function` 方法來計算每個樣本的異常分數。原始的 `decision_function` 產生的分數通常是負值較大代表越異常（或路徑長度越短）。
    * **分數轉換與解讀**：為了更直觀地將高分與高異常風險對應，通常會對原始異常分數進行反向處理（例如，取負值或使用 `1 - score` 的形式）。轉換後，分數越高的樣本被認為是詐欺交易的風險越高。
    * **動態閾值選擇**：由於詐欺交易的比例極低且其異常分數的絕對值可能沒有固定標準，因此採用基於百分位數的方法來決定分類閾值。例如，可以選取測試集（或驗證集）上所有樣本異常分數的第 95 或 96 個百分位數（或其他經過實驗調整的百分位數）作為將樣本劃分為「正常」或「異常/詐欺」的臨界點。高於此閾值的樣本被標記為潛在詐欺。
* **模型參數調整**：
    * `n_estimators`：森林中樹的數量，數量越多通常越穩定，但邊際效益遞減。
    * `max_samples`：構建每棵樹時使用的樣本數量，可以設為 `'auto'` 或具體數值/比例，影響樹的多樣性。
    * `contamination`：數據集中異常點的預期比例，設為 `'auto'` 時算法會自動估計，也可以手動設定一個較小的值（如實際詐欺比例）。
    * `bootstrap`：是否在構建樹時使用有放回抽樣，設為 `True` 通常能增加隨機性。

---
