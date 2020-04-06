---
title: "[程式雜談] Dcard 2020實習 回家作業"
date: 2020-04-06T15:15:06+08:00
draft: false
---
在 [Dcard lab](https://www.facebook.com/dcardlab/) 看到徵實習的消息，我當然選了我最擅長的後端，有趣的是得完成一項作業才能應徵，雖耳聞已久，這倒是我第一次「做作業」。看了一下題目，挺好玩的，不太滿足於僅僅做完要求，又堆疊了一些額外的東西，每項都是第一次碰，多學了不少東西，故來發文記錄。

## 題目
Dcard 每天午夜都有大量使用者湧入抽卡，為了不讓伺服器過載，請設計一個 middleware：

- 限制每小時來自同一個 IP 的請求數量不得超過 1000
- 在 response headers 中加入剩餘的請求數量 (X-RateLimit-Remaining) 以及 rate limit 歸零的時間 (X-RateLimit-Reset)
- 如果超過限制的話就回傳 429 (Too Many Requests)
- 可以使用各種資料庫達成

## 想法

在該求職頁面中的申請條件，有一條是```熟悉go或node.js的框架```，剛好我現在主力語言是 go，雖然以前寫Web都是用 python-django，卻也嚮往用 go 來寫 web，所以在語言上毫不猶豫的就選擇了 go。

資料庫本來想用 PostgreSQL 或 MySQL，因為以前自己寫 Django 或在 Cool 的日子接觸過，算是挺熟悉的了；開寫沒多久，突然想到一直在各種場合遇到 redis，卻從來沒實際操作過，心癢之下就改成 redis。這倒不是一時腦衝的決定，而是題目的情境的確非常適合使用 redis（這可能是出題者的巧思，考驗應徵者對各種資料庫的熟悉程度），因為redis是 key-value，資料之間並沒有關聯性，而且 redis 有天然的 TTL 可以用，在題目的高造訪量的情境下，使用 NoSQL 是最佳解。

## 更多的堆疊

雖然遇到一些 bug，畢竟是挺簡單的 middleware，一兩天就寫完了，後來多寫了幾條自己想做的 TODO，也一一的完成了，包含測試（學到了如何 mock a database）、CI（使用沒用過的 travis CI）、log（把內建的 log 套件換成第三方開發的 sirupsen/logrus，再套一些函數而已）、framework（使用 gin，但因為只是 hello-world，感受不到它的強大 XD）

## 細談程式碼

這個 project 挺值得拿出來談的是我將介面（interface）和實作分離，先在 database.go 定義好這個專案中 db 該有的 method 和回傳值，如下：
```go
type Database interface {
	// Connect to the database server with defined maxIP and timeout
	Init(int, int) error
	// Check if the IP is in database and whether it's forbidden or not
	Find(string) (bool, bool)
	// return X-RateLimit-Remaining and X-RateLimit-Reset
	GetKey(string) (string, string, error)
	// If IP is not found in database, then create one
	SetKey(string) error
	// Increment the visit counter of the IP
	IncrementVisitByIP(string) error
}
```
這樣，我就可以在 middleware 裡面直接使用這個 db 的 method，即使我還沒有寫好實作。
```go
func limitVisit(db Database) gin.HandlerFunc {
	return func(c *gin.Context) {
		ip := c.ClientIP()
		exist, tooMany := db.Find(ip)
		if !exist {
			err := db.SetKey(ip)
			if err != nil {
				log.Fatal("Set redis key", err)
			}
		} else {
			err := db.IncrementVisitByIP(ip)
			if err != nil {
				log.Fatal("Increment redis key", err)
			}
		}
		remaining, ttl, err := db.GetKey(ip)
        // 以下省略
```
也就是說，我可以留到最後再來考慮實作 db 的細節，例如我可以猶豫這個 db 的系統要用 PostgreSQL 還是 redis，但不管用哪一個，只要符合 interface 所定義的內容，想用哪個就用哪個。

Go 的 interface 實作是隱式的，換句話說，我不用宣告 redisServer 這個 struct 實作 (implement) Database，只要我寫好 interface 定義的 method 即可。
```go
func (db *redisServer) Find(ipaddr string) (existed bool, toomuch bool) 
func (db *redisServer) GetKey(ipaddr string) (string, string, error)
func (db *redisServer) SetKey(ipaddr string) error
func (db *redisServer) IncrementVisitByIP(ipaddr string) error
```
除了 implement 介面定義的內容，當然也可以加一些新的 method 以方便使用，我就多宣告了 init 和 reset

```go
func (db *redisServer) Reset() 
func (db *redisServer) Init(maxIP int, timeout int) error
```

至於怎麼把 redisServer 傳到 middleware？
```go
func setupRouter(db Database) *gin.Engine {
	r := gin.Default()
	r.GET("/", limitVisit(db), hello)
	return r
}
func main() {
	var db redisServer
	if err := db.Init(1000, 3600); err != nil {
		log.Fatal("redis server", err)
	}
	db.Reset()
	r := setupRouter(&db)
	r.Run()
}
```

如果之後有一天，自己或其他人發現這個專案有其他新功能要添加等等因素，而想要把 redis 換成 MySQL，也只要實作介面，再把 main 的 redisServer 換成別的 struct 就好了，是不是簡單又方便呢？
