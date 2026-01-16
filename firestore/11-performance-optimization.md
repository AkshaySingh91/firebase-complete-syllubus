# 11. PERFORMANCE OPTIMIZATION

## 1. What This Chapter Covers

This chapter teaches you how to optimize Firestore performance and costs for your Notes application. You'll learn:

- How to optimize query performance with proper indexes and scoping
- How to reduce read operations through caching and limiting
- How to optimize write operations with batching and smart updates
- How to manage indexes effectively
- How to handle connections and memory efficiently
- How to minimize costs through efficient data access patterns

Performance optimization is crucial because it directly impacts user experience, app responsiveness, and billing costs. Poor performance leads to slow apps, frustrated users, and high bills. Understanding query optimization, caching strategies, and cost-effective patterns helps you build fast, efficient applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Query Performance

Optimize queries to reduce latency and costs.

#### Composite Indexes

Create indexes for compound queries:

```dart
// Query requires composite index
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .where('isPinned', isEqualTo: true)
    .orderBy('createdAt', descending: true);

// Firestore will prompt you to create index
// Or create manually in Console → Firestore → Indexes
```

**Index creation:**
```json
{
  "collectionGroup": "notes",
  "queryScope": "COLLECTION",
  "fields": [
    {"fieldPath": "userId", "order": "ASCENDING"},
    {"fieldPath": "isPinned", "order": "ASCENDING"},
    {"fieldPath": "createdAt", "order": "DESCENDING"}
  ]
}
```

**Best practices:**
- Create indexes as you develop (Firestore provides links)
- Monitor index usage in Console
- Remove unused indexes to save storage
- Indexes are created automatically for single-field queries

#### Query Scope (Collection vs Collection Group)

Choose appropriate query scope:

**Collection query (faster):**
```dart
// Query single collection
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .get();
```

**Collection group query (slower, broader):**
```dart
// Query across all subcollections with same name
final query = FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

**Recommendation:** Use collection queries when possible. Use collection group queries only when you need to search across multiple collections.

#### Avoid Client-Side Filtering

Filter on server, not client:

```dart
// BAD: Read all, filter client-side
final allNotes = await FirebaseFirestore.instance
    .collection('notes')
    .get();

final workNotes = allNotes.docs
    .where((doc) => (doc.data()['tags'] as List).contains('work'))
    .toList();
// Problem: Reads all documents, filters after

// GOOD: Filter server-side
final workNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContains: 'work')
    .get();
// Benefit: Only reads matching documents
```

**When client-side filtering is acceptable:**
- Complex logic that can't be expressed in queries
- Small result sets (< 10 documents)
- One-time operations (not in loops)

### 2.2 Read Optimization

Reduce read operations to improve performance and reduce costs.

#### Select Specific Fields

**Note:** Firestore doesn't support selecting specific fields. You always read entire documents. However, you can structure data to minimize document size:

```dart
// BAD: Large document with everything
{
  "title": "Note",
  "content": "Very long content...",
  "fullContent": "Even longer...",
  "history": [...1000 items...],
  "metadata": {...large object...}
}

// GOOD: Split into smaller documents
{
  "title": "Note",
  "contentPreview": "First 500 chars...",
}

// Separate documents for large data
/notes/{noteId}/fullContent/{contentId}
/notes/{noteId}/history/{historyId}
```

#### Limit Result Sets

Always use `limit()` to reduce reads:

```dart
// BAD: Reads all documents
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .get();

// GOOD: Limit results
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .limit(20)
    .get();
```

**Pagination:**
```dart
// Load pages of 20 documents
var query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .orderBy('createdAt', descending: true)
    .limit(20);

var snapshot = await query.get();

// Next page
if (snapshot.docs.isNotEmpty) {
  var nextQuery = query.startAfter([snapshot.docs.last]);
  var nextSnapshot = await nextQuery.get();
}
```

#### Cache Frequently Accessed Data

Cache data to reduce reads:

```dart
class NotesCache {
  static final Map<String, DocumentSnapshot> _cache = {};
  static DateTime? _lastRefresh;
  static const cacheDuration = Duration(minutes: 5);
  
  static Future<DocumentSnapshot> getNote(String noteId) async {
    // Check cache
    if (_cache.containsKey(noteId)) {
      final cached = _cache[noteId]!;
      final age = DateTime.now().difference(
        (cached.metadata.serverTimestamp as Timestamp).toDate()
      );
      
      if (age < cacheDuration) {
        return cached;
      }
    }
    
    // Fetch from Firestore
    final doc = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .get();
    
    // Update cache
    _cache[noteId] = doc;
    return doc;
  }
  
  static void invalidate(String noteId) {
    _cache.remove(noteId);
  }
}
```

**Firestore offline cache:**
```dart
// Firestore automatically caches data
// Enable persistence (usually enabled by default)
await FirebaseFirestore.instance.enablePersistence();

// Reads from cache when offline
final doc = await noteRef.get(); // Uses cache if offline
```

### 2.3 Write Optimization

Optimize write operations for performance.

#### Batch Writes

Use batch writes for multiple operations:

```dart
// BAD: Multiple individual writes
for (var note in notes) {
  await noteRef.set(note.data); // N network calls
}

// GOOD: Single batch write
final batch = FirebaseFirestore.instance.batch();
for (var note in notes) {
  batch.set(note.ref, note.data);
}
await batch.commit(); // 1 network call
```

**Benefits:**
- Single network round trip
- Atomic operation
- Faster execution
- Lower cost (same number of writes, but fewer network calls)

#### Avoid Overwriting Unchanged Documents

Only update when data actually changes:

```dart
// BAD: Always updates, even if unchanged
await noteRef.update({
  'title': currentTitle,
  'content': currentContent,
});

// GOOD: Check if changed before updating
final currentDoc = await noteRef.get();
final currentData = currentDoc.data()!;

if (currentData['title'] != newTitle ||
    currentData['content'] != newContent) {
  await noteRef.update({
    'title': newTitle,
    'content': newContent,
    'updatedAt': FieldValue.serverTimestamp(),
  });
}
```

**Client-side optimization:**
```dart
// Track changes locally
class NoteEditor {
  Map<String, dynamic> _originalData = {};
  Map<String, dynamic> _changes = {};
  
  void updateField(String field, dynamic value) {
    if (_originalData[field] != value) {
      _changes[field] = value;
    } else {
      _changes.remove(field);
    }
  }
  
  Future<void> save() async {
    if (_changes.isNotEmpty) {
      await noteRef.update({
        ..._changes,
        'updatedAt': FieldValue.serverTimestamp(),
      });
    }
  }
}
```

#### Use Appropriate Write Types

Choose the right write method:

**set() with merge:**
```dart
// Use when document might not exist
await noteRef.set({
  'title': 'New Title',
}, SetOptions(merge: true));
```

**update():**
```dart
// Use when document definitely exists
await noteRef.update({
  'title': 'Updated Title',
});
```

**FieldValue operations:**
```dart
// Use for atomic operations
await noteRef.update({
  'viewCount': FieldValue.increment(1), // Atomic, no read needed
  'tags': FieldValue.arrayUnion(['work']), // Atomic
});
```

**Recommendation:** Use `FieldValue.increment()`, `arrayUnion()`, etc. when possible. They're atomic and don't require reading the document first.

### 2.4 Index Management

Manage indexes effectively for optimal performance.

#### Single-Field Indexes

Created automatically for single-field queries:

```dart
// Automatic index created
.where('userId', isEqualTo: userId)
.orderBy('createdAt')
```

**No manual creation needed** for single-field indexes.

#### Composite Indexes

Required for compound queries:

```dart
// Requires composite index
.where('userId', isEqualTo: userId)
.where('isPinned', isEqualTo: true)
.orderBy('createdAt')
```

**Creating indexes:**
1. Firestore provides link in error message
2. Click link to create automatically
3. Or create manually in Console → Firestore → Indexes

**Index definition:**
```json
{
  "collectionGroup": "notes",
  "queryScope": "COLLECTION",
  "fields": [
    {"fieldPath": "userId", "order": "ASCENDING"},
    {"fieldPath": "isPinned", "order": "ASCENDING"},
    {"fieldPath": "createdAt", "order": "DESCENDING"}
  ]
}
```

#### Index Exclusion

Exclude fields from automatic indexing:

```dart
// In firestore.indexes.json
{
  "indexes": [],
  "fieldOverrides": [
    {
      "collectionGroup": "notes",
      "fieldPath": "content",
      "indexes": [
        {"order": "ASCENDING", "queryScope": "COLLECTION"},
        {"order": "DESCENDING", "queryScope": "COLLECTION"},
        {"arrayConfig": "CONTAINS", "queryScope": "COLLECTION"}
      ]
    }
  ]
}
```

**Use when:**
- Field is large (text content)
- Field is rarely queried
- You want to reduce index storage

#### Index Monitoring

Monitor index usage:

**Firebase Console:**
1. Go to Firestore → Indexes
2. View index usage statistics
3. Identify unused indexes
4. Remove unused indexes

**Best practices:**
- Review indexes quarterly
- Remove indexes not used in 30+ days
- Monitor index build time
- Keep index count reasonable (< 100 per collection)

### 2.5 Connection Management

Manage Firestore connections efficiently.

#### Stream Lifecycle

Properly manage stream subscriptions:

```dart
class NotesListState extends State<NotesList> {
  StreamSubscription<QuerySnapshot>? _subscription;
  
  @override
  void initState() {
    super.initState();
    
    final query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: currentUserId);
    
    _subscription = query.snapshots().listen(
      (snapshot) {
        setState(() {
          // Update state
        });
      },
      onError: (error) {
        // Handle error
      },
    );
  }
  
  @override
  void dispose() {
    _subscription?.cancel(); // Important: Cancel to free resources
    super.dispose();
  }
}
```

**Best practices:**
- Always cancel subscriptions in `dispose()`
- Use StreamBuilder for automatic lifecycle management
- Limit number of active listeners
- Share listeners via state management when possible

#### Reconnection Handling

Firestore handles reconnection automatically:

```dart
// Firestore automatically reconnects
query.snapshots().listen((snapshot) {
  // Automatically reconnects when network available
  // No manual reconnection needed
});
```

**Monitor connection state:**
```dart
query.snapshots(includeMetadataChanges: true).listen((snapshot) {
  if (snapshot.metadata.isFromCache) {
    // Offline - showing cached data
    showOfflineIndicator();
  } else {
    // Online - fresh data
    hideOfflineIndicator();
  }
});
```

#### Offline Queue

Firestore queues writes when offline:

```dart
// Write when offline
await noteRef.update({'title': 'New Title'});
// Queued automatically, syncs when online

// Check if write is pending
noteRef.snapshots(includeMetadataChanges: true).listen((snapshot) {
  if (snapshot.metadata.hasPendingWrites) {
    // Write is queued, not yet synced
    showPendingIndicator();
  }
});
```

**Best practices:**
- Show pending indicator for queued writes
- Handle write failures gracefully
- Don't rely on immediate write confirmation when offline

### 2.6 Memory Management

Manage memory efficiently for large datasets.

#### Large Result Sets

Avoid loading all data at once:

```dart
// BAD: Loads all notes into memory
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .get();

final allNotes = snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
// Problem: Could be thousands of documents in memory

// GOOD: Paginate
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .limit(20)
    .get();

final notes = snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
// Benefit: Only 20 documents in memory
```

#### Pagination for Large Data

Implement pagination:

```dart
class PaginatedNotesLoader {
  DocumentSnapshot? _lastDocument;
  final int pageSize = 20;
  final List<Note> _loadedNotes = [];
  
  Future<List<Note>> loadNextPage(String userId) async {
    var query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .orderBy('createdAt', descending: true)
        .limit(pageSize);
    
    if (_lastDocument != null) {
      query = query.startAfter([_lastDocument!]);
    }
    
    final snapshot = await query.get();
    
    if (snapshot.docs.isNotEmpty) {
      _lastDocument = snapshot.docs.last;
    }
    
    final newNotes = snapshot.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();
    
    _loadedNotes.addAll(newNotes);
    return newNotes;
  }
  
  void clear() {
    _loadedNotes.clear();
    _lastDocument = null;
  }
}
```

#### Stream Disposal

Dispose streams to free memory:

```dart
class NoteDetailState extends State<NoteDetail> {
  StreamSubscription<DocumentSnapshot>? _noteSubscription;
  StreamSubscription<QuerySnapshot>? _commentsSubscription;
  
  @override
  void initState() {
    super.initState();
    
    _noteSubscription = noteRef.snapshots().listen(...);
    _commentsSubscription = commentsRef.snapshots().listen(...);
  }
  
  @override
  void dispose() {
    // Cancel all subscriptions
    _noteSubscription?.cancel();
    _commentsSubscription?.cancel();
    super.dispose();
  }
}
```

**Memory leak prevention:**
- Always cancel subscriptions
- Use StreamBuilder for automatic management
- Limit number of active listeners
- Clear large data structures when not needed

### 2.7 Cost Optimization

Minimize Firestore costs through efficient patterns.

#### Minimize Reads/Writes

Reduce number of operations:

```dart
// BAD: Multiple reads
for (var noteId in noteIds) {
  final doc = await noteRef.doc(noteId).get(); // N reads
}

// GOOD: Single query with whereIn
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where(FieldPath.documentId, whereIn: noteIds.take(10).toList())
    .get(); // 1 read (but limited to 10 IDs)
```

**Batch operations:**
```dart
// BAD: Individual writes
for (var note in notes) {
  await noteRef.set(note.data); // N writes, N network calls
}

// GOOD: Batch write
final batch = FirebaseFirestore.instance.batch();
for (var note in notes) {
  batch.set(note.ref, note.data);
}
await batch.commit(); // N writes, 1 network call
```

#### Efficient Queries

Design queries to minimize reads:

```dart
// BAD: Reads all, filters client-side
final allNotes = await collection('notes').get();
final workNotes = allNotes.docs
    .where((doc) => doc.data()['tags'].contains('work'))
    .toList();

// GOOD: Filter server-side
final workNotes = await collection('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

**Query optimization:**
- Always use `where()` filters
- Use `limit()` to reduce results
- Use `orderBy()` only when needed
- Create indexes for compound queries

#### Data Retention Policies

Archive or delete old data:

```dart
// Archive old notes
Future<void> archiveOldNotes() async {
  final sixMonthsAgo = DateTime.now().subtract(Duration(days: 180));
  final oldNotes = await FirebaseFirestore.instance
      .collection('notes')
      .where('createdAt', isLessThan: Timestamp.fromDate(sixMonthsAgo))
      .where('isArchived', isEqualTo: false)
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

**Or delete old data:**
```dart
// Delete notes older than 1 year
Future<void> deleteOldNotes() async {
  final oneYearAgo = DateTime.now().subtract(Duration(days: 365));
  final oldNotes = await FirebaseFirestore.instance
      .collection('notes')
      .where('createdAt', isLessThan: Timestamp.fromDate(oneYearAgo))
      .where('isArchived', isEqualTo: true)
      .limit(500) // Process in batches
      .get();
  
  final batch = FirebaseFirestore.instance.batch();
  for (var doc in oldNotes.docs) {
    batch.delete(doc.reference);
  }
  await batch.commit();
}
```

#### Caching Strategies

Cache data to reduce reads:

**Client-side cache:**
```dart
class NotesCache {
  static final Map<String, DocumentSnapshot> _cache = {};
  static const cacheDuration = Duration(minutes: 5);
  
  static Future<DocumentSnapshot> getNote(String noteId) async {
    // Check cache first
    if (_cache.containsKey(noteId)) {
      return _cache[noteId]!;
    }
    
    // Fetch and cache
    final doc = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .get();
    
    _cache[noteId] = doc;
    return doc;
  }
}
```

**Firestore offline cache:**
```dart
// Enable persistence (default)
await FirebaseFirestore.instance.enablePersistence();

// Reads from cache when available
final doc = await noteRef.get(); // Uses cache if offline
```

## 3. Practical Examples

### Example: Optimized Notes List with Pagination

```dart
class OptimizedNotesList extends StatefulWidget {
  @override
  State<OptimizedNotesList> createState() => _OptimizedNotesListState();
}

class _OptimizedNotesListState extends State<OptimizedNotesList> {
  final List<Note> _notes = [];
  DocumentSnapshot? _lastDocument;
  bool _isLoading = false;
  bool _hasMore = true;
  static const int pageSize = 20;
  
  @override
  void initState() {
    super.initState();
    _loadMore();
  }
  
  Future<void> _loadMore() async {
    if (_isLoading || !_hasMore) return;
    
    setState(() => _isLoading = true);
    
    var query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: currentUserId)
        .where('isArchived', isEqualTo: false)
        .orderBy('createdAt', descending: true)
        .limit(pageSize);
    
    if (_lastDocument != null) {
      query = query.startAfter([_lastDocument!]);
    }
    
    final snapshot = await query.get();
    
    if (snapshot.docs.isEmpty) {
      setState(() {
        _hasMore = false;
        _isLoading = false;
      });
      return;
    }
    
    final newNotes = snapshot.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();
    
    setState(() {
      _notes.addAll(newNotes);
      _lastDocument = snapshot.docs.last;
      _isLoading = false;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _notes.length + (_hasMore ? 1 : 0),
      itemBuilder: (context, index) {
        if (index == _notes.length) {
          // Load more trigger
          _loadMore();
          return Center(child: CircularProgressIndicator());
        }
        return NoteTile(note: _notes[index]);
      },
    );
  }
}
```

### Example: Batch Update with Change Detection

```dart
Future<void> updateNotesIfChanged(
  List<MapEntry<String, Map<String, dynamic>>> updates,
) async {
  final batch = FirebaseFirestore.instance.batch();
  int actualUpdates = 0;
  
  // Read current documents
  final refs = updates.map((e) => 
    FirebaseFirestore.instance.collection('notes').doc(e.key)
  ).toList();
  
  final snapshots = await Future.wait(refs.map((ref) => ref.get()));
  
  // Only update if changed
  for (int i = 0; i < updates.length; i++) {
    final noteId = updates[i].key;
    final newData = updates[i].value;
    final currentDoc = snapshots[i];
    
    if (!currentDoc.exists) continue;
    
    final currentData = currentDoc.data()!;
    bool hasChanges = false;
    final changes = <String, dynamic>{};
    
    for (var key in newData.keys) {
      if (currentData[key] != newData[key]) {
        changes[key] = newData[key];
        hasChanges = true;
      }
    }
    
    if (hasChanges) {
      batch.update(currentDoc.reference, {
        ...changes,
        'updatedAt': FieldValue.serverTimestamp(),
      });
      actualUpdates++;
    }
  }
  
  if (actualUpdates > 0) {
    await batch.commit();
    print('Updated $actualUpdates of ${updates.length} notes');
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Pagination vs Load all:**
- **Pagination:** Better for large datasets, lower memory usage, faster initial load
- **Load all:** Simpler code, but poor performance for large datasets
- **Recommendation:** Always paginate for lists. Load all only for small, fixed datasets (< 20 items).

**Client cache vs Firestore cache:**
- **Client cache:** More control, custom TTL, app-specific logic
- **Firestore cache:** Automatic, works offline, no extra code
- **Recommendation:** Use Firestore cache for most cases. Add client cache for frequently accessed, rarely changing data.

**Batch writes vs Individual writes:**
- **Batch writes:** Faster, atomic, fewer network calls
- **Individual writes:** Simpler code, but slower and more expensive
- **Recommendation:** Use batch writes for multiple operations. Use individual writes for single operations.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Not using limit() on queries**
```dart
// BAD: Reads all documents
final snapshot = await collection('notes').get();

// GOOD: Always limit
final snapshot = await collection('notes').limit(20).get();
```

**2. Client-side filtering instead of server-side**
```dart
// BAD: Reads all, filters client
final all = await collection('notes').get();
final filtered = all.docs.where(...).toList();

// GOOD: Filter server-side
final filtered = await collection('notes').where(...).get();
```

**3. Not canceling stream subscriptions**
```dart
// BAD: Memory leak
query.snapshots().listen(...); // Never canceled

// GOOD: Cancel in dispose
_subscription?.cancel();
```

**4. Overwriting unchanged documents**
```dart
// BAD: Always updates
await noteRef.update({'title': currentTitle});

// GOOD: Check if changed
if (currentTitle != newTitle) {
  await noteRef.update({'title': newTitle});
}
```

### Performance Pitfalls

**1. Too many active listeners**
```dart
// BAD: Multiple listeners for same data
StreamBuilder(stream: query.snapshots(), ...),
StreamBuilder(stream: query.snapshots(), ...),
StreamBuilder(stream: query.snapshots(), ...),

// GOOD: Single listener, share via state management
```

**2. Not using indexes for compound queries**
```dart
// BAD: Query fails or is slow
.where('userId', isEqualTo: userId)
.where('isPinned', isEqualTo: true)
.orderBy('createdAt') // Needs index

// GOOD: Create composite index
```

**3. Loading large datasets into memory**
```dart
// BAD: All notes in memory
final all = await collection('notes').get();
final notes = all.docs.map(...).toList();

// GOOD: Paginate
final page = await collection('notes').limit(20).get();
```

## 6. Summary & Checklist

### Summary

- **Query performance** is optimized with composite indexes, proper scoping, and server-side filtering
- **Read optimization** reduces operations through limiting, caching, and efficient queries
- **Write optimization** uses batching, change detection, and appropriate write types
- **Index management** ensures queries are fast and indexes are maintained
- **Connection management** properly handles streams and reconnection
- **Memory management** uses pagination and proper stream disposal
- **Cost optimization** minimizes reads/writes through efficient patterns and caching

### Checklist: You Are Ready When You Can...

- [ ] Create composite indexes for compound queries
- [ ] Use `limit()` on all queries to reduce reads
- [ ] Filter queries server-side instead of client-side
- [ ] Implement pagination for large datasets
- [ ] Use batch writes for multiple operations
- [ ] Check for changes before updating documents
- [ ] Cancel stream subscriptions in `dispose()`
- [ ] Monitor and manage indexes effectively
- [ ] Use Firestore offline cache
- [ ] Implement data retention policies
- [ ] Optimize queries to minimize reads
- [ ] Handle connection state and offline scenarios

### Verification Steps

**Test your optimizations:**

1. **Test query performance:**
```dart
final stopwatch = Stopwatch()..start();
final snapshot = await query.limit(20).get();
stopwatch.stop();
print('Query took: ${stopwatch.elapsedMilliseconds}ms');
```

2. **Test pagination:**
```dart
// Load first page
var page1 = await query.limit(20).get();
assert(page1.size == 20);

// Load second page
var page2 = await query.startAfter([page1.docs.last]).limit(20).get();
assert(page2.size <= 20);
```

3. **Test batch writes:**
```dart
final batch = FirebaseFirestore.instance.batch();
for (var i = 0; i < 10; i++) {
  batch.set(refs[i], data[i]);
}
await batch.commit(); // Should be faster than 10 individual writes
```

### Next Steps

Once you can optimize performance, you're ready to:
- Understand limits and quotas (Chapter 12)
- Handle errors effectively (Chapter 13)
- Deploy to production (Chapter 16)
