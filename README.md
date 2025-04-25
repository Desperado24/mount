# mount
Mount

## IM 系统技术方案文档

### 1. **概述**

本IM系统方案主要围绕会话数据管理、分页刷新机制和并发控制进行设计，以确保系统能够处理高并发场景下的会话更新、分页刷新、数据一致性和性能优化。本文档详细描述了系统实现的技术原理、实现细节、优缺点及优化方案。

### 2. **系统架构设计**

IM系统主要包括以下几个核心模块：

1. **会话数据管理**：
    - 存储和管理用户的IM会话数据，包括会话列表、消息记录等。
    - 采用 **乐观锁** 和 **重试机制** 来确保数据一致性。
   
2. **分页刷新机制**：
    - 使用 **PagingSource** 和 **Pager** 进行数据的分页加载。
    - 通过 **ViewModel** 和 **LiveData** 实现分页数据的异步加载和展示。
    - 在会话数据更新后，通过事件通知刷新分页数据，确保数据一致性。

3. **并发控制与锁优化**：
    - 使用 **Mutex** 进行线程安全控制，避免多个线程同时修改会话数据导致的数据冲突。
    - 在高并发场景下，使用 **乐观锁** 和 **重试机制** 来避免锁的竞争。

4. **错误处理与重试机制**：
    - 在数据库操作失败时，通过重试机制确保数据的最终一致性。
    - 优化了数据库操作和分页刷新，以减少系统负担。

### 3. **会话更新机制**

#### 3.1 **乐观锁与重试机制**

会话更新操作使用了 **乐观锁** 来避免并发冲突。每次更新会检查当前会话的版本号是否与数据库中的版本号一致，确保不会覆盖其他线程或操作对该会话的修改。

具体实现如下：

```kotlin
private suspend fun updateSessionOptimisticLockWithRetry(session: Session) {
    var retryCount = 0
    while (retryCount < 3) {
        val db = IMDatabase.getInstance()
        if (db == null || !db.isOpen) {
            break
        }
        val rows = sessionDao?.updateSessionWithVersion()
        if (rows ?: 0 > 0) {
            _sessionUpdateEvents.emit(System.currentTimeMillis()) // 刷新分页数据
            unreadRefreshTrigger.emit(Unit)
            ALog.i(TAG, "updateSessionOptimisticLockWithRetry ${session.lastMsgId}")
            return // 成功，退出
        }
        ALog.i(TAG, "updateSessionOptimisticLockWithRetry fail ${session.lastMsgId}")
        retryCount++
        // 如果版本号不匹配，更新版本号
        val latest = sessionDao?.getSessionByUid(session.sessionUid)
        if (latest == null) {
            break
        }
        if (latest.version > session.version) {
            session.version = latest.version
        }
        // 如果库中的未读消息数大于0，且大于当前会话的未读数，则更新未读数
        if (latest.unReadNum > 0 && session.unReadNum > 0 && latest.unReadNum > session.unReadNum) {
            session.unReadNum = latest.unReadNum
        }
    }
}
```

#### **技术分析与细节：**

- **乐观锁**：通过 `version` 字段控制并发更新，确保数据一致性。
- **重试机制**：如果数据库更新失败，则会进行最多 3 次的重试。
- **数据一致性**：如果 `version` 不一致，重新获取数据库中的会话数据并更新本地会话。
- **分页刷新**：更新成功后，通过 `_sessionUpdateEvents.emit(System.currentTimeMillis())` 通知 ViewModel 进行分页刷新，确保 UI 展示的是最新的会话数据。

### 4. **分页与刷新机制**

#### 4.1 **分页加载与刷新**
IM系统的分页与刷新机制采用了 Paging 组件，结合手动刷新与自动刷新逻辑来确保会话数据在不同场景下都能高效更新。
这个方案的核心目标是避免加载过多的会话数据，同时确保用户在不同情况下都能获得最新的会话信息。
#### 4.2 **为什么使用 Paging**
在IM系统中，用户的会话列表可能包含大量数据。若采用传统的全量加载方式，性能会受到很大影响，尤其是在网络状况较差或设备性能不足时。
Paging 组件提供了一个优秀的解决方案，通过按需加载数据并缓存结果来优化性能，减少内存占用。
Paging 的核心功能如下：
分页加载：按页加载数据，避免一次性加载过多数据。

内存优化：只有当前页面的数据被加载到内存中，减少内存压力。

数据库和网络资源节约：减少了数据库查询和网络请求的频率。

在分页加载过程中，系统会使用 **PagingSource** 和 **Pager** 来加载分页数据。通过 `PagingData` 和 `PagingSource`，系统能够高效地从数据库中获取分页数据。
val pager = Pager(
    config = PagingConfig(pageSize = 20),
    pagingSourceFactory = { getSessionPagerList() }
)

val pagingData = pager.flow.cachedIn(viewModelScope)

此时，getSessionPagerList() 方法会负责从数据库中查询数据，并将数据分页返回。
分页逻辑采用了 PagingSource 来管理加载数据的行为，通过 PagingConfig 配置页大小，并将数据流 PagingData 提供给 UI 层进行展示。

优点：
性能优化：分批加载会话数据，避免全量加载造成的内存压力和UI卡顿。

灵活性：通过配置 pageSize 和缓存机制，能够适应不同的数据量和不同的网络环境。

#### 4.3 **数据库更新失败与分页更新的关系**
当数据库更新失败时，会影响 会话数据 的一致性和 分页刷新 的准确性。具体来说，以下几个问题可能导致分页数据无法更新：

乐观锁冲突：当更新会话数据时，如果数据库中的版本号与更新时传递的版本号不一致（如数据被其他线程修改）。在这种情况下，分页数据不会被更新，UI层会继续展示旧的数据。

事务失败：当进行批量更新时，若数据库事务发生错误，可能导致部分数据更新失败。如果在这种情况下没有妥善处理重试或回滚机制，分页数据也会保持不一致。

网络问题或数据库不可用：在网络问题或数据库不可用时，更新操作会失败，分页数据无法及时刷新，用户可能会看到过时的数据。


#### 4.3 **刷新机制**
刷新机制主要通过两种方式触发：

手动刷新：用户主动触发刷新，例如通过下拉刷新等方式。
IM系统中的手动刷新通过 manualRefresh.emit(Unit) 触发：
viewModelScope.launch { manualRefresh.emit(Unit) }


自动刷新：系统在会话数据（如未读数、最后更新时间等）更新后自动触发刷新。
**分页刷新机制**的核心在于，通过监听 sessionUpdateEvents 事件实现。
val rawRefresh = merge(
    imRepository.getSessionUpdateEvents(),
    manualRefresh.map { System.currentTimeMillis() }
)

每当会话数据（如未读数）发生变化时，IM系统会通过该事件触发数据的刷新，确保会话列表始终展示最新的数据。
```kotlin
 if (rows ?: 0 > 0) {
            _sessionUpdateEvents.emit(System.currentTimeMillis()) // 刷新分页数据
            unreadRefreshTrigger.emit(Unit)
            return // 成功，退出
        }
```
#### 4.3 **采用 throttleWithTimeout 控制刷新频率**
在高频率的会话更新场景下，频繁地触发刷新会对系统性能造成很大影响，尤其是在网络或数据库较为繁忙时。为了避免过于频繁的刷新，我们引入了 throttleWithTimeout 来限制刷新频率。
rawRefresh
    .throttleWithTimeout(1_000) // 每秒最多刷新一次
throttleWithTimeout 会限制流的触发频率，确保每秒最多只刷新一次数据，这样即使会话数据多次更新，UI刷新也不会过于频繁，减少了性能开销。


#### **技术分析与细节：**

- **更新失败处理**：数据库更新失败时，系统会通过重试机制保证最终更新成功。若更新失败，分页数据不会及时更新。通过 `_sessionUpdateEvents.emit` 来触发分页刷新，确保分页数据同步。
- **PagingSource**：数据分页通过 `PagingSource` 进行管理，系统根据不同的 `tabType` 来加载不同类型的数据。
- **刷新策略**：通过事件驱动的方式来触发分页刷新，避免了频繁的全量刷新，提高了系统性能。

#### **方案优点：**

- **高效的分页加载**：通过 `PagingSource` 和 `Pager` 来实现数据的按需加载，避免了加载所有数据造成的性能问题。
- **实时数据同步**：通过 `_sessionUpdateEvents` 通知 ViewModel 刷新数据，确保会话数据的实时性。
- **良好的用户体验**：通过乐观锁和重试机制，确保用户能够看到最新的会话数据，避免因数据库冲突而导致的数据错误。

### 5. **高并发与锁优化**

#### 5.1 **锁优化**

由于 IM 系统会处理高并发的会话更新请求，采用 **锁优化机制** 来避免并发冲突。为每个会话（`sessionUid`）分配一个独立的锁，并使用 **Mutex** 控制并发操作。

```kotlin
private val sessionLocks = ConcurrentHashMap<Long, Mutex>()

fun getSessionLock(sessionUid: Long): Mutex {
    return sessionLocks[sessionUid] ?: synchronized(sessionLocks) {
        sessionLocks.getOrPut(sessionUid) { Mutex() }
    }
}
```

#### **技术分析与细节：**

- **每个会话独立锁**：通过 `Mutex` 实现对每个会话的独立加锁，避免了全局锁的竞争。
- **ConcurrentHashMap**：使用线程安全的 `ConcurrentHashMap` 来存储每个会话的锁，保证在高并发环境下的线程安全。
- **锁释放**：每次锁定的资源都在操作完成后及时释放，避免死锁的发生。

#### **方案优点：**

- **提高并发性能**：通过每个会话分配独立锁，减少锁争用，提高系统的并发能力。
- **减少死锁风险**：通过 `Mutex` 确保每个会话的数据操作是线程安全的，并通过合理的锁释放机制减少死锁的风险。

### 6. **错误处理与重试机制**

#### 6.1 **错误处理**

在数据库操作失败时，系统会尝试进行重试。每次重试都会进行版本号的更新，确保操作最终能够成功。具体实现如下：

```kotlin
repeat(maxRetry) { attempt ->
    try {
        // 操作
    } catch (e: Exception) {
        // 异常处理，尝试重试
        releaseLocksSafely(acquired)
        throw e
    } finally {
        releaseLocksSafely(acquired)
    }
}
```

#### **技术分析与细节：**

- **异常捕获与重试**：如果数据库操作失败，系统会捕获异常并进行最大重试次数的重试，确保数据的最终一致性。
- **锁的安全释放**：每次操作完成后，不论成功还是失败，都通过 `releaseLocksSafely` 方法安全释放锁，防止死锁的发生。

#### **方案优点：**

- **高可靠性**：通过重试机制确保在高并发场景下数据最终能被正确更新。
- **安全性**：通过 `Mutex` 锁确保数据的一致性，同时确保在出现异常时锁能够被安全释放。

---

### **总结**

1. **乐观锁与重试机制**：通过乐观锁和重试机制，确保会话更新操作在并发环境下的数据一致性。
2. **分页刷新机制**：通过事件通知 ViewModel 刷新分页数据，确保 UI 显示最新的数据，避免了数据滞后的问题。
3. **并发控制与锁优化**：通过对每个会话分配独立锁，减少了锁的竞争，提高了并发性能。
4. **错误处理与重试机制**：通过重试机制确保数据库操作的可靠性，减少了因并发冲突导致的数据不一致问题。

