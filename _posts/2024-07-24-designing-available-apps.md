---
layout: post
title: Available Applications with Golang
date: 2024-07-24 21:55:00
description: Designing Available Applications with Golang
tags: golang code availability resty
categories: Golang
featured: true
---

# Designing Available Applications with Golang

When working within a context where you cannot or do not want to change your infrastructure on distributed systems, an important question arises: How can I make my application more available?

Available applications should only experience a maximum of 5% errors. If your error rate is higher than this, you have a problem, or worse, you might not even know you have a problem. But don't worry, we're talking about untreated errors or issues that arise because you haven't managed your responses or status codes properly. Here, we'll use Golang and Resty to help solve many of these issues.

## Let's Move On!

Imagine you have an application that needs to call another microservice to retrieve product details, using only the product ID:


```go
package main

import (
    "fmt"
    "github.com/go-resty/resty/v2"
)

func main() {
    client := resty.New()

    // Example array of product IDs
    ids := []string{"123", "456", "789"}

    // Make the request
    resp, err := client.R().
        SetBody(ids).
        SetResult(&[]Product{}).
        Post("https://example.com/api/products")

    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }

    products := resp.Result().(*[]Product)
    fmt.Printf("Products: %v\n", *products)
}

// Product represents a product with details
type Product struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Price float64 `json:"price"`
}
```

If this request fails 50% of the time, what should you do? Oftentimes, we'll call someone to inform them about the issue and work on a solution. But if you don't have that option, you need to solve the problem yourself. Here are some approaches that can help:

### Retry

Retrying can help you get a successful response after initially encountering an error. In cases of transitory failures, like network issues or temporary overloads, you can increase the availability of your application by retrying the request.

#### Be Careful:

When implementing retries, consider the following points:
1. Retrying can improve availability but might also strain the client application.
2. Ensure the third-party application has a rate limit setup to handle retries.

#### Retry Implementation with Resty

Here's how you can implement the retry approach using Resty's built-in mechanism:

```go
package main

import (
    "errors"
    "fmt"
    "github.com/go-resty/resty/v2"
    "time"
)

func main() {
    client := resty.New()

    // Configure retries
    client.
        SetRetryCount(3).
        SetRetryWaitTime(5 * time.Second).
        SetRetryMaxWaitTime(20 * time.Second).
        SetRetryAfter(func(client *resty.Client, resp *resty.Response) (time.Duration, error) {
            if resp.StatusCode() == 429 { // Assuming 429 is the status code for quota exceeded
                return 0, errors.New("quota exceeded")
            }
            return 0, nil
        })

    // Example array of product IDs
    ids := []string{"123", "456", "789"}

    // Make the request
    resp, err := client.R().
        SetBody(ids).
        SetResult(&[]Product{}).
        Post("https://example.com/api/products")

    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }

    products := resp.Result().(*[]Product)
    fmt.Printf("Products: %v\n", *products)
}

// Product represents a product with details
type Product struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Price float64 `json:"price"`
}
```

### Backoff

Backoff strategies help mitigate issues caused by retries by specifying intervals between each retry attempt. You can use either an exponential or linear backoff. However, consider the following points:

#### Be Careful:

1. Backoff may not be necessary if the client application guarantees delivery.
2. For real-time applications, the delay introduced by backoff might be unacceptable.
3. Scheduled workloads where the timing of the response is predictable.
4. Unique contexts where repeated requests may cause unwanted side effects.
5. Stress tests or workload simulations.

> Get in mind
>
> We can handle it using another patterns or using it together, like *Circuit Breaker*, but we can talk about it in another post.

## Conclusion

These approaches can significantly improve the availability of your application. However, always adapt these methods to your specific context. Test, analyze, and validate to ensure the problem of availability is caused by third parties and understand how these changes will impact them. Thorough testing and evaluation can powerfully enhance your application's robustness.

