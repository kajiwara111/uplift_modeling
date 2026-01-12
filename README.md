# 選択バイアス下における Uplift Modeling
## ― 実務的なターゲティング最適化と Policy Value 評価 ―

## 概要
本リポジトリは、**施策（介入）がランダムに付与されない観察データ**を前提に、  
**予算制約下で「誰に介入すべきか（Top-K）」を最適化**し、その意思決定の価値を **Policy Value（AIPW）** で評価するための実践的なフローをまとめたものです。

単に Uplift（介入効果スコア）が高い順にリストを作るだけでなく、「そのリスト通りに運用した場合、アウトカムがどれだけ改善するか」を、  DR/AIPW（傾向スコアモデルとアウトカムモデルの両方を使う評価器）で推定・比較します。

**重要（呼称の注意）**  
本リポジトリで扱う `DR-style` は、標準的な DR（ATE に対する二重頑健性）を強く主張するものではありません。  
**overlap（比較可能領域）を重視してランキングを安定化するための residual correction（安定化ヒューリスティック）**として導入しています。最終的な意思決定は **AIPW による Policy Value** で評価します。

---

## スコープ

### やること
- 観察データからの Uplift 推定（T-Learner / DR-style）
- Top-K Policy の設計と評価（Policy Value / AIPW）
- Robustness Check（seed変更、ランダム介入との比較）
- 本番運用への示唆（オンライン評価、探索枠の考え方）

### やらないこと（意図的に）
- 未観測交絡に対する感度分析
- 最適 Policy の直接学習（Policy Learning）
- 厳密な理論的 Cross-fitting（最小限の実装に留める）
- 例外処理やクラス設計などプロダクションレベルのコーディング

---

## 主な結果（サマリ）

詳細は `02_uplift_modeling.ipynb` のまとめセクションを参照。

| 観点 | 結果（要点） |
|:---|:---|
| **Top-K介入の効果** | TreatNone（介入なし）に対して、Top-K介入は **Incremental Value がプラス**（AIPW） |
| **ターゲティング価値** | **Model Top-K vs Random Top-K** の比較で、「ランダムより良いターゲティングか」を確認 |
| **K（予算）依存** | 運用する Top-K（例：5% / 10% / 20%）によって強いモデルが変わりうることを確認 |
| **今回のデータでの推奨** | 運用 K に合わせて **T(trim)** / **T(trim+overlap)** / **DR-style(trim+overlap)** を比較し、主指標（AIPW）で採用を決めるのが実務的だと確認 |

---

## 4象限の図（Uplift Modelingの基本概念）

Uplift Modelingでは、介入効果 τ(x) と介入なしの結果 Y(0) の組み合わせで対象者を分類します。

|                    | τ(x) < 0（介入で悪化） | τ(x) > 0（介入で改善） |
|:------------------:|:---------------------:|:---------------------:|
| **Y(0)が高い**（放置でも良い結果） | 睡眠者（Sleeping Dogs）→ **避けたい層** | 確実反応（Sure Things）→ 介入してもしなくても良い |
| **Y(0)が低い**（放置だと悪い結果） | 確実不反応（Lost Causes）→ 介入しても無駄 | **説得可能（Persuadables）→ 狙いたい層（Top-K候補）** |

横軸：τ(x) = Y(1) - Y(0)（介入効果）  
縦軸：Y(0)（介入なしの結果）

---

## Quickstart（Google Colabで実行前提）

Colab 上で **Notebook を上から順に実行**すれば再現できます（ローカル環境構築不要）。

### 1. Colab で開く
1. このリポジトリを開き、対象の `.ipynb` をクリック  
2. 画面上部の **「Open in Colab」**（または Colab にアップロード）で開く  

### 2. 依存ライブラリのインストール（必要な場合のみ）
Colab の最初のセルで以下を実行します（環境により不要な場合があります）。

`!pip -q install pandas numpy scikit-learn lightgbm matplotlib seaborn`

### 3. 実行順（推奨）
1. `01_EDA.ipynb`：EDAによってその後のモデリング方針を確認　選択バイアス / overlap 診断、設計判断（トリミング・重みづけの方針）  
2. `02_uplift_modeling.ipynb`：3モデル比較 → Top-K Policy 作成 → Policy Value 評価（主指標）  

---

## なぜこのアプローチが必要なのか

一般的な予測モデルでは解決できない、以下の「意思決定の課題」に対処するために本手法を採用しています。

### 1. 「介入すべき対象」の特定（Who）
元々結果が良い人ではなく、「介入によって結果が改善する人（効果が出る層）」を特定する必要があります。
- 放置しても良い結果が出る層への無駄なコスト投下（死重損失）を防ぐ  
- 介入の効果が最大化する層（Top-K）にリソースを集中する  

### 2. 観察データの「偏り」への対処（Bias / Overlap）
利用データは RCT ではなく観察データのため、処置群と対照群の属性分布が異なる（選択バイアスがある）可能性があります。  
本リポジトリでは以下の分担で対処します。

- **トリミング（trim）**：極端なPS領域を除外し、**比較可能性（スコープ）を確保**する
- **AIPW（主評価）**：観測交絡の調整を行い、Policy Value（意思決定価値）を推定する
- **overlap weights（学習のオプション）**：比較可能領域を相対的に重視し、**ランキングを安定化**させる

---

## 想定するビジネス目的
- **施策の非ランダム性**：過去のデータに選択バイアスが含まれている  
- **予算制約**：全員には介入できず、対象を絞る必要がある  
- **Top-K 意思決定**：スコア上位 K% に介入するリストを作成したい  
- **事後評価**：ABテストが難しい状況でも、施策の価値を推定したい  

---

## 想定する利用シーン（この設計がハマるケース）

本リポジトリの設計は、**「介入するか／しないかの判断が入れ替わりうる“境界層”で、意思決定（Top-K）を改善したい」**
という状況を主に想定しています。

### 1) 想定するビジネス目的
- **予算制約**のもとで、上位K%だけに介入したい（Top-K介入）
- 重要なのは「効果を当てること」より、**介入リストを運用したときの価値（Policy Value）**
- 特に価値が大きいのは、**介入確率が極端ではなく、介入する／しないが微妙な層（傾向スコアが0.5付近）の判断を良くする**こと

### 2) なぜ overlap weights を学習に使うのか
傾向スコアが極端な領域（e(x)が0や1に近い）では、片方の群のデータがほとんどなく、  
Uplift推定が外挿になりやすく、ランキングが不安定になりがちです。

そこで学習では、比較可能性が高い **overlap領域（e(x)≈0.5）** を相対的に重視するために、**overlap weighting** を用いて、Top-K境界付近の順位を安定化させます。

- 傾向スコア： e(x) = P(T=1 | X=x)
- overlap weights（群ごとの重み付け）：
  - treated（T=1）： w_i = 1 - e(x_i)
  - control（T=0）： w_i = e(x_i)

直感的には「overlap population（ATO）」を重視する重み付けです。  

なお e(x)(1-e(x)) は “両群が十分に存在しうる領域（e≈0.5）” の重要度を表す量としてよく登場しますが、  
本リポジトリで用いる個体重みは上記のように **群ごとに定義される**点に注意してください。

### 3) DR-style（overlap-tilted residual correction）の位置づけ
本リポジトリで用いる `DR-style` は、標準的な DR（ATE に対する二重頑健性）を強く主張するものではなく、overlap領域を重視して分散を抑える“安定化目的の擬似アウトカム設計”です。

- 標準DR（IPW）は理論的に魅力がある一方、サンプルサイズが小さい／PSが偏る設定では分散が大きくなりやすく、  
  実務では重みのクリッピング等の追加設計が必要になりがちです。
- 本リポジトリでは勉強会向けに、まず「実務で使える意思決定フロー」を示すことを優先し、  
  **ランキングの安定性**に寄与しやすい DR-style を比較対象として扱います。

### 4) 最終判断は Policy Value（AIPW）
学習側でどの重み付け・推定器を使っても、最終的な意思決定は  
AIPW による Policy Value（Incremental Value）で評価します。

- 推定対象（estimand）はトリミング後集団における ATE（ATE@trim）として固定し、  
  同一の物差しで Top-K policy を比較します。

### 5) この設計がハマりにくいケース（注意）
- 効果の信号が **PSの端（ただしtrim内）** に強く偏っている場合、  
  overlap重視によりその信号が弱まり、Top-Kの当たりを落とす可能性があります。
- 母集団全体（ATE）への一般化を強く求める場合は、  
  標準DR（IPW）＋トリミング＋クリッピング等を含む別設計を検討してください。

---

## データセット：IHDP (Infant Health and Development Program)

### 元データの背景
IHDP は、1980年代に米国で実施された **低出生体重児を対象とした早期介入プログラムの RCT** に基づいています。

| 項目 | 内容 |
|:---|:---|
| **対象** | 低出生体重児（2,500g以下）、早産児 |
| **介入** | 家庭訪問、育児教育、発達支援センター通所（生後〜3歳） |
| **アウトカム** | 子どもの認知発達スコア（IQテスト等） |
| **目的** | 早期介入が発達遅延を防げるか検証 |

### 因果推論ベンチマークとしての IHDP
本リポジトリで使用しているのは、Hill (2011) が作成した **半合成データ** です。  
元の RCT データを加工し、**意図的に選択バイアスを導入**することで、観察データにおける因果推論手法のベンチマークとして広く利用されています。

### データ構成
| 変数 | 説明 |
|:---|:---|
| `treatment` | 介入有無（0/1） |
| `outcome` | 観測アウトカム（合成値） |
| `mu0` / `mu1` | 非介入時／介入時の真の期待アウトカム（検証用、学習には不使用） |
| `x1-x6` | 連続変数（出生体重、頭囲、在胎週数、母親年齢 等） |
| `x7-x15` | 二値変数（性別、双子、母親の学歴・喫煙 等） |
| `x16-x25` | サイトダミー（施設／地域） |

### 参考文献
- Hill, J. L. (2011). *Bayesian Nonparametric Modeling for Causal Inference.* JCGS.
- Infant Health and Development Program (1990). *Enhancing the outcomes of low-birth-weight, premature infants.* JAMA.

---

## 全体フロー

1. **EDA（基礎分析）**
   - 処置割当の偏りの確認  
   - 傾向スコア（PS）分布と Overlap（重なり）の診断  
   - SMD による共変量バランスの確認  

2. **設計判断（EDAに基づく）**
   - PS が極端な観測が一定割合存在 → 外挿リスクがある  
   - **固定トリミング（例：0.05）**でスコープを定義（比較可能性の確保）
   - （必要に応じて）overlap weights を学習に導入してランキングを安定化させる

3. **Uplift Modeling（3モデル比較）**
   - **T (trim)**：T-learner（トリミングのみ）
   - **T (trim + overlap)**：T-learner（overlap weightsで学習）
   - **DR-style (trim + overlap)**：residual-corrected uplift（overlap重視の安定化）

4. **Policy 評価（主評価）**
   - Uplift 上位 K% に介入する Policy を定義  
   - AIPW で Policy Value（Incremental Value）を推定  
   - AUUC / Qini は参考指標（ランキング形状の確認）

5. **Robustness Check**
   - Seed 依存性の確認（再学習によるブレ）
   - **Model Top-K vs Random Top-K**（ターゲティング価値の検証）

6. **本番運用への示唆**
   - オンライン評価（ABテスト）の設計指針
   - 探索枠の設定（Exploration-Exploitation のバランス）

---

## Uplift Modeling の位置づけ（本リポジトリの思想）

本リポジトリにおいて Uplift Modeling は、  
**「個人ごとの効果（CATE）を高精度に当てること」自体が目的ではありません**。

Uplift スコアは **Top-K policy（介入ルール）を作るための中間表現**であり、  
最終的な意思決定の良し悪しは **Policy Value（AIPW）**で判断します。

---

## 評価指標の考え方

本分析では、
- Upliftスコアで「誰に介入するか（ランキング）」を作り、
- Policy Value（AIPW*で「その決め方で本当に得をするか（増分価値）」を評価します。

| 指標 | 役割 | 扱い |
|:---|:---|:---|
| **Uplift スコア** | ランキング作成 | スコア単体では意思決定を評価しない（policy作成のための中間表現） |
| **Policy Value（AIPW）** | **主指標** | Top-K policy を運用した場合の **期待アウトカム（増分）** を推定するため最重視 |
| **AUUC / Qini** | 参考指標 | ランキング形状のチェック用。最終判断には使用しない |

---

### 学習（ランキング）と評価（価値推定）を分ける理由
観察データでは、**ランキングを安定して作ること**と、**policyの価値を（選択バイアスを補正して）見積もること**は別タスクです。  
そのため本リポジトリでは次の分担を採用します。

| フェーズ | 手法 | 対象母集団（解釈） | 重視すること |
|:---|:---|:---|:---|
| **Uplift推定（ランキング）** | T / DR-style（必要に応じて overlap weights） | トリミング後データ（外挿の影響を抑えたい） | 順位（Top-K）の安定性 |
| **Policy Value評価（主評価）** | **AIPW** | **トリミング後集団**を母集団とみなす | 意思決定としての解釈性（「K人に打ったらどれだけ伸びるか」） |

> 主評価では、推定対象を **トリミング後集団の平均効果（ATE@trim）**として固定し、同じ物差しでモデルを比較します。  
> overlap weights は「学習の安定化策」であり、常に性能が上がるとは限らないため、実証的に比較します。

---

## 推定対象（ATE / ATT / ATO）の考え方
本リポジトリの主評価（Policy Value）は、トリミング後集団を母集団とみなした意思決定価値を推定することにあります。  
したがって、推定対象（estimand）としては **ATE@trim（トリミング後集団での平均効果）**を基本に据えます。

| 推定対象 | 平均を取る対象集団 | ひとことで |
|---|---|---|
| **ATE** | 母集団全体 | 全員に打ったら平均どれだけ効く？ |
| **ATT** | 実際に介入された人 | これまで打ってきた人に効いた？ |
| **ATC** | 実際に介入されなかった人 | 打ってない人に打ったら効く？ |
| **ATO** | 比較可能性が高い人を重視（重み付き） | 比べやすい人中心だとどれだけ効く？（※主評価ではなく補助的観点） |

※AIPWは「バイアス補正の推定量」、ATE/ATT/ATOは「推定対象（誰を平均するか）」の概念です。混同しないように分けて記述しています。

---

## リポジトリ構成
| ファイル | 内容 |
|:---|:---|
| `01_EDA.ipynb` | データ理解、選択バイアス診断、設計判断（トリミング・重みづけの方針） |
| `02_uplift_modeling.ipynb` | 3モデル比較（T(trim), T(trim+overlap), DR-style(trim+overlap)）、Top-K policy作成、Policy Value評価 |
| `README.md` | 本ドキュメント |

## 参考文献

本リポジトリのアプローチは、以下の研究を参考にしています。

### Policy Evaluation / Policy Learning（観察データからの意思決定評価・学習）
- Athey, S., & Wager, S. (2021). **Policy Learning with Observational Data.** *Econometrica*, 89(1), 133-161.
  - https://arxiv.org/abs/1702.02896
  - ※本リポジトリでは Policy Learning（最適化）ではなく、Policy Value（AIPW）による評価を主軸としています

### Meta-Learners（T-Learner / X-Learner 等の統一的フレームワーク）
- Künzel, S. R., Sekhon, J. S., Bickel, P. J., & Yu, B. (2019). **Metalearners for estimating heterogeneous treatment effects using machine learning.** *Proceedings of the National Academy of Sciences*, 116(10), 4156-4165.
  - https://www.pnas.org/doi/10.1073/pnas.1804597116

### Doubly Robust / DR-Learner（二重頑健推定量による CATE 推定）
- Kennedy, E. H. (2023). **Towards optimal doubly robust estimation of heterogeneous causal effects.** *Electronic Journal of Statistics*, 17(2), 3008-3049.
  - https://arxiv.org/abs/2004.14497

### Overlap Weights（比較可能領域を重視した重み付け）
- Li, F., Morgan, K. L., & Zaslavsky, A. M. (2018). **Balancing Covariates via Propensity Score Weighting.** *Journal of the American Statistical Association*, 113(521), 390-400.
  - https://arxiv.org/abs/1404.1785

### Trimming（傾向スコアが極端な領域の除外）
- Crump, R. K., Hotz, V. J., Imbens, G. W., & Mitnik, O. A. (2009). **Dealing with limited overlap in estimation of average treatment effects.** *Biometrika*, 96(1), 187-199.
  - https://academic.oup.com/biomet/article/96/1/187/216834