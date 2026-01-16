# 4. READING DATA

## 1. What This Chapter Covers

This chapter teaches you how to read data from Firestore in your Notes application. You'll learn:

- How to fetch single documents and collections
- How to query and filter data efficiently
- How to paginate results for large datasets
- When to use one-time reads vs real-time listeners
- How to work with QuerySnapshot and DocumentSnapshot objects

Reading data efficiently is crucial because every read operation costs money and impacts app performance. Understanding query methods, pagination, and snapshot properties helps you build fast, cost-effective applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Get Document

Reading a single document is the simplest Firestore operation.

#### get() Method

The `get()` method fetches a document once. It returns a `DocumentSnapshot`:

```dart
final docRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

final docSnapshot = await docRef.get();
```

#### DocumentSnapshot

A `DocumentSnapshot` contains the document data and metadata:

```dart
if (docSnapshot.exists) {
  final data = docSnapshot.data(); // Map<String, dynamic>?
  print('Title: ${data?['title']}');
} else {
  print('Document does not exist');
}
```

**Key properties:**
- `exists` - Boolean indicating if document exists
- `id` - Document ID string
- `data()` - Returns all fields as Map, or null if doesn't exist
- `reference` - DocumentReference to this document
- `get(fieldPath)` - Get specific field value

#### Handling Null Documents

Documents might not exist. Always check before accessing data:

```dart
final snapshot = await docRef.get();

if (snapshot.exists) {
  final data = snapshot.data()!; // Safe to use ! after exists check
  final title = data['title'] as String;
} else {
  // Handle missing document
  print('Note not found');
}
```

**Alternative with null safety:**
```dart
final data = snapshot.data();
if (data != null) {
  final title = data['title'] as String? ?? 'Untitled';
}
```

#### From Reference

You can get a document directly from a reference:

```dart
// If you have a DocumentReference
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

final snapshot = await noteRef.get();
```

**Common pattern:** Store references in other documents:
```dart
// In user document
{
  "favoriteNoteRef": "/notes/note123"
}

// Later, get the note
final userDoc = await userRef.get();
final favoriteNotePath = userDoc.data()?['favoriteNoteRef'];
if (favoriteNotePath != null) {
  final noteRef = FirebaseFirestore.instance.doc(favoriteNotePath);
  final noteDoc = await noteRef.get();
}
```

### 2.2 Get Collection

Reading multiple documents from a collection.

#### get() on Collection Reference

Fetch all documents in a collection:

```dart
final collectionRef = FirebaseFirestore.instance.collection('notes');
final querySnapshot = await collectionRef.get();
```

**Warning:** This reads ALL documents. For large collections, use queries with limits.

#### QuerySnapshot

A `QuerySnapshot` contains multiple documents:

```dart
final querySnapshot = await collectionRef.get();

print('Total documents: ${querySnapshot.size}');
print('Is empty: ${querySnapshot.empty}');

// Access documents
for (var doc in querySnapshot.docs) {
  print('Document ID: ${doc.id}');
  print('Data: ${doc.data()}');
}
```

**Key properties:**
- `docs` - List of DocumentSnapshot objects
- `size` - Number of documents
- `empty` - Boolean, true if no documents
- `metadata` - Query metadata (hasPendingWrites, isFromCache)
- `forEach()` - Iterator over documents

#### DocumentSnapshot List

Access individual documents from QuerySnapshot:

```dart
final querySnapshot = await collectionRef.get();

// Method 1: Iterate with for loop
for (var docSnapshot in querySnapshot.docs) {
  final data = docSnapshot.data();
  final title = data['title'] as String;
}

// Method 2: Convert to list
final documents = querySnapshot.docs.toList();

// Method 3: Use forEach
querySnapshot.docs.forEach((doc) {
  print('${doc.id}: ${doc.data()}');
});
```

### 2.3 One-time Reads vs Real-time

Firestore supports two read patterns.

#### get() for One-time Read

Use `get()` when you need data once:

```dart
// Load note details screen
final noteDoc = await noteRef.get();
final noteData = noteDoc.data();
```

**Use cases:**
- Loading initial screen data
- One-time data fetches
- Background operations
- When you don't need updates

#### onSnapshot for Real-time Updates

Use `onSnapshot()` to listen for changes:

```dart
final unsubscribe = noteRef.snapshots().listen((snapshot) {
  if (snapshot.exists) {
    final data = snapshot.data();
    // Update UI with new data
  }
});

// Don't forget to cancel when done
unsubscribe.cancel();
```

**Use cases:**
- Live data that changes (collaborative notes)
- Real-time dashboards
- Chat applications
- When multiple users edit same data

**Important:** Always cancel listeners to prevent memory leaks and unnecessary reads.

### 2.4 Query Methods

Query methods filter and sort documents.

#### where()

Filter documents by field values:

```dart
// Equal to
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'user123');

// Not equal to
.where('isArchived', isNotEqualTo: true);

// Less than / Greater than
.where('wordCount', isLessThan: 1000);
.where('createdAt', isGreaterThan: yesterday);

// Less than or equal / Greater than or equal
.where('rating', isLessThanOrEqualTo: 5);
.where('priority', isGreaterThanOrEqualTo: 1);

// Array contains
.where('tags', arrayContains: 'work');

// Array contains any
.where('tags', arrayContainsAny: ['work', 'personal']);

// In (up to 10 values)
.where('status', whereIn: ['draft', 'published']);

// Not in (up to 10 values)
.where('status', whereNotIn: ['deleted', 'archived']);
```

**Limitations:**
- `whereIn` and `whereNotIn` limited to 10 values
- Can't combine `whereIn` with `arrayContainsAny` in same query
- Need composite index for multiple where clauses (see 2.5)

#### orderBy()

Sort documents by field:

```dart
// Ascending (default)
.orderBy('createdAt');

// Descending
.orderBy('createdAt', descending: true);

// Multiple orderBy (requires index)
.orderBy('isPinned', descending: true)
.orderBy('createdAt', descending: true);
```

**Rules:**
- Must create index for multiple `orderBy()` calls
- When using `where()` with `orderBy()`, orderBy field usually needs to be different or indexed

#### limit()

Limit number of documents returned:

```dart
// First 10 documents
.limit(10);

// Last 10 documents
.limitToLast(10);
```

**Use cases:**
- Pagination
- Top N results
- Reducing read costs

#### Cursor Methods

Navigate to specific positions in query results:

```dart
// Start at specific document
.startAt([snapshot]);

// Start after specific document
.startAfter([snapshot]);

// End at specific document
.endAt([snapshot]);

// End before specific document
.endBefore([snapshot]);
```

**Common pattern for pagination:**
```dart
// First page
var query = collectionRef
    .orderBy('createdAt')
    .limit(10);

var snapshot = await query.get();
var lastDoc = snapshot.docs.last;

// Next page
var nextQuery = collectionRef
    .orderBy('createdAt')
    .startAfter([lastDoc])
    .limit(10);
```

### 2.5 Compound Queries

Combining multiple query conditions.

#### Multiple where Conditions

You can chain multiple `where()` calls:

```dart
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'user123')
    .where('isArchived', isEqualTo: false)
    .where('tags', arrayContains: 'work');
```

**Limitation:** Most compound queries require a composite index.

#### Query Limitations with Indexes

Firestore needs indexes for:
- Queries with multiple `where()` on different fields
- Queries with `where()` + `orderBy()` on different fields
- Queries with `orderBy()` on multiple fields

**Error message:**
```
The query requires an index. You can create it here: [link]
```

Click the link to create the index automatically, or create manually in Firebase Console.

#### Composite Indexes

Composite indexes combine multiple fields:

**Example index needed:**
```dart
// This query needs an index on (userId, isArchived, createdAt)
.where('userId', isEqualTo: 'user123')
.where('isArchived', isEqualTo: false)
.orderBy('createdAt');
```

**Creating indexes:**
1. Firebase Console → Firestore → Indexes
2. Click "Create Index"
3. Add collection and fields
4. Wait for index to build (can take minutes)

**Or use firestore.indexes.json:**
```json
{
  "indexes": [
    {
      "collectionGroup": "notes",
      "queryScope": "COLLECTION",
      "fields": [
        {"fieldPath": "userId", "order": "ASCENDING"},
        {"fieldPath": "isArchived", "order": "ASCENDING"},
        {"fieldPath": "createdAt", "order": "DESCENDING"}
      ]
    }
  ]
}
```

Deploy: `firebase deploy --only firestore:indexes`

### 2.6 Pagination

Pagination loads data in chunks to reduce reads and improve performance.

#### Using limit() with Cursors

**Basic pagination:**
```dart
class NotesPaginator {
  DocumentSnapshot? lastDocument;
  final int pageSize = 20;

  Future<List<Map<String, dynamic>>> loadNextPage(String userId) async {
    var query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .orderBy('createdAt', descending: true)
        .limit(pageSize);

    if (lastDocument != null) {
      query = query.startAfter([lastDocument!]);
    }

    final snapshot = await query.get();
    
    if (snapshot.docs.isNotEmpty) {
      lastDocument = snapshot.docs.last;
    }

    return snapshot.docs.map((doc) => doc.data()).toList();
  }

  void reset() {
    lastDocument = null;
  }
}
```

#### Infinite Scroll Pattern

Load more data as user scrolls:

```dart
class NotesListState extends State<NotesList> {
  final List<Map<String, dynamic>> notes = [];
  DocumentSnapshot? lastDoc;
  bool isLoading = false;
  bool hasMore = true;

  Future<void> loadMore() async {
    if (isLoading || !hasMore) return;

    setState(() => isLoading = true);

    var query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: currentUserId)
        .orderBy('createdAt', descending: true)
        .limit(20);

    if (lastDoc != null) {
      query = query.startAfter([lastDoc!]);
    }

    final snapshot = await query.get();

    if (snapshot.docs.isEmpty) {
      hasMore = false;
    } else {
      final newNotes = snapshot.docs.map((doc) => doc.data()).toList();
      setState(() {
        notes.addAll(newNotes);
        lastDoc = snapshot.docs.last;
      });
    }

    setState(() => isLoading = false);
  }
}
```

#### Page Navigation

Navigate to specific pages (less common, requires offset):

```dart
// Note: Firestore doesn't support offset() directly
// You must use cursors (startAt/startAfter)

Future<List<Map<String, dynamic>>> getPage(
  int pageNumber,
  int pageSize,
) async {
  // For page N, you need to load pages 1 through N-1 first
  // Or maintain cursor positions for each page
  // This is why infinite scroll is preferred
}
```

**Recommendation:** Use infinite scroll instead of page numbers. It's more efficient and better UX.

### 2.7 QuerySnapshot Properties

Understanding QuerySnapshot helps you work with query results.

#### docs (List of Documents)

Access all documents:

```dart
final snapshot = await query.get();
final documents = snapshot.docs; // List<QueryDocumentSnapshot>

for (var doc in documents) {
  print('${doc.id}: ${doc.data()}');
}
```

#### size (Document Count)

Get number of documents:

```dart
final snapshot = await query.get();
print('Found ${snapshot.size} notes');
```

**Note:** `size` counts documents in current query result, not total in collection.

#### empty (Boolean)

Check if query returned no documents:

```dart
final snapshot = await query.get();
if (snapshot.empty) {
  print('No notes found');
} else {
  print('Found ${snapshot.size} notes');
}
```

#### metadata (Query Metadata)

Access query metadata:

```dart
final snapshot = await query.get();
final metadata = snapshot.metadata;

if (metadata.isFromCache) {
  print('Data loaded from cache');
}

if (metadata.hasPendingWrites) {
  print('Document has pending writes');
}
```

**Use cases:**
- Show "offline" indicator when `isFromCache` is true
- Detect when local changes haven't synced yet

#### forEach() Iterator

Iterate over documents:

```dart
final snapshot = await query.get();
snapshot.docs.forEach((doc) {
  print('${doc.id}: ${doc.data()}');
});
```

### 2.8 DocumentSnapshot Properties

Understanding DocumentSnapshot helps you extract data.

#### id (Document ID)

Get document ID:

```dart
final snapshot = await docRef.get();
print('Document ID: ${snapshot.id}');
```

#### exists (Boolean)

Check if document exists:

```dart
final snapshot = await docRef.get();
if (snapshot.exists) {
  // Document exists, safe to call data()
} else {
  // Document doesn't exist
}
```

#### data() (Map of Fields)

Get all fields as Map:

```dart
final snapshot = await docRef.get();
final data = snapshot.data(); // Map<String, dynamic>?

if (data != null) {
  final title = data['title'] as String?;
  final content = data['content'] as String?;
}
```

**Type safety:** Always cast fields to expected types:
```dart
final title = data['title'] as String? ?? 'Untitled';
final wordCount = data['wordCount'] as int? ?? 0;
final isPinned = data['isPinned'] as bool? ?? false;
```

#### get(fieldPath)

Get specific field value:

```dart
final snapshot = await docRef.get();

// Simple field
final title = snapshot.get('title') as String?;

// Nested field
final userName = snapshot.get('author.displayName') as String?;

// With default value
final title = snapshot.get('title') ?? 'Untitled';
```

**FieldPath for nested fields:**
```dart
final fieldPath = FieldPath(['author', 'displayName']);
final userName = snapshot.get(fieldPath) as String?;
```

#### reference (DocumentReference)

Get reference to the document:

```dart
final snapshot = await docRef.get();
final ref = snapshot.reference;

// Use reference for updates
await ref.update({'title': 'New Title'});
```

## 3. Practical Examples

### Example: Loading User's Notes

**Simple list:**
```dart
Future<List<Note>> loadUserNotes(String userId) async {
  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .orderBy('createdAt', descending: true)
      .get();

  return snapshot.docs.map((doc) {
    final data = doc.data();
    return Note(
      id: doc.id,
      title: data['title'] as String,
      content: data['content'] as String,
      createdAt: (data['createdAt'] as Timestamp).toDate(),
    );
  }).toList();
}
```

### Example: Search Notes by Tag

```dart
Future<List<Note>> searchNotesByTag(String userId, String tag) async {
  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .where('tags', arrayContains: tag)
      .orderBy('createdAt', descending: true)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Get Pinned Notes

```dart
Future<List<Note>> getPinnedNotes(String userId) async {
  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .where('isPinned', isEqualTo: true)
      .orderBy('createdAt', descending: true)
      .limit(10)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Paginated Notes with Loading States

```dart
class NotesRepository {
  DocumentSnapshot? _lastDocument;
  static const int pageSize = 20;

  Future<NotesPage> loadNotesPage(String userId) async {
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

    final notes = snapshot.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();

    return NotesPage(
      notes: notes,
      hasMore: snapshot.docs.length == pageSize,
    );
  }

  void resetPagination() {
    _lastDocument = null;
  }
}

class NotesPage {
  final List<Note> notes;
  final bool hasMore;

  NotesPage({required this.notes, required this.hasMore});
}
```

### Example: Handling Document Not Found

```dart
Future<Note?> getNoteById(String noteId) async {
  try {
    final doc = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .get();

    if (doc.exists) {
      return Note.fromFirestore(doc);
    } else {
      return null; // Note doesn't exist
    }
  } catch (e) {
    print('Error loading note: $e');
    return null;
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**One-time read (get()) vs Real-time listener (onSnapshot):**
- **get():** Lower cost, simpler code, no cleanup needed
- **onSnapshot:** Higher cost (listens continuously), automatic updates, requires cleanup
- **Recommendation:** Use `get()` for static data, `onSnapshot` for collaborative/live data

**Pagination with cursors vs offset:**
- **Cursors:** Efficient, works with large datasets, Firestore native
- **Offset:** Not supported directly, would require loading all previous pages
- **Recommendation:** Always use cursors for pagination

**Multiple where() vs Single where() with filtering:**
- **Multiple where():** Server-side filtering, efficient, requires indexes
- **Client-side filtering:** More reads, simpler queries, no indexes needed
- **Recommendation:** Use multiple `where()` for server-side filtering. Only filter client-side for complex logic.

**limit() size:**
- **Small (10-20):** Faster loads, more requests, better for mobile
- **Large (50-100):** Fewer requests, slower loads, more data transfer
- **Recommendation:** Start with 20-30 documents per page. Adjust based on document size and network conditions.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Reading entire collection without limit**
```dart
// BAD: Reads all documents (expensive!)
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .get();

// GOOD: Use limit and pagination
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .limit(20)
    .get();
```

**2. Not checking if document exists**
```dart
// BAD: Crashes if document doesn't exist
final data = snapshot.data()!;
final title = data['title'];

// GOOD: Check existence first
if (snapshot.exists) {
  final data = snapshot.data()!;
  final title = data['title'] as String?;
}
```

**3. Forgetting to cancel listeners**
```dart
// BAD: Memory leak, continues reading
noteRef.snapshots().listen((snapshot) {
  // Update UI
});

// GOOD: Cancel when done
StreamSubscription? _subscription;

void initState() {
  _subscription = noteRef.snapshots().listen((snapshot) {
    // Update UI
  });
}

void dispose() {
  _subscription?.cancel();
  super.dispose();
}
```

**4. Not handling null/optional fields**
```dart
// BAD: Assumes field always exists
final title = data['title'] as String;

// GOOD: Handle null/optional fields
final title = data['title'] as String? ?? 'Untitled';
```

### Performance Pitfalls

**1. Not using indexes for compound queries**
```dart
// BAD: Query fails or is slow without index
.where('userId', isEqualTo: userId)
.where('isArchived', isEqualTo: false)
.orderBy('createdAt');

// GOOD: Create composite index first
// Firebase will prompt you with link
```

**2. Over-fetching data**
```dart
// BAD: Reads entire document when you only need title
final doc = await noteRef.get();
final title = doc.data()?['title'];

// GOOD: Consider if you can restructure to need less data
// Or accept that Firestore reads entire documents (by design)
```

**3. Too many small queries instead of one larger query**
```dart
// BAD: Multiple queries (expensive)
for (var noteId in noteIds) {
  final doc = await noteRef.doc(noteId).get();
}

// GOOD: Single query with whereIn (if < 10 IDs)
final snapshot = await collectionRef
    .where(FieldPath.documentId, whereIn: noteIds)
    .get();
```

**4. Not using pagination for large lists**
```dart
// BAD: Loads all notes at once
final snapshot = await collectionRef.get();
final allNotes = snapshot.docs.map(...).toList();

// GOOD: Paginate results
final snapshot = await collectionRef
    .limit(20)
    .get();
```

### Query Mistakes

**1. Using whereIn with more than 10 values**
```dart
// BAD: Fails if more than 10 IDs
.where(FieldPath.documentId, whereIn: manyNoteIds);

// GOOD: Split into batches of 10
for (var batch in _chunkList(manyNoteIds, 10)) {
  final snapshot = await collectionRef
      .where(FieldPath.documentId, whereIn: batch)
      .get();
}
```

**2. Combining incompatible query operators**
```dart
// BAD: Can't combine whereIn with arrayContainsAny
.where('tags', whereIn: ['tag1', 'tag2'])
.where('tags', arrayContainsAny: ['tag3', 'tag4']);

// GOOD: Use separate queries or restructure data
```

**3. OrderBy without where on same field**
```dart
// BAD: Requires index, might be inefficient
.where('userId', isEqualTo: userId)
.orderBy('createdAt');

// GOOD: If possible, orderBy on userId field too
// Or ensure index exists
```

## 6. Summary & Checklist

### Summary

- **get()** fetches documents once. Use for one-time reads.
- **onSnapshot()** listens for real-time updates. Remember to cancel listeners.
- **Query methods** (where, orderBy, limit) filter and sort documents server-side.
- **Compound queries** require composite indexes. Firebase provides links to create them.
- **Pagination** uses cursors (startAfter) with limit() for efficient data loading.
- **QuerySnapshot** contains multiple documents. Use `docs`, `size`, `empty` properties.
- **DocumentSnapshot** represents a single document. Check `exists` before accessing `data()`.

### Checklist: You Are Ready When You Can...

- [ ] Fetch a single document using `get()` and handle non-existent documents
- [ ] Read all documents from a collection with proper error handling
- [ ] Use `where()` to filter documents by field values
- [ ] Use `orderBy()` to sort query results
- [ ] Implement pagination using `limit()` and `startAfter()`
- [ ] Create composite indexes for compound queries
- [ ] Choose between `get()` and `onSnapshot()` based on use case
- [ ] Access document data using `data()` and handle null/optional fields
- [ ] Use QuerySnapshot properties (`docs`, `size`, `empty`) effectively
- [ ] Cancel real-time listeners to prevent memory leaks
- [ ] Handle query errors and missing indexes
- [ ] Implement infinite scroll pattern for large datasets

### Verification Steps

**Test your reading operations:**

1. **Test single document read:**
```dart
final doc = await FirebaseFirestore.instance
    .collection('notes')
    .doc('test-note-id')
    .get();

assert(doc.exists || !doc.exists); // Should not crash
if (doc.exists) {
  print('Document data: ${doc.data()}');
}
```

2. **Test collection query:**
```dart
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'test-user')
    .limit(10)
    .get();

print('Found ${snapshot.size} documents');
snapshot.docs.forEach((doc) {
  print('${doc.id}: ${doc.data()}');
});
```

3. **Test pagination:**
```dart
var query = FirebaseFirestore.instance
    .collection('notes')
    .orderBy('createdAt')
    .limit(5);

var page1 = await query.get();
print('Page 1: ${page1.size} documents');

if (page1.docs.isNotEmpty) {
  var page2 = await query.startAfter([page1.docs.last]).get();
  print('Page 2: ${page2.size} documents');
}
```

### Next Steps

Once you can read data efficiently, you're ready to:
- Write data to Firestore (Chapter 5)
- Build complex queries with filtering (Chapter 6)
- Set up real-time listeners for live updates (Chapter 7)
