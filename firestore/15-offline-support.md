# 15. OFFLINE SUPPORT

## 1. What This Chapter Covers

This chapter teaches you how to implement offline support in Firestore for your Notes application. You'll learn:

- How Firestore's offline capabilities work (cache, pending writes)
- How to configure persistence settings
- How Firestore behaves when offline
- How to handle offline writes and sync conflicts
- How to manage cache size and clearing
- How to implement syncing strategies
- How to access and display offline data

Offline support is crucial because users often have poor connectivity or no internet access. Understanding offline capabilities, cache management, and sync strategies helps you build apps that work seamlessly offline and provide a consistent user experience.

## 2. Core Concepts (Simple & Clear)

### 2.1 Offline Capabilities

Firestore provides built-in offline support for reading and writing data.

#### Local Cache

Firestore automatically caches data locally for offline access:

```dart
// Enable persistence (usually enabled by default)
await FirebaseFirestore.instance.enablePersistence();

// Data is cached automatically
// Works offline after first read
final doc = await noteRef.get(); // Uses cache if offline
```

**Cache behavior:**
- Automatically caches documents you read
- Persists across app restarts
- Updates when online
- Available immediately when offline

#### Pending Writes Queue

Writes are queued when offline and synced when online:

```dart
// Write when offline
await noteRef.update({'title': 'New Title'});
// Queued automatically, syncs when online

// Check if write is pending
noteRef.snapshots(includeMetadataChanges: true).listen((snapshot) {
  if (snapshot.metadata.hasPendingWrites) {
    print('Write is pending sync');
  } else {
    print('Write synced to server');
  }
});
```

**Write queue features:**
- Writes queued automatically when offline
- Synced in order when connection restored
- Appears in queries immediately (optimistic updates)
- Persists across app restarts

#### Offline Reads

Read operations work offline using cached data:

```dart
// Read works offline if data was previously cached
final doc = await noteRef.get(); // Uses cache if offline

// Listeners work offline too
noteRef.snapshots().listen((snapshot) {
  // Receives cached data when offline
  if (snapshot.metadata.isFromCache) {
    print('Data from cache (offline)');
  } else {
    print('Data from server (online)');
  }
});
```

**Offline read behavior:**
- Returns cached data immediately
- No network requests when offline
- Shows last known state
- Updates automatically when online

### 2.2 Persistence Settings

Configure how Firestore caches data.

#### PersistenceEnabled

Enable or disable persistence:

```dart
// Enable persistence (default on mobile)
await FirebaseFirestore.instance.enablePersistence();

// Or configure in settings
final firestore = FirebaseFirestore.instance;
firestore.settings = const Settings(
  persistenceEnabled: true,
);
```

**Important:** Persistence must be enabled before any Firestore operations.

**Platform differences:**
- **Mobile (iOS/Android):** Persistence enabled by default
- **Web:** Must enable explicitly
- **Desktop:** Must enable explicitly

#### Cache Size

Configure cache size limit:

```dart
// Unlimited cache (default)
firestore.settings = const Settings(
  persistenceEnabled: true,
  cacheSizeBytes: Settings.CACHE_SIZE_UNLIMITED,
);

// Or limit cache size (e.g., 100 MB)
firestore.settings = const Settings(
  persistenceEnabled: true,
  cacheSizeBytes: 100 * 1024 * 1024, // 100 MB
);
```

**Cache size options:**
- `Settings.CACHE_SIZE_UNLIMITED`: No limit (default)
- Size in bytes: Custom limit (e.g., 100 MB)

**Best practice:** Use unlimited cache unless you have specific memory constraints.

#### Persistence Mode

Persistence is always enabled or disabled (no modes):

```dart
// Either persistence is on or off
await FirebaseFirestore.instance.enablePersistence();

// Check if persistence is enabled
final settings = firestore.settings;
print('Persistence: ${settings.persistenceEnabled}');
```

### 2.3 Offline Behavior

Understand how Firestore behaves when offline.

#### Reading Cached Data

Reads automatically use cache when offline:

```dart
// This works offline if data was previously cached
final doc = await noteRef.get();

if (doc.metadata.isFromCache) {
  print('Reading from cache (offline)');
} else {
  print('Reading from server (online)');
}
```

**Cache-first behavior:**
- Returns cached data immediately
- Updates from server when online
- No error thrown when offline

#### Write Queue Management

Writes are automatically queued when offline:

```dart
// Write when offline
await noteRef.update({'title': 'New Title'});
// Automatically queued, no error

// Check queue status
final doc = await noteRef.get();
if (doc.metadata.hasPendingWrites) {
  print('Write is in queue, waiting to sync');
}
```

**Queue behavior:**
- Writes succeed immediately (optimistic updates)
- Queued for sync when online
- Syncs automatically when connection restored
- Maintains write order

#### Sync When Online

Automatic sync when connection restored:

```dart
// Monitor sync status
noteRef.snapshots(includeMetadataChanges: true).listen((snapshot) {
  final metadata = snapshot.metadata;
  
  if (metadata.hasPendingWrites) {
    // Write is queued
    showPendingIndicator();
  } else if (metadata.isFromCache) {
    // Using cached data (offline)
    showOfflineIndicator();
  } else {
    // Synced with server (online)
    hideIndicators();
  }
});
```

**Sync behavior:**
- Automatic when connection restored
- No manual sync needed
- Maintains order of writes
- Resolves conflicts automatically (last write wins)

### 2.4 Offline Writes

Handle writes when offline and track sync status.

#### Adding to Queue

Writes are automatically added to queue:

```dart
// Write when offline - automatically queued
Future<void> updateNoteOffline(String noteId, Map<String, dynamic> data) async {
  try {
    await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .update(data);
    
    // Write succeeds immediately (optimistic update)
    print('Write queued for sync');
  } catch (e) {
    // Only fails if there's a validation error, not because offline
    print('Write failed: $e');
  }
}
```

**Optimistic updates:**
- Writes appear immediately in UI
- Data visible in queries right away
- Syncs in background when online
- No user action needed

#### Tracking Pending Writes

Monitor which writes are pending:

```dart
class PendingWritesTracker {
  static bool hasPendingWrites(DocumentSnapshot snapshot) {
    return snapshot.metadata.hasPendingWrites;
  }
  
  static Stream<bool> watchPendingWrites(DocumentReference ref) {
    return ref.snapshots(includeMetadataChanges: true)
        .map((snapshot) => snapshot.metadata.hasPendingWrites);
  }
}

// Usage
final isPending = PendingWritesTracker.hasPendingWrites(doc);

// Or watch for changes
PendingWritesTracker.watchPendingWrites(noteRef).listen((isPending) {
  if (isPending) {
    showSavingIndicator();
  } else {
    hideSavingIndicator();
  }
});
```

**Show pending indicator:**
```dart
StreamBuilder<DocumentSnapshot>(
  stream: noteRef.snapshots(includeMetadataChanges: true),
  builder: (context, snapshot) {
    if (!snapshot.hasData) return CircularProgressIndicator();
    
    final isPending = snapshot.data!.metadata.hasPendingWrites;
    
    return Column(
      children: [
        if (isPending)
          LinearProgressIndicator(),
        NoteContent(data: snapshot.data!.data()),
      ],
    );
  },
);
```

#### Sync Conflicts

Handle conflicts when syncing:

```dart
// Firestore uses "last write wins" for conflicts
// No manual conflict resolution needed

// If you need conflict handling, use transactions
await FirebaseFirestore.instance.runTransaction((transaction) async {
  final snapshot = await transaction.get(noteRef);
  final currentData = snapshot.data()!;
  
  // Merge changes instead of overwrite
  transaction.update(noteRef, {
    ...currentData,
    ...newData,
    'lastModified': FieldValue.serverTimestamp(),
  });
});
```

**Conflict resolution:**
- Default: Last write wins
- Automatic resolution by Firestore
- No manual intervention needed
- Use transactions for complex conflict handling

### 2.5 Cache Management

Manage cache size and clearing.

#### Clearing Cache

Clear cache when needed:

```dart
// Clear all cached data
await FirebaseFirestore.instance.clearPersistence();

// Note: Must be called before any Firestore operations
// Usually during app initialization or logout
```

**When to clear cache:**
- User logs out
- Switching accounts
- App reset
- Debugging/testing

**Important:** Must clear cache before any Firestore operations. Can't clear while operations are active.

#### Cache Size Limits

Monitor and limit cache size:

```dart
// Set cache size limit
firestore.settings = const Settings(
  persistenceEnabled: true,
  cacheSizeBytes: 100 * 1024 * 1024, // 100 MB
);

// When limit reached, Firestore automatically removes least recently used data
```

**Cache eviction:**
- Automatic when size limit reached
- Removes least recently used data
- Keeps frequently accessed data
- No manual management needed

#### Memory Cache

Firestore uses both disk and memory cache:

```dart
// Disk cache: Persists across app restarts
// Memory cache: Faster access, cleared on app close

// Both managed automatically by Firestore
// No configuration needed
```

**Cache layers:**
- **Memory cache:** Fast, in-memory
- **Disk cache:** Persistent, slower
- Automatic management between layers

### 2.6 Syncing Strategy

Implement syncing strategies for your app.

#### Manual Sync

Force sync when needed:

```dart
// Firestore syncs automatically, but you can trigger operations
Future<void> ensureSynced() async {
  // Write an operation to trigger sync
  await FirebaseFirestore.instance
      .collection('_sync')
      .doc('trigger')
      .set({'timestamp': FieldValue.serverTimestamp()});
  
  // Or just wait for automatic sync
  // Firestore syncs automatically when online
}
```

**Note:** Firestore syncs automatically. Manual sync is rarely needed.

#### Automatic Sync

Firestore syncs automatically:

```dart
// No code needed - automatic sync
// When connection restored:
// 1. Pending writes sync automatically
// 2. Cached data updates from server
// 3. Listeners receive updates

// Monitor sync status
query.snapshots(includeMetadataChanges: true).listen((snapshot) {
  if (!snapshot.metadata.isFromCache) {
    print('Synced with server');
  }
});
```

**Automatic sync behavior:**
- Triggers when connection restored
- Syncs pending writes in order
- Updates cache from server
- Notifies listeners of changes

#### Background Sync

Sync happens in background:

```dart
// Sync happens automatically in background
// No need to keep app open

// Check sync status
Future<void> checkSyncStatus() async {
  final doc = await noteRef.get();
  
  if (doc.metadata.hasPendingWrites) {
    print('Waiting to sync');
  } else if (doc.metadata.isFromCache) {
    print('Using cached data (offline)');
  } else {
    print('Synced with server');
  }
}
```

### 2.7 Offline Data Access

Access and display offline data to users.

#### Loading Cached Data

Load data from cache:

```dart
// Data loads from cache automatically when offline
Future<Note?> getNoteOffline(String noteId) async {
  try {
    final doc = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .get();
    
    if (doc.exists) {
      return Note.fromFirestore(doc);
    }
    return null;
  } catch (e) {
    // Error only if data was never cached
    return null;
  }
}
```

**Cache-first loading:**
- Returns cached data immediately
- No network delay
- Works seamlessly offline
- Updates when online

#### Stale Data Handling

Handle stale data gracefully:

```dart
class OfflineDataHandler {
  static bool isDataStale(DocumentSnapshot snapshot, {Duration maxAge = const Duration(hours: 1)}) {
    if (snapshot.metadata.isFromCache) {
      // Check when data was last updated
      final updatedAt = snapshot.data()?['updatedAt'] as Timestamp?;
      if (updatedAt != null) {
        final age = DateTime.now().difference(updatedAt.toDate());
        return age > maxAge;
      }
    }
    return false;
  }
  
  static Widget buildNoteWithStaleWarning(DocumentSnapshot snapshot) {
    final isStale = isDataStale(snapshot);
    
    return Column(
      children: [
        if (isStale)
          Container(
            color: Colors.orange,
            padding: EdgeInsets.all(8),
            child: Text('Showing cached data (may be outdated)'),
          ),
        NoteView(data: snapshot.data()),
      ],
    );
  }
}
```

#### User Notifications

Notify users about offline status:

```dart
class OfflineIndicator extends StatelessWidget {
  final Query query;
  
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<QuerySnapshot>(
      stream: query.snapshots(includeMetadataChanges: true),
      builder: (context, snapshot) {
        if (!snapshot.hasData) return SizedBox.shrink();
        
        final isFromCache = snapshot.data!.metadata.isFromCache;
        final hasPendingWrites = snapshot.data!.metadata.hasPendingWrites;
        
        if (isFromCache && !hasPendingWrites) {
          return Container(
            color: Colors.blue,
            padding: EdgeInsets.all(8),
            child: Row(
              children: [
                Icon(Icons.cloud_off, color: Colors.white),
                SizedBox(width: 8),
                Text(
                  'Offline - Showing cached data',
                  style: TextStyle(color: Colors.white),
                ),
              ],
            ),
          );
        }
        
        if (hasPendingWrites) {
          return Container(
            color: Colors.orange,
            padding: EdgeInsets.all(8),
            child: Row(
              children: [
                CircularProgressIndicator(color: Colors.white),
                SizedBox(width: 8),
                Text(
                  'Syncing changes...',
                  style: TextStyle(color: Colors.white),
                ),
              ],
            ),
          );
        }
        
        return SizedBox.shrink();
      },
    );
  }
}
```

## 3. Practical Examples

### Example: Offline-First Notes App

```dart
class OfflineNotesService {
  final FirebaseFirestore _firestore;
  
  OfflineNotesService(this._firestore);
  
  // Write works offline
  Future<void> createNote(Note note) async {
    await _firestore.collection('notes').add(note.toFirestore());
    // Queued if offline, syncs automatically when online
  }
  
  // Read works offline (if cached)
  Future<List<Note>> getNotes(String userId) async {
    final snapshot = await _firestore
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .get();
    
    return snapshot.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();
  }
  
  // Watch for sync status
  Stream<bool> watchSyncStatus(String noteId) {
    return _firestore
        .collection('notes')
        .doc(noteId)
        .snapshots(includeMetadataChanges: true)
        .map((snapshot) => snapshot.metadata.hasPendingWrites);
  }
}
```

### Example: Offline-Aware UI

```dart
class NotesListWithOfflineSupport extends StatelessWidget {
  final String userId;
  
  @override
  Widget build(BuildContext context) {
    final query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId);
    
    return Column(
      children: [
        OfflineIndicator(query: query),
        StreamBuilder<QuerySnapshot>(
          stream: query.snapshots(includeMetadataChanges: true),
          builder: (context, snapshot) {
            if (snapshot.hasError) {
              return Text('Error: ${snapshot.error}');
            }
            
            if (snapshot.connectionState == ConnectionState.waiting) {
              return CircularProgressIndicator();
            }
            
            if (!snapshot.hasData) {
              return Text('No notes found');
            }
            
            final notes = snapshot.data!.docs
                .map((doc) => Note.fromFirestore(doc))
                .toList();
            
            return ListView.builder(
              itemCount: notes.length,
              itemBuilder: (context, index) {
                final note = notes[index];
                final doc = snapshot.data!.docs[index];
                final isPending = doc.metadata.hasPendingWrites;
                
                return ListTile(
                  title: Text(note.title),
                  trailing: isPending
                      ? SizedBox(
                          width: 16,
                          height: 16,
                          child: CircularProgressIndicator(strokeWidth: 2),
                        )
                      : null,
                );
              },
            );
          },
        ),
      ],
    );
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Cache-first vs Network-first:**
- **Cache-first:** Firestore default, works offline, fast
- **Network-first:** Would require custom implementation
- **Recommendation:** Use Firestore's cache-first approach (default).

**Automatic sync vs Manual sync:**
- **Automatic:** Firestore default, no code needed
- **Manual:** More control, but complex
- **Recommendation:** Use automatic sync. Add manual triggers only if needed.

**Show offline indicator vs Silent:**
- **Show indicator:** Users know they're offline, better UX
- **Silent:** Cleaner UI, but users may be confused
- **Recommendation:** Show subtle offline indicator when applicable.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Not enabling persistence**
```dart
// BAD: Persistence not enabled (web/desktop)
final firestore = FirebaseFirestore.instance;
// Works online only

// GOOD: Enable persistence
await FirebaseFirestore.instance.enablePersistence();
```

**2. Enabling persistence after operations**
```dart
// BAD: Enable after using Firestore
final doc = await noteRef.get(); // Operation before persistence
await FirebaseFirestore.instance.enablePersistence(); // Too late!

// GOOD: Enable before any operations
await FirebaseFirestore.instance.enablePersistence();
final doc = await noteRef.get(); // Now works offline
```

**3. Not handling cached data**
```dart
// BAD: Assumes fresh data
final doc = await noteRef.get();
final note = Note.fromFirestore(doc); // May be stale if offline

// GOOD: Check cache status
final doc = await noteRef.get();
if (doc.metadata.isFromCache) {
  // Show stale indicator
}
final note = Note.fromFirestore(doc);
```

## 6. Summary & Checklist

### Summary

- **Offline capabilities:** Local cache, pending writes queue, offline reads work automatically
- **Persistence settings:** Enable persistence, configure cache size
- **Offline behavior:** Reads from cache, writes queued, syncs when online
- **Offline writes:** Automatically queued, tracked via metadata, sync conflicts handled
- **Cache management:** Clear cache when needed, set size limits, automatic eviction
- **Syncing strategy:** Automatic sync by default, no manual intervention needed
- **Offline data access:** Load from cache, handle stale data, notify users

### Checklist: You Are Ready When You Can...

- [ ] Enable persistence before Firestore operations
- [ ] Understand that reads work offline from cache
- [ ] Understand that writes queue automatically when offline
- [ ] Track pending writes using metadata
- [ ] Show offline/pending indicators to users
- [ ] Handle stale data gracefully
- [ ] Configure cache size if needed
- [ ] Clear cache when appropriate (logout, etc.)
- [ ] Test offline functionality
- [ ] Monitor sync status
- [ ] Handle offline writes correctly
- [ ] Provide user feedback for offline state

### Verification Steps

**Test your offline support:**

1. **Test offline reads:**
```dart
// Read data online first
final doc1 = await noteRef.get();

// Disable network (emulator or device settings)
// Read again - should use cache
final doc2 = await noteRef.get();
assert(doc2.metadata.isFromCache == true);
```

2. **Test offline writes:**
```dart
// Disable network
// Write - should queue
await noteRef.update({'title': 'New Title'});

// Check pending writes
final doc = await noteRef.get();
assert(doc.metadata.hasPendingWrites == true);

// Re-enable network
// Should sync automatically
```

3. **Test persistence:**
```dart
// Enable persistence
await FirebaseFirestore.instance.enablePersistence();

// Read data
await noteRef.get();

// Restart app
// Data should still be available offline
```

### Next Steps

Once you can implement offline support, you're ready to:
- Deploy to production (Chapter 16)
- Monitor and optimize your app
