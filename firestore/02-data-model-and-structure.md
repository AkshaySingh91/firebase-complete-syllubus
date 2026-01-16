# 2. DATA MODEL & STRUCTURE

## 1. What This Chapter Covers

This chapter explains how to structure data in Firestore for a production Notes application. You'll learn:

- How collections and documents organize your data
- What types of data you can store in fields
- How to model relationships between users, notes, and tags
- When to use subcollections vs root collections
- How collection group queries enable powerful searches

Understanding Firestore's data model is essential because it directly impacts query performance, security rules complexity, and billing costs. Poor data modeling leads to expensive queries, complex security rules, and slow app performance.

## 2. Core Concepts (Simple & Clear)

### 2.1 Understanding Collections

**Collections** are containers that hold documents. Think of them like folders in a filing cabinet.

#### Root Collections

Root collections live at the top level of your database. They're accessed directly:

```
/collectionName/documentId
```

**Example:**
```
/users/user123
/notes/note456
/tags/tag789
```

Root collections are best for:
- Independent entities (users, global settings)
- Data you query across all users
- Top-level resources that don't belong to a specific parent

#### Subcollections

Subcollections live inside documents. They create a hierarchical structure:

```
/collectionName/documentId/subcollectionName/subdocumentId
```

**Example:**
```
/users/user123/notes/note456
/users/user123/settings/preferences
```

Subcollections are best for:
- Data that logically belongs to a parent document
- Scoping queries to a specific user's data
- Organizing related data hierarchically

#### Top-level vs Nested Collections

**Top-level collection approach:**
```
/notes/note123
  - userId: "user123"
  - title: "My Note"
  - content: "..."
```

**Nested subcollection approach:**
```
/users/user123/notes/note123
  - title: "My Note"
  - content: "..."
```

**Key difference:** With top-level collections, you query all notes and filter by userId. With subcollections, you query only that user's notes directly. Subcollections provide better data isolation and simpler security rules.

### 2.2 Understanding Documents

**Documents** are the actual data containers. Each document is like a JSON object with a unique ID.

#### Document IDs

Firestore generates IDs automatically, or you can provide custom IDs:

**Auto-generated ID:**
```javascript
// Firestore creates: "aB3xK9mP2qR7tY"
const docRef = collection(db, 'notes').doc();
```

**Custom ID:**
```javascript
// You provide: "my-custom-note-id"
const docRef = collection(db, 'notes').doc('my-custom-note-id');
```

**When to use custom IDs:**
- You have a natural identifier (username, email, slug)
- You need predictable, readable IDs
- You're importing existing data

**When to use auto-generated IDs:**
- You don't have a natural identifier
- You want guaranteed uniqueness
- You prefer shorter, URL-safe IDs

#### Document Size Limits

Each document has a **1 MB size limit**. This includes:
- All field values
- Field names
- Metadata

**What fits in 1 MB:**
- ~500,000 characters of text
- ~100-200 typical note documents
- Small embedded arrays and maps

**What doesn't fit:**
- Large binary data (use Cloud Storage instead)
- Massive arrays (consider subcollections)
- Very long text content (split into chunks or use Storage)

#### Document Structure

Documents contain fields. Each field has a name and a value:

```json
{
  "title": "My First Note",
  "content": "This is the note content...",
  "createdAt": "2024-01-15T10:30:00Z",
  "userId": "user123",
  "tags": ["work", "important"],
  "isPinned": true
}
```

Documents are schemaless - you can add or remove fields without migrations. However, for production apps, maintain consistent field names and types across documents.

### 2.3 Field Values

Firestore supports several data types for field values.

#### Primitive Types

**String:** Text data
```json
"title": "My Note"
```

**Number:** Integers and floating-point numbers
```json
"wordCount": 150,
"rating": 4.5
```

**Boolean:** True or false
```json
"isPinned": true,
"isArchived": false
```

**Timestamp:** Date and time
```json
"createdAt": Timestamp(2024, 1, 15, 10, 30, 0)
```

**Null:** Absence of a value
```json
"deletedAt": null
```

**Bytes:** Binary data (limited use, prefer Cloud Storage for large files)
```json
"thumbnail": <bytes>
```

#### Complex Types

**Map (Object):** Nested key-value pairs
```json
"author": {
  "userId": "user123",
  "displayName": "John Doe",
  "email": "john@example.com"
}
```

**Array:** Ordered list of values
```json
"tags": ["work", "personal", "urgent"],
"collaborators": ["user123", "user456"]
```

Arrays can contain:
- Primitives: `["tag1", "tag2"]`
- Maps: `[{"userId": "user123", "role": "editor"}]`
- Mixed types (not recommended for consistency)

#### Nested Objects

You can nest maps deeply, but keep nesting shallow (2-3 levels) for query simplicity:

**Good (shallow nesting):**
```json
{
  "metadata": {
    "createdBy": "user123",
    "lastModified": {
      "userId": "user123",
      "timestamp": "2024-01-15T10:30:00Z"
    }
  }
}
```

**Avoid (deep nesting):**
```json
{
  "level1": {
    "level2": {
      "level3": {
        "level4": {
          "value": "too deep"
        }
      }
    }
  }
}
```

Deep nesting makes queries complex and security rules harder to write.

## 3. Practical Examples

### Example Firestore Structure for Notes App

Here's a complete data model for a Notes application with users, notes, tags, and sharing:

```
/users/{userId}
  - displayName: "John Doe"
  - email: "john@example.com"
  - createdAt: Timestamp
  - settings: {
      theme: "dark",
      fontSize: 14
    }

/notes/{noteId}
  - title: "Meeting Notes"
  - content: "Discussion about Q1 goals..."
  - userId: "user123"  // Reference to owner
  - tags: ["work", "meetings"]
  - isPinned: false
  - isArchived: false
  - createdAt: Timestamp
  - updatedAt: Timestamp
  - wordCount: 245
  - collaborators: [
      {
        userId: "user456",
        role: "editor",
        addedAt: Timestamp
      }
    ]

/tags/{tagId}
  - name: "work"
  - userId: "user123"  // Tag owner
  - color: "#FF5733"
  - noteCount: 12
  - createdAt: Timestamp

/sharedNotes/{shareId}
  - noteId: "note123"
  - sharedBy: "user123"
  - sharedWith: "user456"
  - permission: "read"  // or "write"
  - sharedAt: Timestamp
```

### Example Data (JSON-like)

**User Document:**
```json
{
  "userId": "user123",
  "displayName": "John Doe",
  "email": "john@example.com",
  "createdAt": "2024-01-01T00:00:00Z",
  "settings": {
    "theme": "dark",
    "fontSize": 14,
    "notificationsEnabled": true
  }
}
```

**Note Document:**
```json
{
  "noteId": "note456",
  "title": "Q1 Planning Meeting",
  "content": "We discussed the roadmap for Q1. Key points:\n1. Feature X\n2. Feature Y\n3. Timeline adjustments",
  "userId": "user123",
  "tags": ["work", "meetings", "planning"],
  "isPinned": true,
  "isArchived": false,
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-16T14:20:00Z",
  "wordCount": 28,
  "collaborators": [
    {
      "userId": "user456",
      "displayName": "Jane Smith",
      "role": "editor",
      "addedAt": "2024-01-15T11:00:00Z"
    }
  ]
}
```

**Tag Document:**
```json
{
  "tagId": "tag789",
  "name": "work",
  "userId": "user123",
  "color": "#FF5733",
  "noteCount": 12,
  "createdAt": "2024-01-01T00:00:00Z"
}
```

### How This Maps to Notes App

**User Collection:**
- Stores user profile information
- One document per user
- ID matches Firebase Auth UID

**Notes Collection:**
- Stores all notes as top-level documents
- `userId` field links note to owner
- Enables querying all notes or filtering by user
- Supports sharing via `collaborators` array

**Tags Collection:**
- Stores tag definitions (name, color)
- `userId` field scopes tags to users
- `noteCount` maintained for quick stats
- Notes reference tags by name in `tags` array

**Shared Notes Collection (Alternative Pattern):**
- Tracks sharing relationships separately
- Enables querying "notes shared with me"
- Supports different permission levels

## 4. Common Design Decisions

### Why Choose This Approach

**Top-level notes collection with userId field:**
- **Pros:** Simple queries across all notes, easy to implement search, straightforward pagination
- **Cons:** Requires filtering in queries, security rules check userId field

**Alternative: Subcollections (`/users/{userId}/notes/{noteId}`):**
- **Pros:** Natural data isolation, simpler security rules, automatic user scoping
- **Cons:** Harder to query across all users, collection group queries needed for search

**Recommendation:** Use top-level collections for notes if you need global search or admin features. Use subcollections if notes are strictly user-private and you don't need cross-user queries.

### Tags: Array vs Separate Collection

**Array approach (tags in note document):**
```json
{
  "tags": ["work", "personal"]
}
```
- **Pros:** Simple, fast reads, no extra queries
- **Cons:** Can't store tag metadata (color, description), harder to update tag names globally

**Separate collection approach:**
```json
// In note:
"tagIds": ["tag123", "tag456"]

// In tags collection:
/tags/tag123 { "name": "work", "color": "#FF5733" }
```
- **Pros:** Rich tag metadata, easy to update tag properties, can track tag usage
- **Cons:** Requires additional queries, more complex data model

**Recommendation:** Use arrays for simple tag names. Use a separate collection if you need tag colors, descriptions, or tag management features.

### Sharing: Embedded vs Separate Collection

**Embedded collaborators array:**
```json
{
  "collaborators": [
    {"userId": "user456", "role": "editor"}
  ]
}
```
- **Pros:** Single document read, simple structure
- **Cons:** Hard to query "notes shared with me", document grows with many collaborators

**Separate sharedNotes collection:**
```json
/sharedNotes/share123 {
  "noteId": "note456",
  "sharedWith": "user789",
  "permission": "read"
}
```
- **Pros:** Easy to query shared notes, supports many sharing relationships
- **Cons:** Requires additional queries, more complex updates

**Recommendation:** Use embedded array for <10 collaborators per note. Use separate collection for many shares or complex permission models.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Storing large arrays in documents**
```json
// BAD: Array with 1000+ items
"allNotes": ["note1", "note2", ..., "note1000"]

// GOOD: Use subcollection or separate query
/users/{userId}/notes/{noteId}
```

**2. Deep nesting (4+ levels)**
```json
// BAD: Too deep
"user": {
  "profile": {
    "settings": {
      "preferences": {
        "theme": "dark"  // 4 levels deep
      }
    }
  }
}

// GOOD: Flatten structure
"userTheme": "dark"
"userSettings": {
  "theme": "dark",
  "fontSize": 14
}
```

**3. Inconsistent field names**
```json
// BAD: Mixed naming
{ "created_at": "..." }
{ "createdAt": "..." }
{ "CreatedAt": "..." }

// GOOD: Consistent camelCase
{ "createdAt": "..." }
```

**4. Storing computed values that should be calculated**
```json
// BAD: Storing derived data that changes frequently
"fullName": "John Doe"  // If firstName/lastName change, this becomes stale

// GOOD: Compute on client or use Cloud Functions
"firstName": "John",
"lastName": "Doe"
```

### Cost/Performance Pitfalls

**1. Reading entire collections unnecessarily**
```javascript
// BAD: Reads all notes
const snapshot = await getDocs(collection(db, 'notes'));

// GOOD: Filter and limit
const q = query(
  collection(db, 'notes'),
  where('userId', '==', currentUserId),
  limit(20)
);
```

**2. Not using indexes for compound queries**
```javascript
// Requires composite index:
where('userId', '==', 'user123'),
where('isPinned', '==', true),
orderBy('createdAt', 'desc')
```
Firestore will prompt you to create the index, but forgetting to create it causes runtime errors.

**3. Over-fetching data**
```json
// BAD: Storing entire user object in every note
{
  "author": {
    "userId": "user123",
    "displayName": "John",
    "email": "john@example.com",
    "avatar": "https://...",
    "bio": "Long bio text...",
    // ... 20 more fields
  }
}

// GOOD: Store only userId, fetch user separately when needed
{
  "userId": "user123"
}
```

### Security-Related Warnings

**1. Exposing user data in document IDs**
```javascript
// BAD: User email in document ID
/users/john@example.com

// GOOD: Use UID
/users/user123
```

**2. Trusting client-side data structure**
Always validate data structure in security rules:
```javascript
// Security rule should enforce structure
match /notes/{noteId} {
  allow write: if request.resource.data.keys().hasAll(['title', 'content', 'userId']);
}
```

**3. Not scoping queries by userId**
```javascript
// BAD: Query without user filter (security risk)
const q = query(collection(db, 'notes'));

// GOOD: Always filter by authenticated user
const q = query(
  collection(db, 'notes'),
  where('userId', '==', auth.currentUser.uid)
);
```

## 6. Summary & Checklist

### Summary

- **Collections** organize documents. Use root collections for independent entities, subcollections for hierarchical data.
- **Documents** store your data as key-value pairs with a 1 MB size limit.
- **Field values** can be primitives (string, number, boolean) or complex types (map, array).
- **Data modeling** choices (top-level vs subcollections, arrays vs separate collections) impact query performance and security rules complexity.
- **Collection group queries** let you search across all subcollections with the same name.

### Checklist: You Are Ready When You Can...

- [ ] Explain when to use root collections vs subcollections
- [ ] Choose between auto-generated and custom document IDs for your use case
- [ ] Identify when a document might exceed the 1 MB limit
- [ ] Decide between storing tags as arrays vs a separate collection
- [ ] Model a one-to-many relationship (user to notes)
- [ ] Model a many-to-many relationship (notes to tags)
- [ ] Explain when to use collection group queries
- [ ] Avoid common mistakes like deep nesting and over-fetching
- [ ] Design a data structure that supports your app's query patterns

### Next Steps

Once you understand data modeling, you're ready to:
- Set up Firestore in your project (Chapter 3)
- Read and write data (Chapters 4-5)
- Query and filter your data (Chapter 6)
