# 9. DATA MODELING PATTERNS

## 1. What This Chapter Covers

This chapter teaches you how to design effective data models in Firestore for your Notes application. You'll learn:

- Core principles for Firestore data modeling (access patterns first, denormalization)
- How to structure user, notes, and tags data models
- Relationship patterns (one-to-one, one-to-many, many-to-many)
- When to use subcollections vs root collections
- Anti-patterns to avoid (deep nesting, large documents)
- Scalability patterns for high-traffic applications

Effective data modeling is crucial because it directly impacts query performance, security rules complexity, and billing costs. Understanding Firestore's document model, relationship patterns, and scalability techniques helps you build efficient, maintainable applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Data Modeling Principles

Firestore data modeling follows different principles than relational databases.

#### Access Patterns First

Design your data model based on how you'll query it:

```dart
// Think about queries first:
// "Get all notes for a user"
// "Get notes by tag"
// "Get notes created this week"

// Then design structure:
/users/{userId}
  - displayName, email, etc.

/notes/{noteId}
  - userId: "user123"  // For filtering by user
  - tags: ["work", "personal"]  // For filtering by tag
  - createdAt: Timestamp  // For date range queries
```

**Process:**
1. List all queries your app needs
2. Design data structure to support those queries efficiently
3. Denormalize data if needed for query performance

**Example queries for Notes app:**
- Get user's notes (filtered by userId)
- Get notes by tag (arrayContains)
- Get pinned notes (isPinned == true)
- Get notes created in date range (createdAt range query)
- Get shared notes (collaborators arrayContains userId)

#### Denormalization for Performance

Duplicate data to avoid joins and improve query performance:

```dart
// BAD: Normalized (requires multiple reads)
/notes/{noteId}
  - userId: "user123"
  - authorRef: "/users/user123"  // Need to read user separately

// GOOD: Denormalized (single read)
/notes/{noteId}
  - userId: "user123"
  - authorName: "John Doe"  // Duplicated for quick access
  - authorEmail: "john@example.com"  // Duplicated
```

**When to denormalize:**
- Data read together frequently
- Data changes infrequently
- Query performance is critical
- Data size is small

**When to normalize:**
- Data changes frequently (keep single source of truth)
- Data is large (avoid duplication)
- Data is rarely read together

**Example:**
```dart
// Denormalize author info in notes (read together, changes rarely)
/notes/{noteId}
  - title: "Meeting Notes"
  - authorId: "user123"
  - authorName: "John Doe"  // Denormalized
  - authorAvatar: "https://..."  // Denormalized

// Keep user document as source of truth
/users/{userId}
  - displayName: "John Doe"  // Update here, sync to notes
  - email: "john@example.com"
```

#### Collection vs Subcollection Decisions

Choose based on query patterns:

**Root collection:**
```dart
/notes/{noteId}
  - userId: "user123"
```
- Query across all users: `collection('notes').where('tags', arrayContains: 'work')`
- Global search
- Admin features

**Subcollection:**
```dart
/users/{userId}/notes/{noteId}
```
- Query only user's notes: `collection('users').doc(userId).collection('notes')`
- Natural data isolation
- Simpler security rules

**Decision factors:**
- Need to query across users? → Root collection
- Data is user-private? → Subcollection
- Need collection group queries? → Subcollection with same name

### 2.2 User Data Model

Structure user data for efficient access.

#### User Profile Document

Store user profile information:

```dart
/users/{userId}
{
  "displayName": "John Doe",
  "email": "john@example.com",
  "avatarUrl": "https://...",
  "bio": "Software developer",
  "createdAt": Timestamp,
  "lastSeenAt": Timestamp,
}
```

**Best practices:**
- Use Firebase Auth UID as document ID
- Store frequently accessed data in user document
- Keep document size reasonable (< 1 MB)

#### User Preferences

Store user settings and preferences:

```dart
/users/{userId}
{
  "preferences": {
    "theme": "dark",
    "fontSize": 14,
    "notificationsEnabled": true,
    "language": "en",
  }
}
```

**Alternative: Separate preferences document**
```dart
/users/{userId}/preferences/{prefId}
{
  "theme": "dark",
  "fontSize": 14,
}
```

**When to use separate document:**
- Preferences are large
- Preferences change frequently
- You want to version preferences

#### User Statistics

Store aggregated user statistics:

```dart
/users/{userId}
{
  "stats": {
    "noteCount": 42,
    "totalWordCount": 15000,
    "tagsCount": 12,
    "lastNoteCreatedAt": Timestamp,
  }
}
```

**Keep statistics updated:**
- Update when notes are created/deleted
- Use transactions or batch operations
- Consider using Cloud Functions for complex aggregations

### 2.3 Notes Data Model

Structure notes for efficient queries and updates.

#### Note Document Structure

Design note document to support your queries:

```dart
/notes/{noteId}
{
  "title": "Meeting Notes",
  "content": "Discussion about Q1 goals...",
  "userId": "user123",  // For filtering by user
  "authorName": "John Doe",  // Denormalized
  "tags": ["work", "meetings"],  // For tag queries
  "isPinned": false,
  "isArchived": false,
  "isPublic": false,
  "wordCount": 245,
  "createdAt": Timestamp,
  "updatedAt": Timestamp,
  "lastViewedAt": Timestamp,
  "collaborators": [
    {
      "userId": "user456",
      "displayName": "Jane Smith",  // Denormalized
      "role": "editor",
      "addedAt": Timestamp,
    }
  ],
  "collaboratorIds": ["user456"],  // For arrayContains queries
}
```

**Key fields:**
- `userId`: For user filtering
- `tags`: Array for tag queries
- `collaboratorIds`: Separate array for efficient queries
- Denormalized author info for quick display

#### Metadata (created, updated)

Always include timestamps:

```dart
{
  "createdAt": FieldValue.serverTimestamp(),
  "updatedAt": FieldValue.serverTimestamp(),
  "lastViewedAt": FieldValue.serverTimestamp(),
}
```

**Use cases:**
- Sorting by date
- Date range queries
- Showing "last updated" to users
- Analytics and reporting

#### Content Storage

Store content efficiently:

**Small content (< 100 KB):**
```dart
/notes/{noteId}
{
  "content": "Full note content here...",
}
```

**Large content (> 100 KB):**
```dart
/notes/{noteId}
{
  "contentPreview": "First 500 characters...",
  "contentRef": "/notes/note123/fullContent",
}

/notes/{noteId}/fullContent/{contentId}
{
  "text": "Very long content...",
  "chunkIndex": 0,
}
```

**Or use Cloud Storage:**
```dart
/notes/{noteId}
{
  "contentPreview": "First 500 characters...",
  "contentStoragePath": "notes/note123/content.txt",
}
```

#### Tags as Array or Subcollection

Choose based on your needs:

**Array approach:**
```dart
/notes/{noteId}
{
  "tags": ["work", "personal", "urgent"],
}
```

**Pros:**
- Simple queries: `where('tags', arrayContains: 'work')`
- Fast reads (no extra queries)
- Good for simple tag names

**Cons:**
- Can't store tag metadata (color, description)
- Hard to update tag names globally
- Limited to simple strings

**Subcollection approach:**
```dart
/notes/{noteId}
{
  "tagIds": ["tag123", "tag456"],
}

/tags/{tagId}
{
  "name": "work",
  "color": "#FF5733",
  "description": "Work-related notes",
  "noteCount": 42,
}
```

**Pros:**
- Rich tag metadata
- Easy to update tag properties
- Can track tag usage

**Cons:**
- Requires additional queries
- More complex data model

**Recommendation:** Use arrays for simple tags. Use subcollection if you need tag colors, descriptions, or tag management features.

### 2.4 Tags Data Model

Structure tags for efficient queries and management.

#### Tag Document

Store tag information:

```dart
/tags/{tagId}
{
  "name": "work",
  "userId": "user123",  // Tag owner
  "color": "#FF5733",
  "description": "Work-related notes",
  "noteCount": 42,  // Denormalized count
  "createdAt": Timestamp,
  "lastUsedAt": Timestamp,
}
```

**Or use tag name as ID:**
```dart
/tags/{tagName}  // e.g., "work"
{
  "color": "#FF5733",
  "noteCount": 42,
}
```

**Pros of name as ID:**
- Simpler queries (no need to look up tag ID)
- Natural uniqueness per user

**Cons:**
- Can't have special characters easily
- Harder to rename tags

#### Tag Metadata

Store additional tag information:

```dart
/tags/{tagId}
{
  "name": "work",
  "metadata": {
    "color": "#FF5733",
    "icon": "briefcase",
    "category": "professional",
    "isSystemTag": false,
  }
}
```

#### Tag Usage Counts

Maintain tag usage counts:

```dart
/tags/{tagId}
{
  "name": "work",
  "noteCount": 42,  // Updated when notes are tagged/untagged
}
```

**Update pattern:**
```dart
// When adding tag to note
await FirebaseFirestore.instance.runTransaction((transaction) async {
  // Add tag to note
  transaction.update(noteRef, {
    'tags': FieldValue.arrayUnion(['work']),
  });
  
  // Increment tag count
  final tagSnapshot = await transaction.get(tagRef);
  transaction.update(tagRef, {
    'noteCount': (tagSnapshot.data()?['noteCount'] ?? 0) + 1,
  });
});
```

### 2.5 Relationship Patterns

Model relationships between entities.

#### One-to-One

Store related data in the same document or reference:

**Embedded:**
```dart
/users/{userId}
{
  "profile": {
    "displayName": "John",
    "bio": "...",
  },
  "settings": {
    "theme": "dark",
  }
}
```

**Referenced:**
```dart
/users/{userId}
{
  "settingsRef": "/users/user123/settings/prefs",
}

/users/{userId}/settings/prefs
{
  "theme": "dark",
}
```

**Use embedded when:**
- Data is always read together
- Data is small
- Data changes infrequently

#### One-to-Many (Subcollection)

Use subcollections for one-to-many relationships:

```dart
/users/{userId}/notes/{noteId}
{
  "title": "Note",
  "content": "...",
}
```

**Benefits:**
- Natural hierarchy
- Automatic data isolation
- Simpler security rules
- Easy to query user's notes

**Query:**
```dart
final userNotes = await FirebaseFirestore.instance
    .collection('users')
    .doc(userId)
    .collection('notes')
    .get();
```

#### One-to-Many (Reference)

Use references when you need to query across parents:

```dart
/notes/{noteId}
{
  "userId": "user123",  // Reference to user
  "title": "Note",
}

/users/{userId}
{
  "displayName": "John",
}
```

**Benefits:**
- Query across all notes
- Global search
- Collection group queries

**Query:**
```dart
final allWorkNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

#### Many-to-Many

Model many-to-many relationships:

**Array approach (simple):**
```dart
/notes/{noteId}
{
  "tags": ["work", "personal"],
}

/tags/{tagId}
{
  "name": "work",
  "noteIds": ["note1", "note2"],  // Reverse reference
}
```

**Junction collection (complex):**
```dart
/notes/{noteId}
{
  "title": "Note",
}

/tags/{tagId}
{
  "name": "work",
}

/noteTags/{noteTagId}
{
  "noteId": "note123",
  "tagId": "tag456",
  "addedAt": Timestamp,
}
```

**Use junction collection when:**
- You need relationship metadata (when added, who added)
- Relationships are complex
- You need to query relationships efficiently

### 2.6 Subcollection vs Root Collection

Choose based on query patterns and data isolation needs.

#### When to Use Subcollections

Use subcollections when:

**Data is user-scoped:**
```dart
/users/{userId}/notes/{noteId}
```
- Notes belong to specific user
- Never need to query across all users
- Natural data isolation

**Simpler security rules:**
```javascript
match /users/{userId}/notes/{noteId} {
  allow read, write: if request.auth.uid == userId;
}
```

**Hierarchical data:**
```dart
/notes/{noteId}/comments/{commentId}
/notes/{noteId}/versions/{versionId}
```

#### When to Use Root Collections

Use root collections when:

**Need to query across users:**
```dart
/notes/{noteId}
  - userId: "user123"
```
- Global search
- Admin features
- Analytics across all users

**Many-to-many relationships:**
```dart
/notes/{noteId}
/tags/{tagId}
```
- Notes can have multiple tags
- Tags can belong to multiple notes

**Shared/public data:**
```dart
/publicNotes/{noteId}
```
- Public notes visible to all users

#### Query Considerations

**Subcollection queries:**
```dart
// Query user's notes
collection('users').doc(userId).collection('notes').get()

// Can't easily query across users
// Need collection group query
collectionGroup('notes').where('tags', arrayContains: 'work')
```

**Root collection queries:**
```dart
// Query all notes
collection('notes').where('tags', arrayContains: 'work').get()

// Filter by user
collection('notes')
    .where('userId', isEqualTo: userId)
    .where('tags', arrayContains: 'work')
    .get()
```

### 2.7 Anti-Patterns to Avoid

Common mistakes that hurt performance and maintainability.

#### Deep Nesting

Avoid nesting too deeply:

```dart
// BAD: 4+ levels deep
{
  "user": {
    "profile": {
      "settings": {
        "preferences": {
          "theme": "dark"  // Too deep!
        }
      }
    }
  }
}

// GOOD: Flatten structure
{
  "userTheme": "dark",
  "userSettings": {
    "theme": "dark",
    "fontSize": 14,
  }
}
```

**Problems with deep nesting:**
- Complex queries
- Hard to update
- Security rules become complex

#### Large Documents

Keep documents under 1 MB:

```dart
// BAD: Large document
{
  "content": "Very long content...",  // Could be 2 MB
  "history": [...1000 items...],
  "attachments": [...large data...],
}

// GOOD: Split into subcollections
{
  "contentPreview": "First 500 chars...",
}

/notes/{noteId}/fullContent/{contentId}
{
  "text": "Full content...",
}

/notes/{noteId}/history/{historyId}
{
  "change": "...",
  "timestamp": Timestamp,
}
```

#### Unnecessary References

Don't create references when you can embed:

```dart
// BAD: Unnecessary reference
{
  "authorRef": "/users/user123",  // Need extra read
}

// GOOD: Denormalize if read together
{
  "authorId": "user123",
  "authorName": "John Doe",  // Denormalized
}
```

**Use references when:**
- Data changes frequently
- Data is large
- Data is rarely read together

#### NoSQL Array of Arrays

Avoid nested arrays:

```dart
// BAD: Array of arrays
{
  "tags": [
    ["work", "urgent"],  // Nested array
    ["personal"],
  ]
}

// GOOD: Flat array
{
  "tags": ["work", "urgent", "personal"]
}
```

**Problems:**
- Can't query nested arrays easily
- Complex to update
- Hard to maintain

### 2.8 Scalability Patterns

Design for scale from the start.

#### Sharding Strategies

Shard data for high write volume:

**Distributed counters:**
```dart
/counters/noteViews
  /shards/shard0 { count: 1000 }
  /shards/shard1 { count: 1200 }
  /shards/shard2 { count: 1100 }
```

**Sharded collections:**
```dart
/notes_shard0/{noteId}
/notes_shard1/{noteId}
/notes_shard2/{noteId}
```

**Shard selection:**
```dart
final shardId = hash(noteId) % numShards;
final shardRef = collection('notes_shard$shardId');
```

#### Partitioning

Partition data by access patterns:

**By user:**
```dart
/users/{userId}/notes/{noteId}
```

**By date:**
```dart
/notes_2024_01/{noteId}
/notes_2024_02/{noteId}
```

**By region:**
```dart
/notes_us/{noteId}
/notes_eu/{noteId}
```

#### Time-Based Collections

Use time-based collections for time-series data:

```dart
/notes_2024_01/{noteId}  // January 2024
/notes_2024_02/{noteId}  // February 2024
```

**Benefits:**
- Easier to archive old data
- Better query performance
- Can delete old collections

**Query across partitions:**
```dart
// Query current month
final currentMonth = DateTime.now();
final monthCollection = 'notes_${currentMonth.year}_${currentMonth.month}';
final notes = await collection(monthCollection).get();
```

## 3. Practical Examples

### Example: Complete Notes App Data Model

```dart
// User document
/users/{userId}
{
  "displayName": "John Doe",
  "email": "john@example.com",
  "avatarUrl": "https://...",
  "createdAt": Timestamp,
  "stats": {
    "noteCount": 42,
    "totalWordCount": 15000,
  },
  "preferences": {
    "theme": "dark",
    "fontSize": 14,
  }
}

// Note document (root collection for global queries)
/notes/{noteId}
{
  "title": "Meeting Notes",
  "content": "Discussion about Q1 goals...",
  "contentPreview": "Discussion about...",  // First 200 chars
  "userId": "user123",
  "authorName": "John Doe",  // Denormalized
  "authorAvatar": "https://...",  // Denormalized
  "tags": ["work", "meetings"],
  "tagIds": ["tag123", "tag456"],  // For efficient queries
  "isPinned": false,
  "isArchived": false,
  "isPublic": false,
  "wordCount": 245,
  "viewCount": 10,
  "likeCount": 5,
  "createdAt": Timestamp,
  "updatedAt": Timestamp,
  "collaborators": [
    {
      "userId": "user456",
      "displayName": "Jane Smith",
      "role": "editor",
      "addedAt": Timestamp,
    }
  ],
  "collaboratorIds": ["user456"],
}

// Tag document
/tags/{tagId}
{
  "name": "work",
  "userId": "user123",
  "color": "#FF5733",
  "noteCount": 12,
  "createdAt": Timestamp,
  "lastUsedAt": Timestamp,
}

// Shared notes (junction collection for many-to-many)
/sharedNotes/{shareId}
{
  "noteId": "note123",
  "sharedBy": "user123",
  "sharedWith": "user456",
  "permission": "read",  // or "write"
  "sharedAt": Timestamp,
}
```

### Example: Query Patterns

```dart
// Get user's notes
final userNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .orderBy('createdAt', descending: true)
    .get();

// Get notes by tag
final workNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: userId)
    .where('tags', arrayContains: 'work')
    .get();

// Get shared notes
final sharedNotes = await FirebaseFirestore.instance
    .collection('sharedNotes')
    .where('sharedWith', isEqualTo: userId)
    .get();

// Get public notes
final publicNotes = await FirebaseFirestore.instance
    .collection('notes')
    .where('isPublic', isEqualTo: true)
    .orderBy('createdAt', descending: true)
    .limit(50)
    .get();
```

## 4. Common Design Decisions

### Why Choose This Approach

**Root collection vs Subcollection:**
- **Root collection:** Global queries, admin features, cross-user search
- **Subcollection:** User-scoped data, simpler security, natural isolation
- **Recommendation:** Use root collections if you need global queries. Use subcollections for user-private data.

**Array tags vs Tag subcollection:**
- **Array:** Simple, fast, good for basic tags
- **Subcollection:** Rich metadata, tag management, more complex
- **Recommendation:** Start with arrays. Move to subcollection if you need tag colors, descriptions, or management features.

**Denormalization vs Normalization:**
- **Denormalize:** Frequently read together, changes rarely, small data
- **Normalize:** Changes frequently, large data, rarely read together
- **Recommendation:** Denormalize author info in notes. Keep user document as source of truth.

**Embedded vs Referenced:**
- **Embedded:** Always read together, small, changes rarely
- **Referenced:** Large, changes frequently, rarely read together
- **Recommendation:** Embed user preferences in user document. Reference large content or frequently changing data.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Deep nesting (4+ levels)**
```dart
// BAD: Too deep
{
  "user": {
    "profile": {
      "settings": {
        "preferences": {"theme": "dark"}
      }
    }
  }
}

// GOOD: Flatten
{
  "userTheme": "dark",
  "userSettings": {"theme": "dark"}
}
```

**2. Large documents (> 1 MB)**
```dart
// BAD: Could exceed 1 MB
{
  "content": veryLongContent,
  "history": [...1000 items...],
}

// GOOD: Split into subcollections
{
  "contentPreview": "...",
}
// /notes/{noteId}/fullContent/{contentId}
// /notes/{noteId}/history/{historyId}
```

**3. Not denormalizing frequently read data**
```dart
// BAD: Extra read needed
{
  "authorRef": "/users/user123",
}

// GOOD: Denormalize
{
  "authorId": "user123",
  "authorName": "John Doe",
}
```

**4. Using references when embedding is better**
```dart
// BAD: Unnecessary reference
{
  "settingsRef": "/users/user123/settings",
}

// GOOD: Embed if small and read together
{
  "settings": {"theme": "dark"},
}
```

### Performance Pitfalls

**1. Not designing for access patterns**
```dart
// BAD: Structure doesn't support queries
{
  "data": {...},  // Can't query efficiently
}

// GOOD: Structure supports queries
{
  "userId": "user123",  // For filtering
  "tags": ["work"],  // For tag queries
  "createdAt": Timestamp,  // For date queries
}
```

**2. Over-normalization**
```dart
// BAD: Too many reads
// Need to read note, then user, then tags separately

// GOOD: Denormalize for read performance
{
  "title": "Note",
  "authorName": "John",  // Denormalized
  "tags": ["work"],  // Embedded
}
```

**3. Not using indexes**
```dart
// BAD: Query fails or is slow
.where('userId', isEqualTo: userId)
.where('isPinned', isEqualTo: true)
.orderBy('createdAt')  // Needs composite index

// GOOD: Create required indexes
// Firestore provides link to create index
```

## 6. Summary & Checklist

### Summary

- **Access patterns first:** Design data structure based on how you'll query it
- **Denormalization:** Duplicate data for read performance when it changes rarely
- **Subcollections:** Use for user-scoped, hierarchical data with natural isolation
- **Root collections:** Use for global queries, cross-user data, admin features
- **Relationship patterns:** Choose embedded vs referenced vs subcollection based on access patterns
- **Anti-patterns:** Avoid deep nesting, large documents, unnecessary references
- **Scalability:** Use sharding, partitioning, and time-based collections for scale

### Checklist: You Are Ready When You Can...

- [ ] Design data model based on access patterns (queries first)
- [ ] Decide when to denormalize vs normalize data
- [ ] Choose between root collections and subcollections
- [ ] Structure user, notes, and tags data models effectively
- [ ] Model one-to-one, one-to-many, and many-to-many relationships
- [ ] Avoid deep nesting (keep to 2-3 levels)
- [ ] Keep documents under 1 MB (split large data)
- [ ] Denormalize frequently read data
- [ ] Use embedded data when appropriate
- [ ] Implement sharding strategies for high write volume
- [ ] Use time-based collections for time-series data
- [ ] Design for scalability from the start

### Verification Steps

**Test your data model:**

1. **Verify query patterns:**
```dart
// Test all your app's queries
final userNotes = await collection('notes')
    .where('userId', isEqualTo: userId)
    .get();

final workNotes = await collection('notes')
    .where('tags', arrayContains: 'work')
    .get();
```

2. **Verify document size:**
```dart
final doc = await noteRef.get();
final data = doc.data()!;
final jsonString = jsonEncode(data);
final sizeInBytes = utf8.encode(jsonString).length;
print('Document size: ${sizeInBytes / 1024} KB');
assert(sizeInBytes < 1024 * 1024); // < 1 MB
```

3. **Verify denormalization:**
```dart
// Check if you can display note without extra reads
final note = await noteRef.get();
final data = note.data()!;
print('Author: ${data['authorName']}');  // Should be denormalized
// Should not need to read user document
```

### Next Steps

Once you can design effective data models, you're ready to:
- Implement security rules (Chapter 10)
- Optimize performance (Chapter 11)
- Handle errors and edge cases (Chapter 13)
