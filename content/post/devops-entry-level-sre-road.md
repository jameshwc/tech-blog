---
title: "[DevOps] 初階 DevOps/SRE 工程師是如何煉成的"
date: 2021-03-12T15:07:03+08:00
description: "初階 DevOps/SRE 工程師是如何煉成的"
draft: false
---

> 本文同步發表於 [ptt (soft_job)](https://www.ptt.cc/bbs/Soft_Job/M.1615535884.A.7C7.html)
## 前言

背景是學生，大約兩年的 SA/DevOps 學習經驗，剛拿到 ByteDance 的 SRE offer，所以應該可以算是 Entry-level 的 SRE 了，會想寫這篇分享是因為看到滿多人對 DevOps/SRE 的印象是很吃經驗，不太可能讓新鮮人做；對這種印象算一半認同（我就是反例XD），另一方面也想讓有興趣的人知道該如何入門這個領域。

詳細的背景在[前一篇 SRE 面試文](https://www.ptt.cc/bbs/Soft_Job/M.1614755910.A.7C3.html)寫得比較多。

### 什麼是 DevOps/SRE?

我無意在這篇談 DevOps 的商業或管理價值，也無意細分 DevOps Engineer 和 SRE 的區別，很概括且從技術角度來說，DevOps 的重點是

1. 減少從設計、開發（需求、程式碼）到測試、部署（程式）的時間
2. 加強回饋機制（包括但不限於監控、告警）
3. 過程中持續快速的疊代、學習

（改寫 DevOps 三步工作法）

文末會附 DevOps 相關的書單。

## 技能樹

這一部份會以 https://roadmap.sh/devops 搭配講解。

以下的順序以我個人學習、接觸的時間軸做排列：

### 語言

建議 Python、Go、JavaScript 三者至少要一個精通、一個熟練，第三個可以作為輔助。會推薦這三個語言是因為這三個語言要寫自動化的小工具時都很方便；其次這三個語言各自有強項：

- Python：易於和其他人協作、精於 ML（對，SRE 有可能會需要 ML 輔助）
- Go：很多 DevOps 的工具包含 Prometheus, Kubernetes, Docker, Drone 都是 Go 寫成的，要寫網頁後端也很輕鬆
- JavaScript：要寫簡單的網頁前端一定要會 JS，像 aws cdk 也是 TypeScript 的支援比較豐富

但也不是說一定要學這三個語言，例如學 java 就可以結合 jenkins 生態系，所以就看怎麼運用自己的優勢

### Linux/Shell Script

如果一開始接觸的是 Windows 環境，可以去裝 WSL 體驗 Linux；不管如何，走這一行一定要學習 Live in terminal，基本的 cd, ls 就不用說了，跟字串處理有關的 grep, sed, awk, cut 也都要很熟；還有像 wget, curl 等等，要列出所有會用到的指令和工具實在是列不完。有好的搜尋能力的話，stackoverflow 會是你很好的朋友；對 git 也應該至少要會基本的並可以用在專案上。

Linux 除了主流的 ubuntu 以外也可以多嘗試其他 OS，例如 CentOS、Alpine 等等，這部份可以在挑選雲端的虛擬機器 或是 run container 的時候去多多嘗試；另外對 Linux 的觀念包含檔案系統、process management, DNS, DHCP 等等也應該要有基本認知。

### 架站 / SA 相關

會 1 門程式語言，而且對 Linux 夠熟之後，就可以嘗試架站了。

克難一點是可以用自己的機器架，不過建議還是去租雲端的機器（例如 aws, gcp, azure），雖然有可能要花錢（免費的方案不是速度很慢就是不能用太久），但有 public IP 和 24 小時不停機就是方便，也能學到更多東西。

我個人很推用架站來學習，因為在過程中可以學到：

1. 處理網址要了解 DNS, ip, 域名的概念
2. (如果是雲端環境) 學習如何 ssh, live in terminal
3. 設定 Web Server (nginx, apache, etc.)
4. 寫網站前端（http, css, js）、後端（python, go, etc.)
5. 想要一個域名、一台機器但對應到多個網站時，如何設定 Reverse Proxy 和 VM/Docker
6. 跟第三方簽證書設定 https 
7. (如果要寄註冊認證信) 裝 Email Server (SMTP, Reverse DNS, DNS Server)
8. 在 Server 上 Debug 
9. 監控網站流量、機器狀態

至於網站要寫什麼，如果沒有想法可以往購物車或需要註冊登入的網站去發想；新手建議先從前後端混合的框架開始寫（例如 Python 的 Django），比較不需要太多 JavaScript 的知識。

也可以偷懶不寫程式碼，架 WordPress 或跟會寫網站的朋友合作，但學到的東西就會少很多，也容易淪為純 Ops。

### CI/CD

網站有雛型之後，慢慢的會開始覺得本機開發到要更新 server 的程式的流程很麻煩，特別是在頻繁更新和 debug 的時候，這正是 DevOps 要解決的主要問題：縮短 Developers 和 Operation 的距離；具體的解決方式便是引入 CI/CD 的 Pipeline。

CI/CD 簡單來說即是讓程式碼的 build, test, deploy 自動化，使得 developers 只要 push 到版控工具（github/gitlab, etc.），後面就有機器自動化的更新 server 的程式。

有滿多工具可以做到 CI/CD，新手若無頭緒我會建議使用 GitLab 內建的 CI/CD，結合他們自家的版控功能做一條龍；也可以看自己擅長的語言決定用 jenkins 或 drone 或其他工具，大同小異。

如果用 GitLab，推薦自己架一個 GitLab 和 Runner (跑 CI 的環境) ，有人寫了很方便的 docker-compose 可以一行架起來。

### 容器（Containers）

隨著網站規模愈來愈大，可能會在這台機器上架好幾個網站，gitlab, blog, prometheus 等等，除非必要，這些服務都建議容器化用 Docker/Docker-compose 跑，過程中會對 Containers 比較熟；如果有興趣也可以玩 Kubernetes 或類似的容器管理平台，但 k8s 水很深，慎入。

### 寫小工具 / 接觸開源

如果前面的部份都摸得差不多了，可以加強 Develop 的程度，去多摸一門語言，或是深入研究本來會的語言的特性、OOP；也可以嘗試寫一些小工具，例如爬蟲、middleware、metrics exporter 等等。

同時在這個階段儘可能的去接觸開源，一開始會覺得挫折、看不懂是難免的，對規模較大的 repo 欣賞它的架構、規模小的 repo 嘗試去看懂裡面的 code。

廣泛閱讀 open-source 的專案、技術文章，是這個階段進步最快的方式。

### 以專案為本

大量閱讀 open-source 的程式碼和技術文章的過程中，可能會讀到很多沒用過的技術，但也比較能區分 Clean/Dirty Code，這時候可以嘗試做比較大型的專案，套用想學習的技術。

如果有資源，可以做一個純雲端的專案，畢竟會徵 SRE 的公司很少有不上雲的，而且雲端服務會提供很多服務，例如 Load Balancer、Auto Scaling 等等；又例如 SQL 要架在 EC2 還是用 Aurora 這些取捨都挺值得玩味的。
（個人對 aws 比較熟，所以例子都舉 a 家的）

實習也是做專案的方式之一，如果沒辦法實習，看能不能儘量接觸多人開發的專案，會對於軟體開發的流程更熟悉，例如切 staging/production 環境、開發如何切 branch 等等。

這裡節錄部份我以前做過的 project 和用到的技術：

- LeetCode 爬蟲 (Go)
- Dcard 後端面試作業 (Go/Gin, Redis, Travis CI)
- 做 LineBot CI/CD Pipeline (aws: Route53, EKS (k8s), DynamoDB, S3; Vault)
- PTT 爬蟲 (Go, Goroutine, Channel)
- Blackbox Monitoring (Prometheus, Grafana, AlertManager)
- RESTful API Server (Go/Gin, jwt, ELK, MySQL, Unit/Integration Tests, Redis, Prometheus, Vue/TypeScript, azure: AKS, VM)

### 補足 OS、Networking 知識

說得直白一點就是為了面試做準備啦，但這些知識或多或少也會在實戰中用到。

## 結論

在面試的 Q&A 環節，我問 ByteDance 的面試官「一個 SRE 應該具備哪些特質」，他回答我要能臨危不亂、Reactive、Think out of the Box，後者直翻是跳脫框架，但從面試官的解釋比較像是全局思考；我個人會解讀出兩種層次，第一個層次是不能僅僅只在意 config 怎麼設定，而要考慮整個架構的邏輯，包含前面提到的取捨，這才是體現一個 SRE 價值之所在；第二個層次是不要被工具綁架了，DevOps 注重的是流程和文化，最近體驗到的一個例子是已經有 Python 的自動化的 script，就沒必要引入其他的 CI/CD 工具，目的有達成最重要，這也是最近小的在努力的方向。

除了技術以外，如果想要研究 DevOps 方法論的可以讀 鳳凰專案、DevOps Handbook，這兩本是直接與 DevOps 相關；另外也可以讀一些管理學的書包含高德拉撤的目標、第五項修煉，或是精實相關的書。

SRE 一生都在和複雜系統打交道，也可以看看反脆弱和黑天鵝這一系列的書，會對於一些神奇的方法論（例如 Chaos Engineering）比較理解緣由。