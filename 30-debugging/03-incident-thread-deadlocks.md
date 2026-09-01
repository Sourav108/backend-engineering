# Incident 03: Thread Deadlocks in Concurrency (Inconsistent Lock Ordering)

---

## 1. Symptoms & Alert
- **Alert**: `HTTP 504 Gateway Timeout` on `/api/v1/accounts/transfer`.
- **Customer Impact**: Account-to-account funds transfers completely frozen; incoming API requests queued up until Tomcat worker thread pool hit $100\%$ saturation.

---

## 2. Metric & Telemetry Anomalies
- **Micrometer Metrics**: `tomcat.threads.busy` reached max capacity ($200/200$ threads).
- **CPU & Memory**: CPU dropped to nearly $0\%$; memory usage flat (threads were completely blocked waiting for locks).

---

## 3. Diagnostic Steps

### Step 1: Capture JVM Thread Dump via `jcmd`
```bash
kubectl exec -it account-service-99b8f-v9x11 -n production -- jcmd 1 Thread.print > /tmp/threaddump.txt
```

### Step 2: Inspect Deadlock Section of Thread Dump
```text
Found one Java-level deadlock:
=============================
"http-nio-8080-exec-12":
  waiting to lock monitor 0x00007f8a1c004a80 (object 0x0000000712345678, a com.backend.Account),
  which is held by "http-nio-8080-exec-45"

"http-nio-8080-exec-45":
  waiting to lock monitor 0x00007f8a1c007b20 (object 0x0000000787654321, a com.backend.Account),
  which is held by "http-nio-8080-exec-12"

Java stack information for the threads listed above:
===================================================
"http-nio-8080-exec-12":
	at com.backend.AccountService.transferFunds(AccountService.java:24)
	- waiting to lock <0x0000000712345678> (a com.backend.Account)
	- locked <0x0000000787654321> (a com.backend.Account)

"http-nio-8080-exec-45":
	at com.backend.AccountService.transferFunds(AccountService.java:24)
	- waiting to lock <0x0000000787654321> (a com.backend.Account)
	- locked <0x0000000712345678> (a com.backend.Account)
```

---

## 4. Root Cause Analysis
- Thread 12 was executing `transfer(Account 101, Account 202)`. It locked Account 101 first, then attempted to acquire Account 202.
- Concurrently, Thread 45 was executing `transfer(Account 202, Account 101)`. It locked Account 202 first, then attempted to acquire Account 101.
- Because lock acquisition order was **non-deterministic and dependent on input argument order**, the two threads formed a circular wait condition (**Coffman Deadlock Condition**), freezing both threads and cascading across all 200 Tomcat worker threads.

---

## 5. Immediate Mitigation
Restart the service pods to release the frozen threads and clear the deadlock state:
```bash
kubectl rollout restart deployment account-service -n production
```

---

## 6. Permanent Fix
Enforce **Global Deterministic Lock Ordering** (e.g. always acquiring locks in ascending order of `accountId`):

```java
// BEFORE (VULNERABLE TO CIRCULAR DEADLOCK)
public void transferFunds(Account from, Account to, long amount) {
    synchronized (from) {
        synchronized (to) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// AFTER (DETERMINISTIC GLOBAL LOCK ORDERING)
public void transferFunds(Account from, Account to, long amount) {
    Account firstLock = from.getId() < to.getId() ? from : to;
    Account secondLock = from.getId() < to.getId() ? to : from;

    synchronized (firstLock) {
        synchronized (secondLock) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

---

## 7. Postmortem Action Items
- [x] Replace Java-level mutex synchronization with database-level row locks (`SELECT ... FOR UPDATE ORDER BY id ASC`).
- [x] Configure automated thread deadlock detection metrics in Micrometer (`jvm.threads.deadlocked`).
