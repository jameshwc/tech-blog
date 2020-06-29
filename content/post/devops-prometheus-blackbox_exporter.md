---
title: "[DevOps] 從外部監控網站狀態：Prometheus & Blackbox exporter"
date: 2020-06-29T15:58:24+08:00
draft: false
---

> 此篇文章為個人在系上服務之團隊的期末報告，略加修改

## Introduction

6 月中時，Printing (Papercut) 網頁壞掉 (Status code: 500)，因為同學知道我是 PC 組，有管 Printing 的機器，來密我我再轉傳到 PC 組群組才得以解決；那時我便覺得這樣的流程太沒效率：我們總是要等 user 給 feedback 才會知道機器壞掉了，而且系上大部分人應該都不知道是確切是誰負責管 Papercut 的，加添了回報的困難性；另一方面個人對 Monitor 的知識也滿缺乏的，所以就想說趁這個 Final Project 的機會來練練手。

## Prometheus & Blackbox Exporter

### Why Prometheus?

Prometheus 是一個用 go 寫成的監控預警框架，因其開源、時間序列資料庫的特性，廣受業界所使用在監控各式網站與服務。其缺點是圖表不精緻，所以我們可以引入 Grafana 來改善。

### Grafana

Grafana 是一套開源的 dashboard 系統，可以很方便的結合 Prometheus metrics，並顯示精美的圖表。許多較常見的監控指標，例如 nginx，在 Grafana 官方的 Community 都有人做好現成的 dashboard，只要指定 json ID，幾秒鐘就可以產生如下圖的圖表。

![](https://i.imgur.com/esUS1Wr.png)

### Blackbox Monitor

Blackbox Monitor 相對於 Whitebox Monitor，也就是我們不需要知道機器或服務內部的狀況，**透過外部的手段來了解服務有沒有正常運行。**

如果以 nginx-based 的 web server 來當例子，在機器內部寫一個 metrics，固定每 1 分鐘輸出 uptime 多久、1 小時內多少流量、cpu 使用率多少，這是 **Whitebox Monitor (從內部匯出資訊)**；
Send requests 到各個 Endpoints，或者將 test data 送到 API，看該 endpoint 的狀態碼是不是如預期，或者 response 有沒有跟預期一樣，這是 **Blackbox Monitor (不需要知道內部的資訊)**。

Prometheus 對於 Blackbox Monitor 與 Whitebox Monitor 這兩者都支援，且以 Whitebox Monitor 為主要的實踐。但為什麼我這次是使用 Blackbox Monitor 呢？一方面是我不想在 Production 的環境動一些有的沒的，怕一不小心影響到服務；另一方面以使用情境，也就是看服務有沒有掛掉這件事，似乎不需要用到 Whitebox Monitor；再者，Blackbox 的好處是非常的泛用，只要輸入網址就能運作，所以這樣的方式不僅適用於 Papercut，也可以監控其它服務、網站等等。

### Workflow

#### Instance

- Blackbox Exporter
- Prometheus
- AlertManager

---

Blackbox Exporter 每 n 秒送 request 到 Papercut，記錄狀態碼、Response time 等等，Prometheus 也每 n 秒抓下 Blackbox Exporter 的資料，並存在 Prometheus 的資料庫內。如果 Prometheus 發現狀態碼並非 200，就會送 alerts 到 AlertManager，AlertManager 再根據設定將告警的內容用 webhook 送到 Slack bot。

請參考下圖：
![](https://i.imgur.com/oXC2VD3.jpg)

---

## Configuration

### Prometheus & Grafana Deployment

docker-compose.yml
```yaml
version: '3.2'
services:
  prometheus:
    image: prom/prometheus
    ports:
    - 9090:9090
    command:
    - --config.file=/etc/prometheus/prometheus.yml
    - --web.route-prefix=/
    - --web.external-url=https://prometheus.jameshsu.csie.org # set for reverse proxy
    volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
    - ./slack_rules.yml:/etc/prometheus/slack_rules.yml
  grafana:
    image: grafana/grafana
    volumes:
    - ./grafana:/var/lib/grafana
    environment:
    - GF_SECURITY_ADMIN_PASSWORD=password
    - GF_SERVER_ROOT_URL=https://grafana.jameshsu.csie.org # set for reverse proxy
    depends_on:
    - prometheus
    ports:
    - '3000:3000'
```

### Blackbox Exporter

用 [github repo](https://github.com/prometheus/blackbox_exporter) 上預設的 config。

### Prometheus

Prometheus.yml
```yaml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 52.229.61.229:9093 # Alertmanager's address
rule_files:
    - "slack_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'blackbox_papercut'
    metrics_path: /probe
    params:
      module: ['http_2xx']  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - https://printing.csie.ntu.edu.tw    # Target to probe with http.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: linux1.csie.org:9115  # The blackbox exporter's real address.
```

slack_rules.yml
```yaml
groups:
- name: test
  rules:
  - alert: EndpointDown
    expr: probe_http_status_code != 200
    for: 15s
    labels:
      severity: "critical"
      instance: "https://printing.csie.ntu.edu.tw"
    annotations:
      summary: "Endpoint {{ $labels.instance }} down"
```

### AlertManager

alert-config.yml
```yaml
global:
  resolve_timeout: 2m
receivers:
  - name: slack-pc
    slack_configs:
      - api_url: "https://hooks.slack.com/services/def/abc/123456"
        channel: "#pc"
        send_resolved: true
        text: |-
            {{ range .Alerts }}{{ .Annotations.message }}
            {{ end }}
        title: "{{ .CommonAnnotations.env }}: {{ .CommonAnnotations.summary }}"
route:
  group_by:
    - instance
    - severity
  group_interval: 10s
  group_wait: 10s
  receiver: slack-pc
  repeat_interval: 1m
  routes:
    - receiver: slack-pc
```