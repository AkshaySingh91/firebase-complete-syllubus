# 6. QUERIES & FILTERING

## 1. What This Chapter Covers

This chapter teaches you how to build efficient queries and filters in Firestore for your Notes application. You'll learn:

- How to filter documents using equality, comparison, and array operators
- How to combine multiple conditions in composite queries
- How to query across subcollections using collection group queries
- How to optimize queries for performance and cost
- When indexes are required and how to create them

Building efficient queries is critical because poorly designed queries lead to high costs, slow performance, and user frustration. Understanding query operators, index requirements, and optimization techniques helps you build fast, cost-effective applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Equality Queries

Equality queries match documents where a field equals (or doesn't equal) a specific value.

#### where('field', isEqualTo: value)

Find documents where a field exactly matches a value:

```dart
// Find notes by specific user
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'user123');

// Find notes with specific status
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', isEqualTo: 'published');

// Find notes with exact tag
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('category', isEqualTo: 'work');
```

**Use cases:**
- Filtering by user ID
- Finding documents with specific status
- Matching exact string values
- Filtering by boolean flags

**Example for Notes app:**
```dart
// Get all pinned notes for a user
final pinnedNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('isPinned', isEqualTo: true)
    .get();
```

#### where('field', isNotEqualTo: value)

Find documents where a field does NOT equal a value:

```dart
// Find notes that are not archived
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', isNotEqualTo: 'archived');

// Find notes not in draft status
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', isNotEqualTo: 'draft');
```

**Limitations:**
- `isNotEqualTo` requires an index when combined with other where clauses
- Can't be combined with `orderBy` on a different field without an index
- Less efficient than positive equality checks

**Use cases:**
- Excluding specific statuses
- Finding active vs inactive documents
- Filtering out deleted items

**Example:**
```dart
// Get all active (non-archived, non-deleted) notes
final activeNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('status', isNotEqualTo: 'archived')
    .where('status', isNotEqualTo: 'deleted')
    .get();
```

**Note:** For multiple exclusions, consider using `whereIn` with allowed values instead of multiple `isNotEqualTo` clauses.

### 2.2 Comparison Queries

Comparison queries match documents based on numeric, date, or string ordering.

#### where('field', isGreaterThan: value)

Find documents where field value is greater than the specified value:

```dart
// Find notes with word count greater than 1000
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('wordCount', isGreaterThan: 1000);

// Find notes created after a specific date
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('createdAt', isGreaterThan: DateTime(2024, 1, 1));
```

#### where('field', isGreaterThanOrEqualTo: value)

Includes documents where field equals the value:

```dart
// Find notes with word count >= 500
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('wordCount', isGreaterThanOrEqualTo: 500);
```

#### where('field', isLessThan: value)

Find documents where field value is less than the specified value:

```dart
// Find notes with word count less than 100
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('wordCount', isLessThan: 100);
```

#### where('field', isLessThanOrEqualTo: value)

Includes documents where field equals the value:

```dart
// Find notes with word count <= 500
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('wordCount', isLessThanOrEqualTo: 500);
```

**Range queries:**
```dart
// Find notes with word count between 100 and 1000
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('wordCount', isGreaterThanOrEqualTo: 100)
    .where('wordCount', isLessThanOrEqualTo: 1000);
```

**Date range queries:**
```dart
// Find notes created in the last week
final weekAgo = DateTime.now().subtract(Duration(days: 7));
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('createdAt', isGreaterThanOrEqualTo: Timestamp.fromDate(weekAgo));
```

**String comparison:**
```dart
// Find notes with title starting with 'A' through 'M'
// Note: String comparison is lexicographic (alphabetical)
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('title', isGreaterThanOrEqualTo: 'A')
    .where('title', isLessThan: 'N');
```

### 2.3 Array Queries

Array queries check if arrays contain specific values.

#### where('tags', arrayContains: 'value')

Find documents where an array field contains a specific value:

```dart
// Find notes tagged with 'work'
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContains: 'work');

// Find notes with specific tag
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContains: 'important');
```

**Behavior:**
- Matches if array contains the exact value
- Works with strings, numbers, booleans
- For objects, entire object must match

**Example for Notes app:**
```dart
// Get all work-related notes
final workNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('tags', arrayContains: 'work')
    .get();
```

#### where('tags', arrayContainsAny: ['value1', 'value2'])

Find documents where an array field contains ANY of the specified values:

```dart
// Find notes tagged with 'work' OR 'personal'
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContainsAny: ['work', 'personal']);

// Find notes with any of multiple tags
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContainsAny: ['urgent', 'important', 'critical']);
```

**Limitations:**
- Limited to 10 values in the array
- Can't combine with `arrayContains` in the same query
- Requires index when combined with other where clauses

**Use cases:**
- Finding documents with any of several tags
- Multi-category filtering
- Flexible tag-based search

**Example:**
```dart
// Get notes with any priority tag
final priorityNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('tags', arrayContainsAny: ['urgent', 'important', 'critical'])
    .get();
```

#### Array Membership

Understanding how array queries work:

**String arrays:**
```dart
// Document: {tags: ['work', 'personal', 'urgent']}
.where('tags', arrayContains: 'work') // Matches
.where('tags', arrayContains: 'home') // Doesn't match
```

**Object arrays:**
```dart
// Document: {
//   collaborators: [
//     {userId: 'user123', role: 'editor'},
//     {userId: 'user456', role: 'viewer'}
//   ]
// }

// This matches (entire object must match)
.where('collaborators', arrayContains: {
  'userId': 'user123',
  'role': 'editor'
})

// This doesn't match (partial match not supported)
.where('collaborators', arrayContains: {'userId': 'user123'})
```

**Note:** For object arrays, you typically query by a separate field that stores user IDs:
```dart
// Better structure for querying
// Document: {
//   collaboratorIds: ['user123', 'user456'],
//   collaborators: [{userId: 'user123', role: 'editor'}, ...]
// }

.where('collaboratorIds', arrayContains: 'user123')
```

### 2.4 IN and NOT IN Queries

IN and NOT IN queries match documents where a field value is in (or not in) a list of values.

#### where('category', whereIn: ['value1', 'value2'])

Find documents where field value matches ANY value in the list:

```dart
// Find notes in 'work' OR 'personal' category
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('category', whereIn: ['work', 'personal']);

// Find notes with specific statuses
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', whereIn: ['draft', 'published']);
```

**Limitations:**
- Maximum 10 values in the array
- Requires index when combined with other where clauses
- Can't combine `whereIn` with `arrayContainsAny` in same query

**Use cases:**
- Filtering by multiple exact values
- Status-based filtering
- Category selection

**Example:**
```dart
// Get notes in multiple categories
final notes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('category', whereIn: ['work', 'personal', 'ideas'])
    .get();
```

#### where('status', whereNotIn: ['value1', 'value2'])

Find documents where field value does NOT match any value in the list:

```dart
// Find notes that are not archived or deleted
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', whereNotIn: ['archived', 'deleted']);

// Exclude multiple statuses
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('status', whereNotIn: ['draft', 'archived', 'deleted']);
```

**Limitations:**
- Maximum 10 values in the array
- Requires index when combined with other where clauses
- Less efficient than positive filters

**Use cases:**
- Excluding multiple statuses
- Finding active documents
- Filtering out unwanted categories

**Example:**
```dart
// Get active notes (exclude archived and deleted)
final activeNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('status', whereNotIn: ['archived', 'deleted'])
    .get();
```

**Handling more than 10 values:**
```dart
// If you need to exclude more than 10 values, split into multiple queries
Future<List<QueryDocumentSnapshot>> getActiveNotes(String userId) async {
  final excludedStatuses = ['archived', 'deleted', 'draft', ...]; // > 10 items
  
  if (excludedStatuses.length <= 10) {
    return (await FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .where('status', whereNotIn: excludedStatuses)
        .get()).docs;
  }
  
  // Split into chunks of 10
  final allDocs = <QueryDocumentSnapshot>[];
  for (var chunk in _chunkList(excludedStatuses, 10)) {
    final snapshot = await FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .where('status', whereNotIn: chunk)
        .get();
    allDocs.addAll(snapshot.docs);
  }
  
  // Remove duplicates
  return allDocs.toSet().toList();
}
```

### 2.5 Composite Queries

Composite queries combine multiple conditions to create complex filters.

#### Multiple Conditions

Chain multiple `where()` clauses:

```dart
// Find pinned, non-archived notes for a user
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('isPinned', isEqualTo: true)
    .where('isArchived', isEqualTo: false)
    .where('wordCount', isGreaterThan: 100);
```

**Order matters for some queries:**
- Equality filters should come first
- Range filters should be on the same field or require index
- `orderBy` usually needs to match a where clause or require index

#### Index Requirements

Most composite queries require composite indexes:

**Simple query (no index needed):**
```dart
// Single where clause - uses automatic single-field index
.where('userId', isEqualTo: 'user123')
```

**Composite query (index required):**
```dart
// Multiple where clauses on different fields - needs composite index
.where('userId', isEqualTo: 'user123')
.where('isPinned', isEqualTo: true)
.where('isArchived', isEqualTo: false)
.orderBy('createdAt')
```

**Creating indexes:**
1. Run the query - Firestore will show an error with a link
2. Click the link to create the index automatically
3. Or create manually in Firebase Console → Firestore → Indexes

**Index definition example:**
```json
{
  "collectionGroup": "notes",
  "queryScope": "COLLECTION",
  "fields": [
    {"fieldPath": "userId", "order": "ASCENDING"},
    {"fieldPath": "isPinned", "order": "ASCENDING"},
    {"fieldPath": "isArchived", "order": "ASCENDING"},
    {"fieldPath": "createdAt", "order": "DESCENDING"}
  ]
}
```

#### Query Cursors with Compound Queries

Cursors work with composite queries but require careful ordering:

```dart
// First page
var query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('isPinned', isEqualTo: true)
    .orderBy('createdAt', descending: true)
    .limit(20);

var snapshot = await query.get();
var lastDoc = snapshot.docs.last;

// Next page
var nextQuery = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .where('isPinned', isEqualTo: true)
    .orderBy('createdAt', descending: true)
    .startAfter([lastDoc])
    .limit(20);
```

**Important:** The `orderBy` field must be included in the composite index, and cursor values must match the query structure.

### 2.6 Collection Group Queries

Collection group queries search across all subcollections with the same name, regardless of parent document.

#### Query Across Subcollections

Normal queries only search one collection. Collection group queries search all subcollections with matching names:

**Normal query (single collection):**
```dart
// Only searches /users/user123/notes
final query = FirebaseFirestore.instance
    .collection('users')
    .doc('user123')
    .collection('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

**Collection group query (all subcollections):**
```dart
// Searches /users/*/notes across all users
final query = FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

**Use cases:**
- Searching all users' notes (admin feature)
- Finding notes across all users with specific tags
- Global search functionality
- Analytics across all subcollections

**Example for Notes app:**
```dart
// Find all notes with 'work' tag across all users
final allWorkNotes = await FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

#### Using collectionGroup

The `collectionGroup()` method queries across collections with the same name:

```dart
// Query all 'notes' subcollections
final query = FirebaseFirestore.instance.collectionGroup('notes');

// Can combine with where clauses
final query = FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('isPublic', isEqualTo: true)
    .where('tags', arrayContains: 'public')
    .orderBy('createdAt', descending: true)
    .limit(50);
```

**Important considerations:**
- Requires collection group index (different from regular composite index)
- Can be expensive (searches many collections)
- Use security rules to restrict access appropriately
- Consider cost implications for large datasets

**Collection group index:**
Collection group queries require special indexes. Firestore will prompt you to create them, or create in Console → Firestore → Indexes → Collection Group tab.

### 2.7 Query Optimization

Optimizing queries reduces costs and improves performance.

#### Selecting Fields

**Note:** Firestore doesn't support selecting specific fields. You always read entire documents. However, you can structure your data to minimize document size:

**Bad structure (large documents):**
```dart
// Document contains everything
{
  title: 'Note',
  content: 'Very long content...',
  fullContent: 'Even longer full content...',
  metadata: {...large object...},
  history: [...many items...]
}
```

**Better structure (smaller documents):**
```dart
// Main document (small)
{
  title: 'Note',
  content: 'Preview...',
  contentRef: '/notes/note123/fullContent'
}

// Separate document for large content
/notes/note123/fullContent {
  text: 'Very long content...'
}
```

#### Limiting Results

Always use `limit()` to reduce reads:

```dart
// BAD: Reads all documents
final snapshot = await collectionRef.get();

// GOOD: Limit results
final snapshot = await collectionRef
    .where('userId', isEqualTo: currentUserId)
    .limit(20)
    .get();
```

**Pagination:**
```dart
// Load pages of 20 documents
final snapshot = await collectionRef
    .where('userId', isEqualTo: currentUserId)
    .orderBy('createdAt')
    .limit(20)
    .get();
```

#### Avoiding Client-Side Filtering

Filter on the server, not the client:

```dart
// BAD: Reads all, filters client-side
final allNotes = await collectionRef.get();
final workNotes = allNotes.docs
    .where((doc) => (doc.data()['tags'] as List).contains('work'))
    .toList();

// GOOD: Filter server-side
final workNotes = await collectionRef
    .where('tags', arrayContains: 'work')
    .get();
```

**When client-side filtering is acceptable:**
- Complex logic that can't be expressed in queries
- Small result sets (< 10 documents)
- One-time operations (not in loops)

#### Proper Indexing

Create indexes for all composite queries:

**Automatic indexes:**
- Single-field queries
- Single-field orderBy

**Manual indexes required:**
- Multiple where clauses on different fields
- where() + orderBy() on different fields
- Multiple orderBy() clauses
- Collection group queries

**Index best practices:**
- Create indexes as you develop (Firestore provides links)
- Test queries in development before production
- Monitor index usage in Firebase Console
- Remove unused indexes to save storage

**Checking index status:**
```dart
// Firestore will throw error with index link if index missing
try {
  final snapshot = await query.get();
} on FirebaseException catch (e) {
  if (e.code == 'failed-precondition') {
    print('Index required: ${e.message}');
    // Follow link in error message to create index
  }
}
```

## 3. Practical Examples

### Example: Filter Notes by Multiple Criteria

```dart
Future<List<Note>> getFilteredNotes({
  required String userId,
  bool? isPinned,
  bool? isArchived,
  String? category,
  int? minWordCount,
  List<String>? tags,
}) async {
  var query = FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId);

  if (isPinned != null) {
    query = query.where('isPinned', isEqualTo: isPinned);
  }

  if (isArchived != null) {
    query = query.where('isArchived', isEqualTo: isArchived);
  }

  if (category != null) {
    query = query.where('category', isEqualTo: category);
  }

  if (minWordCount != null) {
    query = query.where('wordCount', isGreaterThanOrEqualTo: minWordCount);
  }

  if (tags != null && tags.isNotEmpty) {
    if (tags.length == 1) {
      query = query.where('tags', arrayContains: tags.first);
    } else {
      query = query.where('tags', arrayContainsAny: tags.take(10).toList());
    }
  }

  query = query.orderBy('createdAt', descending: true).limit(50);

  final snapshot = await query.get();
  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Search Notes by Date Range

```dart
Future<List<Note>> getNotesInDateRange({
  required String userId,
  required DateTime startDate,
  required DateTime endDate,
}) async {
  final startTimestamp = Timestamp.fromDate(startDate);
  final endTimestamp = Timestamp.fromDate(endDate);

  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .where('createdAt', isGreaterThanOrEqualTo: startTimestamp)
      .where('createdAt', isLessThanOrEqualTo: endTimestamp)
      .orderBy('createdAt', descending: true)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Find Notes with Any Priority Tag

```dart
Future<List<Note>> getPriorityNotes(String userId) async {
  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .where('tags', arrayContainsAny: [
        'urgent',
        'important',
        'critical',
        'high-priority'
      ])
      .orderBy('createdAt', descending: true)
      .limit(20)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Collection Group Query for Public Notes

```dart
Future<List<Note>> getPublicNotesByTag(String tag) async {
  final snapshot = await FirebaseFirestore.instance
      .collectionGroup('notes')
      .where('isPublic', isEqualTo: true)
      .where('tags', arrayContains: tag)
      .orderBy('createdAt', descending: true)
      .limit(50)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

### Example: Exclude Multiple Statuses

```dart
Future<List<Note>> getActiveNotes(String userId) async {
  // Use whereIn with allowed statuses instead of whereNotIn
  final allowedStatuses = ['draft', 'published', 'shared'];
  
  final snapshot = await FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .where('status', whereIn: allowedStatuses)
      .orderBy('createdAt', descending: true)
      .get();

  return snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Equality vs Comparison queries:**
- **Equality:** Fast, simple, no index needed for single field
- **Comparison:** More flexible, requires index for ranges
- **Recommendation:** Use equality when possible. Use comparison for ranges and sorting.

**arrayContains vs arrayContainsAny:**
- **arrayContains:** Single value, simpler, more efficient
- **arrayContainsAny:** Multiple values (up to 10), more flexible
- **Recommendation:** Use `arrayContains` for single tag. Use `arrayContainsAny` for multiple tags.

**whereIn vs Multiple OR queries:**
- **whereIn:** Single query, efficient, limited to 10 values
- **Multiple queries:** More flexible, requires merging results client-side
- **Recommendation:** Use `whereIn` when <= 10 values. Split into multiple queries and merge if > 10.

**Collection group vs Separate collection:**
- **Collection group:** Natural hierarchy, scoped to parent, requires collection group index
- **Separate collection:** Easier global queries, simpler indexes
- **Recommendation:** Use subcollections for user-scoped data. Use root collections if you need frequent cross-user queries.

**Client-side vs Server-side filtering:**
- **Server-side:** Efficient, reduces reads, requires proper indexes
- **Client-side:** More flexible, but reads all documents first
- **Recommendation:** Always filter server-side when possible. Only filter client-side for complex logic.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Not using limit() on queries**
```dart
// BAD: Reads all documents (expensive!)
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .get();

// GOOD: Always limit results
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .limit(20)
    .get();
```

**2. Using whereIn with more than 10 values**
```dart
// BAD: Fails if more than 10 values
final manyIds = List.generate(15, (i) => 'id$i');
final snapshot = await collectionRef
    .where(FieldPath.documentId, whereIn: manyIds)
    .get();

// GOOD: Split into chunks
for (var chunk in _chunkList(manyIds, 10)) {
  final snapshot = await collectionRef
      .where(FieldPath.documentId, whereIn: chunk)
      .get();
  // Process results
}
```

**3. Combining incompatible query operators**
```dart
// BAD: Can't combine whereIn with arrayContainsAny
final snapshot = await collectionRef
    .where('category', whereIn: ['work', 'personal'])
    .where('tags', arrayContainsAny: ['urgent', 'important'])
    .get(); // Error!

// GOOD: Use separate queries or restructure
```

**4. Not creating required indexes**
```dart
// BAD: Query fails without index
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .where('isPinned', isEqualTo: true)
    .orderBy('createdAt')
    .get(); // Fails: index required

// GOOD: Create composite index first
// Firestore provides link in error message
```

### Performance Pitfalls

**1. Client-side filtering instead of server-side**
```dart
// BAD: Reads all, filters client-side
final allNotes = await collectionRef.get();
final workNotes = allNotes.docs
    .where((doc) => doc.data()['tags'].contains('work'))
    .toList();

// GOOD: Filter server-side
final workNotes = await collectionRef
    .where('tags', arrayContains: 'work')
    .get();
```

**2. Not using pagination for large result sets**
```dart
// BAD: Loads all notes at once
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .get(); // Could be thousands of documents!

// GOOD: Paginate
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .limit(20)
    .get();
```

**3. Over-complex queries that could be simpler**
```dart
// BAD: Complex query with many conditions
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .where('status', isNotEqualTo: 'archived')
    .where('status', isNotEqualTo: 'deleted')
    .where('status', isNotEqualTo: 'draft')
    .get();

// GOOD: Use whereIn with allowed values
final snapshot = await collectionRef
    .where('userId', isEqualTo: userId)
    .where('status', whereIn: ['published', 'shared'])
    .get();
```

**4. Collection group queries without considering cost**
```dart
// BAD: Expensive collection group query without limit
final snapshot = await FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('tags', arrayContains: 'work')
    .get(); // Could read millions of documents!

// GOOD: Always limit collection group queries
final snapshot = await FirebaseFirestore.instance
    .collectionGroup('notes')
    .where('tags', arrayContains: 'work')
    .limit(50)
    .get();
```

## 6. Summary & Checklist

### Summary

- **Equality queries** (`isEqualTo`, `isNotEqualTo`) match exact values
- **Comparison queries** (`isGreaterThan`, `isLessThan`, etc.) match ranges
- **Array queries** (`arrayContains`, `arrayContainsAny`) search within arrays
- **IN queries** (`whereIn`, `whereNotIn`) match multiple values (max 10)
- **Composite queries** combine multiple conditions and require indexes
- **Collection group queries** search across all subcollections with the same name
- **Query optimization** reduces costs through limiting, proper indexing, and server-side filtering

### Checklist: You Are Ready When You Can...

- [ ] Filter documents using equality operators (`isEqualTo`, `isNotEqualTo`)
- [ ] Use comparison operators for range queries (`isGreaterThan`, `isLessThan`, etc.)
- [ ] Query array fields using `arrayContains` and `arrayContainsAny`
- [ ] Use `whereIn` and `whereNotIn` for multiple value matching (respecting 10-value limit)
- [ ] Build composite queries with multiple `where()` clauses
- [ ] Create and understand composite indexes for complex queries
- [ ] Use collection group queries to search across subcollections
- [ ] Optimize queries by limiting results and filtering server-side
- [ ] Handle index requirements and create indexes when needed
- [ ] Choose appropriate query operators based on use case
- [ ] Avoid common mistakes like client-side filtering and missing limits
- [ ] Understand when to use collection groups vs separate collections

### Verification Steps

**Test your query operations:**

1. **Test equality query:**
```dart
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'test-user')
    .limit(10)
    .get();

assert(snapshot.size <= 10);
print('Found ${snapshot.size} notes');
```

2. **Test composite query:**
```dart
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'test-user')
    .where('isPinned', isEqualTo: true)
    .orderBy('createdAt')
    .limit(10)
    .get();

// If this fails, create the required composite index
print('Composite query successful: ${snapshot.size} results');
```

3. **Test array query:**
```dart
final snapshot = await FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContains: 'work')
    .limit(10)
    .get();

print('Found ${snapshot.size} notes with work tag');
```

### Next Steps

Once you can build efficient queries, you're ready to:
- Set up real-time listeners for live updates (Chapter 7)
- Use transactions for complex operations (Chapter 8)
- Apply advanced data modeling patterns (Chapter 9)
