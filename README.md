# TinyCoT
TinyCoT is a lightweight TCP relay server for ATAK/CoT clients. It supports multiple teams, event replay, optional CoT XMl dumping, and persistent state using JSON with automatic rotation. It also supports uploading Data Packages. 

Everything will be here soon. TinyCot was created to run on hardware with limited resources. It's not a replacement for enterprise-grade TAK servers and doesn't attempt to compete with them. At the moment there's no TLS support for clients because... I don't need it. :)

### ASCII diagram of TinyCoT's main structures and goroutines (functions)
<details>
  <summary>See below</summary>
<pre>
                         ┌──────────────┐
                         │   OS Signal  │
                         └──────┬───────┘
                                │
                                ▼
                       ┌───────────────────┐
                       │Graceful Shutdown  │
                       │  startGraceful... │
                       └─┬─────────────────┘
                         │
           ┌─────────────┴─────────────────┐
           ▼                               ▼
┌───────────────────┐           ┌───────────────────┐
│Marti HTTP Server  │           │ TCP Listener      │
│ (HTTP /Marti API) │           │ Accept() Loop     │
└───────────────────┘           └─────┬─────────────┘
                                        │
                ┌───────────────────────┴──────────────────────────┐
                ▼                                                  ▼
         ┌───────────────┐                                 ┌────────────────┐
         │ handleUpload  │                                 │ handleConn     │
         │ (DP POST)     │                                 │ (CoT TCP Client)
         └─────┬─────────┘                                 └───────┬────────┘
               │                                                   │
       ┌───────┴────────┐                                 ┌────────┴─────────┐
       ▼                ▼                                 ▼                  ▼
┌─────────────┐   ┌───────────────┐                 ┌───────────┐       ┌──────────────┐
│ Temp DP File│   │ DPInfo Struct │                 │ buf String│       │ sendChan[200]│
│   (disk)    │   │  dpMap in mem │                 │ accumulate│       │ message queue│
└─────┬───────┘   └─────┬─────────┘                 └───────────┘       └──────┬───────┘
      │                 │                                        │             │
      │                 │                                        │             │
      │                 ▼                                        ▼             ▼
      │          ┌───────────────┐                       ┌────────────────┐  ┌────────────┐
      │          │ SQLite dpDB   │                       │ Heartbeat      │  │ sender()   │
      │          │  write DPInfo │                       │ ticker 25s     │  │ goroutine  │
      │          └───────────────┘                       └────────────────┘  └────────────┘
      │                 │                                        │             │
      │                 ▼                                        ▼             ▼
      │          ┌───────────────┐                       ┌─────────────────────────┐
      │          │ persistent DP │<--------------------->│ broadcastCoTAsync()     │
      │          │   storage     │                       │ (send to all clients)   │
      │          └───────────────┘                       └─────────────────────────┘
      │
      ▼
┌───────────────┐
│ Files on Disk │
│ hash.zip      │
└───────────────┘
</details>

#### Legend / Notes
- Each client connection => 1 <code>handleConn</code> goroutine + 1 <code>sender</code> goroutine + 1 heartbeat ticker.
- <code>sendChan[200]</code> buffers messages to client; if client is slow, memory can grow (200*msg size).
- <code>buf</code> String in <code>handleConn</code> uses string concatenation; large messages → more GC pressure.
- DP uploads use temp file + streaming hash → memory efficient even for large files.
- <code>dpMap</code> in <code>Server</code> stores DP metadata in memory; scales with number of packages.
- SQLite persists <code>DPInfo</code>; <code>modernc.org/sqlite</code> is fully Go → safe for ARM64.
- Graceful shutdown closes all connections, stops tickers and goroutines.
</pre>

Estimated memory consumption values for 100 clients projected is ~ 80–90 MB.
