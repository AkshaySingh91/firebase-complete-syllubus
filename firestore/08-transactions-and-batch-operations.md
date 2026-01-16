# 8. TRANSACTIONS & BATCH OPERATIONS

## 1. What This Chapter Covers

This chapter teaches you how to use transactions and batch operations in Firestore for your Notes application. You'll learn:

- When to use transactions vs batch writes
- How transactions ensure atomic read-modify-write operations
- How to handle transaction conflicts and retries
- Transaction limitations and how to work within them
- How to implement distributed counters for high-write scenarios
- How to use batch operations for bulk updates

Transactions and batch operations are essential for maintaining data consistency when multiple documents need to be updated together. Understanding when to use each approach, their limitations, and patterns like distributed counters helps you build reliable, scalable applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 When to Use Transactions

Transactions ensure atomic operations - all reads and writes succeed together or fail together.

#### Atomic Read-Modify-Write

Transactions are needed when you must read current data before writing:

```dart
// BAD: Race condition possible
final doc = await noteRef.get();
final currentViews = doc.data()?['viewCount'] ?? 0;
await noteRef.update({'viewCount': currentViews + 1});
// Another client might have updated viewCount between read and write!

// GOOD: Atomic transaction
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final currentViews = snapshot.data()?['viewCount'] ?? 0;
  transaction.update(noteRef, {'viewCount': currentViews + 1});
});
```

**Use transactions when:**
- You need to read before writing
- Multiple documents must be updated based on current values
- You need to prevent race conditions
- Data consistency is critical

#### Concurrent Updates

Transactions handle concurrent updates automatically:

```dart
// Two users like a note simultaneously
// Transaction ensures both increments are applied correctly
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final currentLikes = snapshot.data()?['likeCount'] ?? 0;
  transaction.update(noteRef, {'likeCount': currentLikes + 1});
});
```

**How it works:**
- Transaction reads current value
- If another transaction modified the document, current transaction retries
- Retries continue until transaction succeeds or times out

#### Data Consistency

Transactions ensure related documents stay consistent:

```dart
// Update note and user's note count atomically
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Read both documents
  final noteSnapshot = await transaction.get(noteRef);
  final userSnapshot = await transaction.get(userRef);
  
  // Calculate based on current values
  final noteData = noteSnapshot.data()!;
  final userData = userSnapshot.data()!;
  
  // Update both atomically
  transaction.update(noteRef, {
    'isArchived': true,
    'archivedAt': FieldValue.serverTimestamp(),
  });
  
  transaction.update(userRef, {
    'archivedNoteCount': (userData['archivedNoteCount'] ?? 0) + 1,
    'activeNoteCount': (userData['activeNoteCount'] ?? 0) - 1,
  });
});
```

### 2.2 Transaction Lifecycle

Understanding how transactions work helps you use them effectively.

#### runTransaction<T>(transaction)

Execute a transaction with automatic retry:

```dart
final result = await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Read phase
  final snapshot = await transaction.get(noteRef);
  
  // Modify phase
  final data = snapshot.data()!;
  final newWordCount = data['wordCount'] + 100;
  
  // Write phase
  transaction.update(noteRef, {'wordCount': newWordCount});
  
  // Return result
  return newWordCount;
});

print('New word count: $result');
```

**Transaction function:**
- Must be idempotent (safe to retry)
- Can read multiple documents
- Can write to multiple documents
- Returns a value of type T

#### Read Phase

All reads happen first:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // All reads happen in this phase
  final note1 = await transaction.get(noteRef1);
  final note2 = await transaction.get(noteRef2);
  final user = await transaction.get(userRef);
  
  // Use read data to calculate writes
  // ...
});
```

**Rules:**
- All reads must complete before any writes
- Reads see a consistent snapshot of data
- No writes are visible to reads in the same transaction

#### Write Phase

All writes happen after reads:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Read phase
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  
  // Write phase (all writes together)
  transaction.update(noteRef, {'title': 'New Title'});
  transaction.update(userRef, {'noteCount': data['noteCount'] + 1});
  transaction.set(tagRef, {'name': 'work', 'count': 1});
});
```

**Rules:**
- All writes are applied atomically
- Either all writes succeed or all fail
- Writes are not visible to other transactions until commit

#### Commit Phase

Transaction commits all writes together:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Transaction automatically commits after function completes
  // If any document was modified by another transaction,
  // this transaction retries automatically
});
```

**Automatic retry:**
- If documents were modified during transaction, it retries
- Retries up to a limit (usually 5 attempts)
- Fails if retries exhausted or timeout reached

#### Retry on Conflict

Transactions automatically retry on conflicts:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final currentViews = snapshot.data()?['viewCount'] ?? 0;
  
  // If another transaction updated viewCount,
  // this transaction retries with new value
  transaction.update(noteRef, {'viewCount': currentViews + 1});
});
```

**Conflict scenarios:**
- Another transaction modified a document you read
- Transaction retries with fresh data
- Retries continue until success or timeout

### 2.3 Transaction Limitations

Understanding limitations helps you design effective transactions.

#### Cannot Read After Write

You cannot read a document after writing to it in the same transaction:

```dart
// BAD: Cannot read after write
await FirebaseFirestore.instance.runTransaction((transaction) async {
  transaction.update(noteRef, {'title': 'New Title'});
  final snapshot = await transaction.get(noteRef); // ERROR!
});

// GOOD: Read before write
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  // Use snapshot data
  transaction.update(noteRef, {'title': 'New Title'});
});
```

**Workaround:** Store values in variables:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  
  // Calculate new values
  final newTitle = 'Updated: ${data['title']}';
  final newWordCount = data['wordCount'] + 100;
  
  // Write using calculated values
  transaction.update(noteRef, {
    'title': newTitle,
    'wordCount': newWordCount,
  });
});
```

#### Maximum 25 MB Read per Transaction

Transactions are limited to 25 MB of read data:

```dart
// BAD: Might exceed 25 MB if collection is large
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(collectionRef); // Could be huge!
  // ...
});

// GOOD: Limit reads or use batch operations
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(
    collectionRef.limit(100) // Limit to reasonable size
  );
  // ...
});
```

**Best practice:** Keep transaction reads small. Use batch operations for large updates.

#### Maximum 5 MB Write per Transaction

Transactions are limited to 5 MB of write data:

```dart
// BAD: Large document might exceed 5 MB
await FirebaseFirestore.instance.runTransaction((transaction) async {
  transaction.set(noteRef, {
    'content': veryLargeContent, // Might exceed 5 MB
  });
});

// GOOD: Keep writes small, split large operations
// Use batch operations for bulk writes
```

#### Timeout Handling

Transactions have timeout limits:

```dart
try {
  await FirebaseFirestore.instance.runTransaction((transaction) async {
    // Transaction logic
  });
} on FirebaseException catch (e) {
  if (e.code == 'deadline-exceeded') {
    // Transaction timed out
    // Handle timeout (retry, show error, etc.)
  } else if (e.code == 'aborted') {
    // Transaction aborted (too many retries)
    // Handle abort
  }
}
```

**Timeout causes:**
- Too many retries due to conflicts
- Transaction takes too long
- Network issues

**Best practice:** Keep transactions fast. Avoid long-running operations inside transactions.

### 2.4 WriteBatch

Batch operations perform multiple writes atomically without reads. (See Chapter 5 for details, summarized here.)

#### Creating WriteBatch

Create a batch and add operations:

```dart
final batch = FirebaseFirestore.instance.batch();

batch.set(noteRef1, {'title': 'Note 1'});
batch.update(noteRef2, {'title': 'Updated'});
batch.delete(noteRef3);

await batch.commit();
```

#### Multiple set/update/delete

Mix different operation types:

```dart
final batch = FirebaseFirestore.instance.batch();

// Create
batch.set(newNoteRef, noteData);

// Update
batch.update(existingNoteRef, updates);

// Delete
batch.delete(oldNoteRef);

await batch.commit();
```

#### commit() Method

Commit executes all operations atomically:

```dart
try {
  await batch.commit();
  print('Batch committed successfully');
} on FirebaseException catch (e) {
  print('Batch failed: ${e.message}');
}
```

#### Maximum 500 Operations

Batches are limited to 500 operations:

```dart
// Split into multiple batches if > 500 operations
for (var chunk in _chunkList(operations, 500)) {
  final batch = FirebaseFirestore.instance.batch();
  for (var op in chunk) {
    batch.set(op.ref, op.data);
  }
  await batch.commit();
}
```

**Transactions vs Batches:**
- **Transactions:** Can read before writing, automatic retry on conflicts, limited to 25 MB read / 5 MB write
- **Batches:** No reads, no retries, limited to 500 operations, faster for bulk writes

### 2.5 Read-Modify-Write Pattern

The read-modify-write pattern ensures you work with current data.

#### Reading Current State

Read documents to get current values:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Read current state
  final noteSnapshot = await transaction.get(noteRef);
  final userSnapshot = await transaction.get(userRef);
  
  final noteData = noteSnapshot.data()!;
  final userData = userSnapshot.data()!;
  
  // Use current values for calculations
});
```

#### Calculating New Values

Calculate new values based on current state:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  
  // Calculate based on current values
  final currentViews = data['viewCount'] ?? 0;
  final newViews = currentViews + 1;
  
  final currentWordCount = data['wordCount'] ?? 0;
  final newWordCount = calculateWordCount(newContent);
  
  // Write calculated values
  transaction.update(noteRef, {
    'viewCount': newViews,
    'wordCount': newWordCount,
  });
});
```

#### Writing Updates

Write all updates atomically:

```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Read
  final noteSnapshot = await transaction.get(noteRef);
  final tagSnapshot = await transaction.get(tagRef);
  
  // Calculate
  final noteData = noteSnapshot.data()!;
  final tagData = tagSnapshot.data()!;
  
  // Write
  transaction.update(noteRef, {
    'tags': FieldValue.arrayUnion(['work']),
  });
  
  transaction.update(tagRef, {
    'noteCount': (tagData['noteCount'] ?? 0) + 1,
  });
});
```

#### Retry on Conflict

Transactions automatically retry on conflicts:

```dart
// Transaction automatically handles retries
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  // If noteRef was modified, transaction retries automatically
  // with fresh data
  final data = snapshot.data()!;
  transaction.update(noteRef, {'viewCount': data['viewCount'] + 1});
});
```

**Important:** Transaction function must be idempotent (safe to retry multiple times).

### 2.6 Counter Pattern

Distributed counters handle high-write scenarios by sharding.

#### Distributed Counters

Single document counters fail under high write volume:

```dart
// BAD: Single document counter (bottleneck)
// /counters/noteViews { count: 1000000 }
// Many transactions updating same document = conflicts and retries

// GOOD: Sharded counter
// /counters/noteViews/shard0 { count: 250000 }
// /counters/noteViews/shard1 { count: 250000 }
// /counters/noteViews/shard2 { count: 250000 }
// /counters/noteViews/shard3 { count: 250000 }
// Total = sum of all shards
```

**Implementation:**
```dart
Future<void> incrementViewCount(String noteId) async {
  // Choose random shard
  final shardId = Random().nextInt(10); // 10 shards
  final shardRef = FirebaseFirestore.instance
      .collection('counters')
      .doc('noteViews')
      .collection('shards')
      .doc('shard$shardId');
  
  // Increment shard (low conflict probability)
  await shardRef.update({
    'count': FieldValue.increment(1),
  });
}

Future<int> getViewCount(String noteId) async {
  // Sum all shards
  final shardsSnapshot = await FirebaseFirestore.instance
      .collection('counters')
      .doc('noteViews')
      .collection('shards')
      .get();
  
  int total = 0;
  for (var shard in shardsSnapshot.docs) {
    total += shard.data()['count'] as int;
  }
  
  return total;
}
```

#### Subcollections for Scaling

Use subcollections to organize shards:

```
/counters/noteViews
  /shards/shard0 { count: 1000 }
  /shards/shard1 { count: 1200 }
  /shards/shard2 { count: 1100 }
  ...
```

**Benefits:**
- Each shard updated independently
- Low conflict probability
- Scales to millions of writes
- Easy to add more shards

#### Sharding for High Write Volume

Determine number of shards based on write volume:

```dart
class DistributedCounter {
  final String counterName;
  final int numShards;
  
  DistributedCounter(this.counterName, {this.numShards = 10});
  
  Future<void> increment() async {
    final shardId = Random().nextInt(numShards);
    final shardRef = FirebaseFirestore.instance
        .collection('counters')
        .doc(counterName)
        .collection('shards')
        .doc('shard$shardId');
    
    await shardRef.update({
      'count': FieldValue.increment(1),
    });
  }
  
  Future<int> getCount() async {
    final shardsSnapshot = await FirebaseFirestore.instance
        .collection('counters')
        .doc(counterName)
        .collection('shards')
        .get();
    
    return shardsSnapshot.docs.fold<int>(
      0,
      (sum, shard) => sum + (shard.data()['count'] as int? ?? 0),
    );
  }
}
```

**Shard count guidelines:**
- 10 shards: Up to ~100 writes/second
- 100 shards: Up to ~1000 writes/second
- 1000 shards: Up to ~10000 writes/second

## 3. Practical Examples

### Example: Atomic Note Creation with User Update

```dart
Future<String> createNoteWithUserUpdate({
  required String userId,
  required String title,
  required String content,
}) async {
  String? noteId;
  
  await FirebaseFirestore.instance.runTransaction((transaction) async {
    // Read user document
    final userRef = FirebaseFirestore.instance
        .collection('users')
        .doc(userId);
    final userSnapshot = await transaction.get(userRef);
    
    if (!userSnapshot.exists) {
      throw Exception('User not found');
    }
    
    // Create note
    noteId = FirebaseFirestore.instance.collection('notes').doc().id;
    final noteRef = FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId);
    
    transaction.set(noteRef, {
      'title': title,
      'content': content,
      'userId': userId,
      'createdAt': FieldValue.serverTimestamp(),
      'wordCount': content.split(' ').length,
    });
    
    // Update user's note count
    final userData = userSnapshot.data()!;
    transaction.update(userRef, {
      'noteCount': (userData['noteCount'] ?? 0) + 1,
      'lastNoteCreatedAt': FieldValue.serverTimestamp(),
    });
  });
  
  return noteId!;
}
```

### Example: Atomic Tag Updates

```dart
Future<void> addTagToNote(String noteId, String tagName) async {
  await FirebaseFirestore.instance.runTransaction((transaction) async {
    // Read note and tag
    final noteRef = FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId);
    final tagRef = FirebaseFirestore.instance
        .collection('tags')
        .doc(tagName);
    
    final noteSnapshot = await transaction.get(noteRef);
    final tagSnapshot = await transaction.get(tagRef);
    
    if (!noteSnapshot.exists) {
      throw Exception('Note not found');
    }
    
    final noteData = noteSnapshot.data()!;
    final currentTags = (noteData['tags'] as List<dynamic>?) ?? [];
    
    // Check if tag already exists
    if (currentTags.contains(tagName)) {
      return; // Tag already exists
    }
    
    // Update note
    transaction.update(noteRef, {
      'tags': FieldValue.arrayUnion([tagName]),
    });
    
    // Update tag count
    if (tagSnapshot.exists) {
      final tagData = tagSnapshot.data()!;
      transaction.update(tagRef, {
        'noteCount': (tagData['noteCount'] ?? 0) + 1,
      });
    } else {
      transaction.set(tagRef, {
        'name': tagName,
        'noteCount': 1,
        'createdAt': FieldValue.serverTimestamp(),
      });
    }
  });
}
```

### Example: Incrementing View Count with Distributed Counter

```dart
class NoteViewCounter {
  static const int numShards = 10;
  
  static Future<void> increment(String noteId) async {
    final shardId = Random().nextInt(numShards);
    final shardRef = FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .collection('viewCountShards')
        .doc('shard$shardId');
    
    await shardRef.set({
      'count': FieldValue.increment(1),
    }, SetOptions(merge: true));
  }
  
  static Future<int> getCount(String noteId) async {
    final shardsSnapshot = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .collection('viewCountShards')
        .get();
    
    int total = 0;
    for (var shard in shardsSnapshot.docs) {
      total += shard.data()['count'] as int? ?? 0;
    }
    
    return total;
  }
}
```

### Example: Batch Archive Multiple Notes

```dart
Future<void> archiveNotes(List<String> noteIds, String userId) async {
  final batch = FirebaseFirestore.instance.batch();
  final userRef = FirebaseFirestore.instance
      .collection('users')
      .doc(userId);
  
  for (var noteId in noteIds) {
    final noteRef = FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId);
    
    batch.update(noteRef, {
      'isArchived': true,
      'archivedAt': FieldValue.serverTimestamp(),
    });
  }
  
  // Update user's archived count
  batch.update(userRef, {
    'archivedNoteCount': FieldValue.increment(noteIds.length),
    'activeNoteCount': FieldValue.increment(-noteIds.length),
  });
  
  await batch.commit();
}
```

### Example: Transaction with Error Handling

```dart
Future<bool> transferNoteOwnership({
  required String noteId,
  required String fromUserId,
  required String toUserId,
}) async {
  try {
    await FirebaseFirestore.instance.runTransaction((transaction) async {
      // Read note and both users
      final noteRef = FirebaseFirestore.instance
          .collection('notes')
          .doc(noteId);
      final fromUserRef = FirebaseFirestore.instance
          .collection('users')
          .doc(fromUserId);
      final toUserRef = FirebaseFirestore.instance
          .collection('users')
          .doc(toUserId);
      
      final noteSnapshot = await transaction.get(noteRef);
      final fromUserSnapshot = await transaction.get(fromUserRef);
      final toUserSnapshot = await transaction.get(toUserRef);
      
      if (!noteSnapshot.exists) {
        throw Exception('Note not found');
      }
      
      final noteData = noteSnapshot.data()!;
      if (noteData['userId'] != fromUserId) {
        throw Exception('User does not own this note');
      }
      
      // Update note ownership
      transaction.update(noteRef, {
        'userId': toUserId,
        'transferredAt': FieldValue.serverTimestamp(),
      });
      
      // Update user note counts
      final fromUserData = fromUserSnapshot.data()!;
      final toUserData = toUserSnapshot.data()!;
      
      transaction.update(fromUserRef, {
        'noteCount': (fromUserData['noteCount'] ?? 0) - 1,
      });
      
      transaction.update(toUserRef, {
        'noteCount': (toUserData['noteCount'] ?? 0) + 1,
      });
    });
    
    return true;
  } on FirebaseException catch (e) {
    if (e.code == 'deadline-exceeded') {
      print('Transaction timed out');
      return false;
    } else if (e.code == 'aborted') {
      print('Transaction aborted (too many retries)');
      return false;
    }
    rethrow;
  } catch (e) {
    print('Error: $e');
    return false;
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Transactions vs Batch operations:**
- **Transactions:** Use when you need to read before writing, handle concurrent updates, ensure consistency
- **Batches:** Use for bulk writes without reads, faster execution, no retries
- **Recommendation:** Use transactions for read-modify-write. Use batches for bulk writes without reads.

**Single counter vs Distributed counter:**
- **Single counter:** Simple, works for low write volume (< 1 write/second)
- **Distributed counter:** Complex, works for high write volume (> 10 writes/second)
- **Recommendation:** Start with single counter. Move to distributed counter when you see transaction conflicts.

**Transaction retry vs Manual retry:**
- **Automatic retry:** Built into transactions, handles conflicts automatically
- **Manual retry:** More control, but usually unnecessary
- **Recommendation:** Rely on automatic retry. Add manual retry only for specific error cases.

**FieldValue.increment vs Transaction:**
- **FieldValue.increment:** Atomic, no reads needed, works for simple increments
- **Transaction:** More flexible, can read before increment, handles complex logic
- **Recommendation:** Use `FieldValue.increment` for simple counters. Use transactions for complex increments based on current values.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Reading after writing in transaction**
```dart
// BAD: Cannot read after write
await FirebaseFirestore.instance.runTransaction((transaction) async {
  transaction.update(noteRef, {'title': 'New'});
  final snapshot = await transaction.get(noteRef); // ERROR!
});

// GOOD: Read before write
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  transaction.update(noteRef, {'title': 'New'});
});
```

**2. Not handling transaction errors**
```dart
// BAD: No error handling
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Transaction logic
});

// GOOD: Handle errors
try {
  await FirebaseFirestore.instance.runTransaction((transaction) async {
    // Transaction logic
  });
} on FirebaseException catch (e) {
  if (e.code == 'deadline-exceeded') {
    // Handle timeout
  } else if (e.code == 'aborted') {
    // Handle abort
  }
}
```

**3. Non-idempotent transaction functions**
```dart
// BAD: Not idempotent (unsafe to retry)
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  
  // This increments every retry, even if transaction already succeeded!
  transaction.update(noteRef, {
    'viewCount': data['viewCount'] + 1,
  });
  
  // Side effect that happens on every retry
  sendEmail(); // BAD: Email sent multiple times!
});

// GOOD: Idempotent (safe to retry)
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  
  // Safe to retry - reads current value each time
  transaction.update(noteRef, {
    'viewCount': data['viewCount'] + 1,
  });
  
  // No side effects in transaction
});
```

**4. Using transactions for simple operations**
```dart
// BAD: Unnecessary transaction
await FirebaseFirestore.instance.runTransaction((transaction) async {
  transaction.update(noteRef, {'title': 'New Title'});
});

// GOOD: Simple update (no read needed)
await noteRef.update({'title': 'New Title'});
```

### Performance Pitfalls

**1. Reading too much data in transaction**
```dart
// BAD: Might exceed 25 MB limit
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(
    collectionRef // Could be huge!
  );
});

// GOOD: Limit reads
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(
    collectionRef.limit(100)
  );
});
```

**2. Long-running operations in transaction**
```dart
// BAD: Long operation causes timeout
await FirebaseFirestore.instance.runTransaction((transaction) async {
  await Future.delayed(Duration(seconds: 10)); // Too long!
  // Transaction logic
});

// GOOD: Keep transactions fast
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Fast operations only
  final snapshot = await transaction.get(noteRef);
  transaction.update(noteRef, updates);
});
```

**3. Not using distributed counters for high write volume**
```dart
// BAD: Single counter (bottleneck)
// /counters/views { count: 1000000 }
// Many transactions = conflicts

// GOOD: Distributed counter
// /counters/views/shard0 { count: 100000 }
// /counters/views/shard1 { count: 100000 }
// ... (10 shards)
```

## 6. Summary & Checklist

### Summary

- **Transactions** ensure atomic read-modify-write operations with automatic retry on conflicts
- **Transaction lifecycle** includes read phase, write phase, and commit phase with automatic retries
- **Transaction limitations** include no reads after writes, 25 MB read limit, 5 MB write limit, and timeouts
- **Batch operations** perform multiple writes atomically without reads (max 500 operations)
- **Read-modify-write pattern** reads current state, calculates new values, and writes atomically
- **Distributed counters** use sharding to handle high write volume by reducing conflicts

### Checklist: You Are Ready When You Can...

- [ ] Choose between transactions and batch operations based on use case
- [ ] Use transactions for read-modify-write operations
- [ ] Understand transaction lifecycle (read phase, write phase, commit)
- [ ] Handle transaction errors (timeouts, aborts)
- [ ] Avoid reading after writing in transactions
- [ ] Keep transactions within size limits (25 MB read, 5 MB write)
- [ ] Write idempotent transaction functions (safe to retry)
- [ ] Use batch operations for bulk writes without reads
- [ ] Implement distributed counters for high-write scenarios
- [ ] Understand when to use FieldValue.increment vs transactions
- [ ] Handle transaction conflicts and retries
- [ ] Optimize transactions for performance (keep them fast)

### Verification Steps

**Test your transaction operations:**

1. **Test basic transaction:**
```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final data = snapshot.data()!;
  transaction.update(noteRef, {
    'viewCount': (data['viewCount'] ?? 0) + 1,
  });
});

// Verify update
final doc = await noteRef.get();
print('View count: ${doc.data()?['viewCount']}');
```

2. **Test transaction with multiple documents:**
```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final noteSnapshot = await transaction.get(noteRef);
  final userSnapshot = await transaction.get(userRef);
  
  transaction.update(noteRef, {'isArchived': true});
  transaction.update(userRef, {
    'archivedNoteCount': FieldValue.increment(1),
  });
});

// Verify both updated
```

3. **Test distributed counter:**
```dart
// Increment
await NoteViewCounter.increment('note123');

// Get count
final count = await NoteViewCounter.getCount('note123');
print('Total views: $count');
```

### Next Steps

Once you can use transactions and batch operations, you're ready to:
- Apply advanced data modeling patterns (Chapter 9)
- Implement security rules (Chapter 10)
- Optimize performance (Chapter 11)
