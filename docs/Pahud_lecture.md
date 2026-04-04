# Pahud 演講

> 這是一場由一位資深工程師（曾任職 PC Home、愛情公寓、AWS）分享的技術演講，涵蓋從 Web 2.0 時代到 AI Agent 時代的職涯歷程、技術趨勢觀察與人生建議。

---

## 1. .com 時代的浪潮與抉擇

90 年代末，網路泡沫正值高峰。講者在 2000 年大四時便投入了一家 .com 公司工作，拿到了當時相當不錯的薪資報酬。這讓他開始認真思考一個問題：是否應該繼續完成學業，還是直接投身這波席捲全球的網路浪潮？這個在學業與機會之間的拉扯，是那個年代許多技術人共同面對的抉擇。

---

## 2. Web 2.0 時代與基礎建設的磨練

隨即在2004年左右進入了所謂的 Web 2.0 時代。當時最具代表性的產品是 Flickr——滑鼠移上去背景會變成淡黃色，點一下就能編輯，這在當時是令人驚豔的 UI 設計。Web 2.0 的核心概念是 User Generated Content（UGC），從 1.0 的被動式入口網站（門戶網站），轉變為用戶主動產生內容。那個年代的代表產品包括 Flickr、明日報個人新聞網、無名小站等。

講者進入 PCHome 工作時，整個系統跑的是大型Sun系統，他覺得太難用且效能差，於是一口氣全部換成 Linux。這個決定讓他奠定了紮實的 Linux 系統管理基礎。明日報個人新聞網、PCHome 相簿等服務都是他和同事用這套方法救起來的，他和一群同事們一起用光碟一台一台主機安裝，把學長教的東西派上用場。

這段經歷的核心啟示是：基礎功夫的重要性。即便是門戶網站等級的大型系統，只要有紮實的基礎，就能搞定。這套從學長那裡學到的 Linux 運維能力，後來在愛情公寓、上海創業時期都持續發揮作用。

**提到的技術：** Sun Solaris、Linux、Web 2.0、UGC

---

## 3. 上海創業時期：CDN、機房與早期基礎設施

2006 年左右，講者跟隨幾位附中學弟前往上海創業（愛情公寓）。他和幾位優秀的夥伴搞定中國的 CDN、機房、Load Balancer 等基礎設施。那個年代還沒有「雲端」的概念，所有東西都是實體設備：

- 一台 L4 Switch Load Balancer（如 NetScaler）要幾十萬台幣，甚至要買兩台才能高可用
- NAS 存儲設備（如 NetApp）也要好幾百萬台幣
- 這些都是非常大的資本投資

講者強調，他從 PCHome 學到的那套 Linux 運維技能，在上海依然完全適用。技術的可遷移性是職涯中非常重要的資產——你在一個地方學到的紮實基礎，可以帶到下一個戰場繼續使用。

**提到的技術/服務：** CDN、Load Balancer（NetScaler）、NAS（NetApp）

---

## 4. 攝影帶來的意外人生轉折

在上海期間，講者大量投入攝影，用底片拍攝，堅持與眾不同的風格。他的一張在北疆拍攝的照片獲得了 Flickr 年度第一名，還被選為 Flickr 第一本寫真集的封面，自由時報也進行了採訪報導。

攝影把他帶到了許多意想不到的地方：背包旅行去印度（兩次，自己坐火車）、去恆河看燒屍體思考人生、去古巴拍婚紗照。他和後來的妻子（上海人）在印度沙漠裡自己帶著淘寶買的便宜婚紗，用底片自拍婚紗照。

這段經歷的核心觀點是：

- **做任何事情都要做到與眾不同**——不一定是最好，但一定要跟別人不一樣。這個性格特質貫穿了講者的整個職涯。
- **一個決定的受益者可能不是你自己**——講者去紐約後，收穫最大的反而是他的妻子（在紐約接案、做 Manager 管理美國人）和女兒（在曼哈頓念書、英文突飛猛進）。做決定時要考慮對家人的連帶影響。

---

## 5. 從零開始進入 AWS：證照策略與職涯轉型

2014-2015 年，講者從愛情公寓離開後，才第一次接觸 AWS，連 EC2、ELB 是什麼都不知道。但他認定 AWS 是雲端領域走在最前面的，決定「拿下它」。

問題是：沒用過任何 AWS 服務，怎麼去面試？他的策略是：

1. 發現 AWS 有證照體系（當時只有 5 張證照）
2. 花 3 個月時間把 5 張證照全部考到
3. 成為台灣第一個 All Five Certified Holder

去面試時，因為有了證照的加持，講者至少在 Knowledge 層面已經不怕了。最終成功進入 AWS 擔任 SA（Solutions Architect）。

**核心策略：** 當你沒有實戰經驗時，用證照來證明你的知識深度，至少消除面試中「你懂不懂」的疑慮。這不是最好的方式，但在零經驗的情況下是有效的敲門磚，並且展現 insist on the highest bar 的 leadership principle.

---

## 6. Serverless 浪潮與擁抱新技術的心態

進入 AWS 後，講者在 2019 年從一般 SA 轉型為 Specialist SA，專注於 Serverless 和 Container 領域（Lambda、API Gateway、Event Driven Architecture）。

這次轉型的背景是：AWS 服務越來越多，不可能什麼都會，必須 Specialize 某一塊領域。而當時最紅的就是 Serverless。

講者描述了第一次接觸 Serverless 時的震撼：以前所有東西都要自己架機器，現在一個 Function 會在你不知道的某台機器上被觸發，處理完就消失了，什麼機器都不用管。他的第一反應是「完了，工程師要失業了」。

但他很快意識到：**當你發現以前學的東西快要沒用的時候，就是要擁抱你覺得即將崛起的新東西。你不會失業，你只會被更需要。但如果你緊抓著已經過時的東西不放，那就不好說了。**

他的做法是：全力投入，把自己變成這塊領域最懂的人。當其他同事搞不懂、每個人都來問你的時候，你就贏了。

**提到的技術：** AWS Lambda、API Gateway、Fargate、Event Driven Architecture、Serverless、Container

---

## 7. CDK 開源貢獻：如何影響全世界

講者在 AWS 期間發現了 CDK（Cloud Development Kit）和 CloudFormation 的威力——用程式碼描述基礎設施，按一下全部自動生成。他又一次覺得「工程師要失業了」，但隨即決定：這是未來的趨勢，一定要做到台灣最強。

關鍵是：**老闆沒有叫他寫 CDK**。老闆要他服務客戶、創造 revenue。CDK 沒有直接的 revenue。但他堅持投入，因為他認定這是未來趨勢。

他的具體做法：

1. **在 GitHub 上搶先提交 PR**——每當 AWS 發布新服務或新功能，他第一個寫出 CDK 的 PR（當時沒有 AI，全部手寫）
2. **被打回來就學習**——那個 L7 Principal SDE 不斷打回他的 PR，他就從每次被打回來的 feedback 中學習。從 L7 身上學到的東西非常珍貴
3. **終於被 merge**——當 PR 被 merge 的那一刻，等於是他定義了全世界 AWS 用戶使用這個服務時的程式介面
4. **立刻去 Twitter 宣傳**——每次 PR 被 merge，馬上去 Twitter 發文, Tag 這個領域最有影響力的人或者團隊leader。最重要的是讓那些領域裡最重要的人轉發你的東西。一次轉發等於中一次樂透，全世界這塊領域的人都會知道你

這個策略的結果：日本社群稱他為「CDK 最速男」。2022 年，那位 L7 Principal SDE 主動找他，說「全世界只有你能做這個職缺」——Open Source Community Manager，一個在亞洲從未有過的職位。面試過程幾乎就是聊天，因為對方早就認識他了。

**核心方法論：**
- 找到你認為的未來趨勢，即使老闆沒要求也要投入
- 從頂尖的人身上學習（被打回來是最好的學習機會）
- 做完好事只是一半，另一半是讓別人看見（Visibility + Marketing）
- 讓最重要的人轉發你的成果，建立你在該領域的知名度

**提到的技術：** AWS CDK、CloudFormation、GitHub PR

---

## 8. 紐約生活與 AI 時代的日常實踐

2022 年講者轉調到紐約，至今四年。他分享了 AI 如何改變他的日常生活：

每天早上開車送女兒上學時（Tesla FSD 自動駕駛），他會按住語音助理，用中文或英文問各種技術問題。例如：

- 「昨天有一個 supply chain attack 的新聞，你可以解釋什麼是 supply chain attack 嗎？」
- Tesla 的 Grok 可以自動連網、即時抓取 Twitter 資訊，用中文回答, 可惜不能產生 markdown 甚至寄出來，也就是討論完無法留下筆記
- 於是他用 OpenClaw（龍蝦）做同樣的事，背後是 GPT-5.4 也能做到即時查新聞的需求

更進一步，他會要求 AI 深入技術細節：
- 「幫我看那個 OpenClaw 的 Source Code，他的 SecretRef 是怎麼實現的？」
- AI 會去翻 GitHub Source Code，最後告訴他整個實現機制（例如寫了一個 Secret Provider）

**完全不需要打開電腦**，在通勤途中就能透過語音與 AI 助理完成技術學習和資訊獲取。學完之後直接發 Facebook 分享心得。

講者認為這個變化已經在發生，每個人都應該開始找到用語音與隨身 AI 助理溝通的方式：Delegate 給它去 investigate、figure out implementation，讓它用 ASCII 畫架構圖、用中文解釋給你聽。

**提到的工具/服務：** Tesla FSD、Grok、OpenClaw、GPT、Facebook

---

## 9. 現在是 AI 的「iPhone Moment」

講者認為現在的 AI 浪潮就像 2006 年 Steve Jobs 發布 iPhone 的時刻。iPhone 當年顛覆了整個中國的移動互聯網（微博、微信等），現在 AI 正在做同樣的事。

他梳理了這個浪潮的時間線：
- 2023 年左右：GPT-3.5 出現
- 2024 年：Cursor、Windsurf、Claude 等 Coding Agent 崛起
- Claude Code 的關鍵轉折：推出訂閱制。講者認為 Claude Code 的成功不只是技術厲害，而是那個決定用訂閱制的人做了最關鍵的決策。搭配 Boris（產品負責人）每週快速迭代新功能，整個產品就活起來了
- 2025-2026 年：N8N、Claude Code、OpenClaw 等工具爆發。OpenClaw 在 GitHub 上的新星數量已超過 Linux 和 React，排名第一

更重要的信號是**廠商全部跳進來了**：
- NVIDIA 派工程師直接成為 Maintainer（做 NemoClaw）  
- Microsoft Teams, 365、Slack、Telegram 都派原廠工程師支持 Integration
- 字節跳動（做「如夢」的團隊）宣布第一個官方 OpenClaw ClawHub China mirror 成立
- QQ 也跳進來支持

講者指出，上次有這樣廠商全面投入的盛況，可能是 10 年前的 Kubernetes 或 20 年前的 Linux Kernel。

**提到的工具/服務：** GPT-3.5、Cursor、Windsurf、Claude Code、N8N、OpenClaw、NVIDIA NemoClaw、Kubernetes

---

## 10. 趨勢一：Browser Automation 的兩種場景

講者觀察到 Browser Automation 正在發生重大變化，分為兩種場景：

### 場景一：重複性自動化（Automation）

把瀏覽器操作配置好後，讓它定期執行（每小時、每天）。這種場景應該把 Browser 跑在遠端的 SaaS 或 Managed 平台（如 Amazon Bedrock AgentCore Browser Tool），而不是本機。

關鍵原則：**一定要把 Gateway 和 Browser 解耦（decouple）**，不要讓瀏覽器拖垮你的 Gateway。使用 CDP（Chrome DevTools Protocol）去控制遠端瀏覽器。

講者的實際案例：他設計了一個自動化流程，讓 Agent Browser 每小時自動進入 X.com（Twitter）首頁，往下滾 80 次（每次休息 1.5 秒），抓取所有貼文，過濾掉廢話，根據作者粉絲數計算權重，最後產生不帶偏見的三五句話摘要報導，發布到 https://x.deepsrt.com

### 場景二：Local Debugging 與 Scraping

這種場景是在本機操作瀏覽器，典型用途包括：

1. **前端開發 Debugging**——Google 的 Chrome Web DevTools 現在提供 MCP 支持，讓 Agent 連上你正在使用的瀏覽器，自動抓取 console log、error，自己修改再 try
2. **Purpose-built 的搜尋與爬蟲**——例如找機票、訂酒店

講者詳細解釋了他的機票搜尋系統架構：

- 他寫了一個 **Bridge**（用 Rust 寫的），封裝高階方法（如 `search_ticket`、`search_hotels`、`search_jobs`）
- Bridge 透過 Chrome Extension 操作瀏覽器（如 SkyScanner、Booking.com）
- Agent 只需要呼叫高階的 MCP 方法，不需要知道底層實現
- 例如：告訴 Agent「幫我找四腿票去巴黎，TPE 出發，外站日本或韓國，長榮航空」，Agent 就會自動組合搜尋條件

**封裝高階方法的好處：**
1. **Agent 不需要 Skill**——再笨的模型都看得懂 `search_ticket(date, from, to)` 這種高階方法
2. **實現邏輯被保護**——Agent 看不到背後是 SkyScanner 還是其他服務，攻擊面大幅縮小、從商業來講也保護了實現邏輯不被看見
3. **可透過信任網路遠端存取**——Bridge 可以 listen 在 Tailscale VPN 網路上，遠端的 Agent 也能呼叫

講者還提到，AI 會自己判斷搜尋的合理性——例如發現夏天去歐洲不可能拿到 3 萬以下的票價，就會自動縮小範圍到 10-11 月，最終找到 2 萬多的四腿票。

**提到的技術/工具：** CDP（Chrome DevTools Protocol）、Chrome Extension、MCP（Model Context Protocol）、Rust、Bridge 架構、Sky Scanner、Booking.com、Tailscale、AWS Agent Core Browser Tool、Apps.dip.com

---

## 11. 趨勢二：Isolation（隔離）的三個層級

講者強調 AI Agent 的執行環境隔離是非常重要的安全議題，並介紹了三個層級的演進：

### 第一層：Host Level Isolation（主機層級）

最直覺的做法——找一台沒在用的機器（如 Mac Mini），把 Agent 裝進去。好處是不會搞掛你自己的電腦，但風險很明顯：Agent 有完整權限安裝任何軟體、存取環境變數中的 Credentials、複製你的 Skill 程式碼 - 幾乎可以做任何事。

### 第二層：Kubernetes Pod Level Isolation（Pod 層級）

把 Coding Agent 跑在 Kubernetes 的 Pod 裡面。雖然 Pod 不是最強的隔離，但至少：
- 一個 Pod 可以讓他只跑一隻 Agent
- Agent 無法存取主機的其他地方，因為被困在Pod裡面
- Pod 裡面連 sudo 都沒有，想要 `apt-get install` 或 `npm install` 都不行

講者自己在 Zeabur 上用 K3S 的 Pod 跑 Agent，並維護了一個 Helm Chart。他設定了自動化流程：OpenClaw 每次發布新版本，他的 GitHub Action 會自動偵測、打 tag、觸發 Helm Chart release，Pod 自動更新。

### 第三層：MicroVM Isolation（微型虛擬機層級）

每個 MicroVM 有自己的 Kernel，隔離強度最高。代表性的實現是 NVIDIA 的 NemoClaw，使用 OpenShell 技術。

MicroVM 的核心特性是：每個 MicroVM 擁有自己獨立的 guest kernel，不與其他 VM 共用主機 kernel，隔離強度遠高於 container。即使某個執行環境被攻破，也不會影響其他 MicroVM。

在架構設計上，Gateway（大腦）和 Tool Calling（手腳、眼睛）被刻意解耦，各自跑在獨立的執行環境中，無法互相影響。這是邏輯層面的設計決策，而 MicroVM 則是在底層提供了強隔離的技術保障，兩者相輔相成。不共用主機kernel

講者預測，到年底，萬一各家大廠發布 Managed 的 Agent 服務時，一定會採用 Firecracker 等級的 MicroVM 強隔離。

**提到的技術：** Kubernetes、K3S、Helm Chart、Pod、MicroVM、NVIDIA Nemo Cloud、OpenShard、Firecracker、Zeabur、GitHub Action

---

## 12. 趨勢三：IDE 正在消亡，主動式隨身助理崛起

講者認為 IDE 正在走向衰退——不是因為不好用，而是開發者不再需要坐在電腦前才能把東西寫出來。

核心觀點：**Bring your intention into reality**。你可以用手機、語音、Telegram、Discord、iMessage，甚至開車時跟 AI 對話，把你的 intention（意圖）轉換成現實。

具體例子：
- Claude Code 的 Dispatch 功能可以用 Telegram 遠端操作
- 你可以 walk away 去遛狗，CC 在幫你工作，你只需要一支手機用語音跟它講話

**主動式隨身助理（Proactive Agent）的設計：**

講者建議讓你的 Agent 設計成 proactive——定期自我 review 過去的對話和執行紀錄，主動提出改進方案。不是你問它「下次怎麼做更好」，而是它自己時間到就反思，然後告訴你，你說「好，去做」就行了。你是老闆。

**Coding Agent in the Cloud：**

講者預測到年底之前，所有你現在在本地使用的 Coding Agent（Google Gemini、Anthropic Claude、GitHub Copilot、Codex、CC 等）全部都會有雲端版本。區別是你可以用瀏覽器或手機語音控制它，而且可以有多個平行工作，吃同一個訂閱。

他自己已經在實現這件事：當用戶在 GitHub 開 issue，他的 K3S Pod 裡跑的 Kiro CLI 會在 60 秒內抓取 issue、執行 Skill 進行 triage、甚至開始寫 PR。透過一個用 Rust 寫的 Agent Broker，左手用 ACP(Agent Communication Protocol)，右手連到 Discord/Telegram/Slack，實現完整的雲端近似 OpenClaw 的體驗卻不需要安裝龍蝦，就連 emoji 表情包也會自動切換，提供滿滿的情緒價值。

**PM 與工程師的界線模糊化：**

- PM 不需要 Programmer 就能做出 POC，只要會講話就行
- 工程師也要用 PM 的角度思考，格局要更大
- 新創公司已經開始用 generic title，例如 Anthropic 的研究和工程職缺全部統一叫 "Member of Technical Staff"（MTS），從新進員工到共同創辦人都一樣，期望每個人同時具備 PM 和 Builder 的能力

**提到的工具/概念：** Claude Code Dispatch、Telegram、Discord、Agent Communication Protocol、Agent Broker、Rust、K3S、Kiro CLI, MTS

---

## 13. Survival Tips：AI 時代的生存法則

### Coding CLI 是你唯一的介面

不要再去 Facebook 問「大大」了。你有 CC（Claude Code）、Codex 等 Coding CLI，所有問題直接問它。

講者最常用的 prompt 技巧：
```
ELI5 ZHTW with ASCII  <貼上網址或文章>
```
意思是「Explain Like I'm 5」——把我當五歲小孩，用繁體中文、畫 ASCII 圖解釋給我聽。AI 會翻遍 Source Code，畫出架構圖，用中文寫出完整的流程。

**Source Code 就是真理**——自媒體講的、大大說的，都不如你自己用工具去 Figure out 來得真實。

### 保持強烈的好奇心

資訊取得已經如此容易，任何念頭都可以讓 Agent 幫你挖掘背後發生了什麼事。不斷問、不斷學習，最終你會在某個領域成為最強的。

### 讓最強的人看見你（Visibility）

1. 找到你所在領域最強的人
2. 不要臉地去 @ 他、勾搭他
3. 當他轉發你一次，等於中一次樂透
4. 中十次樂透之後，他們缺人時第一個就找你

**做完一件事情做得很好，只是把事情做了一半。剩下那一半是推出去讓更多人看見、認可。後者甚至比前者更重要。**

### AI 不會取代你，AI 創造更多機會

講者用自己的經歷反覆印證：
- 大學裝電腦 → 沒人裝電腦了 → 但有了新的工作
- 自己架機器 → Serverless 不用架了 → 但 Serverless 專家更被需要
- 手寫程式 → CI/CD 自動化 → 測試工程師去學 Jenkins、GitHub Action
- 每一次「要失業了」的恐慌，最終都變成了新的機會

但也有現實面：**AI Intensifies your work load**——以前做三天的事，現在一天做完，老闆就會塞更多工作給你。要注意調整節奏、適可而止、適當 push back。

### 掌握自己的職涯

- 想清楚你夢想的工作是什麼，用盡一切努力去得到
- 如果現在的環境沒有發揮餘地，想想下一個認同你的老闆在哪裡
- OpenAI、Anthropic 在日本和新加坡都有辦公室了，有機會就去面試
- 如果有出國工作、出差、念書的機會掉下來，不要排斥——那是上帝幫你開的門
- 去你這輩子最不可能去的國家旅行，那會是你一輩子最珍貴的資產和回憶
- 如果出現了生命中的另一半，不要放過，互相扶持去迎接新的挑戰

---

## 附錄：演講中提到的工具與技術清單

| 類別 | 名稱 |
|------|------|
| 雲端平台 | AWS（EC2、ELB、ALB、Lambda、API Gateway、Fargate、EKS、Amazon Q） |
| IaC 工具 | AWS CDK、CloudFormation |
| AI/Coding Agent | Claude Code、Cursor、Windsurf、GitHub Copilot、Codex、Kiro CLI、Amazon Q |
| AI 模型/服務 | GPT-3.5、GPT-5.4、Grok、Claude |
| 瀏覽器自動化 | CDP（Chrome DevTools Protocol）、Chrome Extension、MCP、Agent Browser |
| 容器/編排 | Kubernetes、K3S、Helm Chart、Pod、Docker |
| 虛擬化/隔離 | MicroVM、Firecracker、NVIDIA Nemo Cloud、OpenShard |
| 網路/VPN | Tailscale |
| 開發平台 | GitHub、GitHub Action |
| 通訊整合 | Discord、Telegram、Slack、Agent Communication Protocol |
| 自動化平台 | N8N、OpenClaw |
| 旅遊工具 | Sky Scanner、Booking.com |
| 部署平台 | Zeabur |
| 程式語言 | Rust（Bridge、Agent Broker） |
| 作業系統 | Linux、Sun Solaris |
| 其他 | Tesla FSD、https://x.deepsrt.com |
