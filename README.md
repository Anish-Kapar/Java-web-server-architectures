# ☕ Java Web Server Architectures

> 🔍 Comparing **Single-Threaded**, **Multi-Threaded**, and **Thread Pool** server designs under high concurrent load using Apache JMeter.

![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)
![Apache JMeter](https://img.shields.io/badge/Apache%20JMeter-D22128?style=for-the-badge&logo=apachejmeter&logoColor=white)
![Socket Programming](https://img.shields.io/badge/Socket-Programming-007396?style=for-the-badge&logo=java&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

---

## 📖 Overview

This project implements **three Java socket-based server architectures** to demonstrate how different concurrency models handle real-world load.

Each server is stress-tested with **10,000 requests at ~1,000 req/sec** using Apache JMeter to measure throughput, response time, latency, and scalability.

---

## 🏛️ Architectures

### 🔴 1. Single-Threaded Server

Processes one client request at a time. All requests are handled **sequentially** on a single thread — no concurrency at all.

```
Client → ServerSocket → Single Thread → Handle Request → Respond
                              ↑
                   (next client waits here)
```

**⚙️ How it works:**
- Server opens port and calls `accept()` in a loop
- Each accepted client is handled **before** the next one is accepted
- If processing is slow → every other client waits in queue

**Characteristics:**
- ✅ Simple to implement and debug
- ⚠️ Requests queue up — each waits for the previous to finish
- ❌ Completely blocks under concurrent load
- ❌ Not suitable for production


**📊 JMeter Results — 10,000 requests @ 1,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples | 11,000 requests |
| 🚀 Throughput | ~85 req/sec (~5,154 / min) |
| ⏱️ Average Response Time | **631 ms** |
| 📉 Median Response Time | **24 ms** |
| 📐 Std. Deviation | **1,329 ms** (very inconsistent) |
| ✅ Successful Responses | Partial (many connections refused) |
| 💾 Response Size | ~23 bytes per response |
| 🔎 Bottleneck | 1 thread handling 1,000 req/sec load |

> ⚠️ **Key insight:** Server could only handle ~85 req/sec out of 1,000 req/sec being sent. The high deviation (1,329 ms) shows extremely inconsistent performance — some requests were fast (24ms median) but others timed out completely.

---

### 🟡 2. Multi-Threaded Server

Spawns a **new thread for every incoming client request**, enabling true concurrency. Multiple clients are served simultaneously.

```
Client 1 ──→ Thread 1 ──→ Handle & Respond
Client 2 ──→ Thread 2 ──→ Handle & Respond
Client 3 ──→ Thread 3 ──→ Handle & Respond
Client N ──→ Thread N ──→ Handle & Respond
```

**⚙️ How it works:**
- Server calls `accept()` → immediately spawns a `new Thread()` for that client
- Main thread goes back to `accept()` right away
- Each client gets its own dedicated thread

**Characteristics:**
- ✅ True concurrent request handling
- ✅ Each client gets immediate response
- ✅ Much faster than single-threaded under load
- ⚠️ 1,000 clients → 1,000 threads created simultaneously
- ❌ Unbounded thread creation can crash JVM under extreme load
- ❌ High memory + CPU overhead per thread

**📊 JMeter Results — 10,000 requests @ 1,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples | 10,000+ requests |
| 🚀 Throughput | Significantly higher than Single-Threaded |
| ⏱️ Average Response Time | Reduced compared to Single-Threaded |
| 📐 Std. Deviation | Lower than Single-Threaded |
| ✅ Successful Responses | Higher success rate |
| 🔎 Bottleneck | Unbounded thread creation under heavy load |

> ⚠️ **Key insight:** Big improvement over single-threaded, but under 1,000 req/sec, the JVM spawns 1,000 threads at once — this becomes a resource problem at very high concurrency.

---

### 🟢 3. Thread Pool Server ⭐ Recommended

Uses a **fixed pool of reusable worker threads**. Requests are queued and picked up by available threads — no overhead from creating/destroying threads per request.

```
Client 1 ──→ ┐
Client 2 ──→ ├──→ Request Queue ──→ [ Thread 1 ]──→ Respond
Client 3 ──→ ├──→                   [ Thread 2 ]──→ Respond
Client N ──→ ┘                      [ Thread 3 ]──→ Respond
                                    [    ...   ]
                                    [ Thread 10]──→ Respond
```

**⚙️ How it works (`poolSize = 10`):**
- `Executors.newFixedThreadPool(10)` creates exactly 10 worker threads
- Every accepted client is submitted via `threadPool.execute()`
- If all 10 threads are busy → new clients wait in queue (not rejected)
- Threads are **reused** — zero creation overhead per request

**Characteristics:**
- ✅ Controlled, bounded thread usage (max 10 threads)
- ✅ Threads are reused — no creation/destruction overhead
- ✅ Graceful queue management under burst load
- ✅ Consistent and predictable performance
- ✅ Production-ready architecture (used by Tomcat, Jetty, etc.)

**📊 JMeter Results — 10,000 requests @ 1,000 req/sec:**

| Metric | Value |
|--------|-------|
| 📦 Total Samples | 10,000+ requests |
| 🚀 Throughput | Highest & most stable |
| ⏱️ Average Response Time | Lowest & most consistent |
| 📐 Std. Deviation | Lowest (most predictable) |
| ✅ Successful Responses | Best success rate |
| 🔎 Advantage | Fixed 10 threads handle all load via queue |

> ✅ **Key insight:** With only 10 threads, this server handles the same 10,000 request load more efficiently than spawning 1,000 threads. Less memory, lower CPU overhead, better throughput consistency.

---

## 📈 Performance Comparison

| Architecture | Throughput | Avg Response Time | Std Deviation | Scalability | Thread Usage |
|---|---|---|---|---|---|
| 🔴 Single-Threaded | ~85 req/sec | **631 ms** | **1,329 ms** | ❌ Poor | 1 thread |
| 🟡 Multi-Threaded | High ↑ | Moderate ↓ | Medium | ⚠️ Good | 1 thread per client |
| 🟢 Thread Pool | Highest ↑↑ | Lowest ↓↓ | Lowest | ✅ Best | Fixed 10 threads |

---

## 🧪 JMeter Test Setup

| Parameter | Value |
|---|---|
| 🛠️ Tool | Apache JMeter |
| 📦 Total Requests | 10,000 |
| ⚡ Simulated Load | ~1,000 requests/sec (10 sec ramp) |
| 🌐 Protocol | TCP Socket |
| 🖥️ Server Host | localhost:8010 |
| 📐 Metrics Tracked | Throughput, Avg Response Time, Median, Std Deviation, Latency, Connect Time, Bytes |

### 📋 JMeter Metrics Explained

| Metric | What it means |
|---|---|
| **Sample Time (ms)** | Total time server took to respond |
| **Latency** | Time from request sent → first byte received |
| **Connect Time** | Time to establish the TCP socket connection |
| **Throughput** | Requests successfully handled per second |
| **Median** | Response time for the middle 50% of requests (more reliable than average) |
| **Std Deviation** | How inconsistent the response times are — lower is better |

---

## 🏁 Conclusion

| | Architecture | Verdict |
|---|---|---|
| 🔴 | **Single-Threaded** | Simple but not scalable. Only 85 req/sec under 1,000 req/sec load. High deviation (1,329ms) means extremely unpredictable performance. |
| 🟡 | **Multi-Threaded** | Good concurrency improvement but dangerous under heavy load — unbounded thread creation can exhaust JVM memory. |
| 🟢 | **Thread Pool** | Best of all three. Fixed threads + queue = production-ready, consistent, and resource-efficient. This is how real servers (Tomcat, Jetty) work. |

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| ☕ Java (Socket Programming) | Core server implementation |
| 🧵 `Thread` / `Runnable` | Multi-threaded server |
| 🔄 `ExecutorService` + `Executors.newFixedThreadPool()` | Thread Pool server |
| 📊 Apache JMeter | Load & performance testing |

---

<p align="center">Made with ❤️ using Java</p>
