# 7. REAL-TIME LISTENERS

## 1. What This Chapter Covers

This chapter teaches you how to set up real-time listeners in Firestore for your Notes application. You'll learn:

- How to listen to document and collection changes in real-time
- How to handle snapshot events and detect changes
- How to manage listener lifecycle and prevent memory leaks
- How to work with metadata to detect offline/cache states
- How to optimize listeners for performance and cost
- How to handle errors and implement reconnection strategies

Real-time listeners are essential for collaborative features, live updates, and responsive user experiences. However, they consume resources and can be expensive if not managed properly. Understanding listener lifecycle, metadata, and optimization helps you build efficient real-time applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Setting Up Listeners

Real-time listeners automatically receive updates when data changes in Firestore.

#### .snapshots() Method

The `snapshots()` method returns a stream that emits updates whenever data changes:

```dart
// Listen to a single document
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

final stream = noteRef.snapshots();
```

**For collections:**
```dart
// Listen to a collection query
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'user123');

final stream = query.snapshots();
```

#### Stream<QuerySnapshot> or Stream<DocumentSnapshot>

Listeners return different stream types:

**Document listener:**
```dart
// Returns Stream<DocumentSnapshot>
final docStream = noteRef.snapshots();

docStream.listen((DocumentSnapshot snapshot) {
  if (snapshot.exists) {
    final data = snapshot.data();
    print('Document updated: $data');
  }
});
```

**Collection listener:**
```dart
// Returns Stream<QuerySnapshot>
final queryStream = query.snapshots();

queryStream.listen((QuerySnapshot snapshot) {
  print('Collection updated: ${snapshot.size} documents');
  for (var doc in snapshot.docs) {
    print('Document: ${doc.id}');
  }
});
```

#### Listening to Collection or Document

**Single document listener:**
```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('note123');

noteRef.snapshots().listen((snapshot) {
  if (snapshot.exists) {
    final data = snapshot.data()!;
    // Update UI with new data
  } else {
    // Document was deleted
  }
});
```

**Collection query listener:**
```dart
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .orderBy('createdAt', descending: true)
    .limit(20);

query.snapshots().listen((snapshot) {
  // snapshot.docs contains all matching documents
  // Updates automatically when documents are added, modified, or removed
  final notes = snapshot.docs.map((doc) => Note.fromFirestore(doc)).toList();
  // Update UI
});
```

### 2.2 Snapshot Events

Snapshots provide information about document and collection state.

#### QuerySnapshot (for Collections)

`QuerySnapshot` contains all documents matching a query:

```dart
query.snapshots().listen((QuerySnapshot snapshot) {
  // Total number of documents
  print('Total documents: ${snapshot.size}');
  
  // Check if empty
  if (snapshot.empty) {
    print('No documents found');
  }
  
  // Access all documents
  for (var doc in snapshot.docs) {
    print('Document ID: ${doc.id}');
    print('Data: ${doc.data()}');
  }
  
  // Metadata
  final metadata = snapshot.metadata;
  if (metadata.isFromCache) {
    print('Data from cache (offline)');
  }
});
```

**Document changes:**
```dart
query.snapshots().listen((QuerySnapshot snapshot) {
  // Get document changes
  snapshot.docChanges.forEach((change) {
    switch (change.type) {
      case DocumentChangeType.added:
        print('Document added: ${change.doc.id}');
        break;
      case DocumentChangeType.modified:
        print('Document modified: ${change.doc.id}');
        break;
      case DocumentChangeType.removed:
        print('Document removed: ${change.doc.id}');
        break;
    }
  });
});
```

#### DocumentSnapshot (for Single Document)

`DocumentSnapshot` represents a single document:

```dart
noteRef.snapshots().listen((DocumentSnapshot snapshot) {
  if (snapshot.exists) {
    final data = snapshot.data()!;
    print('Document exists: $data');
    
    // Check if document has pending writes
    if (snapshot.metadata.hasPendingWrites) {
      print('Document has local changes not yet synced');
    }
  } else {
    print('Document does not exist');
  }
});
```

#### Change Events (added, modified, removed)

Detect specific types of changes:

```dart
query.snapshots().listen((QuerySnapshot snapshot) {
  for (var change in snapshot.docChanges) {
    switch (change.type) {
      case DocumentChangeType.added:
        // New document added to query results
        final newNote = Note.fromFirestore(change.doc);
        // Add to UI
        break;
        
      case DocumentChangeType.modified:
        // Existing document was modified
        final updatedNote = Note.fromFirestore(change.doc);
        // Update in UI
        break;
        
      case DocumentChangeType.removed:
        // Document removed from query results
        final removedNoteId = change.doc.id;
        // Remove from UI
        break;
    }
  }
});
```

**Using change indices:**
```dart
snapshot.docChanges.forEach((change) {
  final index = change.newIndex; // New position in results
  final oldIndex = change.oldIndex; // Previous position
  
  switch (change.type) {
    case DocumentChangeType.added:
      // Insert at newIndex
      break;
    case DocumentChangeType.modified:
      // Move from oldIndex to newIndex if order changed
      break;
    case DocumentChangeType.removed:
      // Remove from oldIndex
      break;
  }
});
```

### 2.3 Listener Lifecycle

Properly managing listener lifecycle prevents memory leaks and unnecessary costs.

#### Subscribing to Stream

Subscribe to the stream and store the subscription:

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
        // Handle updates
        setState(() {
          // Update state
        });
      },
      onError: (error) {
        // Handle errors
        print('Listener error: $error');
      },
    );
  }
}
```

#### Canceling Subscription

Always cancel subscriptions to prevent memory leaks:

```dart
@override
void dispose() {
  _subscription?.cancel();
  super.dispose();
}
```

**Important:** If you don't cancel subscriptions, listeners continue running in the background, consuming resources and generating read costs.

#### StreamBuilder in Flutter

`StreamBuilder` automatically manages subscription lifecycle:

```dart
StreamBuilder<QuerySnapshot>(
  stream: FirebaseFirestore.instance
      .collection('notes')
      .where('userId', isEqualTo: currentUserId)
      .snapshots(),
  builder: (context, snapshot) {
    if (snapshot.hasError) {
      return Text('Error: ${snapshot.error}');
    }
    
    if (snapshot.connectionState == ConnectionState.waiting) {
      return CircularProgressIndicator();
    }
    
    if (!snapshot.hasData || snapshot.data!.empty) {
      return Text('No notes found');
    }
    
    final notes = snapshot.data!.docs
        .map((doc) => Note.fromFirestore(doc))
        .toList();
    
    return ListView.builder(
      itemCount: notes.length,
      itemBuilder: (context, index) {
        return NoteTile(note: notes[index]);
      },
    );
  },
);
```

**Benefits:**
- Automatically subscribes when widget builds
- Automatically cancels when widget disposes
- Handles connection states
- Built-in error handling

#### Using dispose() Properly

Always clean up in `dispose()`:

```dart
class NoteDetailState extends State<NoteDetail> {
  StreamSubscription<DocumentSnapshot>? _noteSubscription;
  StreamSubscription<QuerySnapshot>? _commentsSubscription;
  
  @override
  void initState() {
    super.initState();
    
    // Listen to note
    _noteSubscription = noteRef.snapshots().listen((snapshot) {
      setState(() {
        // Update note
      });
    });
    
    // Listen to comments
    _commentsSubscription = commentsRef.snapshots().listen((snapshot) {
      setState(() {
        // Update comments
      });
    });
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

### 2.4 Metadata Changes

Metadata provides information about data source and sync status.

#### Listen to Source Changes

Detect whether data comes from cache or server:

```dart
query.snapshots().listen((QuerySnapshot snapshot) {
  final metadata = snapshot.metadata;
  
  if (metadata.isFromCache) {
    // Data is from local cache (offline)
    showOfflineIndicator();
  } else {
    // Data is from server (online)
    hideOfflineIndicator();
  }
});
```

#### hasPendingWrites

Check if document has local changes not yet synced:

```dart
noteRef.snapshots().listen((DocumentSnapshot snapshot) {
  if (snapshot.metadata.hasPendingWrites) {
    // Document has local changes waiting to sync
    showPendingIndicator();
  } else {
    // All changes synced
    hidePendingIndicator();
  }
});
```

**Use case:** Show "Saving..." indicator:
```dart
StreamBuilder<DocumentSnapshot>(
  stream: noteRef.snapshots(),
  builder: (context, snapshot) {
    if (!snapshot.hasData) return CircularProgressIndicator();
    
    final hasPendingWrites = snapshot.data!.metadata.hasPendingWrites;
    
    return Column(
      children: [
        if (hasPendingWrites)
          Text('Saving...', style: TextStyle(color: Colors.orange)),
        NoteContent(data: snapshot.data!.data()),
      ],
    );
  },
);
```

#### isFromCache

Determine if data is from cache (offline) or server (online):

```dart
query.snapshots().listen((QuerySnapshot snapshot) {
  final isFromCache = snapshot.metadata.isFromCache;
  
  if (isFromCache) {
    // User is offline, showing cached data
    showOfflineBanner();
  } else {
    // User is online, showing fresh data
    hideOfflineBanner();
  }
});
```

#### Source Enum (cache, server, default)

More granular source information:

```dart
query.snapshots(includeMetadataChanges: true).listen((snapshot) {
  final source = snapshot.metadata.source;
  
  switch (source) {
    case Source.server:
      print('Data from server');
      break;
    case Source.cache:
      print('Data from cache');
      break;
    case Source.defaultSource:
      print('Data from default source (cache or server)');
      break;
  }
});
```

**includeMetadataChanges:**
```dart
// Listen to metadata changes too (cache/server transitions)
query.snapshots(includeMetadataChanges: true).listen((snapshot) {
  // This fires even when only metadata changes (cache -> server)
  // Useful for showing offline/online indicators
});
```

### 2.5 Performance Considerations

Optimizing listeners reduces costs and improves performance.

#### Minimize Listener Count

Each active listener consumes resources:

```dart
// BAD: Multiple listeners for same data
final listener1 = notesRef.snapshots().listen(...);
final listener2 = notesRef.snapshots().listen(...);
final listener3 = notesRef.snapshots().listen(...);

// GOOD: Single listener, share data via state management
final listener = notesRef.snapshots().listen((snapshot) {
  // Update shared state (Provider, Riverpod, etc.)
  notesProvider.updateNotes(snapshot.docs);
});
```

**Best practice:** Use state management to share listener data across widgets instead of creating multiple listeners.

#### Use Proper Query Scopes

Limit listener scope to only necessary data:

```dart
// BAD: Listens to all notes
final allNotes = FirebaseFirestore.instance
    .collection('notes')
    .snapshots();

// GOOD: Listen only to user's notes with limit
final userNotes = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .orderBy('createdAt', descending: true)
    .limit(20)
    .snapshots();
```

**Benefits:**
- Fewer documents to sync
- Lower read costs
- Faster updates
- Less bandwidth usage

#### Offline Data Handling

Firestore caches data automatically for offline access:

```dart
// Enable offline persistence (usually enabled by default)
await FirebaseFirestore.instance.enablePersistence();

// Listeners work offline
query.snapshots().listen((snapshot) {
  if (snapshot.metadata.isFromCache) {
    // Show offline indicator
  }
  // Data is still available from cache
});
```

**Offline behavior:**
- Listeners continue working with cached data
- Writes are queued and sync when online
- No errors thrown for offline reads
- Automatic reconnection when online

### 2.6 Error Handling in Listeners

Handle errors gracefully to prevent crashes and provide good UX.

#### onError Callback

Handle errors in stream subscription:

```dart
_subscription = query.snapshots().listen(
  (snapshot) {
    // Handle data
  },
  onError: (error) {
    // Handle errors
    if (error is FirebaseException) {
      switch (error.code) {
        case 'permission-denied':
          showError('Permission denied');
          break;
        case 'unavailable':
          showError('Service unavailable');
          break;
        default:
          showError('Error: ${error.message}');
      }
    }
  },
);
```

#### Stream Error Handling

Handle errors in StreamBuilder:

```dart
StreamBuilder<QuerySnapshot>(
  stream: query.snapshots(),
  builder: (context, snapshot) {
    if (snapshot.hasError) {
      return ErrorWidget(
        message: snapshot.error.toString(),
        onRetry: () {
          // Retry logic
        },
      );
    }
    
    // Normal UI
  },
);
```

#### Reconnection Strategies

Firestore automatically reconnects, but you can add retry logic:

```dart
class NotesListener {
  StreamSubscription<QuerySnapshot>? _subscription;
  int _retryCount = 0;
  static const maxRetries = 3;
  
  void startListening() {
    _subscription = query.snapshots().listen(
      (snapshot) {
        _retryCount = 0; // Reset on success
        // Handle data
      },
      onError: (error) {
        if (_retryCount < maxRetries) {
          _retryCount++;
          Future.delayed(Duration(seconds: _retryCount * 2), () {
            startListening(); // Retry
          });
        } else {
          // Max retries reached, show error
          showError('Failed to connect after $maxRetries attempts');
        }
      },
    );
  }
}
```

**Note:** Firestore handles reconnection automatically. Manual retry is usually unnecessary unless you have specific requirements.

## 3. Practical Examples

### Example: Real-time Notes List

```dart
class NotesListWidget extends StatelessWidget {
  final String userId;
  
  @override
  Widget build(BuildContext context) {
    final query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .orderBy('createdAt', descending: true)
        .limit(50);
    
    return StreamBuilder<QuerySnapshot>(
      stream: query.snapshots(),
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Text('Error: ${snapshot.error}');
        }
        
        if (snapshot.connectionState == ConnectionState.waiting) {
          return CircularProgressIndicator();
        }
        
        if (!snapshot.hasData || snapshot.data!.empty) {
          return Text('No notes found');
        }
        
        final notes = snapshot.data!.docs
            .map((doc) => Note.fromFirestore(doc))
            .toList();
        
        return ListView.builder(
          itemCount: notes.length,
          itemBuilder: (context, index) {
            return NoteTile(note: notes[index]);
          },
        );
      },
    );
  }
}
```

### Example: Collaborative Note Editing

```dart
class CollaborativeNoteEditor extends StatefulWidget {
  final String noteId;
  
  @override
  State<CollaborativeNoteEditor> createState() => _CollaborativeNoteEditorState();
}

class _CollaborativeNoteEditorState extends State<CollaborativeNoteEditor> {
  StreamSubscription<DocumentSnapshot>? _subscription;
  final TextEditingController _controller = TextEditingController();
  bool _isSaving = false;
  
  @override
  void initState() {
    super.initState();
    
    final noteRef = FirebaseFirestore.instance
        .collection('notes')
        .doc(widget.noteId);
    
    _subscription = noteRef.snapshots().listen((snapshot) {
      if (snapshot.exists) {
        final data = snapshot.data()!;
        final content = data['content'] as String? ?? '';
        
        // Only update if content changed (avoid cursor jumps)
        if (_controller.text != content) {
          _controller.text = content;
        }
        
        setState(() {
          _isSaving = snapshot.metadata.hasPendingWrites;
        });
      }
    });
  }
  
  @override
  void dispose() {
    _subscription?.cancel();
    _controller.dispose();
    super.dispose();
  }
  
  void _onContentChanged(String newContent) {
    // Update Firestore (triggers listener for other users)
    FirebaseFirestore.instance
        .collection('notes')
        .doc(widget.noteId)
        .update({
          'content': newContent,
          'updatedAt': FieldValue.serverTimestamp(),
        });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_isSaving)
          LinearProgressIndicator(),
        TextField(
          controller: _controller,
          onChanged: _onContentChanged,
          maxLines: null,
        ),
      ],
    );
  }
}
```

### Example: Live Unread Count

```dart
class UnreadCountWidget extends StatelessWidget {
  final String userId;
  
  @override
  Widget build(BuildContext context) {
    final query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: userId)
        .where('isRead', isEqualTo: false);
    
    return StreamBuilder<QuerySnapshot>(
      stream: query.snapshots(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return SizedBox.shrink();
        }
        
        final unreadCount = snapshot.data!.size;
        
        if (unreadCount == 0) {
          return SizedBox.shrink();
        }
        
        return Badge(
          label: Text('$unreadCount'),
          child: Icon(Icons.notifications),
        );
      },
    );
  }
}
```

### Example: Offline Indicator

```dart
class OfflineIndicator extends StatelessWidget {
  final Query query;
  
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<QuerySnapshot>(
      stream: query.snapshots(includeMetadataChanges: true),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return SizedBox.shrink();
        }
        
        final isFromCache = snapshot.data!.metadata.isFromCache;
        
        if (isFromCache) {
          return Container(
            color: Colors.orange,
            padding: EdgeInsets.all(8),
            child: Text(
              'Offline - Showing cached data',
              style: TextStyle(color: Colors.white),
            ),
          );
        }
        
        return SizedBox.shrink();
      },
    );
  }
}
```

### Example: Document Change Tracking

```dart
class NotesListWithChanges extends StatefulWidget {
  @override
  State<NotesListWithChanges> createState() => _NotesListWithChangesState();
}

class _NotesListWithChangesState extends State<NotesListWithChanges> {
  final List<Note> _notes = [];
  
  @override
  void initState() {
    super.initState();
    
    final query = FirebaseFirestore.instance
        .collection('notes')
        .where('userId', isEqualTo: currentUserId)
        .orderBy('createdAt', descending: true);
    
    query.snapshots().listen((QuerySnapshot snapshot) {
      for (var change in snapshot.docChanges) {
        switch (change.type) {
          case DocumentChangeType.added:
            setState(() {
              _notes.insert(
                change.newIndex,
                Note.fromFirestore(change.doc),
              );
            });
            break;
            
          case DocumentChangeType.modified:
            setState(() {
              _notes[change.oldIndex] = Note.fromFirestore(change.doc);
              // Reorder if needed
              if (change.oldIndex != change.newIndex) {
                final note = _notes.removeAt(change.oldIndex);
                _notes.insert(change.newIndex, note);
              }
            });
            break;
            
          case DocumentChangeType.removed:
            setState(() {
              _notes.removeAt(change.oldIndex);
            });
            break;
        }
      }
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _notes.length,
      itemBuilder: (context, index) {
        return NoteTile(note: _notes[index]);
      },
    );
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Real-time listener vs One-time read:**
- **Listener:** Use for data that changes frequently, collaborative features, live updates
- **One-time read:** Use for static data, initial loads, background operations
- **Recommendation:** Use listeners for user-facing data that needs to stay current. Use one-time reads for data that rarely changes.

**StreamBuilder vs Manual subscription:**
- **StreamBuilder:** Automatic lifecycle, simpler code, good for simple cases
- **Manual subscription:** More control, better for complex state management
- **Recommendation:** Use StreamBuilder for simple UI updates. Use manual subscriptions with state management for complex apps.

**includeMetadataChanges: true vs false:**
- **true:** Fires on cache/server transitions, useful for offline indicators
- **false:** Only fires on data changes, more efficient
- **Recommendation:** Use `true` only when you need to show offline/online status. Use `false` otherwise.

**Single listener vs Multiple listeners:**
- **Single listener:** More efficient, share data via state management
- **Multiple listeners:** Simpler per-widget, but less efficient
- **Recommendation:** Prefer single listener with state management for shared data. Multiple listeners are fine for widget-specific data.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Not canceling subscriptions**
```dart
// BAD: Memory leak
void initState() {
  query.snapshots().listen((snapshot) {
    // Handle data
  });
  // Subscription never canceled!
}

// GOOD: Cancel in dispose
StreamSubscription? _subscription;

void initState() {
  _subscription = query.snapshots().listen(...);
}

void dispose() {
  _subscription?.cancel();
  super.dispose();
}
```

**2. Creating multiple listeners for same data**
```dart
// BAD: Multiple listeners
Widget build(BuildContext context) {
  return Column(
    children: [
      StreamBuilder(stream: query.snapshots(), ...), // Listener 1
      StreamBuilder(stream: query.snapshots(), ...), // Listener 2
      StreamBuilder(stream: query.snapshots(), ...), // Listener 3
    ],
  );
}

// GOOD: Single listener, share via state management
final notesProvider = StreamProvider((ref) => query.snapshots());

Widget build(BuildContext context) {
  final notes = ref.watch(notesProvider);
  // Use notes in multiple widgets
}
```

**3. Not handling errors**
```dart
// BAD: Crashes on error
query.snapshots().listen((snapshot) {
  // No error handling
});

// GOOD: Handle errors
query.snapshots().listen(
  (snapshot) {
    // Handle data
  },
  onError: (error) {
    // Handle error
    print('Error: $error');
  },
);
```

**4. Listening to too much data**
```dart
// BAD: Listens to all notes
final allNotes = FirebaseFirestore.instance
    .collection('notes')
    .snapshots();

// GOOD: Limit scope
final userNotes = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: currentUserId)
    .limit(20)
    .snapshots();
```

### Performance Pitfalls

**1. Not using proper query scopes**
```dart
// BAD: Listens to entire collection
collection('notes').snapshots()

// GOOD: Filter and limit
collection('notes')
    .where('userId', isEqualTo: userId)
    .limit(50)
    .snapshots()
```

**2. Updating UI on every metadata change**
```dart
// BAD: Rebuilds on cache/server transitions
query.snapshots(includeMetadataChanges: true).listen((snapshot) {
  setState(() {
    // Rebuilds even when only metadata changes
  });
});

// GOOD: Only update on data changes
query.snapshots().listen((snapshot) {
  setState(() {
    // Only rebuilds on actual data changes
  });
});
```

**3. Not debouncing rapid updates**
```dart
// BAD: Updates UI on every change (could be rapid)
query.snapshots().listen((snapshot) {
  setState(() {
    // Updates immediately, could be expensive
  });
});

// GOOD: Debounce rapid updates
Timer? _debounceTimer;
query.snapshots().listen((snapshot) {
  _debounceTimer?.cancel();
  _debounceTimer = Timer(Duration(milliseconds: 300), () {
    setState(() {
      // Update after debounce
    });
  });
});
```

## 6. Summary & Checklist

### Summary

- **Real-time listeners** use `snapshots()` to receive automatic updates when data changes
- **DocumentSnapshot** represents a single document, **QuerySnapshot** represents query results
- **Change events** (added, modified, removed) let you detect specific types of changes
- **Listener lifecycle** must be managed - always cancel subscriptions in `dispose()`
- **Metadata** (isFromCache, hasPendingWrites) provides information about data source and sync status
- **Performance** is optimized by minimizing listeners, using proper query scopes, and limiting data
- **Error handling** prevents crashes and provides good user experience

### Checklist: You Are Ready When You Can...

- [ ] Set up a listener for a single document using `snapshots()`
- [ ] Set up a listener for a collection query
- [ ] Handle DocumentSnapshot and QuerySnapshot in listeners
- [ ] Detect document changes (added, modified, removed) using `docChanges`
- [ ] Cancel subscriptions properly in `dispose()` to prevent memory leaks
- [ ] Use StreamBuilder for automatic lifecycle management
- [ ] Check metadata to detect offline state (`isFromCache`)
- [ ] Check metadata to detect pending writes (`hasPendingWrites`)
- [ ] Use `includeMetadataChanges` when you need offline indicators
- [ ] Optimize listeners by limiting query scope and minimizing listener count
- [ ] Handle errors in listeners using `onError` callback
- [ ] Choose between real-time listeners and one-time reads based on use case
- [ ] Implement collaborative features using real-time listeners

### Verification Steps

**Test your listener setup:**

1. **Test document listener:**
```dart
final noteRef = FirebaseFirestore.instance
    .collection('notes')
    .doc('test-note');

noteRef.snapshots().listen((snapshot) {
  print('Document updated: ${snapshot.exists}');
  if (snapshot.exists) {
    print('Data: ${snapshot.data()}');
  }
});

// Update document in another client/console
// Should see update in listener
```

2. **Test collection listener:**
```dart
final query = FirebaseFirestore.instance
    .collection('notes')
    .where('userId', isEqualTo: 'test-user')
    .limit(10);

query.snapshots().listen((snapshot) {
  print('Collection updated: ${snapshot.size} documents');
  snapshot.docChanges.forEach((change) {
    print('${change.type}: ${change.doc.id}');
  });
});
```

3. **Test subscription cancellation:**
```dart
final subscription = query.snapshots().listen(...);
// Verify subscription is active
assert(subscription.isPaused == false);

// Cancel
subscription.cancel();

// Verify canceled
// (Subscription will not receive further updates)
```

### Next Steps

Once you can set up real-time listeners, you're ready to:
- Use transactions for complex operations (Chapter 8)
- Apply advanced data modeling patterns (Chapter 9)
- Implement security rules (Chapter 10)
