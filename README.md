package main

import (
	"context"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"
)

var (
	count uint64
)

func main() {
	logger := log.New(os.Stdout, "http-server: ", log.LstdFlags)

	server := &http.Server{
		Addr: ":8080",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			logger.Printf("Received request %s %s\n", r.Method, r.URL.Path)

			atomic.AddUint64(&count, 1)

			time.Sleep(10 * time.Second)

			body, err := ioutil.ReadAll(r.Body)
			if err != nil {
				logger.Printf("Failed to read request body: %v\n", err)
				http.Error(w, "Failed to read request body", http.StatusBadRequest)
				return
			}

			logger.Printf("Request body: %s\n", string(body))
		}),
	}

	go func() {
		logger.Println("Starting server...")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Fatalf("Failed to start server: %v", err)
		}
	}()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	<-sigCh

	logger.Println("Shutting down server...")
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		logger.Fatalf("Failed to shut down server: %v", err)
	}

	logger.Printf("Processed %d requests\n", count)
}
