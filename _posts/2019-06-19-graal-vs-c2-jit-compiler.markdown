---
layout: post
title: Graal vs. C2 JIT compiler
subtitle: How to speed things up
date: 2019-06-17 00:00:00 +0000
tags: [Computer Science, Scala, Graal, JVM]
---

## How to speed things up

At the Scala Days conference in Lausanne this year there were a few interesting talks,
and no, I am not talking about Shapeless for Scala 3.

The first talk that I found quite interesting was about deploying a Scala application in Google Cloud Run 
([a managed compute platform that automatically scales your stateless containers](https://cloud.google.com/run/))
by James Ward and Josh Suereth.  
One of the main concern about serverless and JVM languages is around cold start and performances.
Surprisingly (at least for me) there is a "simple" answer to this problem: GraalVM.

The idea is ahead-of-time compilation to produce a standalone executable
(aka a native image) by using **GraalVM**.

An approach presented was to have a docker image like:

```
FROM oracle/graalvm-ce:10.0.0 as builder

WORDKIR /app
COPY . /app
RUN gu install native-image
RUN ./sbt graalvm-native-image:packageBin

FROM apline:3.9.4

COPY --from=builder /app/target/graalvm-native-image/app /app
CMD ["/app", "-web"]
```

Being new to Graal I was quite confused at that point,
and I soon realised most other people are, so it's important to emphasize the following statement:  
*Graal and GraalVM are not the same thing.*

Graal is a JIT (just in time) compiler.
GraalVM is an umbrella for a variety of technologies including Graal and SubstrateVM (SVM).

By using GraalVM, you can effectively run your Scala application outside of the JVM.
You can run it on Android and iOS for example
(a more in-depth talk about this topic was given by *Vojin Jovanovic* titled *Run Scala Faster with GraalVM on any Platform*).

In the reality not every application can (or should?) run outside of the JVM.
On the other hand, thank to the layered compiler architecture,
it is possible to replace the traditional C2 JIT compiler with Graal while still running on the JVM.

This concept was further explored by *Chris Thalinger* talk about 
*Performance tuning Twitter services with Graal and ML*.  
The speaker, who is part of the [#TwitterVMTeam](https://twitter.com/hashtag/twittervmteam), 
presented the ambitious "Bayesian optimisation as a service", a machine learning software 
that automatically tunes applications running in the cloud!

According to Thalinger, Twitter currently runs the majority of its scala services using this strategy
with impressive advantages including drastic reduction of CPU usage and latency up to 28%,
which (especially for a company like Twitter) translate directly to money savings on the infrastructure
and a better user experience.


## Graal for SpringerLink

It is actually surprisingly simple to try out Graal.
In fact, if you're already on JDK11 (SpringerLink was not yet),
you only need to add the following settings:

`-XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler`

I've decided to go YOLO and try it out on the main link.springer.com application,
responsible for serving most of the content available on the website.

The experiment was about deploying two distinct applications (one instance each)
that only differ from those settings and run some load testing 
using a simple command line tool (siege).

The load testing was run with 25 simultaneous users,
for 1 minute and with no delays between requests:  
`siege -c 25 -b -t 1M https://...`:


### 1st attempt

Straight to the results:

|                         | Graal            | C2               |
|-------------------------|------------------|------------------|
| Transactions            | 11412 hits       | 8839 hits        |  
| Availability            | 100.00 %         | 99.84 %          |
| Elapsed time            | 59.92 secs       | 59.54 secs       |   
| Data transferred        | 79.86 MB         | 61.95 MB         | 
| Response time           | 0.13 secs        | 0.17 secs        |  
| Transaction rate        | 190.45 trans/sec | 148.45 trans/sec |                 
| Throughput              | 1.33 MB/sec      | 1.04 MB/sec      |    
| Concurrency             | 24.84            | 24.84            |
| Successful transactions | 0                | 0                |
| Failed transactions     | 0                | 14               |
| Longest transaction     | 0.54             | 1.16             |
| Shortest transaction    | 0.00             | 0.00             |


At first this was looking very good.
Too good in fact: I was hitting an article not found page.

｡ﾟ(ﾟ∩´﹏`∩ﾟ)ﾟ｡

Please take a moment to laugh at me now.

Lesson learned: do not start load testing stuff that you're not absolutely sure is working fine.


### Second attempt

Despite the dramatic findings, the results were still encouraging
so I run the test again on a working article page:

|                         | Graal           | C2              |
|-------------------------|-----------------|-----------------|
| Transactions            | 5859 hits       | 3964 hits       |
| Availability            | 100.00 %        | 100.00 %        |
| Elapsed time            | 59.02 secs      | 59.11 secs      |
| Data transferred        | 96.62 MB        | 65.34 MB        |
| Response time           | 0.25 secs       | 0.37 secs       |
| Transaction rate        | 99.27 trans/sec | 67.06 trans/sec |
| Throughput              | 1.64 MB/sec     | 1.11 MB/sec     |
| Concurrency             | 24.79           | 24.67           |
| Successful transactions | 5859            | 3964            |
| Failed transactions     | 0               | 0               |
| Longest transaction     | 4.60            | 7.50            |
| Shortest transaction    | 0.00            | 0.00            |


To make sure external factors (e.g. cache) didn't play a fundamental role here, 
I run a second test on the same article page, considering everything was warm at this point:

|                         | Graal           | C2              |
|-------------------------|-----------------|-----------------|
| Transactions            | 8117 hits       | 5536 hits       |
| Availability            | 100.00 %        | 100.00 %        |
| Elapsed time            | 59.83 secs      | 59.82 secs      |
| Data transferred        | 133.87 MB       | 91.36 MB        |
| Response time           | 0.18 secs       | 0.27 secs       |
| Transaction rate        | 135.67 trans/sec| 92.54 trans/sec |
| Throughput              | 2.24 MB/sec     | 1.53 MB/sec     |
| Concurrency             | 24.74           | 24.84           |
| Successful transactions | 8117            | 5536            |
| Failed transactions     | 0               | 0               |
| Longest transaction     | 3.23            | 5.93            |
| Shortest transaction    | 0.01            | 0.00            |

Two obvious things to notice:
1. performances of both applications improved significantly compared to the first run, and 
2. Graal performances were significantly better than C2 in both occasions


### Conclusion

I did not have time to verify improvements in terms of CPU and memory,
and I could not extensively test the application to prove it works as expected
(probably the most important factor).

However, tests suggest that the Graal application can potential handle 
47% more transaction per seconds, with a 66% faster response in average.

Surely worth a try!
