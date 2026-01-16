# 13. ERROR HANDLING & BEST PRACTICES

## 1. What This Chapter Covers

This chapter teaches you how to handle errors and follow best practices in Firestore for your Notes application. You'll learn:

- Common Firestore errors and how to handle them
- Error handling strategies and patterns
- Retry logic with exponential backoff
- Code organization patterns (repository, service layer)
- Testing strategies with emulators and mocks
- Debugging techniques and monitoring

Error handling is critical because it directly impacts user experience and app reliability. Poor error handling leads to crashes, data loss, and frustrated users. Understanding common errors, retry strategies, and best practices helps you build robust, production-ready applications.

## 2. Core Concepts (Simple & Clear)

### 2.1 Common Errors

Firestore operations can fail for various reasons. Understanding common errors helps you handle them gracefully.

#### PERMISSION_DENIED

Occurs when security rules block an operation:

```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'permission-denied') {
    // Security rules blocked the operation
    showError('You do not have permission to update this note');
  }
}
```

**Common causes:**
- User not authenticated
- User doesn't own the document
- Security rules validation failed
- Missing required fields

**Handling:**
```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'permission-denied') {
    // Check authentication
    if (FirebaseAuth.instance.currentUser == null) {
      // Prompt user to sign in
      navigateToLogin();
    } else {
      // User doesn't have permission
      showError('You cannot modify this note');
    }
  }
}
```

#### NOT_FOUND

Occurs when trying to read or update a non-existent document:

```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'not-found') {
    // Document doesn't exist
    showError('Note not found');
  }
}
```

**Handling:**
```dart
// Check if document exists before updating
final doc = await noteRef.get();
if (doc.exists) {
  await noteRef.update({'title': 'New Title'});
} else {
  // Document doesn't exist, create it
  await noteRef.set({'title': 'New Title'});
}

// Or use update() with error handling
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'not-found') {
    // Create document if it doesn't exist
    await noteRef.set({'title': 'New Title'});
  }
}
```

#### ALREADY_EXISTS

Occurs when trying to create a document that already exists:

```dart
try {
  await noteRef.create({'title': 'New Note'});
} on FirebaseException catch (e) {
  if (e.code == 'already-exists') {
    // Document already exists
    showError('Note already exists');
  }
}
```

**Handling:**
```dart
// Use set() with merge instead of create()
await noteRef.set({
  'title': 'New Note',
}, SetOptions(merge: true));

// Or check existence first
final doc = await noteRef.get();
if (!doc.exists) {
  await noteRef.create({'title': 'New Note'});
} else {
  // Document already exists
  showError('Note already exists');
}
```

#### ABORTED (Transaction Conflicts)

Occurs when a transaction conflicts with another transaction:

```dart
try {
  await FirebaseFirestore.instance.runTransaction((transaction) async {
    final snapshot = await transaction.get(noteRef);
    transaction.update(noteRef, {
      'viewCount': (snapshot.data()?['viewCount'] ?? 0) + 1,
    });
  });
} on FirebaseException catch (e) {
  if (e.code == 'aborted') {
    // Transaction aborted due to conflict
    // Firestore automatically retries, but may exhaust retries
    showError('Update failed. Please try again.');
  }
}
```

**Handling:**
```dart
// Transactions retry automatically
// Add manual retry if needed
Future<void> updateWithRetry(int maxRetries) async {
  for (int i = 0; i < maxRetries; i++) {
    try {
      await FirebaseFirestore.instance.runTransaction((transaction) async {
        final snapshot = await transaction.get(noteRef);
        transaction.update(noteRef, {
          'viewCount': (snapshot.data()?['viewCount'] ?? 0) + 1,
        });
      });
      return; // Success
    } on FirebaseException catch (e) {
      if (e.code == 'aborted' && i < maxRetries - 1) {
        // Wait before retry
        await Future.delayed(Duration(milliseconds: 100 * (i + 1)));
        continue;
      }
      rethrow;
    }
  }
}
```

#### RESOURCE_EXHAUSTED

Occurs when quotas are exceeded or rate limits are hit:

```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'resource-exhausted') {
    // Quota exceeded or rate limited
    showError('Too many requests. Please try again later.');
  }
}
```

**Common causes:**
- Free tier daily quota exceeded
- Write rate limit exceeded (too many writes to same document)
- Too many concurrent operations

**Handling:**
```dart
// Implement rate limiting
class RateLimiter {
  static final Map<String, DateTime> _lastRequest = {};
  static const minDelay = Duration(seconds: 1);
  
  static Future<void> throttle(String key) async {
    final lastRequest = _lastRequest[key];
    if (lastRequest != null) {
      final timeSinceLastRequest = DateTime.now().difference(lastRequest);
      if (timeSinceLastRequest < minDelay) {
        await Future.delayed(minDelay - timeSinceLastRequest);
      }
    }
    _lastRequest[key] = DateTime.now();
  }
}

// Use rate limiter
await RateLimiter.throttle('update-note-$noteId');
await noteRef.update({'title': 'New Title'});
```

#### Unauthenticated

Occurs when user is not authenticated:

```dart
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'unauthenticated') {
    // User not authenticated
    navigateToLogin();
  }
}
```

**Handling:**
```dart
// Check authentication before operations
Future<void> updateNote(String noteId, Map<String, dynamic> data) async {
  final user = FirebaseAuth.instance.currentUser;
  if (user == null) {
    throw Exception('User must be authenticated');
  }
  
  try {
    await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .update(data);
  } on FirebaseException catch (e) {
    if (e.code == 'unauthenticated') {
      // Token expired, refresh or re-authenticate
      await user.getIdToken(true); // Refresh token
      // Retry operation
      await FirebaseFirestore.instance
          .collection('notes')
          .doc(noteId)
          .update(data);
    } else {
      rethrow;
    }
  }
}
```

### 2.2 Error Handling Strategies

Implement robust error handling throughout your application.

#### Try-Catch Blocks

Use try-catch to handle errors:

```dart
Future<Note?> getNote(String noteId) async {
  try {
    final doc = await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .get();
    
    if (doc.exists) {
      return Note.fromFirestore(doc);
    }
    return null;
  } on FirebaseException catch (e) {
    // Handle Firestore-specific errors
    _handleFirestoreError(e);
    return null;
  } catch (e) {
    // Handle other errors
    print('Unexpected error: $e');
    return null;
  }
}

void _handleFirestoreError(FirebaseException e) {
  switch (e.code) {
    case 'permission-denied':
      showError('Permission denied');
      break;
    case 'not-found':
      showError('Note not found');
      break;
    case 'unavailable':
      showError('Service unavailable. Please try again.');
      break;
    default:
      showError('Error: ${e.message}');
  }
}
```

#### Async Error Handling

Handle errors in async operations:

```dart
Future<void> createNote(Map<String, dynamic> data) async {
  try {
    await FirebaseFirestore.instance
        .collection('notes')
        .add(data);
    
    showSuccess('Note created successfully');
  } on FirebaseException catch (e) {
    _handleFirestoreError(e);
  } catch (e) {
    showError('Failed to create note: $e');
  }
}

// With async error handling in UI
void onCreateNotePressed() async {
  setState(() => _isLoading = true);
  
  try {
    await createNote(_noteData);
  } catch (e) {
    // Error already handled in createNote
  } finally {
    setState(() => _isLoading = false);
  }
}
```

#### Retry Mechanisms

Implement retry logic for transient errors:

```dart
Future<T> retryOperation<T>(
  Future<T> Function() operation,
  {int maxRetries = 3, Duration delay = const Duration(seconds: 1)}
) async {
  for (int i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } on FirebaseException catch (e) {
      // Retry on transient errors
      if (_isTransientError(e) && i < maxRetries - 1) {
        await Future.delayed(delay * (i + 1)); // Exponential backoff
        continue;
      }
      rethrow;
    }
  }
  throw Exception('Operation failed after $maxRetries retries');
}

bool _isTransientError(FirebaseException e) {
  return e.code == 'unavailable' ||
         e.code == 'deadline-exceeded' ||
         e.code == 'resource-exhausted';
}
```

#### User Feedback

Provide clear feedback to users:

```dart
class ErrorHandler {
  static void showError(String message) {
    // Show error message to user
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.red,
        duration: Duration(seconds: 3),
      ),
    );
  }
  
  static void showSuccess(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.green,
        duration: Duration(seconds: 2),
      ),
    );
  }
}

// Usage
try {
  await noteRef.update({'title': 'New Title'});
  ErrorHandler.showSuccess('Note updated');
} on FirebaseException catch (e) {
  ErrorHandler.showError(_getErrorMessage(e));
}
```

### 2.3 Retry Logic

Implement intelligent retry mechanisms for transient failures.

#### Exponential Backoff

Use exponential backoff to avoid overwhelming the server:

```dart
Future<T> retryWithBackoff<T>(
  Future<T> Function() operation,
  {int maxRetries = 5}
) async {
  for (int attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } on FirebaseException catch (e) {
      if (!_isTransientError(e) || attempt == maxRetries - 1) {
        rethrow;
      }
      
      // Exponential backoff: 1s, 2s, 4s, 8s, 16s
      final delay = Duration(seconds: 1 << attempt);
      await Future.delayed(delay);
    }
  }
  throw Exception('Operation failed after $maxRetries retries');
}

bool _isTransientError(FirebaseException e) {
  return e.code == 'unavailable' ||
         e.code == 'deadline-exceeded' ||
         e.code == 'resource-exhausted';
}
```

#### Transaction Retry Limits

Firestore automatically retries transactions, but they can still fail:

```dart
Future<void> updateWithTransaction(int maxRetries) async {
  for (int i = 0; i < maxRetries; i++) {
    try {
      await FirebaseFirestore.instance.runTransaction((transaction) async {
        final snapshot = await transaction.get(noteRef);
        transaction.update(noteRef, {
          'viewCount': (snapshot.data()?['viewCount'] ?? 0) + 1,
        });
      });
      return; // Success
    } on FirebaseException catch (e) {
      if (e.code == 'aborted' && i < maxRetries - 1) {
        // Wait before retry (transactions retry internally, but we add delay)
        await Future.delayed(Duration(milliseconds: 100 * (i + 1)));
        continue;
      }
      rethrow;
    }
  }
}
```

#### Network Error Handling

Handle network errors gracefully:

```dart
Future<T> executeWithNetworkHandling<T>(Future<T> Function() operation) async {
  try {
    return await operation();
  } on FirebaseException catch (e) {
    if (e.code == 'unavailable') {
      // Network error - check connectivity
      final connectivityResult = await Connectivity().checkConnectivity();
      if (connectivityResult == ConnectivityResult.none) {
        throw NetworkException('No internet connection');
      }
      
      // Retry with backoff
      return await retryWithBackoff(operation);
    }
    rethrow;
  }
}
```

### 2.4 Code Organization

Organize code for maintainability and testability.

#### Repository Pattern

Use repository pattern to abstract Firestore operations:

```dart
abstract class NotesRepository {
  Future<List<Note>> getNotes(String userId);
  Future<Note?> getNote(String noteId);
  Future<String> createNote(Note note);
  Future<void> updateNote(String noteId, Map<String, dynamic> data);
  Future<void> deleteNote(String noteId);
}

class FirestoreNotesRepository implements NotesRepository {
  final FirebaseFirestore _firestore;
  
  FirestoreNotesRepository(this._firestore);
  
  @override
  Future<List<Note>> getNotes(String userId) async {
    try {
      final snapshot = await _firestore
          .collection('notes')
          .where('userId', isEqualTo: userId)
          .get();
      
      return snapshot.docs
          .map((doc) => Note.fromFirestore(doc))
          .toList();
    } on FirebaseException catch (e) {
      throw NotesRepositoryException('Failed to get notes: ${e.message}');
    }
  }
  
  @override
  Future<Note?> getNote(String noteId) async {
    try {
      final doc = await _firestore
          .collection('notes')
          .doc(noteId)
          .get();
      
      if (doc.exists) {
        return Note.fromFirestore(doc);
      }
      return null;
    } on FirebaseException catch (e) {
      if (e.code == 'not-found') {
        return null;
      }
      throw NotesRepositoryException('Failed to get note: ${e.message}');
    }
  }
  
  // Other methods...
}
```

#### Service Layer

Create service layer for business logic:

```dart
class NotesService {
  final NotesRepository _repository;
  
  NotesService(this._repository);
  
  Future<List<Note>> getUserNotes(String userId) async {
    try {
      return await _repository.getNotes(userId);
    } catch (e) {
      // Log error, handle business logic
      throw NotesServiceException('Failed to load notes: $e');
    }
  }
  
  Future<void> archiveNote(String noteId) async {
    try {
      await _repository.updateNote(noteId, {
        'isArchived': true,
        'archivedAt': FieldValue.serverTimestamp(),
      });
    } catch (e) {
      throw NotesServiceException('Failed to archive note: $e');
    }
  }
}
```

#### Provider/Bloc Patterns

Use state management for UI updates:

```dart
// Provider pattern
class NotesProvider extends ChangeNotifier {
  final NotesService _service;
  List<Note> _notes = [];
  bool _isLoading = false;
  String? _error;
  
  NotesProvider(this._service);
  
  List<Note> get notes => _notes;
  bool get isLoading => _isLoading;
  String? get error => _error;
  
  Future<void> loadNotes(String userId) async {
    _isLoading = true;
    _error = null;
    notifyListeners();
    
    try {
      _notes = await _service.getUserNotes(userId);
    } catch (e) {
      _error = e.toString();
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }
}
```

#### Dependency Injection

Use dependency injection for testability:

```dart
// Inject dependencies
class NotesScreen extends StatelessWidget {
  final NotesService _notesService;
  
  NotesScreen({NotesService? notesService})
      : _notesService = notesService ?? NotesService(
          FirestoreNotesRepository(FirebaseFirestore.instance),
        );
  
  // Use _notesService in widget
}
```

### 2.5 Testing

Test Firestore operations with emulators and mocks.

#### Unit Tests with Mocks

Mock Firestore for unit tests:

```dart
// Using mockito
class MockFirestore extends Mock implements FirebaseFirestore {}
class MockCollectionReference extends Mock implements CollectionReference {}
class MockDocumentReference extends Mock implements DocumentReference {}
class MockQuerySnapshot extends Mock implements QuerySnapshot {}

void main() {
  group('NotesRepository', () {
    late MockFirestore mockFirestore;
    late NotesRepository repository;
    
    setUp(() {
      mockFirestore = MockFirestore();
      repository = FirestoreNotesRepository(mockFirestore);
    });
    
    test('getNotes returns list of notes', () async {
      // Arrange
      final mockSnapshot = MockQuerySnapshot();
      when(mockFirestore.collection('notes')).thenReturn(mockCollectionRef);
      when(mockCollectionRef.where('userId', isEqualTo: 'user123')).thenReturn(mockQuery);
      when(mockQuery.get()).thenAnswer((_) async => mockSnapshot);
      when(mockSnapshot.docs).thenReturn([mockDoc1, mockDoc2]);
      
      // Act
      final notes = await repository.getNotes('user123');
      
      // Assert
      expect(notes.length, 2);
    });
  });
}
```

#### Integration Tests

Use Firebase Emulator for integration tests:

```dart
// Setup emulator
void main() {
  setUpAll(() async {
    FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
    await Firebase.initializeApp();
  });
  
  test('create and read note', () async {
    // Create note
    final noteRef = FirebaseFirestore.instance.collection('notes').doc();
    await noteRef.set({
      'title': 'Test Note',
      'userId': 'user123',
    });
    
    // Read note
    final doc = await noteRef.get();
    expect(doc.exists, true);
    expect(doc.data()?['title'], 'Test Note');
  });
}
```

#### Firebase Emulator

Use emulator for local testing:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Use emulator in test/debug mode
  if (kDebugMode) {
    FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
  }
  
  await Firebase.initializeApp();
  runApp(MyApp());
}
```

#### Test Data Seeding

Seed test data for consistent tests:

```dart
Future<void> seedTestData() async {
  final batch = FirebaseFirestore.instance.batch();
  
  // Create test notes
  for (int i = 0; i < 10; i++) {
    final noteRef = FirebaseFirestore.instance.collection('notes').doc('note$i');
    batch.set(noteRef, {
      'title': 'Test Note $i',
      'userId': 'user123',
      'createdAt': FieldValue.serverTimestamp(),
    });
  }
  
  await batch.commit();
}
```

### 2.6 Debugging

Use debugging tools and techniques to troubleshoot issues.

#### Firebase Console

Use Firebase Console to inspect data and operations:

**Data inspection:**
- View documents in Console
- Check document structure
- Verify field values

**Monitoring:**
- Check usage statistics
- Monitor read/write operations
- View error logs

#### Logging Strategies

Implement structured logging:

```dart
class Logger {
  static void logInfo(String message, {Map<String, dynamic>? data}) {
    if (kDebugMode) {
      print('[INFO] $message');
      if (data != null) {
        print('Data: $data');
      }
    }
  }
  
  static void logError(String message, Object error, {StackTrace? stackTrace}) {
    print('[ERROR] $message');
    print('Error: $error');
    if (stackTrace != null) {
      print('Stack trace: $stackTrace');
    }
    
    // Send to error tracking service
    // FirebaseCrashlytics.instance.recordError(error, stackTrace);
  }
}

// Usage
try {
  await noteRef.update({'title': 'New Title'});
  Logger.logInfo('Note updated', data: {'noteId': noteId});
} catch (e, stackTrace) {
  Logger.logError('Failed to update note', e, stackTrace: stackTrace);
}
```

#### Performance Monitoring

Monitor performance metrics:

```dart
Future<T> measurePerformance<T>(String operation, Future<T> Function() fn) async {
  final stopwatch = Stopwatch()..start();
  try {
    final result = await fn();
    stopwatch.stop();
    Logger.logInfo('$operation completed', data: {
      'duration': stopwatch.elapsedMilliseconds,
    });
    return result;
  } catch (e) {
    stopwatch.stop();
    Logger.logError('$operation failed', e, data: {
      'duration': stopwatch.elapsedMilliseconds,
    });
    rethrow;
  }
}

// Usage
await measurePerformance('updateNote', () async {
  await noteRef.update({'title': 'New Title'});
});
```

#### Error Tracking

Track errors with Firebase Crashlytics or similar:

```dart
Future<void> updateNote(String noteId, Map<String, dynamic> data) async {
  try {
    await FirebaseFirestore.instance
        .collection('notes')
        .doc(noteId)
        .update(data);
  } catch (e, stackTrace) {
    // Log to Crashlytics
    FirebaseCrashlytics.instance.recordError(
      e,
      stackTrace,
      reason: 'Failed to update note',
      information: ['noteId: $noteId', 'data: $data'],
    );
    rethrow;
  }
}
```

## 3. Practical Examples

### Example: Comprehensive Error Handler

```dart
class FirestoreErrorHandler {
  static String getErrorMessage(FirebaseException e) {
    switch (e.code) {
      case 'permission-denied':
        return 'You do not have permission to perform this operation';
      case 'not-found':
        return 'The requested resource was not found';
      case 'already-exists':
        return 'This resource already exists';
      case 'aborted':
        return 'Operation was aborted. Please try again';
      case 'resource-exhausted':
        return 'Too many requests. Please try again later';
      case 'unauthenticated':
        return 'You must be signed in to perform this operation';
      case 'unavailable':
        return 'Service unavailable. Please check your connection';
      case 'deadline-exceeded':
        return 'Operation timed out. Please try again';
      default:
        return 'An error occurred: ${e.message}';
    }
  }
  
  static bool isRetryable(FirebaseException e) {
    return e.code == 'unavailable' ||
           e.code == 'deadline-exceeded' ||
           e.code == 'resource-exhausted';
  }
  
  static Future<T> executeWithRetry<T>(
    Future<T> Function() operation,
    {int maxRetries = 3}
  ) async {
    for (int i = 0; i < maxRetries; i++) {
      try {
        return await operation();
      } on FirebaseException catch (e) {
        if (isRetryable(e) && i < maxRetries - 1) {
          await Future.delayed(Duration(seconds: 1 << i)); // Exponential backoff
          continue;
        }
        throw NotesException(getErrorMessage(e), e);
      }
    }
    throw NotesException('Operation failed after $maxRetries retries');
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Try-catch vs Error callbacks:**
- **Try-catch:** Dart/Flutter standard, easier to read, better stack traces
- **Error callbacks:** Less common in Dart, harder to compose
- **Recommendation:** Use try-catch for error handling in Dart.

**Repository pattern vs Direct Firestore calls:**
- **Repository:** Testable, maintainable, abstracted
- **Direct calls:** Simpler, less code, but harder to test
- **Recommendation:** Use repository pattern for production apps.

**Exponential backoff vs Fixed delay:**
- **Exponential backoff:** Reduces server load, standard practice
- **Fixed delay:** Simpler, but may overwhelm server
- **Recommendation:** Use exponential backoff for retries.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Not handling errors**
```dart
// BAD: No error handling
await noteRef.update({'title': 'New Title'});

// GOOD: Handle errors
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  _handleError(e);
}
```

**2. Catching all errors without distinction**
```dart
// BAD: Generic error handling
try {
  await noteRef.update({'title': 'New Title'});
} catch (e) {
  print('Error: $e'); // Doesn't distinguish error types
}

// GOOD: Handle specific errors
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'permission-denied') {
    // Handle permission error
  } else if (e.code == 'not-found') {
    // Handle not found error
  }
}
```

**3. Not retrying transient errors**
```dart
// BAD: No retry for transient errors
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  if (e.code == 'unavailable') {
    showError('Service unavailable'); // Should retry
  }
}

// GOOD: Retry transient errors
await retryWithBackoff(() async {
  await noteRef.update({'title': 'New Title'});
});
```

## 6. Summary & Checklist

### Summary

- **Common errors:** PERMISSION_DENIED, NOT_FOUND, ALREADY_EXISTS, ABORTED, RESOURCE_EXHAUSTED, unauthenticated
- **Error handling:** Use try-catch blocks, handle specific error codes, provide user feedback
- **Retry logic:** Use exponential backoff for transient errors, respect retry limits
- **Code organization:** Use repository pattern, service layer, state management
- **Testing:** Use mocks for unit tests, emulator for integration tests
- **Debugging:** Use Firebase Console, structured logging, performance monitoring, error tracking

### Checklist: You Are Ready When You Can...

- [ ] Handle common Firestore errors (PERMISSION_DENIED, NOT_FOUND, etc.)
- [ ] Implement try-catch blocks for error handling
- [ ] Use exponential backoff for retries
- [ ] Provide clear error messages to users
- [ ] Organize code using repository pattern
- [ ] Write unit tests with mocks
- [ ] Use Firebase Emulator for integration tests
- [ ] Implement structured logging
- [ ] Monitor performance and errors
- [ ] Distinguish between retryable and non-retryable errors
- [ ] Handle authentication errors gracefully
- [ ] Test error scenarios in your application

### Verification Steps

**Test your error handling:**

1. **Test permission errors:**
```dart
// Sign out
await FirebaseAuth.instance.signOut();

// Try to update note
try {
  await noteRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  assert(e.code == 'permission-denied');
}
```

2. **Test not found errors:**
```dart
// Try to update non-existent document
try {
  await nonExistentRef.update({'title': 'New Title'});
} on FirebaseException catch (e) {
  assert(e.code == 'not-found');
}
```

3. **Test retry logic:**
```dart
// Simulate transient error and verify retry
await retryWithBackoff(() async {
  // Operation that might fail
});
```

### Next Steps

Once you can handle errors effectively, you're ready to:
- Integrate with Flutter/Dart (Chapter 14)
- Deploy to production (Chapter 16)
