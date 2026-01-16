# 10. SECURITY RULES

## 1. What This Chapter Covers

This chapter teaches you how to write secure Firestore security rules for your Notes application. You'll learn:

- Security rules syntax and structure
- How to enforce authentication requirements
- How to validate document data on write
- How to control read and write permissions
- How to restrict queries for security
- Common security patterns for real applications
- How to test and deploy security rules safely

Security rules are critical because they are your primary defense against unauthorized access and data corruption. Poorly written rules lead to data breaches, unauthorized access, and billing abuse. Understanding security rules syntax, validation, and common patterns helps you build secure applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Security Rules Syntax

Security rules use a declarative language to define access control.

#### match Statements

`match` statements define which documents the rules apply to:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Match all documents
    match /{document=**} {
      allow read, write: if false; // Deny all by default
    }
    
    // Match specific collection
    match /notes/{noteId} {
      allow read, write: if request.auth != null;
    }
    
    // Match subcollection
    match /users/{userId}/notes/{noteId} {
      allow read, write: if request.auth.uid == userId;
    }
  }
}
```

**Path patterns:**
- `/{document=**}` - Match all documents recursively
- `/notes/{noteId}` - Match documents in notes collection
- `/users/{userId}/notes/{noteId}` - Match subcollection documents
- `/{userId}` - Capture path segment as variable

#### allow Rules

`allow` rules define what operations are permitted:

```javascript
match /notes/{noteId} {
  // Allow all read operations
  allow read: if request.auth != null;
  
  // Allow all write operations
  allow write: if request.auth != null;
  
  // Or specify individual operations
  allow get: if request.auth != null;
  allow list: if request.auth != null;
  allow create: if request.auth != null;
  allow update: if request.auth != null;
  allow delete: if request.auth != null;
}
```

**Operation types:**
- `read` - Includes both `get` and `list`
- `write` - Includes `create`, `update`, and `delete`
- `get` - Read single document
- `list` - Query collection
- `create` - Create new document
- `update` - Update existing document
- `delete` - Delete document

#### request vs resource

`request` contains data about the operation being attempted:

```javascript
match /notes/{noteId} {
  allow update: if 
    // User is authenticated
    request.auth != null &&
    // User owns the note
    request.auth.uid == resource.data.userId &&
    // New title is provided
    request.resource.data.title is string &&
    // Title is not empty
    request.resource.data.title.size() > 0;
}
```

**request properties:**
- `request.auth` - Authentication info (null if not authenticated)
- `request.auth.uid` - User ID
- `request.resource.data` - New document data (for writes)
- `request.resource.data.fieldName` - Specific field in new data

**resource properties:**
- `resource.data` - Current document data (for updates/deletes)
- `resource.data.fieldName` - Specific field in current data

**Key difference:**
- `request.resource.data` - Data being written (create/update)
- `resource.data` - Existing data in database (update/delete)

#### Custom Functions

Define reusable functions for complex logic:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper function
    function isAuthenticated() {
      return request.auth != null;
    }
    
    function isOwner(userId) {
      return request.auth != null && request.auth.uid == userId;
    }
    
    function isValidNote() {
      return request.resource.data.keys().hasAll(['title', 'content', 'userId']) &&
             request.resource.data.title is string &&
             request.resource.data.content is string &&
             request.resource.data.userId is string;
    }
    
    match /notes/{noteId} {
      allow read: if isAuthenticated();
      allow create: if isAuthenticated() && isValidNote();
      allow update: if isOwner(resource.data.userId);
      allow delete: if isOwner(resource.data.userId);
    }
  }
}
```

**Function best practices:**
- Keep functions simple and focused
- Use descriptive names
- Reuse functions across rules
- Functions can call other functions

### 2.2 Authentication Rules

Enforce authentication requirements.

#### request.auth != null

Check if user is authenticated:

```javascript
match /notes/{noteId} {
  // Only authenticated users can read
  allow read: if request.auth != null;
  
  // Only authenticated users can write
  allow write: if request.auth != null;
}
```

#### request.auth.uid

Get the authenticated user's ID:

```javascript
match /notes/{noteId} {
  // User can only read their own notes
  allow read: if request.auth != null &&
               request.auth.uid == resource.data.userId;
  
  // User can only create notes for themselves
  allow create: if request.auth != null &&
                 request.auth.uid == request.resource.data.userId;
}
```

#### User-Specific Data Access

Restrict access to user's own data:

```javascript
match /notes/{noteId} {
  // Read: User must own the note
  allow get: if request.auth != null &&
              request.auth.uid == resource.data.userId;
  
  // List: Can only query own notes (enforced in query)
  allow list: if request.auth != null;
  
  // Create: Must set userId to own ID
  allow create: if request.auth != null &&
                 request.auth.uid == request.resource.data.userId;
  
  // Update: Must own the note
  allow update: if request.auth != null &&
                 request.auth.uid == resource.data.userId;
  
  // Delete: Must own the note
  allow delete: if request.auth != null &&
                 request.auth.uid == resource.data.userId;
}
```

**Important:** For `list` operations, you must also filter queries by `userId` in your application code. Rules can't enforce query filters, only validate that the user is authenticated.

### 2.3 Document Validation

Validate document structure and values on write.

#### validate() Function

Use `validate` to check document structure:

```javascript
match /notes/{noteId} {
  allow create: if request.auth != null &&
                 request.auth.uid == request.resource.data.userId &&
                 validateNote(request.resource.data);
  
  function validateNote(data) {
    return data.keys().hasAll(['title', 'content', 'userId', 'createdAt']) &&
           data.title is string &&
           data.content is string &&
           data.userId is string &&
           data.createdAt is timestamp &&
           data.title.size() > 0 &&
           data.content.size() > 0;
  }
}
```

#### Field Existence Checks

Ensure required fields exist:

```javascript
match /notes/{noteId} {
  allow create: if request.auth != null &&
                 // Required fields must exist
                 request.resource.data.keys().hasAll(['title', 'content', 'userId']) &&
                 // Optional fields can be missing
                 (!('tags' in request.resource.data) || 
                  request.resource.data.tags is list);
}
```

**Check field existence:**
- `'fieldName' in request.resource.data` - Field exists
- `request.resource.data.keys().hasAll(['field1', 'field2'])` - All fields exist
- `!('fieldName' in request.resource.data)` - Field doesn't exist

#### Type Validation

Validate field types:

```javascript
match /notes/{noteId} {
  allow create: if request.auth != null &&
                 request.resource.data.title is string &&
                 request.resource.data.content is string &&
                 request.resource.data.userId is string &&
                 request.resource.data.wordCount is int &&
                 request.resource.data.isPinned is bool &&
                 request.resource.data.createdAt is timestamp &&
                 request.resource.data.tags is list;
}
```

**Type checks:**
- `is string` - String type
- `is int` - Integer type
- `is float` - Floating point number
- `is number` - Integer or float
- `is bool` - Boolean type
- `is timestamp` - Timestamp type
- `is latlng` - GeoPoint type
- `is list` - Array type
- `is map` - Object/Map type

#### Value Constraints

Validate field values:

```javascript
match /notes/{noteId} {
  allow create: if request.auth != null &&
                 // Title length constraints
                 request.resource.data.title.size() > 0 &&
                 request.resource.data.title.size() <= 200 &&
                 // Content length constraints
                 request.resource.data.content.size() > 0 &&
                 request.resource.data.content.size() <= 100000 &&
                 // Word count must be positive
                 request.resource.data.wordCount > 0 &&
                 // Tags array size limit
                 (!('tags' in request.resource.data) ||
                  request.resource.data.tags.size() <= 10);
}
```

**Common constraints:**
- `field.size() > 0` - Non-empty string/array
- `field.size() <= max` - Maximum length/size
- `field >= min && field <= max` - Numeric range
- `field in ['value1', 'value2']` - Enum values

### 2.4 Read Rules

Control read access to documents and collections.

#### get() Permissions

Control single document reads:

```javascript
match /notes/{noteId} {
  // Public read
  allow get: if true;
  
  // Authenticated users only
  allow get: if request.auth != null;
  
  // Owner only
  allow get: if request.auth != null &&
              request.auth.uid == resource.data.userId;
  
  // Owner or public note
  allow get: if request.auth != null &&
              (request.auth.uid == resource.data.userId ||
               resource.data.isPublic == true);
}
```

#### list() Permissions (Collection Access)

Control collection queries:

```javascript
match /notes/{noteId} {
  // Allow listing (querying collection)
  allow list: if request.auth != null;
}
```

**Important:** `list` rules can't enforce query filters. You must filter queries in your application code:

```dart
// In your app code
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId); // Required!
```

#### Query Restrictions

Rules can't restrict query filters, but you can validate query results:

```javascript
match /notes/{noteId} {
  // Allow list, but user must filter by userId in query
  allow list: if request.auth != null;
  
  // For each document returned, validate access
  allow get: if request.auth != null &&
              (request.auth.uid == resource.data.userId ||
               resource.data.isPublic == true);
}
```

**Best practice:** Always filter queries in application code, then rules validate each document.

### 2.5 Write Rules

Control create, update, and delete operations.

#### create Rules

Validate new document creation:

```javascript
match /notes/{noteId} {
  allow create: if request.auth != null &&
                 // User must set userId to their own ID
                 request.auth.uid == request.resource.data.userId &&
                 // Required fields
                 request.resource.data.keys().hasAll(['title', 'content', 'userId']) &&
                 // Field validation
                 request.resource.data.title is string &&
                 request.resource.data.content is string &&
                 // Can't set certain fields
                 !('viewCount' in request.resource.data) &&
                 !('likeCount' in request.resource.data);
}
```

**Common create validations:**
- User is authenticated
- User sets userId to their own ID
- Required fields are present
- Field types are correct
- Computed fields (counts, timestamps) are not set by client

#### update Rules

Validate document updates:

```javascript
match /notes/{noteId} {
  allow update: if request.auth != null &&
                 // User must own the note
                 request.auth.uid == resource.data.userId &&
                 // Can't change userId
                 request.resource.data.userId == resource.data.userId &&
                 // Can't change createdAt
                 request.resource.data.createdAt == resource.data.createdAt &&
                 // Title must be valid if provided
                 (!('title' in request.resource.data) ||
                  (request.resource.data.title is string &&
                   request.resource.data.title.size() > 0));
}
```

**Common update validations:**
- User owns the document
- Immutable fields can't be changed (userId, createdAt)
- Updated fields are valid
- Only allowed fields can be updated

#### delete Rules

Control document deletion:

```javascript
match /notes/{noteId} {
  allow delete: if request.auth != null &&
                // User must own the note
                request.auth.uid == resource.data.userId;
}
```

**Common delete validations:**
- User owns the document
- Document is not locked/protected
- User has delete permission

#### Field-Level Validation

Validate individual fields:

```javascript
match /notes/{noteId} {
  allow update: if request.auth != null &&
                 request.auth.uid == resource.data.userId &&
                 // Validate title if being updated
                 (!('title' in request.resource.data) ||
                  (request.resource.data.title is string &&
                   request.resource.data.title.size() > 0 &&
                   request.resource.data.title.size() <= 200)) &&
                 // Validate tags if being updated
                 (!('tags' in request.resource.data) ||
                  (request.resource.data.tags is list &&
                   request.resource.data.tags.size() <= 10 &&
                   request.resource.data.tags.size() > 0));
}
```

### 2.6 Query Restrictions

Restrict queries to prevent unauthorized data access.

#### allow list: if <condition>

Control which queries are allowed:

```javascript
match /notes/{noteId} {
  // Allow list only if user is authenticated
  allow list: if request.auth != null;
}
```

**Limitation:** Rules can't enforce query filters. You must filter in application code.

#### Query Constraints in Rules

You can't restrict query filters in rules, but you can validate results:

```javascript
match /notes/{noteId} {
  // Allow list
  allow list: if request.auth != null;
  
  // But validate each document
  allow get: if request.auth != null &&
              request.auth.uid == resource.data.userId;
}
```

**Application code must filter:**
```dart
// Required: Filter by userId in app code
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId); // Enforced in app
```

#### Security vs Data Validation

**Security rules:** Enforce access control (who can read/write)
**Data validation:** Ensure data integrity (what data is valid)

```javascript
// Security: Who can access
allow read: if request.auth != null &&
             request.auth.uid == resource.data.userId;

// Validation: What data is valid
allow create: if request.resource.data.title is string &&
               request.resource.data.title.size() > 0;
```

**Best practice:** Use rules for both security and validation.

### 2.7 Common Patterns

Real-world security patterns for common scenarios.

#### Public Read, Auth Write

Public content that anyone can read, but only authenticated users can write:

```javascript
match /publicNotes/{noteId} {
  // Anyone can read
  allow read: if true;
  
  // Only authenticated users can write
  allow write: if request.auth != null;
}
```

#### Owner-Only Access

Users can only access their own documents:

```javascript
match /notes/{noteId} {
  allow read: if request.auth != null &&
              request.auth.uid == resource.data.userId;
  
  allow create: if request.auth != null &&
                 request.auth.uid == request.resource.data.userId;
  
  allow update, delete: if request.auth != null &&
                         request.auth.uid == resource.data.userId;
}
```

#### Shared Document Access

Documents shared with specific users:

```javascript
match /notes/{noteId} {
  allow read: if request.auth != null &&
              (request.auth.uid == resource.data.userId ||
               request.auth.uid in resource.data.collaboratorIds);
  
  allow update: if request.auth != null &&
                 (request.auth.uid == resource.data.userId ||
                  (request.auth.uid in resource.data.collaboratorIds &&
                   getCollaboratorRole(request.auth.uid) == 'editor'));
  
  allow delete: if request.auth != null &&
                 request.auth.uid == resource.data.userId; // Only owner can delete
  
  function getCollaboratorRole(userId) {
    let collaborator = resource.data.collaborators[resource.data.collaborators.hasAny([userId])];
    return collaborator.role;
  }
}
```

#### Admin Bypass

Allow admins to bypass normal rules:

```javascript
match /notes/{noteId} {
  // Admin can do anything
  allow read, write: if request.auth != null &&
                      get(/databases/$(database)/documents/users/$(request.auth.uid)).data.isAdmin == true;
  
  // Normal users
  allow read: if request.auth != null &&
               request.auth.uid == resource.data.userId;
  
  allow create: if request.auth != null &&
                 request.auth.uid == request.resource.data.userId;
  
  allow update, delete: if request.auth != null &&
                         request.auth.uid == resource.data.userId;
}
```

### 2.8 Testing Security Rules

Test rules before deploying to production.

#### Firebase Emulator

Test rules locally with emulator:

```bash
# Start emulator
firebase emulators:start --only firestore

# Rules are loaded from firestore.rules
```

**Test in code:**
```dart
// Connect to emulator
FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);

// Test operations
try {
  await noteRef.get();
  print('Read allowed');
} on FirebaseException catch (e) {
  if (e.code == 'permission-denied') {
    print('Read denied');
  }
}
```

#### Test Rules Playground

Use Firebase Console Rules Playground:

1. Go to Firebase Console → Firestore → Rules
2. Click "Rules Playground"
3. Select operation (get, list, create, update, delete)
4. Enter document path
5. Set authentication
6. Test rule

**Example test:**
- Operation: `get`
- Location: `/notes/note123`
- Authenticated: Yes, UID: `user123`
- Expected: Allow (if user123 owns note123)

#### Unit Testing

Write automated tests for rules:

```dart
// Using @firebase/rules-unit-testing (Node.js)
// Or test in Flutter with emulator

test('User can read own note', () async {
  // Setup: Create note owned by user123
  await adminFirestore.collection('notes').doc('note123').set({
    'title': 'Test',
    'userId': 'user123',
  });
  
  // Test: user123 reads note
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc('note123');
  
  // Should succeed
  final doc = await noteRef.get();
  expect(doc.exists, true);
});

test('User cannot read other user note', () async {
  // Setup: Create note owned by user123
  await adminFirestore.collection('notes').doc('note123').set({
    'title': 'Test',
    'userId': 'user123',
  });
  
  // Test: user456 tries to read note
  // Sign in as user456
  await FirebaseAuth.instance.signInWithEmailAndPassword(
    email: 'user456@example.com',
    password: 'password',
  );
  
  final noteRef = FirebaseFirestore.instance
      .collection('notes')
      .doc('note123');
  
  // Should fail
  expect(
    () => noteRef.get(),
    throwsA(isA<FirebaseException>().having(
      (e) => e.code,
      'code',
      'permission-denied',
    )),
  );
});
```

### 2.9 Production Best Practices

Deploy rules safely to production.

#### Version Control

Store rules in version control:

```bash
# firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Rules here
  }
}
```

**Benefits:**
- Track changes
- Review before deploy
- Rollback if needed
- Team collaboration

#### Staging Testing

Test rules in staging environment first:

```bash
# Deploy to staging project
firebase use staging
firebase deploy --only firestore:rules

# Test thoroughly
# Then deploy to production
firebase use prod
firebase deploy --only firestore:rules
```

#### Gradual Rollout

Deploy rules gradually:

1. Deploy to staging
2. Test thoroughly
3. Deploy to production
4. Monitor for errors
5. Have rollback plan ready

**Rollback:**
```bash
# Revert to previous version
git checkout HEAD~1 firestore.rules
firebase deploy --only firestore:rules
```

## 3. Practical Examples

### Example: Complete Notes App Security Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }
    
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }
    
    function isValidNote(data) {
      return data.keys().hasAll(['title', 'content', 'userId']) &&
             data.title is string &&
             data.content is string &&
             data.userId is string &&
             data.title.size() > 0 &&
             data.content.size() > 0 &&
             data.title.size() <= 200 &&
             data.content.size() <= 100000;
    }
    
    // Users collection
    match /users/{userId} {
      // Users can read any user profile
      allow read: if isAuthenticated();
      
      // Users can only update their own profile
      allow update: if isOwner(userId);
      
      // Users can only create their own profile
      allow create: if isOwner(userId) &&
                     request.resource.data.userId == userId;
    }
    
    // Notes collection
    match /notes/{noteId} {
      // Read: Owner or public note
      allow get: if isAuthenticated() &&
                  (isOwner(resource.data.userId) ||
                   resource.data.isPublic == true);
      
      // List: Authenticated (must filter by userId in query)
      allow list: if isAuthenticated();
      
      // Create: Must be authenticated and set userId to own ID
      allow create: if isAuthenticated() &&
                      request.auth.uid == request.resource.data.userId &&
                      isValidNote(request.resource.data) &&
                      // Can't set computed fields
                      !('viewCount' in request.resource.data) &&
                      !('likeCount' in request.resource.data) &&
                      // createdAt must be server timestamp
                      request.resource.data.createdAt == request.time;
      
      // Update: Must own note
      allow update: if isAuthenticated() &&
                      isOwner(resource.data.userId) &&
                      // Can't change userId
                      request.resource.data.userId == resource.data.userId &&
                      // Can't change createdAt
                      request.resource.data.createdAt == resource.data.createdAt &&
                      // Validate updated fields
                      (!('title' in request.resource.data) ||
                       (request.resource.data.title is string &&
                        request.resource.data.title.size() > 0)) &&
                      (!('tags' in request.resource.data) ||
                       (request.resource.data.tags is list &&
                        request.resource.data.tags.size() <= 10));
      
      // Delete: Must own note
      allow delete: if isAuthenticated() &&
                      isOwner(resource.data.userId);
    }
    
    // Tags collection
    match /tags/{tagId} {
      // Read: Authenticated users
      allow read: if isAuthenticated();
      
      // Create: Authenticated users, must set userId
      allow create: if isAuthenticated() &&
                      request.auth.uid == request.resource.data.userId &&
                      request.resource.data.keys().hasAll(['name', 'userId']) &&
                      request.resource.data.name is string;
      
      // Update: Owner only
      allow update: if isAuthenticated() &&
                      isOwner(resource.data.userId);
      
      // Delete: Owner only
      allow delete: if isAuthenticated() &&
                      isOwner(resource.data.userId);
    }
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Public read vs Authenticated read:**
- **Public read:** Use for public content, reduces authentication overhead
- **Authenticated read:** Use for private content, better security
- **Recommendation:** Use authenticated read for user data. Use public read only for truly public content.

**Owner-only vs Shared access:**
- **Owner-only:** Simpler rules, better performance
- **Shared access:** More flexible, supports collaboration
- **Recommendation:** Start with owner-only. Add shared access when you need collaboration features.

**Validation in rules vs Application:**
- **Rules validation:** Enforced server-side, can't be bypassed
- **Application validation:** Better UX, faster feedback
- **Recommendation:** Validate in both. Rules for security, application for UX.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Allowing all access (test mode)**
```javascript
// BAD: Allows anyone to read/write
match /{document=**} {
  allow read, write: if true;
}

// GOOD: Restrictive rules
match /notes/{noteId} {
  allow read: if request.auth != null &&
               request.auth.uid == resource.data.userId;
}
```

**2. Not validating userId on create**
```javascript
// BAD: User can create note for anyone
allow create: if request.auth != null;

// GOOD: User must set userId to own ID
allow create: if request.auth != null &&
               request.auth.uid == request.resource.data.userId;
```

**3. Not protecting immutable fields**
```javascript
// BAD: User can change userId
allow update: if request.auth != null;

// GOOD: Prevent userId change
allow update: if request.auth != null &&
               request.resource.data.userId == resource.data.userId;
```

**4. Assuming list rules enforce query filters**
```javascript
// BAD: Rules can't enforce query filters
allow list: if request.auth != null;
// User can still query all notes!

// GOOD: Filter in app code AND validate in rules
// App: .where('userId', isEqualTo: currentUserId)
// Rules: allow get: if request.auth.uid == resource.data.userId
```

### Security Pitfalls

**1. Not validating field types**
```javascript
// BAD: No type validation
allow create: if request.auth != null;

// GOOD: Validate types
allow create: if request.auth != null &&
               request.resource.data.title is string &&
               request.resource.data.wordCount is int;
```

**2. Allowing clients to set computed fields**
```javascript
// BAD: Client can set viewCount
allow create: if request.auth != null;

// GOOD: Prevent setting computed fields
allow create: if request.auth != null &&
               !('viewCount' in request.resource.data) &&
               !('likeCount' in request.resource.data);
```

**3. Not testing rules before deployment**
```javascript
// BAD: Deploy without testing
firebase deploy --only firestore:rules

// GOOD: Test first
// 1. Test in emulator
// 2. Test in staging
// 3. Deploy to production
```

## 6. Summary & Checklist

### Summary

- **Security rules** use `match` statements and `allow` rules to control access
- **Authentication** is checked with `request.auth != null` and `request.auth.uid`
- **Document validation** ensures data integrity with type and value checks
- **Read rules** control `get` and `list` operations
- **Write rules** control `create`, `update`, and `delete` operations
- **Query restrictions** require filtering in application code (rules can't enforce filters)
- **Common patterns** include owner-only, shared access, and admin bypass
- **Testing** with emulator and staging ensures rules work before production

### Checklist: You Are Ready When You Can...

- [ ] Write basic security rules with `match` and `allow` statements
- [ ] Enforce authentication requirements
- [ ] Validate document structure and field types
- [ ] Control read access (get and list)
- [ ] Control write access (create, update, delete)
- [ ] Protect immutable fields (userId, createdAt)
- [ ] Prevent clients from setting computed fields
- [ ] Implement owner-only access patterns
- [ ] Implement shared document access patterns
- [ ] Test rules in emulator before deployment
- [ ] Deploy rules to staging before production
- [ ] Understand that list rules don't enforce query filters

### Verification Steps

**Test your security rules:**

1. **Test authenticated access:**
```dart
// Sign in
await FirebaseAuth.instance.signInAnonymously();

// Should succeed
final doc = await noteRef.get();
```

2. **Test unauthenticated access:**
```dart
// Sign out
await FirebaseAuth.instance.signOut();

// Should fail
try {
  await noteRef.get();
} on FirebaseException catch (e) {
  assert(e.code == 'permission-denied');
}
```

3. **Test owner access:**
```dart
// Create note as user123
await noteRef.set({
  'title': 'Test',
  'userId': 'user123',
});

// Sign in as user123
// Should succeed
final doc = await noteRef.get();

// Sign in as user456
// Should fail
```

### Next Steps

Once you can write secure rules, you're ready to:
- Optimize performance (Chapter 11)
- Handle errors and edge cases (Chapter 13)
- Deploy to production (Chapter 16)
