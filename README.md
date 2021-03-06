# Rate limiting for go-redis [![Build Status](https://travis-ci.org/go-redis/rate.svg?branch=v5)](https://travis-ci.org/go-redis/rate)

```go
// +build example

package main

import (
	"fmt"
	"log"
	"net/http"
	"strconv"
	"time"

	timerate "golang.org/x/time/rate"

	"gopkg.in/go-redis/rate.v5"
	"gopkg.in/redis.v5"
)

func handler(w http.ResponseWriter, req *http.Request, rateLimiter *rate.Limiter) {
	userID := "user-12345"
	limit := int64(5)

	rate, reset, allowed := rateLimiter.AllowMinute(userID, limit)
	if !allowed {
		w.Header().Set("X-RateLimit-Limit", strconv.FormatInt(limit, 10))
		w.Header().Set("X-RateLimit-Remaining", strconv.FormatInt(limit-rate, 10))
		w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(reset, 10))
		http.Error(w, "API rate limit exceeded.", 429)
		return
	}

	fmt.Fprintf(w, "Hello world!\n")
	fmt.Fprint(w, "Rate limit remaining: ", strconv.FormatInt(limit-rate, 10))
}

func statusHandler(w http.ResponseWriter, req *http.Request, rateLimiter *rate.Limiter) {
	userID := "user-12345"
	limit := int64(5)

	// With n=0 we just retrieve the current limit.
	rate, reset, allowed := rateLimiter.AllowN(userID, limit, time.Minute, 0)
	fmt.Fprintf(w, "Current rate: %d", rate)
	fmt.Fprintf(w, "Reset: %d", reset)
	fmt.Fprintf(w, "Allowed: %v", allowed)
}

func main() {
	ring := redis.NewRing(&redis.RingOptions{
		Addrs: map[string]string{
			"server1": "localhost:6379",
		},
	})
	fallbackLimiter := timerate.NewLimiter(timerate.Every(time.Second), 100)
	limiter := rate.NewLimiter(ring, fallbackLimiter)

	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		handler(w, req, limiter)
	})

	http.HandleFunc("/status", func(w http.ResponseWriter, req *http.Request) {
		statusHandler(w, req, limiter)
	})

	http.HandleFunc("/favicon.ico", http.NotFound)
	log.Println("listening on localhost:8888...")
	log.Println(http.ListenAndServe("localhost:8888", nil))
}
```
