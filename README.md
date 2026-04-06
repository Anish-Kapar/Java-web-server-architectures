# ☕ Java Web Server Architectures

> 🔍 Comparing **Single-Threaded**, **Multi-Threaded**, and **Thread Pool** server designs under high concurrent load using Apache JMeter.

![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)
![Apache JMeter](https://img.shields.io/badge/Apache%20JMeter-D22128?style=for-the-badge&logo=apachejmeter&logoColor=white)
![Socket Programming](https://img.shields.io/badge/Socket-Programming-007396?style=for-the-badge&logo=java&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

---

## 📖 Overview

This project implements **three Java socket-based server architectures** to demonstrate how different concurrency models handle real-world load.

Each server is stress-tested with **600,000 requests over 60 seconds (~10,000 req/sec)** using Apache JMeter to measure throughput, response time, latency, and scalability.

---

## 🧪 JMeter Test Configuration

| Parameter | Value |
|---|---|
| 🛠️ Tool | Apache JMeter 5.6.2 |
| 📦 Total Requests | 600,000 |
| ⚡ Simulated Load | ~10,000 requests/sec |
| ⏱️ Ramp-up Period | 60 seconds |
| 🌐 Protocol | TCP Socket |
| 🖥️ Server Host | localhost:8010 |

---

## 🏛️ Architectures & Results

### 🔴 1. Single-Threaded Server

Processes one client request at a time. All requests are handled **sequentially** on a single thread — zero concurrency.

```
Client → ServerSocket → Single Thread → Handle Request → Respond
                              ↑
                   (next client waits here)
```

**⚙️ Characteristics:**
- ✅ Simple to implement and debug
- ⚠️ Requests queue up — each waits for the previous to finish
- ❌ Completely blocks under concurrent load
- ❌ Not suitable for production


**📊 JMeter Results — 60,000 requests @ ~1,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples | ~11,000 |
| 🚀 Throughput | **~85 req/sec** |
| ⏱️ Average Response Time | **631 ms** |
| 📉 Median Response Time | **24 ms** |
| 📐 Std. Deviation | **1,329 ms** |
| ❌ Errors | Many (connection refused + timeouts) |
| 💾 Response Size | 22 bytes |

> ⚠️ **What happened:** Server processed only ~85 req/sec out of 1,000 req/sec being sent. The massive deviation of 1,329ms shows extremely unpredictable performance — some requests were fast but most timed out waiting in queue. Eventually the server threw `SocketTimeoutException` repeatedly as load overwhelmed the single thread.

---

### 🟡 2. Multi-Threaded Server

Spawns a **new thread for every incoming client request**, enabling true concurrency.

```
Client 1 ──→ new Thread() ──→ Handle & Respond
Client 2 ──→ new Thread() ──→ Handle & Respond
Client N ──→ new Thread() ──→ Handle & Respond
  ↑
(unbounded — 1 client = 1 new thread always)
```

**⚙️ Characteristics:**
- ✅ True concurrent request handling
- ✅ Each client gets an immediate thread
- ⚠️ 10,000 req/sec → 10,000 threads spawned simultaneously
- ❌ OS hits thread creation limit → `pthread_create failed (EAGAIN)`
- ❌ Unbounded thread creation crashes JVM under extreme load

**📊 JMeter Results — 600,000 requests @ ~10,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples Processed | **~19,304** |
| 🚀 Peak Sample Time | **0–2 ms** (while threads available) |
| ⏱️ Average Response Time | **~190–237 ms** (after degradation) |
| ❌ Total Errors | **2,699 failures** |
| 💥 Crash Point | ~4,723 threads → OS limit hit |
| 💾 Response Size | 22 bytes (0 on failed) |
| 🔎 Failure Reason | `pthread_create failed` — OS ran out of threads |

> ⚠️ **What happened:** Server performed brilliantly at 0–2ms while OS could create threads. Around sample ~4,723, the JVM threw `Failed to start thread "Unknown thread" - pthread_create failed (EAGAIN)` — the OS hit its maximum thread limit. From that point, new connections were rejected causing 2,699 errors out of ~19,304 attempts.

---

### 🟢 3. Thread Pool Server ⭐ Best Architecture

Uses a **fixed pool of 10 reusable worker threads**. Requests queue up and are served by available threads — no thread creation overhead per request.

```
Client 1 ──→ ┐
Client 2 ──→ ├──→ BlockingQueue ──→ [ Worker Thread 1  ] ──→ Respond
Client 3 ──→ ├──→                   [ Worker Thread 2  ] ──→ Respond
Client N ──→ ┘                      [ Worker Thread ... ] ──→ Respond
                                    [ Worker Thread 10 ] ──→ Respond
                     (only 10 threads ever exist — reused forever)
```

**⚙️ Characteristics:**
- ✅ Fixed 10 threads — zero thread creation per request
- ✅ Controlled, bounded resource usage
- ✅ Recovers gracefully after burst spikes
- ⚠️ Queue overflows under extreme load (10,000 req/sec), causing temporary errors
- ✅ Returns to normal after burst — unlike multi-threaded which crashes permanently

**📊 JMeter Results — 600,000 requests @ ~10,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples Processed | **~26,870** |
| 🚀 Peak Sample Time (stable phase) | **0–1 ms** |
| ⏱️ Response Time (early ramp) | **55–62 ms** |
| ⏱️ Average (overall) | **~3,007 ms** (includes queue overflow period) |
| ❌ Total Errors | **3,310** (during peak burst only) |
| 💾 Response Size | 29 bytes |
| ♻️ Recovery | ✅ Server recovered and continued processing |
| 🔎 Key Behaviour | Errors during overflow → auto-recovery → back to 0–1ms |

> ✅ **What happened:** Thread pool started at ~60ms during ramp-up (threads warming up), then settled to **0–1ms** as the pool stabilized. During the extreme 10,000 req/sec burst, the queue temporarily overflowed causing ~3,310 errors (samples ~11,000). **Crucially — the server recovered on its own** and continued processing at 0–1ms (samples ~13,700+), unlike multi-threaded which crashed permanently. Total samples processed (26,870) was also higher than multi-threaded (19,304).

---

## 📈 Head-to-Head Comparison

| Metric | 🔴 Single-Threaded | 🟡 Multi-Threaded | 🟢 Thread Pool |
|---|---|---|---|
| **Samples Processed** | ~11,000 | ~19,304 | **~26,870** |
| **Best Response Time** | 24 ms (median) | 0–2 ms | **0–1 ms** |
| **Avg Response Time** | 631 ms | 190–237 ms | 3,007 ms* |
| **Std Deviation** | 1,329 ms | Medium | Lowest (stable phases) |
| **Total Errors** | Many | 2,699 | 3,310 |
| **Crash / Recovery** | Timeouts loop | 💥 Crashed at ~4,723 threads | ✅ Recovered automatically |
| **Thread Usage** | 1 fixed | 1 per client (unbounded) | **10 fixed (reused)** |
| **Scalability** | ❌ Poor | ⚠️ Good until OS limit | ✅ Best |
| **Production Ready** | ❌ No | ❌ No | ✅ Yes |

> *Thread Pool's higher average is skewed by the queue overflow burst period. In stable operation it runs at 0–1ms — the lowest of all three.

---

## 📋 JMeter Metrics — What They Mean

| Metric | What it means |
|---|---|
| **Sample Time (ms)** | Total time from request sent → response fully received |
| **Latency** | Time from request sent → first byte of response received |
| **Connect Time** | Time to establish the TCP socket connection |
| **Throughput** | Requests successfully handled per second |
| **Std Deviation** | How inconsistent response times are — lower = more predictable |
| **Bytes = 0** | Server rejected/dropped the connection — failed request |

---

## 🏁 Conclusion

| | Architecture | Verdict |
|---|---|---|
| 🔴 | **Single-Threaded** | Only ~85 req/sec under 1,000 req/sec load. Std dev of 1,329ms means wildly unpredictable performance. Not usable at any real scale. |
| 🟡 | **Multi-Threaded** | Fast at 0–2ms initially, but **permanently crashed** at ~4,723 threads. The OS cannot keep up with unbounded thread creation. Dangerous under load. |
| 🟢 | **Thread Pool** | Processed the most requests (26,870), ran at 0–1ms in stable phases, and **recovered automatically** after burst overflow. This is how production servers (Tomcat, Jetty, Netty) actually work. |

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| ☕ Java (Socket Programming) | Core server implementation |
| 🧵 `Thread` / `Runnable` | Multi-threaded server |
| 🔄 `ExecutorService` + `Executors.newFixedThreadPool(10)` | Thread Pool server |
| 📊 Apache JMeter 5.6.2 | Load & performance testing |

---

<p align="center">Made with ❤️ using Java & Apache JMeter</p>
