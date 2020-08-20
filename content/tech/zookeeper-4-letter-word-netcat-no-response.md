---
title: "Zookeeper 4 Letter Word Netcat No Response"
date: 2020-08-19T16:20:00-07:00
slug: ""
description: "Zookeeper 4 letter word commands do not get response via netcat. How to check if Zookeeper is responding and resolve the netcat issue."
keywords: []
draft: false
tags: ["tech", "quickie", "zookeeper", "netcat"]
math: false
toc: true
---

# Problem

To report Zookeeper service health, we use the 4 letter word (_4LW_) command
`ruok` and check that response is `imok`.

`netcat` is the channel via which the message is
sent.

After upgrading the operating system from Ubuntu `16.04` to `18.04`, we see that
intermittently there is no response. Hmm...

# Triage

## Is Zookeeper getting those commands?

A simple bash script can be used to verify this:

```bash
#!/bin/bash

for i in {1..10}
do
  status=$(echo "ruok" | nc localhost 2181)
  echo -e "try ${i}: ${status}"
done
```

Output looks suspect:

```bash
root@zk-node:~# ./check_zk.sh
try 1: imok
try 2: imok
try 3: imok
try 4: imok
try 5: imok
try 6:
try 7: imok
try 8:
try 9: imok
try 10: imok
root@zk-node:~#
```

But Zookeeper service has no issues processing the requests:

```bash
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,054 [myid:] - INFO  [NIOWorkerThread-31:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49880
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,091 [myid:] - INFO  [NIOWorkerThread-22:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49882
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,110 [myid:] - INFO  [NIOWorkerThread-20:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49884
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,125 [myid:] - INFO  [NIOWorkerThread-4:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49886
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,139 [myid:] - INFO  [NIOWorkerThread-3:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49888
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,153 [myid:] - INFO  [NIOWorkerThread-1:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49890
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,166 [myid:] - INFO  [NIOWorkerThread-23:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49892
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,179 [myid:] - INFO  [NIOWorkerThread-11:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49894
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,190 [myid:] - INFO  [NIOWorkerThread-9:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49896
Jul 24 22:21:07 zk-node zookeeper[23515]: 2020-07-24 22:21:07,201 [myid:] - INFO  [NIOWorkerThread-5:NIOServerCnxn@507] - Processing ruok command from /127.0.0.1:49898
```

No errors or warnings or exceptions. Looks like it is responding just fine. 

## 4LW commands do not work consistently

Next, we tried looking for various causes that result in 4LW commands silently
failing.
Bumped into this known Zookeeper issue
<cite>ZOOKEEPER-737: some 4 letter words may fail with netcat (nc) [^1]</cite>.

Now we did not observe any exceptions in ZK logs - probably because we're
using a later version of ZK - `v3.6.1`. But it does seem like a plausible
root cause.

A <cite>StackOverflow post [^2]</cite> points to the same with a fix to add
delay in netcat before closing the channel.

# Solution

## Add delay to netcat before closing the channel

Based on the <cite>StackOverflow post [^2]</cite>, we added the `-q 1` flag;
which adds a 1 second delay after sending the message and before it closes
the channel.

### Result?

```bash
# status=$(echo "ruok" | nc localhost 2181 -q 1)

root@zk-node:~# ./check_zk.sh
try 1: imok
try 2: imok
try 3: imok
try 4: imok
try 5: imok
try 6: imok
try 7: imok
try 8: imok
try 9: imok
try 10: imok
root@zk-node:~#
```

Consistency at last!

The goal of any platform - it works or it doesn't; either is fine, as long as
it does that consistently.

**TIL**

[^1]: https://issues.apache.org/jira/browse/ZOOKEEPER-737
[^2]: https://stackoverflow.com/questions/26182416/zookeeper-server-started-but-ruok-no-output
