# 5. WRITING DATA

## 1. What This Chapter Covers

This chapter teaches you how to write data to Firestore in your Notes application. You'll learn:

- How to create, update, and delete documents
- When to use set() vs update() vs add()
- How to merge data without overwriting existing fields
- How to perform atomic batch operations
- How to use server timestamps and field transforms
- How to handle write conflicts and errors

Writing data correctly is essential because it affects data integrity, user experience, and app reliability. Understanding write methods, merge behavior, and batch operations helps you build robust applications that handle concurrent edits and data consistency.

## 2. Core Concepts (Simple & Clear)

### 2.1 Writing Methods

Firestore provides several methods to write data, each with specific use cases.

#### set() - Create or Replace Document

`set()` creates a document if it doesn't exist, or completely replaces it if it does:

```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

await noteRef.set({
  'title': 'My Note',
  'content': 'Note content here',
  'userId': 'user123',
  'createdAt': FieldValue.serverTimestamp(),
});
```

**Behavior:**
- Creates document if it doesn't exist
- Replaces entire document if it exists (removes all other fields)
- Use when you want to set the complete document

**Warning:** `set()` without merge overwrites the entire document. Any fields not included are deleted.

#### add() - Create with Auto ID

`add()` creates a new document with an auto-generated ID:

```dart
final collectionRef = FirebaseFirestore.instance.collection('notes');

final docRef = await collectionRef.add({
  'title': 'New Note',
  'content': 'Content here',
  'userId': 'user123',
  'createdAt': FieldValue.serverTimestamp(),
});

print('Created document with ID: ${docRef.id}');
```

**Use cases:**
- Creating new documents when you don't need a specific ID
- Generating unique IDs automatically
- Simpler than generating ID manually

#### update() - Modify Specific Fields

`update()` modifies only the specified fields, leaving others unchanged:

```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

await noteRef.update({
  'title': 'Updated Title',
  'updatedAt': FieldValue.serverTimestamp(),
});
```

**Behavior:**
- Only updates specified fields
- Fails if document doesn't exist (throws error)
- Preserves all other fields
- Use when you want to modify specific fields

**Error handling:**
```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'not-found') {
    print('Document does not exist');
  }
}
```

#### create() - Create if Not Exists

`create()` creates a document only if it doesn't exist:

```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

try {
  await noteRef.create({
    'title': 'New Note',
    'content': 'Content',
    'userId': 'user123',
  });
} on FirebaseException catch (e) {
  if (e.code == 'already-exists') {
    print('Document already exists');
  }
}
```

**Use cases:**
- Ensuring document doesn't exist before creating
- Preventing accidental overwrites
- Initializing documents with default values

#### delete() - Remove Document

`delete()` removes a document entirely:

```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

await noteRef.delete();
```

**Behavior:**
- Removes document completely
- Succeeds even if document doesn't exist (no error)
- Subcollections are NOT deleted automatically

**Delete with subcollections:**
```dart
// Delete document
await noteRef.delete();

// Manually delete subcollections if needed
final commentsRef = noteRef.collection('comments');
final commentsSnapshot = await commentsRef.get();
for (var doc in commentsSnapshot.docs) {
  await doc.reference.delete();
}
```

### 2.2 set() Options

`set()` supports options to control merge behavior.

#### SetOptions with Merge

Merge data instead of replacing:

```dart
// Merge: true - merges with existing document
await noteRef.set({
  'title': 'Updated Title',
  'tags': ['work'],
}, SetOptions(merge: true));

// Existing fields are preserved
// Only 'title' and 'tags' are updated
```

**Merge behavior:**
- Fields in the set() call are updated
- Fields not in the set() call remain unchanged
- Document is created if it doesn't exist
- Use when you want partial updates with set()

#### Merge vs Replace

**Without merge (default):**
```dart
// Document exists with: {title: 'Old', content: 'Old content', userId: 'user123'}
await noteRef.set({
  'title': 'New Title',
});
// Result: {title: 'New Title'} - content and userId are deleted!
```

**With merge:**
```dart
// Document exists with: {title: 'Old', content: 'Old content', userId: 'user123'}
await noteRef.set({
  'title': 'New Title',
}, SetOptions(merge: true));
// Result: {title: 'New Title', content: 'Old content', userId: 'user123'}
```

**Recommendation:** Use `update()` for partial updates, or `set()` with merge if you're unsure if document exists.

#### Server Timestamps in set()

You can use server timestamps with set():

```dart
await noteRef.set({
  'title': 'My Note',
  'createdAt': FieldValue.serverTimestamp(),
  'updatedAt': FieldValue.serverTimestamp(),
}, SetOptions(merge: true));
```

### 2.3 update() Behavior

`update()` provides powerful field manipulation capabilities.

#### Field Paths with Dot Notation

Update nested fields using dot notation:

```dart
// Update nested field
await noteRef.update({
  'author.displayName': 'John Doe',
  'author.email': 'john@example.com',
});

// Update array element (by index - be careful!)
// Note: Firestore doesn't support direct array index updates well
// Better to use arrayUnion/arrayRemove
```

**Nested map updates:**
```dart
// Document structure:
// {
//   title: 'Note',
//   metadata: {
//     views: 10,
//     likes: 5
//   }
// }

await noteRef.update({
  'metadata.views': 11,
  'metadata.likes': 6,
});
```

#### Incrementing Counters

Use `FieldValue.increment()` for atomic counter updates:

```dart
// Increment by 1
await noteRef.update({
  'viewCount': FieldValue.increment(1),
});

// Increment by specific amount
await noteRef.update({
  'wordCount': FieldValue.increment(50),
});

// Decrement (use negative number)
await noteRef.update({
  'viewCount': FieldValue.increment(-1),
});
```

**Benefits:**
- Atomic operation (no race conditions)
- Works even if field doesn't exist (starts at 0)
- Safe for concurrent updates

**Example for Notes app:**
```dart
// When user views a note
await noteRef.update({
  'viewCount': FieldValue.increment(1),
  'lastViewedAt': FieldValue.serverTimestamp(),
});
```

#### Array Operations

Firestore provides atomic array operations:

**arrayUnion - Add elements (no duplicates):**
```dart
await noteRef.update({
  'tags': FieldValue.arrayUnion(['work', 'important']),
});

// If tags already contains 'work', it won't be duplicated
// Only 'important' is added if not present
```

**arrayRemove - Remove elements:**
```dart
await noteRef.update({
  'tags': FieldValue.arrayRemove(['work']),
});

// Removes 'work' from tags array if present
```

**Adding to collaborators array:**
```dart
await noteRef.update({
  'collaborators': FieldValue.arrayUnion([
    {'userId': 'user456', 'role': 'editor', 'addedAt': FieldValue.serverTimestamp()}
  ]),
});
```

**Limitation:** `arrayUnion` and `arrayRemove` work with entire array elements, not partial matches. For complex objects, the entire object must match.

#### Map Field Updates

Update nested map fields:

```dart
// Partial map update
await noteRef.update({
  'settings.theme': 'dark',
  'settings.fontSize': 14,
});

// This only updates theme and fontSize
// Other settings fields remain unchanged
```

**Important:** You can't update a map field partially with a new map object. Use dot notation for individual fields:

```dart
// BAD: Replaces entire settings map
await noteRef.update({
  'settings': {'theme': 'dark'}, // Other settings fields are lost!
});

// GOOD: Update individual fields
await noteRef.update({
  'settings.theme': 'dark',
  'settings.fontSize': 14,
});
```

### 2.4 Document Creation

You can create documents with manual or auto-generated IDs.

#### Manual Document ID

Specify the document ID yourself:

```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('my-custom-note-id');

await noteRef.set({
  'title': 'My Note',
  'content': 'Content',
});
```

**Use cases:**
- You have a natural identifier (username, slug, email)
- You need predictable, readable IDs
- Importing existing data with known IDs

**Example:**
```dart
// Create note with slug as ID
final slug = 'meeting-notes-2024';
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc(slug);

await noteRef.set({
  'title': 'Meeting Notes 2024',
  'slug': slug,
  'content': '...',
});
```

#### Auto-Generated Document ID

Let Firestore generate a unique ID:

```dart
// Method 1: Using doc() without ID
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc(); // Auto-generates ID

await noteRef.set({
  'title': 'My Note',
  'content': 'Content',
});

print('Created note with ID: ${noteRef.id}');

// Method 2: Using add()
final docRef = await FirebaseFirestore.instance
    .collection('notes')
    .add({
      'title': 'My Note',
      'content': 'Content',
    });

print('Created note with ID: ${docRef.id}');
```

**Use cases:**
- You don't have a natural identifier
- You want guaranteed uniqueness
- You prefer shorter, URL-safe IDs

#### Using add() Method

`add()` is a convenience method for creating documents with auto IDs:

```dart
final docRef = await FirebaseFirestore.instance
    .collection('notes')
    .add({
      'title': 'New Note',
      'content': 'Content here',
      'userId': currentUserId,
      'createdAt': FieldValue.serverTimestamp(),
    });

// docRef contains the DocumentReference with auto-generated ID
final noteId = docRef.id;
```

**Difference from set() with auto ID:**
- `add()` returns the DocumentReference immediately
- `set()` with `doc()` requires getting the reference first
- Both create documents with auto-generated IDs

### 2.5 Batch Writes

Batch writes perform multiple operations atomically (all succeed or all fail).

#### WriteBatch Class

Create a batch and add operations:

```dart
final batch = FirebaseFirestore.instance.batch();

// Add multiple operations
final note1Ref = FirebaseFirestore.instance.collection('notes').doc('note1');
batch.set(note1Ref, {'title': 'Note 1', 'userId': 'user123'});

final note2Ref = FirebaseFirestore.instance.collection('notes').doc('note2');
batch.set(note2Ref, {'title': 'Note 2', 'userId': 'user123'});

final userRef = FirebaseFirestore.instance.collection('users').doc('user123');
batch.update(userRef, {'noteCount': FieldValue.increment(2)});

// Commit all operations atomically
await batch.commit();
```

#### Multiple Operations in One Batch

You can mix different operation types:

```dart
final batch = FirebaseFirestore.instance.batch();

// Create new note
final newNoteRef = FirebaseFirestore.instance.collection('notes').doc();
batch.set(newNoteRef, {
  'title': 'New Note',
  'userId': 'user123',
  'createdAt': FieldValue.serverTimestamp(),
});

// Update user's note count
final userRef = FirebaseFirestore.instance.collection('users').doc('user123');
batch.update(userRef, {
  'noteCount': FieldValue.increment(1),
});

// Delete old note
final oldNoteRef = FirebaseFirestore.instance.collection('notes').doc('old-note');
batch.delete(oldNoteRef);

// All operations succeed or fail together
await batch.commit();
```

#### Atomic Operations

Batch operations are atomic:

- **All operations succeed** - If commit() succeeds, all operations are applied
- **All operations fail** - If any operation fails, none are applied
- **No partial updates** - You won't have some operations succeed and others fail

**Use cases:**
- Maintaining data consistency across multiple documents
- Updating related documents together
- Ensuring all-or-nothing operations

**Example: Creating note with tag updates:**
```dart
final batch = FirebaseFirestore.instance.batch();

// Create note
final noteRef = FirebaseFirestore.instance.collection('notes').doc();
batch.set(noteRef, {
  'title': 'Meeting Notes',
  'tags': ['work', 'meetings'],
  'userId': 'user123',
});

// Update tag counts
final workTagRef = FirebaseFirestore.instance
    .collection('tags')
    .doc('work');
batch.update(workTagRef, {
  'noteCount': FieldValue.increment(1),
});

final meetingsTagRef = FirebaseFirestore.instance
    .collection('tags')
    .doc('meetings');
batch.update(meetingsTagRef, {
  'noteCount': FieldValue.increment(1),
});

await batch.commit(); // All succeed or all fail
```

#### Maximum 500 Operations per Batch

Batches are limited to 500 operations:

```dart
// BAD: Exceeds limit
final batch = FirebaseFirestore.instance.batch();
for (int i = 0; i < 600; i++) {
  final ref = collectionRef.doc('note$i');
  batch.set(ref, {'title': 'Note $i'});
}
await batch.commit(); // Fails: too many operations

// GOOD: Split into multiple batches
for (var chunk in _chunkList(notes, 500)) {
  final batch = FirebaseFirestore.instance.batch();
  for (var note in chunk) {
    batch.set(note.ref, note.data);
  }
  await batch.commit();
}
```

#### Committing Batch

Commit the batch to execute all operations:

```dart
try {
  await batch.commit();
  print('Batch write successful');
} on FirebaseException catch (e) {
  print('Batch write failed: ${e.message}');
  // Handle error
}
```

**Error handling:** If any operation in the batch fails, the entire batch fails and no operations are applied.

### 2.6 Write Time Behavior

Understanding how Firestore handles timestamps and conflicts.

#### Server Timestamps

Use `FieldValue.serverTimestamp()` for accurate, consistent timestamps:

```dart
await noteRef.set({
  'title': 'My Note',
  'createdAt': FieldValue.serverTimestamp(),
  'updatedAt': FieldValue.serverTimestamp(),
});
```

**Benefits:**
- Uses server time (not client time)
- Consistent across all clients
- Handles clock skew automatically
- Resolves to Timestamp when read

**How it works:**
- During write: Placeholder value is sent
- Server replaces with actual server timestamp
- When read: Returns Timestamp object

#### Client-Side Time vs Server Time

**Client-side time (not recommended):**
```dart
// BAD: Uses client's clock (can be wrong)
await noteRef.set({
  'createdAt': DateTime.now(),
});
```

**Problems:**
- Client's clock might be wrong
- Different clients have different times
- Can cause ordering issues

**Server time (recommended):**
```dart
// GOOD: Uses server time
await noteRef.set({
  'createdAt': FieldValue.serverTimestamp(),
});
```

#### Conflict Resolution

Firestore handles write conflicts automatically:

**Last write wins:**
- If two clients update the same document simultaneously
- The last write to reach the server wins
- No errors are thrown
- Data from first write may be lost

**Using transactions for conflict prevention:**
```dart
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final currentData = snapshot.data()!;
  
  // Modify based on current data
  transaction.update(noteRef, {
    'viewCount': (currentData['viewCount'] as int) + 1,
  });
});
```

Transactions ensure you're working with the latest data and prevent lost updates.

### 2.7 Field Value Transforms

FieldValue provides special operations that execute on the server.

#### FieldValue.delete()

Remove a field from a document:

```dart
await noteRef.update({
  'oldField': FieldValue.delete(),
  'title': 'Updated Title', // Can update and delete in same operation
});
```

**Use cases:**
- Removing optional fields
- Cleaning up deprecated fields
- Conditional field removal

#### FieldValue.serverTimestamp()

Set field to server timestamp:

```dart
await noteRef.update({
  'updatedAt': FieldValue.serverTimestamp(),
});
```

**Always use for timestamps** - ensures consistency across clients.

#### FieldValue.increment(n)

Atomically increment/decrement numeric fields:

```dart
// Increment
await noteRef.update({
  'viewCount': FieldValue.increment(1),
  'wordCount': FieldValue.increment(50),
});

// Decrement
await noteRef.update({
  'viewCount': FieldValue.increment(-1),
});
```

**Benefits:**
- Atomic (no race conditions)
- Works even if field doesn't exist (starts at 0)
- Safe for concurrent updates

#### FieldValue.arrayUnion(elements)

Add elements to array (no duplicates):

```dart
await noteRef.update({
  'tags': FieldValue.arrayUnion(['work', 'important']),
  'collaborators': FieldValue.arrayUnion([
    {'userId': 'user456', 'role': 'editor'}
  ]),
});
```

**Behavior:**
- Adds elements only if not already present
- For objects, entire object must match to be considered duplicate
- Preserves array order

#### FieldValue.arrayRemove(elements)

Remove elements from array:

```dart
await noteRef.update({
  'tags': FieldValue.arrayRemove(['work']),
  'collaborators': FieldValue.arrayRemove([
    {'userId': 'user456', 'role': 'editor'}
  ]),
});
```

**Behavior:**
- Removes matching elements
- For objects, entire object must match
- No error if element doesn't exist

## 3. Practical Examples

### Example: Creating a New Note

```dart
Future<String> createNote({
  required String userId,
  required String title,
  required String content,
  List<String>? tags,
}) async {
  final noteRef = FirebaseFirestore.instance.collection('notes').doc();
  
  await noteRef.set({
    'title': title,
    'content': content,
    'userId': userId,
    'tags': tags ?? [],
    'isPinned': false,
    'isArchived': false,
    'wordCount': content.split(' ').length,
    'createdAt': FieldValue.serverTimestamp(),
    'updatedAt': FieldValue.serverTimestamp(),
  });
  
  return noteRef.id;
}
```

### Example: Updating Note Content

```dart
Future<void> updateNoteContent(String noteId, String newContent) async {
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc(noteId);
  
  await noteRef.update({
    'content': newContent,
    'wordCount': newContent.split(' ').length,
    'updatedAt': FieldValue.serverTimestamp(),
  });
}
```

### Example: Adding Tags to Note

```dart
Future<void> addTagsToNote(String noteId, List<String> newTags) async {
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc(noteId);
  
  await noteRef.update({
    'tags': FieldValue.arrayUnion(newTags),
    'updatedAt': FieldValue.serverTimestamp(),
  });
}
```

### Example: Toggling Pin Status

```dart
Future<void> togglePinStatus(String noteId, bool currentStatus) async {
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc(noteId);
  
  await noteRef.update({
    'isPinned': !currentStatus,
    'updatedAt': FieldValue.serverTimestamp(),
  });
}
```

### Example: Batch Create Notes with User Update

```dart
Future<void> createMultipleNotes({
  required String userId,
  required List<Map<String, dynamic>> notesData,
}) async {
  final batch = FirebaseFirestore.instance.batch();
  final userRef = FirebaseFirestore.instance
      .collection('users')
      .doc(userId);
  
  for (var noteData in notesData) {
    final noteRef = FirebaseFirestore.instance.collection('notes').doc();
    batch.set(noteRef, {
      ...noteData,
      'userId': userId,
      'createdAt': FieldValue.serverTimestamp(),
      'updatedAt': FieldValue.serverTimestamp(),
    });
  }
  
  // Update user's note count
  batch.update(userRef, {
    'noteCount': FieldValue.increment(notesData.length),
  });
  
  await batch.commit();
}
```

### Example: Deleting Note and Updating Counts

```dart
Future<void> deleteNote(String noteId, String userId) async {
  final batch = FirebaseFirestore.instance.batch();
  
  // Delete note
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc(noteId);
  batch.delete(noteRef);
  
  // Decrement user's note count
  final userRef = FirebaseFirestore.instance
      .collection('users')
      .doc(userId);
  batch.update(userRef, {
    'noteCount': FieldValue.increment(-1),
  });
  
  // Decrement tag counts (if note had tags)
  // Note: You'd need to read note first to get tags, then update
  final noteDoc = await noteRef.get();
  if (noteDoc.exists) {
    final tags = noteDoc.data()?['tags'] as List<dynamic>? ?? [];
    for (var tag in tags) {
      final tagRef = FirebaseFirestore.instance
          .collection('tags')
          .doc(tag.toString());
      batch.update(tagRef, {
        'noteCount': FieldValue.increment(-1),
      });
    }
  }
  
  await batch.commit();
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**set() vs update():**
- **set():** Use when creating new documents or replacing entirely
- **update():** Use when modifying specific fields of existing documents
- **set() with merge:** Use when you're unsure if document exists and want partial update
- **Recommendation:** Use `update()` for known existing documents. Use `set()` with merge when document might not exist.

**Manual ID vs Auto-generated ID:**
- **Manual ID:** Use when you have natural identifier (slug, username)
- **Auto-generated ID:** Use when you need guaranteed uniqueness, no natural ID
- **Recommendation:** Prefer auto-generated IDs unless you have a strong reason for manual IDs (SEO, readability, etc.)

**Single writes vs Batch writes:**
- **Single writes:** Simpler, good for independent operations
- **Batch writes:** Atomic, good for related operations that must succeed together
- **Recommendation:** Use batches when operations are related (e.g., creating note + updating counts). Use single writes for independent operations.

**Client timestamps vs Server timestamps:**
- **Client timestamps:** Never use (clock can be wrong)
- **Server timestamps:** Always use (consistent, accurate)
- **Recommendation:** Always use `FieldValue.serverTimestamp()` for any timestamp fields.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Using set() without merge, overwriting document**
```dart
// BAD: Deletes all other fields
await noteRef.set({
  'title': 'New Title',
});
// If document had 'content', 'tags', etc., they're now gone!

// GOOD: Use update() or set() with merge
await noteRef.update({
  'title': 'New Title',
});
// OR
await noteRef.set({
  'title': 'New Title',
}, SetOptions(merge: true));
```

**2. Using update() on non-existent document**
```dart
// BAD: Throws error if document doesn't exist
await noteRef.update({'title': 'New Title'});

// GOOD: Check existence or use set() with merge
if ((await noteRef.get()).exists) {
  await noteRef.update({'title': 'New Title'});
}
// OR
await noteRef.set({
  'title': 'New Title',
}, SetOptions(merge: true));
```

**3. Using client-side DateTime instead of server timestamp**
```dart
// BAD: Client time can be wrong
await noteRef.set({
  'createdAt': DateTime.now(),
});

// GOOD: Use server timestamp
await noteRef.set({
  'createdAt': FieldValue.serverTimestamp(),
});
```

**4. Not handling batch size limits**
```dart
// BAD: May exceed 500 operation limit
final batch = FirebaseFirestore.instance.batch();
for (var note in manyNotes) {
  batch.set(note.ref, note.data);
}
await batch.commit(); // Fails if > 500

// GOOD: Split into chunks
for (var chunk in _chunkList(manyNotes, 500)) {
  final batch = FirebaseFirestore.instance.batch();
  for (var note in chunk) {
    batch.set(note.ref, note.data);
  }
  await batch.commit();
}
```

### Data Integrity Mistakes

**1. Not using atomic operations for counters**
```dart
// BAD: Race condition possible
final doc = await noteRef.get();
final currentCount = doc.data()?['viewCount'] ?? 0;
await noteRef.update({'viewCount': currentCount + 1});

// GOOD: Atomic increment
await noteRef.update({
  'viewCount': FieldValue.increment(1),
});
```

**2. Updating entire map instead of individual fields**
```dart
// BAD: Loses other map fields
await noteRef.update({
  'settings': {'theme': 'dark'}, // Other settings lost!
});

// GOOD: Update individual fields
await noteRef.update({
  'settings.theme': 'dark',
  'settings.fontSize': 14,
});
```

**3. Not using batches for related operations**
```dart
// BAD: Partial failure possible
await noteRef.set({'title': 'Note'});
await userRef.update({'noteCount': FieldValue.increment(1)});
// If second fails, note exists but count is wrong

// GOOD: Atomic batch
final batch = FirebaseFirestore.instance.batch();
batch.set(noteRef, {'title': 'Note'});
batch.update(userRef, {'noteCount': FieldValue.increment(1)});
await batch.commit(); // All or nothing
```

### Performance Pitfalls

**1. Too many individual writes instead of batch**
```dart
// BAD: Multiple round trips
for (var note in notes) {
  await noteRef.set(note.data); // N network calls
}

// GOOD: Single batch
final batch = FirebaseFirestore.instance.batch();
for (var note in notes) {
  batch.set(note.ref, note.data);
}
await batch.commit(); // 1 network call
```

**2. Not handling write errors**
```dart
// BAD: Crashes on error
await noteRef.update({'title': 'New Title'});

// GOOD: Handle errors
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'not-found') {
    // Document doesn't exist
  } else if (e.code == 'permission-denied') {
    // Security rules blocked
  }
}
```

## 6. Summary & Checklist

### Summary

- **set()** creates or replaces documents. Use with `SetOptions(merge: true)` to merge instead of replace.
- **update()** modifies specific fields. Fails if document doesn't exist.
- **add()** creates documents with auto-generated IDs.
- **delete()** removes documents completely.
- **Batch writes** perform multiple operations atomically (max 500 operations).
- **Server timestamps** ensure consistent time across clients. Always use `FieldValue.serverTimestamp()`.
- **Field transforms** (increment, arrayUnion, arrayRemove) provide atomic operations.

### Checklist: You Are Ready When You Can...

- [ ] Create a document using `set()` and handle merge behavior
- [ ] Create a document with auto-generated ID using `add()`
- [ ] Update specific fields using `update()` with error handling
- [ ] Delete documents and understand subcollection behavior
- [ ] Use `set()` with `SetOptions(merge: true)` for partial updates
- [ ] Update nested fields using dot notation
- [ ] Use `FieldValue.increment()` for atomic counter updates
- [ ] Use `FieldValue.arrayUnion()` and `arrayRemove()` for array operations
- [ ] Create batch writes with multiple operations
- [ ] Understand batch size limits (500 operations) and how to handle them
- [ ] Use `FieldValue.serverTimestamp()` instead of client-side time
- [ ] Choose between `set()`, `update()`, and `add()` based on use case
- [ ] Handle write errors and non-existent documents
- [ ] Use batches for related operations that must succeed together

### Verification Steps

**Test your write operations:**

1. **Test document creation:**
```dart
final noteRef = FirebaseFirestore.instance.collection('notes').doc();
await noteRef.set({
  'title': 'Test Note',
  'createdAt': FieldValue.serverTimestamp(),
});

final doc = await noteRef.get();
assert(doc.exists);
print('Document created with ID: ${doc.id}');
```

2. **Test update with merge:**
```dart
// Create document
await noteRef.set({'title': 'Original', 'content': 'Content'});

// Update with merge
await noteRef.set({
  'title': 'Updated',
}, SetOptions(merge: true));

// Verify both fields exist
final doc = await noteRef.get();
final data = doc.data()!;
assert(data['title'] == 'Updated');
assert(data['content'] == 'Content');
```

3. **Test batch write:**
```dart
final batch = FirebaseFirestore.instance.batch();
final ref1 = collectionRef.doc('test1');
final ref2 = collectionRef.doc('test2');

batch.set(ref1, {'title': 'Note 1'});
batch.set(ref2, {'title': 'Note 2'});
await batch.commit();

// Verify both created
final doc1 = await ref1.get();
final doc2 = await ref2.get();
assert(doc1.exists && doc2.exists);
```

### Next Steps

Once you can write data correctly, you're ready to:
- Build complex queries (Chapter 6)
- Set up real-time listeners (Chapter 7)
- Use transactions for complex operations (Chapter 8)
