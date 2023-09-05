+++
author = "angellissa"
title = "Gracefully Splitting Traffic by Proportion in Golang"
date = "2023-09-05"
description = "Gracefully Splitting Traffic by Proportion in Golang"
tags = [
    "markdown",
    "Go",
    "Golang"
]
categories = [
    "Code",
    "Article",
]
+++

In the context of performing gray releases, it's common to divert a portion of traffic to a newly deployed service for small-scale validation. As the feature matures, you gradually increase the traffic directed to the new service. This involves the need to split traffic by proportion. But how can you achieve this precise traffic splitting while minimizing performance overhead?

One common approach is to generate random numbers and check if they fall within a specified range to decide whether to forward traffic to the new service. However, this approach has two main drawbacks:

1. Frequent random number generation introduces performance overhead, especially in high-concurrency scenarios.
2. Random number generation may not be evenly distributed. For example, if you have two services, A and B, with a traffic ratio of 3:7, using random numbers can lead to uneven distribution, such as consecutive requests all landing on service B.

Is there a method that can reduce performance overhead and provide precise traffic splitting? The answer is yes. Here's one implementation approach:

1. Determine the traffic splitting ratio and calculate a base value (base). For example, if the ratio is 3:7, the base is 10.
2. Generate an array (source) with a length equal to the base and fill it with consecutive numbers from 0 to base-1.
3. Shuffle the elements in the source array to ensure that each element's position is random.
4. Create a global counter (queryCount). Increment the counter by 1 for each incoming request, ensuring atomicity.
5. Calculate the remainder of queryCount divided by base to get the value rate and retrieve the corresponding value from the source array (source[rate]).
6. Determine which interval source[rate] falls into to decide the traffic split.

Here's an example Go code implementation:

```go
import (
    "fmt"
    "math/rand"
    "sync/atomic"
)

type TrafficControl struct {
    source     []int
    queryCount uint32
    base       int
    ratio      int
}

func NewTrafficControl(base int, ratio int) *TrafficControl {
    source := make([]int, base)
    for i := 0; i < base; i++ {
        source[i] = i
    }

    rand.Shuffle(base, func(i, j int) {
        source[i], source[j] = source[j], source[i]
    })

    return &TrafficControl{
        source: source,
        base:   base,
        ratio:  ratio,
    }
}

func (t *TrafficControl) Allow() bool {
    rate := t.source[int(atomic.AddUint32(&t.queryCount, 1))%t.base]
    if rate < t.ratio {
        return true
    } else {
        return false
    }
}

func main() {
    trafficCtl := NewTrafficControl(10, 6)
    cnt := 100
    serviceAQueryCnt := 0
    serviceBQueryCnt := 0
    for cnt > 0 {
        if trafficCtl.Allow() {
            serviceAQueryCnt++
        } else {
            serviceBQueryCnt++
        }
        cnt--
    }

    fmt.Printf("service A query count: %v, service B query count %v", serviceAQueryCnt, serviceBQueryCnt)
}
```

The result of running this code is:

```
service A query count: 60, service B query count 40
```

The code's logic is straightforward: by using the modulus operation on a counter, it ensures that traffic is split by the specified proportion. By shuffling the elements of an array, it aims to achieve an even distribution of traffic. Of course, there are other ways to implement traffic splitting. If you have a more ingenious method, please feel free to share it in the comments.