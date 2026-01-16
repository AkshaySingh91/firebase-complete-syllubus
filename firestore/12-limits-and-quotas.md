# 12. LIMITS & QUOTAS

## 1. What This Chapter Covers

This chapter teaches you about Firestore limits and quotas for your Notes application. You'll learn:

- Document size, field count, and nesting depth limits
- Collection naming and organizational limits
- Query operation limits (IN, NOT IN, arrayContainsAny)
- Write operation rate limits and batch constraints
- Read operation quotas and billing tier limits
- Delete operation constraints and large deletion considerations
- Storage limits and data retention policies
- Network request and response size limits
- Pricing model and cost factors

Understanding limits and quotas is crucial because exceeding them causes errors, performance degradation, and unexpected costs. Knowing these constraints helps you design data models, optimize queries, and plan for scale.

## 2. Core Concepts (Simple & Clear)

### 2.1 Document Limits

Each Firestore document has hard limits you must respect.

#### Size: 1 MiB (Maximum)

Each document cannot exceed 1 MiB (1,048,576 bytes):

```dart
// Check document size
final doc = await noteRef.get();
final data = doc.data()!;
final jsonString = jsonEncode(data);
final sizeInBytes = utf8.encode(jsonString).length;

if (sizeInBytes > 1024 * 1024) {
  print('Warning: Document exceeds 1 MiB limit');
}
```

**What counts toward size:**
- All field values
- Field names
- Document metadata (IDs, etc.)

**Handling large content:**
```dart
// BAD: Large content in document
{
  "content": "Very long content...", // Could exceed 1 MB
}

// GOOD: Store large content separately
{
  "contentPreview": "First 500 chars...",
  "contentRef": "/notes/note123/fullContent",
}

// Or use Cloud Storage
{
  "contentPreview": "First 500 chars...",
  "contentStoragePath": "notes/note123/content.txt",
}
```

#### Number of Fields: 20,000 (Maximum)

Each document can have at most 20,000 fields:

```dart
// BAD: Too many fields (unlikely but possible)
{
  "field1": "value1",
  "field2": "value2",
  // ... 20,001 fields
}

// GOOD: Use nested objects or subcollections
{
  "metadata": {
    "field1": "value1",
    "field2": "value2",
    // Group related fields
  }
}
```

**Note:** This limit is rarely reached in practice. Most documents have < 100 fields.

#### Depth: 40 (Maximum)

Documents can be nested up to 40 levels deep:

```dart
// BAD: Too deep (40+ levels)
{
  "level1": {
    "level2": {
      // ... 40+ levels deep
    }
  }
}

// GOOD: Keep nesting shallow (2-3 levels)
{
  "metadata": {
    "author": {
      "name": "John",
      "email": "john@example.com",
    }
  }
}
```

**Best practice:** Keep nesting to 2-3 levels for readability and query simplicity.

### 2.2 Collection Limits

Collections have fewer hard limits but naming conventions to follow.

#### No Hard Limit on Collections

You can create unlimited collections, but practical considerations apply:

```dart
// You can create many collections
/notes
/users
/tags
/sharedNotes
// ... unlimited
```

**Practical limits:**
- Storage costs for each collection
- Index costs for indexed fields
- Query performance for large collections
- Security rules complexity

#### Naming Conventions

Collection names must follow these rules:

**Valid names:**
```dart
/notes
/users
/notes_2024
/user-notes
```

**Invalid names:**
```dart
/__notes__  // Double underscores
/notes/     // Trailing slash
//notes     // Leading slash
```

**Best practices:**
- Use lowercase with underscores
- Avoid special characters
- Keep names descriptive and concise
- Use plural nouns (notes, users, tags)

### 2.3 Query Limits

Query operations have specific limits on values and operations.

#### IN/whereIn: 10 Values Maximum

The `whereIn` clause is limited to 10 values:

```dart
// BAD: More than 10 values
final manyIds = List.generate(15, (i) => 'id$i');
final snapshot = await collection('notes')
    .where(FieldPath.documentId, whereIn: manyIds)
    .get(); // ERROR: Exceeds 10 value limit

// GOOD: Split into chunks of 10
Future<List<DocumentSnapshot>> getNotesByIds(List<String> ids) async {
  final allDocs = <DocumentSnapshot>[];
  
  for (var chunk in _chunkList(ids, 10)) {
    final snapshot = await FirebaseFirestore.instance
        .collection('notes')
        .where(FieldPath.documentId, whereIn: chunk)
        .get();
    allDocs.addAll(snapshot.docs);
  }
  
  return allDocs;
}

List<List<T>> _chunkList<T>(List<T> list, int chunkSize) {
  final chunks = <List<T>>[];
  for (var i = 0; i < list.length; i += chunkSize) {
    chunks.add(list.sublist(i, i + chunkSize > list.length ? list.length : i + chunkSize));
  }
  return chunks;
}
```

#### NOT IN/whereNotIn: 10 Values Maximum

Same 10-value limit applies to `whereNotIn`:

```dart
// Limited to 10 values
.where('status', whereNotIn: ['archived', 'deleted', ...]) // Max 10
```

**Workaround:** Use `whereIn` with allowed values instead:

```dart
// Instead of whereNotIn with many values
.where('status', whereNotIn: ['archived', 'deleted', 'draft', ...]) // Limited

// Use whereIn with allowed values
.where('status', whereIn: ['published', 'shared']) // Better
```

#### arrayContainsAny: 10 Elements Maximum

The `arrayContainsAny` clause is limited to 10 values:

```dart
// BAD: More than 10 values
final manyTags = List.generate(15, (i) => 'tag$i');
final snapshot = await collection('notes')
    .where('tags', arrayContainsAny: manyTags)
    .get(); // ERROR: Exceeds 10 value limit

// GOOD: Split into chunks
for (var chunk in _chunkList(manyTags, 10)) {
  final snapshot = await collection('notes')
      .where('tags', arrayContainsAny: chunk)
      .get();
  // Process results
}
```

### 2.4 Write Limits

Write operations have rate limits to prevent abuse.

#### 1 Write per Document per Second (Sustained)

For sustained writes, limit to 1 write per document per second:

```dart
// BAD: Too many writes to same document
for (var i = 0; i < 100; i++) {
  await noteRef.update({'viewCount': i}); // Could exceed limit
}

// GOOD: Use FieldValue.increment (atomic, no limit concern)
await noteRef.update({
  'viewCount': FieldValue.increment(100),
});
```

**Workaround for high write volume:**
- Use distributed counters (sharded)
- Batch updates together
- Use Cloud Functions for background processing

#### 10,000 Writes per Document per Second (Burst)

Burst writes can reach 10,000/second, but sustained should be 1/second:

```dart
// Burst writes are acceptable for short periods
// But sustained writes should be limited to 1/second per document
```

**Best practice:** Use distributed counters for high-frequency updates (view counts, like counts, etc.).

#### Batch: 500 Operations Maximum

Batch operations are limited to 500 operations:

```dart
// BAD: Exceeds batch limit
final batch = FirebaseFirestore.instance.batch();
for (var i = 0; i < 600; i++) {
  batch.set(refs[i], data[i]); // ERROR: Exceeds 500 limit
}
await batch.commit();

// GOOD: Split into multiple batches
for (var chunk in _chunkList(operations, 500)) {
  final batch = FirebaseFirestore.instance.batch();
  for (var op in chunk) {
    batch.set(op.ref, op.data);
  }
  await batch.commit();
}
```

### 2.5 Read Limits

Read operations have quotas based on billing tier.

#### 50,000 Reads per Document per Day (Free Tier)

Free tier allows 50,000 reads per day across all documents:

**Free tier limits:**
- 50,000 document reads/day
- 20,000 document writes/day
- 20,000 document deletes/day
- 50,000 index entries/day
- 1 GB storage

**Blaze plan (pay-as-you-go):**
- No daily limits
- Pay per operation
- Scales automatically

**Cost optimization:**
- Use caching to reduce reads
- Limit query results with `limit()`
- Use pagination for large lists
- Monitor read usage in Firebase Console

#### Various Limits Based on Billing Plan

**Free tier (Spark):**
- 50K reads/day
- 20K writes/day
- 20K deletes/day
- 1 GB storage

**Blaze plan (pay-as-you-go):**
- No daily limits
- $0.06 per 100K reads
- $0.18 per 100K writes
- $0.02 per 100K deletes
- $0.18/GB/month storage

**Monitoring usage:**
```dart
// Check usage in Firebase Console
// Firestore → Usage tab
// Monitor reads, writes, deletes, storage
```

### 2.6 Delete Limits

Delete operations have constraints for large deletions.

#### Deletion Queue Processing

Large deletions are processed asynchronously:

```dart
// Deleting many documents
final batch = FirebaseFirestore.instance.batch();
for (var doc in docsToDelete) {
  batch.delete(doc.reference);
}
await batch.commit();

// Deletion is processed asynchronously
// Documents may still appear in queries temporarily
```

**Best practice:** Delete in batches and handle asynchronously.

#### Large Deletion Considerations

For very large deletions (> 10,000 documents):

```dart
// Delete in smaller batches
Future<void> deleteManyDocuments(List<DocumentReference> refs) async {
  for (var chunk in _chunkList(refs, 500)) {
    final batch = FirebaseFirestore.instance.batch();
    for (var ref in chunk) {
      batch.delete(ref);
    }
    await batch.commit();
    
    // Optional: Add delay between batches
    await Future.delayed(Duration(milliseconds: 100));
  }
}
```

**Note:** Deleting subcollections requires deleting documents individually. Firestore doesn't cascade delete subcollections.

### 2.7 Storage Limits

Storage has limits on total size and per-document storage.

#### Total Storage Size

**Free tier:** 1 GB total storage

**Blaze plan:** Unlimited (pay $0.18/GB/month)

**Monitor storage:**
```dart
// Check in Firebase Console
// Firestore → Usage → Storage
```

#### Per-Document Storage

Each document limited to 1 MiB (covered in 2.1).

#### Data Retention

No automatic data deletion. You must manage data retention:

```dart
// Archive or delete old data
Future<void> archiveOldNotes() async {
  final sixMonthsAgo = DateTime.now().subtract(Duration(days: 180));
  final oldNotes = await FirebaseFirestore.instance
      .collection('notes')
      .where('createdAt', isLessThan: Timestamp.fromDate(sixMonthsAgo))
      .where('isArchived', isEqualTo: false)
      .limit(500)
      .get();
  
  final batch = FirebaseFirestore.instance.batch();
  for (var doc in oldNotes.docs) {
    batch.update(doc.reference, {
      'isArchived': true,
      'archivedAt': FieldValue.serverTimestamp(),
    });
  }
  await batch.commit();
}
```

**Best practice:** Implement data retention policies to manage storage costs.

### 2.8 Network Limits

Network requests and responses have size limits.

#### Request Size Limits

**Single write request:** 5 MB maximum

```dart
// BAD: Large document (> 5 MB) in single write
await noteRef.set({
  'content': veryLargeContent, // Could exceed 5 MB
});

// GOOD: Split large content
// Store in Cloud Storage or split into chunks
```

#### Response Size Limits

**Query response:** 10 MB maximum

```dart
// BAD: Query returns too much data
final snapshot = await collection('notes').get(); // Could exceed 10 MB

// GOOD: Limit results
final snapshot = await collection('notes')
    .limit(100) // Reasonable limit
    .get();
```

#### Connection Limits

No hard limit on concurrent connections, but practical limits apply:

- Too many concurrent connections can slow performance
- Use connection pooling when possible
- Cancel unused subscriptions

### 2.9 Pricing Model

Understand Firestore pricing to optimize costs.

#### Reads Cost

**Blaze plan:**
- $0.06 per 100,000 document reads
- Applies to `get()`, `list()`, and real-time listeners

**Cost optimization:**
- Cache frequently accessed data
- Use pagination to reduce reads
- Limit query results
- Use offline persistence

#### Writes Cost

**Blaze plan:**
- $0.18 per 100,000 document writes
- Includes `create()`, `update()`, and `set()`

**Cost optimization:**
- Batch multiple writes together
- Only update changed fields
- Use `FieldValue` operations when possible

#### Deletes Cost

**Blaze plan:**
- $0.02 per 100,000 document deletes
- Cheapest operation

**Cost optimization:**
- Batch deletes together
- Archive instead of delete when possible

#### Storage Cost

**Blaze plan:**
- $0.18 per GB per month
- Based on total document size

**Cost optimization:**
- Keep documents small (< 100 KB)
- Archive or delete old data
- Remove unused fields
- Use Cloud Storage for large files

#### Network Bandwidth

**Blaze plan:**
- Egress: $0.12 per GB (first 10 GB free)
- Ingress: Free

**Cost optimization:**
- Limit query results
- Use compression when possible
- Cache data to reduce network usage

## 3. Practical Examples

### Example: Handling whereIn Limit

```dart
Future<List<Note>> getNotesByIds(List<String> noteIds) async {
  final allNotes = <Note>[];
  
  // Split into chunks of 10 (whereIn limit)
  for (var chunk in _chunkList(noteIds, 10)) {
    final snapshot = await FirebaseFirestore.instance
        .collection('notes')
        .where(FieldPath.documentId, whereIn: chunk)
        .get();
    
    final notes = snapshot.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();
    
    allNotes.addAll(notes);
  }
  
  return allNotes;
}
```

### Example: Distributed Counter for High Write Volume

```dart
class ViewCounter {
  static const int numShards = 10;
  
  static Future<void> increment(String noteId) async {
    // Use distributed counter to avoid write rate limits
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

### Example: Checking Document Size

```dart
Future<bool> checkDocumentSize(DocumentReference ref) async {
  final doc = await ref.get();
  if (!doc.exists) return true;
  
  final data = doc.data()!;
  final jsonString = jsonEncode(data);
  final sizeInBytes = utf8.encode(jsonString).length;
  final sizeInMB = sizeInBytes / (1024 * 1024);
  
  if (sizeInMB > 1.0) {
    print('Warning: Document ${ref.id} exceeds 1 MB: ${sizeInMB.toStringAsFixed(2)} MB');
    return false;
  }
  
  return true;
}
```

### Example: Batch Delete with Rate Limiting

```dart
Future<void> deleteManyDocuments(
  List<DocumentReference> refs,
  {int batchSize = 500, Duration delay = const Duration(milliseconds: 100)}
) async {
  for (var chunk in _chunkList(refs, batchSize)) {
    final batch = FirebaseFirestore.instance.batch();
    
    for (var ref in chunk) {
      batch.delete(ref);
    }
    
    await batch.commit();
    
    // Rate limiting between batches
    if (chunk.length == batchSize) {
      await Future.delayed(delay);
    }
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Distributed counter vs Single counter:**
- **Single counter:** Simple, but limited to 1 write/second
- **Distributed counter:** Complex, but handles high write volume
- **Recommendation:** Use single counter for < 1 write/second. Use distributed counter for high-frequency updates.

**Archive vs Delete:**
- **Archive:** Keep data, higher storage cost, can restore
- **Delete:** Remove data, lower storage cost, cannot restore
- **Recommendation:** Archive for important data. Delete for temporary/cache data.

**Chunking whereIn vs Multiple queries:**
- **Chunking:** Handles > 10 values, more code complexity
- **Multiple queries:** Simpler, but more network calls
- **Recommendation:** Use chunking helper function for > 10 values.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Exceeding whereIn limit**
```dart
// BAD: More than 10 values
.where(FieldPath.documentId, whereIn: manyIds) // Fails if > 10

// GOOD: Chunk into groups of 10
for (var chunk in _chunkList(manyIds, 10)) {
  .where(FieldPath.documentId, whereIn: chunk)
}
```

**2. Exceeding document size limit**
```dart
// BAD: Large document
{
  "content": veryLargeContent, // Could exceed 1 MB
}

// GOOD: Split large content
{
  "contentPreview": "...",
  "contentRef": "/notes/note123/fullContent",
}
```

**3. Exceeding batch limit**
```dart
// BAD: More than 500 operations
final batch = FirebaseFirestore.instance.batch();
for (var i = 0; i < 600; i++) {
  batch.set(refs[i], data[i]); // Fails
}

// GOOD: Split into batches
for (var chunk in _chunkList(operations, 500)) {
  // Create new batch for each chunk
}
```

**4. Too many writes to same document**
```dart
// BAD: High-frequency updates
for (var i = 0; i < 100; i++) {
  await noteRef.update({'viewCount': i}); // Rate limited
}

// GOOD: Use FieldValue.increment or distributed counter
await noteRef.update({'viewCount': FieldValue.increment(100)});
```

## 6. Summary & Checklist

### Summary

- **Document limits:** 1 MiB size, 20,000 fields, 40 levels depth
- **Collection limits:** No hard limit, but follow naming conventions
- **Query limits:** whereIn/whereNotIn/arrayContainsAny limited to 10 values
- **Write limits:** 1 write/second sustained, 10,000/second burst, 500 operations per batch
- **Read limits:** 50K/day (free tier), unlimited (Blaze plan with pay-per-use)
- **Delete limits:** Processed asynchronously, delete in batches for large operations
- **Storage limits:** 1 GB (free tier), unlimited with Blaze plan ($0.18/GB/month)
- **Network limits:** 5 MB request, 10 MB response
- **Pricing:** Pay per read/write/delete/storage based on usage

### Checklist: You Are Ready When You Can...

- [ ] Understand document size limit (1 MiB) and how to handle large content
- [ ] Know field count (20,000) and nesting depth (40) limits
- [ ] Handle whereIn/whereNotIn limit (10 values) with chunking
- [ ] Respect write rate limits (1/second sustained) and use distributed counters when needed
- [ ] Understand batch operation limit (500) and split large batches
- [ ] Know read quotas for free tier vs Blaze plan
- [ ] Handle large deletions in batches
- [ ] Monitor storage usage and implement retention policies
- [ ] Understand network size limits (5 MB request, 10 MB response)
- [ ] Calculate costs based on pricing model
- [ ] Optimize operations to reduce costs
- [ ] Monitor usage in Firebase Console

### Verification Steps

**Test your limits awareness:**

1. **Test document size check:**
```dart
final doc = await noteRef.get();
final data = doc.data()!;
final sizeInBytes = utf8.encode(jsonEncode(data)).length;
print('Document size: ${sizeInBytes / 1024} KB');
assert(sizeInBytes < 1024 * 1024); // < 1 MB
```

2. **Test whereIn chunking:**
```dart
final manyIds = List.generate(25, (i) => 'id$i');
final notes = await getNotesByIds(manyIds); // Should handle chunking
assert(notes.length == 25);
```

3. **Test batch limit:**
```dart
final operations = List.generate(600, (i) => Operation(...));
// Should split into 2 batches (500 + 100)
await batchWriteOperations(operations);
```

### Next Steps

Once you understand limits and quotas, you're ready to:
- Handle errors effectively (Chapter 13)
- Optimize for production (Chapter 16)
- Monitor usage and costs (Chapter 16)
