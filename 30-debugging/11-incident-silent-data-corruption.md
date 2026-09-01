# Incident 11: Silent Data Corruption (Lost Update Anomaly)

---

## 1. Symptoms & Alert
- **Alert**: Financial audit reconciliation script detected ledger discrepancies: `LedgerDiscrepancy: -$142,500 total account balance drift`.
- **Customer Impact**: Users submitted balance withdrawals, but total ledger sums failed to balance; customer balances silently overwritten.

---

## 2. Metric & Telemetry Anomalies
- **Error Rates**: HTTP $2\text{xx}$ success rate sat at $100\%$ (Zero exceptions thrown in logs!).
- **Database Logs**: No deadlock errors, no transaction rollbacks; all SQL statements returned `200 OK`.

---

## 3. Diagnostic Steps & Root Cause
Inspecting the Account Service code:

```java
// DISASTER: NON-ATOMIC READ-MODIFY-WRITE WITHOUT LOCKING OR VERSION CHECKS
public void addRewardPoints(Long userId, int pointsToAdd) {
    UserAccount account = accountRepository.findById(userId); // Step 1: Read (Balance = 100)
    
    int newPoints = account.getPoints() + pointsToAdd;        // Step 2: Modify in JVM RAM (100 + 10 = 110)
    account.setPoints(newPoints);
    
    accountRepository.save(account);                          // Step 3: Write back (Balance = 110)
}
```

### The Lost Update Anomaly:
1. Thread A reads `Points = 100` for User 42.
2. Simultaneously, Thread B reads `Points = 100` for User 42.
3. Thread A adds 10 points and saves `Points = 110`.
4. Thread B adds 50 points and saves `Points = 150` (overwriting Thread A's commit!).
5. **Result**: Thread A's 10-point addition was **silently lost forever** with zero errors logged.

---

## 4. Immediate Mitigation
Execute reconciliation migration script recalculating all user account points directly from immutable append-only transaction logs.

---

## 5. Permanent Fix
Implement **Atomic SQL Increments** or **JPA Optimistic Locking (`@Version`)**:

```java
// SOLUTION 1: Atomic Database Increments
@Modifying
@Query("UPDATE UserAccount u SET u.points = u.points + :points WHERE u.id = :userId")
int incrementPointsAtomically(@Param("userId") Long userId, @Param("points") int points);

// SOLUTION 2: Optimistic Locking with Version Field
@Entity
public class UserAccount {
    @Id private Long id;
    private int points;
    
    @Version
    private Long version; // Triggers OptimisticLockException on concurrent conflict!
}
```

---

## 6. Postmortem Action Items
- [x] Add `@Version` column to all financial and inventory JPA entities.
- [x] Run hourly automated reconciliation jobs comparing balances against immutable ledger tables.
