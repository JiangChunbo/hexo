---
title: DataSource 之 Tomcat JDBC
date: 2022-11-29 15:32:17
tags:
- DataSource
---

# 0. 参考引用

https://blog.csdn.net/zhangxin09/article/details/124901850


# 1.1. `ConnectionPool`

## 类属性

### size

连接池中当前活跃的连接数，通常与 `maxActive` 比较，防止超过最大活跃数。

```java
private AtomicInteger size = new AtomicInteger(0);
```

### busy

包含所有正在使用的连接

```java
private BlockingQueue<PooledConnection> busy;
```

### idle

包含所有空闲的连接

```java
private BlockingQueue<PooledConnection> idle;
```


## 方法


### 获得连接


一般思路是: 当需要获得一个 connection 时，从 `idle` 中 `poll` 一个 connection，然后将该 connection 再 `offer` 至 `busy` 队列中。这是最基本的思路。


```java
private PooledConnection borrowConnection(int wait, String username, String password) throws SQLException {

    if (isClosed()) {
        throw new SQLException("Connection pool closed.");
    } // 如果 Pool 关闭，抛出异常

    //get the current time stamp
    long now = System.currentTimeMillis();
    // 看看是否有一个立即可用的 connection
    PooledConnection con = idle.poll();

    while (true) {
        if (con!=null) {
            // 配置该 connection 并返回
            PooledConnection result = borrowConnection(now, con, username, password);
            borrowedCount.incrementAndGet();
            if (result!=null) {
                return result;
            }
        }

        //if we get here, see if we need to create one
        //this is not 100% accurate since it doesn't use a shared
        //atomic variable - a connection can become idle while we are creating
        //a new connection
        // 由于没有使用共享的原子性变量，
        // 因此当我们创建一个新的 connection 的时候，有可能这时候一个 connection 就空闲下来了
        if (size.get() < getPoolProperties().getMaxActive()) {
            // 原子性双重检测，减少 size.addAndGet 的并发
            if (size.addAndGet(1) > getPoolProperties().getMaxActive()) {
                // 如果走到这里，则表示线程并发，有多个线程进入该逻辑，操作是无效的，数量减一
                size.decrementAndGet();
            } else {
                // 可以创建线程，低于 max_active
                return createConnection(now, con, username, password);
            }
        } //end if

        //calculate wait time for this iteration
        long maxWait = wait;
        //if the passed in wait time is -1, means we should use the pool property value
        if (wait==-1) {
            maxWait = (getPoolProperties().getMaxWait()<=0)?Long.MAX_VALUE:getPoolProperties().getMaxWait();
        }

        long timetowait = Math.max(0, maxWait - (System.currentTimeMillis() - now));
        waitcount.incrementAndGet();
        try {
            // 获取一个现存的 connection
            con = idle.poll(timetowait, TimeUnit.MILLISECONDS);
        } catch (InterruptedException ex) {
            if (getPoolProperties().getPropagateInterruptState()) {
                Thread.currentThread().interrupt();
            }
            SQLException sx = new SQLException("Pool wait interrupted.");
            sx.initCause(ex);
            throw sx;
        } finally {
            waitcount.decrementAndGet();
        }
        if (maxWait==0 && con == null) { //no wait, return one if we have one
            if (jmxPool!=null) {
                jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - no wait.");
            }
            throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                    "NoWait: Pool empty. Unable to fetch a connection, none available["+busy.size()+" in use].");
        }
        //we didn't get a connection, lets see if we timed out
        if (con == null) {
            if ((System.currentTimeMillis() - now) >= maxWait) {
                if (jmxPool!=null) {
                    jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - timeout.");
                }
                throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                    "Timeout: Pool empty. Unable to fetch a connection in " + (maxWait / 1000) +
                    " seconds, none available[size:"+size.get() +"; busy:"+busy.size()+"; idle:"+idle.size()+"; lastwait:"+timetowait+"].");
            } else {
                //no timeout, lets try again
                continue;
            }
        }
    } //while
}
```

需要注意是: 

- 原子性 size 加一再减一


### 释放连接

```java
protected void returnConnection(PooledConnection con) {
    if (isClosed()) {
        //if the connection pool is closed
        //close the connection instead of returning it
        release(con);
        return;
    } //end if

    if (con != null) {
        try {
            returnedCount.incrementAndGet();
            con.lock();
            if (con.isSuspect()) {
                if (poolProperties.isLogAbandoned() && log.isInfoEnabled()) {
                    log.info("Connection(" + con + ") that has been marked suspect was returned."
                            + " The processing time is " + (System.currentTimeMillis()-con.getTimestamp()) + " ms.");
                }
                if (jmxPool!=null) {
                    jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.SUSPECT_RETURNED_NOTIFICATION,
                            "Connection(" + con + ") that has been marked suspect was returned.");
                }
            }
            if (busy.remove(con)) {

                if (!shouldClose(con,PooledConnection.VALIDATE_RETURN) && reconnectIfExpired(con)) {
                    con.clearWarnings();
                    con.setStackTrace(null);
                    con.setTimestamp(System.currentTimeMillis());
                    if (((idle.size()>=poolProperties.getMaxIdle()) && !poolProperties.isPoolSweeperEnabled()) || (!idle.offer(con))) {
                        if (log.isDebugEnabled()) {
                            log.debug("Connection ["+con+"] will be closed and not returned to the pool, idle["+idle.size()+"]>=maxIdle["+poolProperties.getMaxIdle()+"] idle.offer failed.");
                        }
                        release(con);
                    }
                } else {
                    if (log.isDebugEnabled()) {
                        log.debug("Connection ["+con+"] will be closed and not returned to the pool.");
                    }
                    release(con);
                } //end if
            } else {
                if (log.isDebugEnabled()) {
                    log.debug("Connection ["+con+"] will be closed and not returned to the pool, busy.remove failed.");
                }
                release(con);
            }
        } finally {
            con.unlock();
        }
    } //end if
} //checkIn
```



**release**

```java
protected void release(PooledConnection con) {
    if (con == null) {
        return;
    }
    try {
        con.lock();
        if (con.release()) {
            // 池中可用连接减一
            size.addAndGet(-1);
            con.setHandler(null);
        }
        releasedCount.incrementAndGet();
    } finally {
        con.unlock();
    }
    // we've asynchronously reduced the number of connections
    // we could have threads stuck in idle.poll(timeout) that will never be
    // notified
    if (waitcount.get() > 0) {
        if (!idle.offer(create(true))) {
            log.warn("Failed to add a new connection to the pool after releasing a connection " +
                    "when at least one thread was waiting for a connection.");
        }
    }
}
```


### 回收无效连接

如果 isPoolSweeperEnabled 为 true，则会注册一个 cleaner

```java
public boolean isPoolSweeperEnabled() {
    boolean timer = getTimeBetweenEvictionRunsMillis()>0;
    boolean result = timer && (isRemoveAbandoned() && getRemoveAbandonedTimeout()>0);
    result = result || (timer && getSuspectTimeout()>0);
    result = result || (timer && isTestWhileIdle());
    result = result || (timer && getMinEvictableIdleTimeMillis()>0);
    result = result || (timer && getMaxAge()>0);
    return result;
}
```



```java
private static synchronized void registerCleaner(PoolCleaner cleaner) {
    unregisterCleaner(cleaner);
    cleaners.add(cleaner);
    if (poolCleanTimer == null) {
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        try {
            Thread.currentThread().setContextClassLoader(ConnectionPool.class.getClassLoader());
            // Create the timer thread in a PrivilegedAction so that a
            // reference to the web application class loader is not created
            // via Thread.inheritedAccessControlContext
            PrivilegedAction<Timer> pa = new PrivilegedNewTimer();
            poolCleanTimer = AccessController.doPrivileged(pa);
        } finally {
            Thread.currentThread().setContextClassLoader(loader);
        }
    }
    poolCleanTimer.schedule(cleaner, cleaner.sleepTime,cleaner.sleepTime);
}
```



