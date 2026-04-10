# Helm 升級導致 PVC 資料消失：一次 openAB 踩坑紀錄

> 透過開源專案 openAB 學習新技術：https://github.com/openabdev/openab

> **版本資訊**
> - Kubernetes / Helm / PVC
> - 更新日期: 2026-04-10

---

在 openAB 上跑的 AI 做了一次版本更新後，裡面的 prompt 檔案全部消失了。

K8s（Kubernetes）是一個容器管理平台，可以用 Docker 來類比，但它能部署在雲端做水平擴展，自動管理容器的生命週期。Helm 是 K8s 的套件管理工具，搭配 Chart（安裝包）來安裝、更新、刪除、回滾應用，讓你不必手動寫一堆 YAML 再一個一個 apply。

而容器裡的檔案預設是暫時的，重啟就沒了。要長期保留資料就得設 PVC（持久化儲存），掛載到容器裡。但這次的問題是：新版 Chart 改了結構，導致 PVC 名稱變了，Helm 升級時就把舊的 PVC 刪掉，資料跟著消失。

這篇完整紀錄這個問題的根因、怎麼發現的、以及之後怎麼避免。

---

## TL;DR

- **問題**：Helm upgrade 後，容器裡的 prompt 檔案全部消失
- **根因**：新版 Chart 改了 values.yaml 結構，PVC 名稱從 `openab` 變成 `openab-kiro`，Helm 把舊 PVC 刪了
- **關鍵機制**：Helm upgrade 會刪除「舊 release 有但新 release 沒有」的資源
- **防禦**：升級前備份、加 `helm.sh/resource-policy: keep`、比對新舊模板
- **適用場景**：任何用 Helm 管理有狀態應用的情境

---

## 解決的問題

### 使用情境

在 openAB 上跑 AI agent，agent 的 prompt 檔案存在容器內。夥伴執行版本升級後，prompt 檔案全部消失。

**預期行為：**
- 升級後 AI agent 正常運作，資料保留

**實際行為：**
- 升級後 prompt 檔案消失，AI agent 無法正常運作

**觸發條件：**
- 新版 Chart 的 values.yaml 結構改變
- PVC 名稱因此改變
- 舊 PVC 沒有設定 `helm.sh/resource-policy: keep`

---

## 核心概念

### K8s、Helm、Chart 的關係

```
K8s（Kubernetes）
  容器管理平台，Docker 的雲端進化版
  ├── 水平擴展（流量大了自動加機器）
  ├── 自動重啟（容器掛了自己拉起來）
  └── 生命週期管理（部署、更新、回滾）

Helm
  K8s 的套件管理工具（像 apt / brew 之於 OS）
  └── 用 Chart 來打包、安裝、更新、刪除、回滾應用

Chart（Helm 的安裝包）
  ├── templates/     K8s 資源的 YAML 模板，用變數佔位符
  ├── values.yaml    變數預設值（token、storage size…）
  └── Chart.yaml     Chart 名稱、版本、container image 版本
```

Helm 的運作流程：

```bash
helm install/upgrade <release> <chart> --set key=value
```

```
參數 + values.yaml
    ↓
填入 templates/ 裡的佔位符
    ↓
產生完整的 K8s YAML
    ↓
送給 K8s 執行（建立/更新 Pod、Service、PVC…）
```

重點：Helm 讓你不必手動寫一堆 YAML 再一個一個 `kubectl apply`。

### PVC（Persistent Volume Claim）

容器裡的檔案預設是暫時的 — 容器重啟就沒了。PVC 是 K8s 提供的持久化儲存機制：

```
沒有 PVC：
  Pod 重啟 → 容器內檔案消失

有 PVC：
  Pod ──mount──→ PVC ──bind──→ Host 磁碟
  Pod 重啟 → PVC 還在 → 資料保留
```

- PVC 是獨立於 Pod 的資源
- 實際資料存在 Host 機器的磁碟上
- 即使 Pod 被刪除重建，只要 PVC 還在，資料就還在

### Helm Upgrade 的資源管理邏輯

這是這次踩坑的關鍵：

```
Helm upgrade 比對邏輯：

  舊 release 的資源清單    vs    新 release 的資源清單
  ─────────────────────         ─────────────────────
  Pod: openab                   Pod: openab
  Service: openab               Service: openab
  PVC: openab          ←──×──   PVC: openab-kiro    ← 名字不同！

  結果：
  - Pod、Service → 更新（名字一樣）
  - PVC openab → 刪除（新 release 裡不存在）
  - PVC openab-kiro → 新建（舊 release 裡沒有）
```

Helm 的邏輯是：**舊 release 有但新 release 沒有的資源 → 刪除**。

---

## 問題根因分析

### 為什麼 PVC 名稱會變？

新版 Chart 為了支援多 Agent，values.yaml 的結構改了：

```yaml
# 舊版 values.yaml
storage: 1Gi
token: <TOKEN>

# 新版 values.yaml（多包一層 agents.kiro）
agents:
  kiro:
    storage: 1Gi
    token: <TOKEN>
```

模板裡用 values 的路徑來組 PVC 名稱，結構一改，產生的名稱就不同了：

```
舊模板 → PVC 名稱: openab
新模板 → PVC 名稱: openab-kiro
```

### 為什麼 Helm 會刪掉舊 PVC？

兩個條件同時成立：

1. 舊 PVC `openab` 在新 release 的資源清單裡不存在
2. 舊 PVC 沒有設定 `helm.sh/resource-policy: keep` annotation

如果有這個 annotation，Helm 就會跳過刪除：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openab
  annotations:
    helm.sh/resource-policy: keep    # ← 告訴 Helm「不要刪我」
```

### 怎麼發現的？

```bash
# 1. 查當前部署的 PVC 名稱
kubectl get pvc
# → 看到 openab

# 2. 用 helm template 預覽新版會產生什麼
helm template <release> <new-chart> --set ...
# → 看到 PVC 名稱變成 openab-kiro

# 3. 比對 → 名字不同 → 升級會出事
```

---

## 操作指引：安全升級 Helm Chart

### 升級前檢查

```bash
# Step 1: 記錄當前 PVC
kubectl get pvc -n <NAMESPACE>

# Step 2: 預覽新版 Chart 會產生什麼資源
helm template <RELEASE> <NEW_CHART> --set key=value

# Step 3: 比對 PVC 名稱是否一致
# 如果不一致 → 進入防護流程
```

### 防護流程（PVC 名稱會變的情況）

```bash
# 方法 1: 備份 PVC 資料
kubectl cp <NAMESPACE>/<POD_NAME>:/path/to/data ./backup/

# 方法 2: 手動加上 resource-policy annotation
kubectl annotate pvc <PVC_NAME> -n <NAMESPACE> \
  helm.sh/resource-policy=keep

# 方法 3: 兩者都做（建議）
```

### 升級後確認

```bash
# 確認舊 PVC 還在（如果加了 keep）
kubectl get pvc -n <NAMESPACE>

# 確認新 Pod 正常運作
kubectl get pods -n <NAMESPACE>

# 如果需要，把舊 PVC 的資料搬到新 PVC
kubectl cp ./backup/ <NAMESPACE>/<NEW_POD_NAME>:/path/to/data
```

---

## 最佳實務

- **有狀態的應用，PVC 模板一律加 `helm.sh/resource-policy: keep`**
  - 這應該是 Chart 作者的責任，但很多 Chart 沒做
  - 使用者可以自己手動加
- **升級前一律跑 `helm template` 比對**
  - 特別注意 PVC、Secret、ConfigMap 等有狀態資源的名稱
- **升級前一律備份 PVC 資料**
  - 即使名稱沒變，也可能有其他問題
- **用 `helm diff` plugin 更方便**
  - `helm diff upgrade <release> <chart>` 可以直接看差異

---

## Troubleshooting

### 症狀：升級後容器內檔案消失

- **可能原因**：PVC 被 Helm 刪除（名稱改變 + 沒有 keep policy）
- **處理方式**：
  1. `kubectl get pvc` 確認 PVC 是否還在
  2. 如果 PVC 已被刪，資料無法復原（除非有備份或底層儲存還在）
  3. 從備份還原，或重新建立資料

### 症狀：升級後出現兩個 PVC

- **可能原因**：加了 `resource-policy: keep` 後，舊 PVC 保留，新 PVC 也建立了
- **處理方式**：
  1. 把舊 PVC 資料搬到新 PVC
  2. 確認新 Pod mount 的是新 PVC
  3. 手動刪除舊 PVC

### 症狀：helm upgrade 報錯 resource already exists

- **可能原因**：手動建立的資源跟 Helm 管理的資源衝突
- **處理方式**：
  1. `kubectl annotate` 加上 `meta.helm.sh/release-name` 和 `meta.helm.sh/release-namespace`
  2. `kubectl label` 加上 `app.kubernetes.io/managed-by=Helm`

---

## 總結

這次踩坑的核心教訓：Helm upgrade 不只是「更新」，它會主動清理不再存在的資源。對於有狀態的應用（PVC、資料庫），升級前的比對和備份是必要步驟，不是可選的。

關鍵概念：
- Helm 用 Chart 模板 + values 產生 K8s 資源
- values 結構改變 → 資源名稱可能改變
- Helm upgrade 刪除「舊有新無」的資源
- `helm.sh/resource-policy: keep` 是 PVC 的保命符

---

## 參考資源

- [openAB GitHub](https://github.com/openabdev/openab)
- [Helm 官方文檔 - Resource Policy](https://helm.sh/docs/howto/charts_tips_and_tricks/#tell-helm-not-to-uninstall-a-resource)
- [Kubernetes PVC 官方文檔](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Helm Diff Plugin](https://github.com/databus23/helm-diff)
