# ZK -- 网络连接（Client端）

ClientCnxn管理client端等socket io，它维护可用的server 列表，并且可以透明地进行server连接切换。

## 主要成员

outgoingQueue和pendingQueue是两个重要的队列，用于存储等待发送和已发送了在等待response的packet。

SendThread负责发送心跳和请求，EventThread负责处理各种event。

```java
//已发送request，在等待response的packet
private final Queue<Packet> pendingQueue = new ArrayDeque<>();
//等待发送的packet
private final LinkedBlockingDeque<Packet> outgoingQueue = new LinkedBlockingDeque<Packet>();
//管理watcher，处理ClientCnxn生成的event
private final ZKWatchManager watchManager;
private long sessionId;
private byte[] sessionPasswd;
final String chrootPath;
//发送request，生成heartbeat，同时作为ReadThread的上游。守护线程。
final SendThread sendThread;
//处理Event/Packet的线程，守护线程
final EventThread eventThread;
//管理client能连接的server host
private final HostProvider hostProvider;
//管理SASL认证？？？
public ZooKeeperSaslClient zooKeeperSaslClient;
//配置信息
private final ZKClientConfig clientConfig;
```

Packet封装了Request、Response、Callback等.

```java
static class Packet {
        RequestHeader requestHeader;
        ReplyHeader replyHeader;
        Record request;
        Record response;
        ByteBuffer bb;
        /** Client's view of the path (may differ due to chroot) **/
        String clientPath;
        /** Servers's view of the path (may differ due to chroot) **/
        String serverPath;
        boolean finished;
        AsyncCallback cb;
        Object ctx;
        WatchRegistration watchRegistration;
        public boolean readOnly;
        WatchDeregistration watchDeregistration;
}
```

从Client的代码可以看出，同步调用create、delete等通过ClientCnxn.submitRequest实现，submitRequest主要逻辑如下。主要是将Packet放入outgoingQueue中，并等待返回结果。

异步调用直接调用queuePacket，将Packet放入outgoingQueue。调用时会传递一个AsyncCallback参数，用于回调。

outgoingQueue中的packet是谁来消费呢？我猜测是由SendThread负责的。

更新：看了ClientCnxnSocket代码，发现outgoingQueue实际是由ClientCnxnSocket消费的，ClientCnxnSocket将outgoingQueue中packet发送出去，并放入到peningQueue中。

```java
public ReplyHeader submitRequest(
      RequestHeader h,
      Record request,
      Record response,
      WatchRegistration watchRegistration,
      WatchDeregistration watchDeregistration) throws InterruptedException {
      ReplyHeader r = new ReplyHeader();
      **Packet packet = queuePacket(**
          h,
          r,
          request,
          response,
          null,
          null,
          null,
          null,
          watchRegistration,
          watchDeregistration);
      synchronized (packet) {
          if (requestTimeout > 0) {
              // Wait for request completion with timeout
              waitForPacketFinish(r, packet);
          } else {
              // Wait for request completion infinitely
              while (!packet.finished) {
                  packet.wait();
              }
          }
      }
      if (r.getErr() == Code.REQUESTTIMEOUT.intValue()) {
          sendThread.cleanAndNotifyState();
      }
      return r;
  }

public Packet queuePacket(
        RequestHeader h,
        ReplyHeader r,
        Record request,
        Record response,
        AsyncCallback cb,
        String clientPath,
        String serverPath,
        Object ctx,
        WatchRegistration watchRegistration,
        WatchDeregistration watchDeregistration) {
        Packet packet = null;

        // Note that we do not generate the Xid for the packet yet. It is
        // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
        // where the packet is actually sent.
        packet = new Packet(h, r, request, response, watchRegistration);
        packet.cb = cb;
        packet.ctx = ctx;
        packet.clientPath = clientPath;
        packet.serverPath = serverPath;
        packet.watchDeregistration = watchDeregistration;
        // The synchronized block here is for two purpose:
        // 1. synchronize with the final cleanup() in SendThread.run() to avoid race
        // 2. synchronized against each packet. So if a closeSession packet is added,
        // later packet will be notified.
        synchronized (state) {
            if (!state.isAlive() || closing) {
                conLossPacket(packet);
            } else {
                // If the client is asking to close the session then
                // mark as closing
                if (h.getType() == OpCode.closeSession) {
                    closing = true;
                }
                **outgoingQueue.add(packet);**
            }
        }
        sendThread.getClientCnxnSocket().packetAdded();
        return packet;
    }
```

## SendThread

**负责向Zookeeper Server发送心跳，并维持与Server的session，调用clientCnxnSocket向server发送packet，并对异常情况进行处理。**

主要成员为ClientCnxnSocket， 有ClientCnxnSocketNetty、ClientCnxnSocketNIO等实现，从名字就可以看出是用于Socket网络通信。

```java
private final ClientCnxnSocket clientCnxnSocket;
```

ClientCnxnSocket生命周期为如下。

doTransport中发送outgoingQueue中packet，发送之后将其放入到pendingQueue，供之后获取结果。

```java
* loop:
 * - try:
 * - - !isConnected()
 * - - - connect()
 * - - doTransport()
 * - catch:
 * - - cleanup()
 * close()
```

SendThread主要流程如下。主要做的事情就是是建立连接、发送数据、关闭连接。

```java
@Override
public void run() {
    clientCnxnSocket.introduce(this, sessionId, outgoingQueue);
    clientCnxnSocket.updateNow();
    clientCnxnSocket.updateLastSendAndHeard();
    int to;
    long lastPingRwServer = Time.currentElapsedTime();
    final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
    InetSocketAddress serverAddress = null;
    while (state.isAlive()) {
        try {
            if (!clientCnxnSocket.isConnected()) {
                // don't re-establish connection if we are closing
                if (closing) {
                    break;
                }
                if (rwServerAddress != null) {
                    serverAddress = rwServerAddress;
                    rwServerAddress = null;
                } else {
                    serverAddress = hostProvider.next(1000);
                }
                onConnecting(serverAddress);
                **startConnect(serverAddress);**
                clientCnxnSocket.updateLastSendAndHeard();
            }

            if (state.isConnected()) {
                // determine whether we need to send an AuthFailed event.
                if (zooKeeperSaslClient != null) {
                    //略....
                }
                to = readTimeout - clientCnxnSocket.getIdleRecv();
            } else {
                to = connectTimeout - clientCnxnSocket.getIdleRecv();
            }

            if (to <= 0) {
                //略....
                throw new SessionTimeoutException(warnInfo);
            }
            if (state.isConnected()) {
                //1000(1 second) is to prevent race condition missing to send the second ping
                //also make sure not to send too many pings when readTimeout is small
                int timeToNextPing = readTimeout / 2
                                     - clientCnxnSocket.getIdleSend()
                                     - ((clientCnxnSocket.getIdleSend() > 1000) ? 1000 : 0);
                //send a ping request either time is due or no packet sent out within MAX_SEND_PING_INTERVAL
                if (timeToNextPing <= 0 || clientCnxnSocket.getIdleSend() > MAX_SEND_PING_INTERVAL) {
										//发送心跳，更新lastSend
                    sendPing();
                    clientCnxnSocket.updateLastSend();
                } else {
                    if (timeToNextPing < to) {
                        to = timeToNextPing;
                    }
                }
            }

            //略....
						//doTransport负责读取packets到incomingBuffer、写outgoing queue packets、更新相关时间戳
            **clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this);**
        } catch (Throwable e) {
            //略....
        }
    }

    synchronized (state) {
        // When it comes to this point, it guarantees that later queued
        // packet to outgoingQueue will be notified of death.
        cleanup();
    }
    **clientCnxnSocket.close();**
    if (state.isAlive()) {
        **eventThread.queueEvent(new WatchedEvent(Event.EventType.None, Event.KeeperState.Disconnected, null));**
    }
    eventThread.queueEvent(new WatchedEvent(Event.EventType.None, Event.KeeperState.Closed, null));
}
```

## EventThread

EventThread负责处理event，包括watch event和Create、Get等请求等结果packet。主要成员：

```java
private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
```

EventThread主要逻辑是从waitingEvents中取event，交给对应的processor进行处理：

```java
public void run() {
    try {
        isRunning = true;
        while (true) {
            Object event = waitingEvents.take();
            if (event == eventOfDeath) {
                wasKilled = true;
            } else {
                processEvent(event);
            }
            if (wasKilled) {
                synchronized (waitingEvents) {
                    if (waitingEvents.isEmpty()) {
                        isRunning = false;
                        break;
                    }
                }
            }
        }
    } catch (InterruptedException e) {
        LOG.error("Event thread exiting due to interruption", e);
    }
}

//处理event的主要逻辑，包括watchEvent、LocalCallBack、各种Response等的处理
private void processEvent(Object event) {
            try {
                if (event instanceof WatcherSetEventPair) {
                    // each watcher will process the event
                    WatcherSetEventPair pair = (WatcherSetEventPair) event;
                    for (Watcher watcher : pair.watchers) {
                        try {
                            watcher.process(pair.event);
                        } catch (Throwable t) {
                            LOG.error("Error while calling watcher.", t);
                        }
                    }
                } else if (event instanceof LocalCallback) {
                    LocalCallback lcb = (LocalCallback) event;
                    if (lcb.cb instanceof StatCallback) {
                        ((StatCallback) lcb.cb).processResult(lcb.rc, lcb.path, lcb.ctx, null);
                    } else if (lcb.cb instanceof DataCallback) {
                        ((DataCallback) lcb.cb).processResult(lcb.rc, lcb.path, lcb.ctx, null, null);
                    } else if (lcb.cb instanceof ACLCallback) {
                        ((ACLCallback) lcb.cb).processResult(lcb.rc, lcb.path, lcb.ctx, null, null);
                    } else if 
											//...
                } else {
                    Packet p = (Packet) event;
                    int rc = 0;
                    String clientPath = p.clientPath;
                    if (p.replyHeader.getErr() != 0) {
                        rc = p.replyHeader.getErr();
                    }
                    if (p.cb == null) {
                        LOG.warn("Somehow a null cb got to EventThread!");
                    } else if (p.response instanceof ExistsResponse
                               || p.response instanceof SetDataResponse
                               || p.response instanceof SetACLResponse) {
                        StatCallback cb = (StatCallback) p.cb;
                        if (rc == Code.OK.intValue()) {
                            if (p.response instanceof ExistsResponse) {
                                cb.processResult(rc, clientPath, p.ctx, ((ExistsResponse) p.response).getStat());
                            } else if (p.response instanceof SetDataResponse) {
                                cb.processResult(rc, clientPath, p.ctx, ((SetDataResponse) p.response).getStat());
                            } else if (p.response instanceof SetACLResponse) {
                                cb.processResult(rc, clientPath, p.ctx, ((SetACLResponse) p.response).getStat());
                            }
                        } else {
                            cb.processResult(rc, clientPath, p.ctx, null);
                        }
                    } else if (p.response instanceof GetDataResponse) {
                        DataCallback cb = (DataCallback) p.cb;
                        GetDataResponse rsp = (GetDataResponse) p.response;
                        if (rc == Code.OK.intValue()) {
                            cb.processResult(rc, clientPath, p.ctx, rsp.getData(), rsp.getStat());
                        } else {
                            cb.processResult(rc, clientPath, p.ctx, null, null);
                        }
                    } else if (p.response instanceof GetACLResponse) {
                       ...
                }
            } catch (Throwable t) {
                LOG.error("Unexpected throwable", t);
            }
        }

    }
```

## ZKWatchManager

管理用户通过API提交的watcher，有add，remove，query等功能。

```java
private final Map<String, Set<Watcher>> dataWatches = new HashMap<>();
private final Map<String, Set<Watcher>> existWatches = new HashMap<>();
private final Map<String, Set<Watcher>> childWatches = new HashMap<>();
private final Map<String, Set<Watcher>> persistentWatches = new HashMap<>();
private final Map<String, Set<Watcher>> persistentRecursiveWatches = new HashMap<>();
private final boolean disableAutoWatchReset;

private volatile Watcher defaultWatcher;
```

## 主要流程

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_network.png)
