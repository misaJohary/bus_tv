---
name: flutter-sqflite
description: Implements local relational database storage using sqflite. Use when building offline-first local data sources, managing migrations, transactions, or caching relational entities.
---

# Flutter SQFlite Local Database Pattern

> **What:** Comprehensive guide for implementing offline-first relational persistence using `sqflite` in Flutter Clean Architecture.
> **When to use:** Local caching, offline message storage, complex relational queries, search indexing, or persistent transactional data.
> **Depends on:** Works alongside `flutter-repository`, `flutter-clean-architecture`, and `flutter-serialization`.

`sqflite` is the official and most reliable SQLite plugin for Flutter. It provides full control over relational schemas, indices, transactions, and migration scripts.

---

## 1. Setup & Dependencies

Add `sqflite` and `path` to `pubspec.yaml`:

```yaml
dependencies:
  sqflite: ^2.4.4
  path: ^1.9.1

dev_dependencies:
  sqflite_common_ffi: ^2.3.4 # For unit & integration testing on desktop/CI
```

---

## 2. Database Service & Migration Architecture

A centralized `AppDatabase` singleton manages opening, schema creation, foreign keys, and versioned migrations:

```dart
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';

class AppDatabase {
  static const String _databaseName = 'app_database.db';
  static const int _databaseVersion = 2;

  Database? _db;

  Future<Database> get database async {
    if (_db != null && _db!.isOpen) return _db!;
    _db = await _initDatabase();
    return _db!;
  }

  Future<Database> _initDatabase() async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, _databaseName);

    return await openDatabase(
      path,
      version: _databaseVersion,
      onConfigure: _onConfigure,
      onCreate: _onCreate,
      onUpgrade: _onUpgrade,
    );
  }

  /// Enable SQLite foreign key constraints
  Future<void> _onConfigure(Database db) async {
    await db.execute('PRAGMA foreign_keys = ON');
  }

  /// Initial table creation (Version 1)
  Future<void> _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE messages (
        id TEXT PRIMARY KEY,
        chat_id TEXT NOT NULL,
        sender_id TEXT NOT NULL,
        content TEXT NOT NULL,
        status TEXT NOT NULL,
        created_at INTEGER NOT NULL
      )
    ''');

    await db.execute('''
      CREATE INDEX idx_messages_chat_id ON messages (chat_id, created_at DESC)
    ''');

    if (version >= 2) {
      await _migrateToV2(db);
    }
  }

  /// Incremental migrations
  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    if (oldVersion < 2) {
      await _migrateToV2(db);
    }
  }

  Future<void> _migrateToV2(Database db) async {
    await db.execute('''
      ALTER TABLE messages ADD COLUMN reply_to_id TEXT
    ''');
  }

  Future<void> close() async {
    if (_db != null && _db!.isOpen) {
      await _db!.close();
      _db = null;
    }
  }
}
```

---

## 3. Table Schema & Column Constants

Never hardcode SQL strings across the app. Define contracts for each table:

```dart
abstract class MessageTable {
  static const String tableName = 'messages';

  static const String colId = 'id';
  static const String colChatId = 'chat_id';
  static const String colSenderId = 'sender_id';
  static const String colContent = 'content';
  static const String colStatus = 'status';
  static const String colCreatedAt = 'created_at';
  static const String colReplyToId = 'reply_to_id';
}
```

---

## 4. Local Data Source Implementation

The Local Data Source encapsulates all database queries and transactions:

```dart
abstract class MessageLocalDataSource {
  Future<void> insertOrUpdateMessage(Map<String, dynamic> messageMap);
  Future<void> insertMessagesBatch(List<Map<String, dynamic>> messages);
  Future<List<Map<String, dynamic>>> getMessagesByChat(String chatId, {int limit = 50, int offset = 0});
  Future<int> deleteMessage(String id);
  Future<void> clearChat(String chatId);
}

class MessageLocalDataSourceImpl implements MessageLocalDataSource {
  final AppDatabase _appDatabase;

  MessageLocalDataSourceImpl(this._appDatabase);

  @override
  Future<void> insertOrUpdateMessage(Map<String, dynamic> messageMap) async {
    final db = await _appDatabase.database;
    await db.insert(
      MessageTable.tableName,
      messageMap,
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
  }

  @override
  Future<void> insertMessagesBatch(List<Map<String, dynamic>> messages) async {
    final db = await _appDatabase.database;
    final batch = db.batch();

    for (final message in messages) {
      batch.insert(
        MessageTable.tableName,
        message,
        conflictAlgorithm: ConflictAlgorithm.replace,
      );
    }

    await batch.commit(noResult: true);
  }

  @override
  Future<List<Map<String, dynamic>>> getMessagesByChat(
    String chatId, {
    int limit = 50,
    int offset = 0,
  }) async {
    final db = await _appDatabase.database;
    return await db.query(
      MessageTable.tableName,
      where: '${MessageTable.colChatId} = ?',
      whereArgs: [chatId],
      orderBy: '${MessageTable.colCreatedAt} DESC',
      limit: limit,
      offset: offset,
    );
  }

  @override
  Future<int> deleteMessage(String id) async {
    final db = await _appDatabase.database;
    return await db.delete(
      MessageTable.tableName,
      where: '${MessageTable.colId} = ?',
      whereArgs: [id],
    );
  }

  @override
  Future<void> clearChat(String chatId) async {
    final db = await _appDatabase.database;
    await db.delete(
      MessageTable.tableName,
      where: '${MessageTable.colChatId} = ?',
      whereArgs: [chatId],
    );
  }
}
```

---

## 5. Type Conversions & SQLite Compatibility

SQLite only supports `INTEGER`, `REAL`, `TEXT`, `BLOB`, and `NULL`. Convert rich Dart types accordingly:

| Dart Type | SQLite Column Type | Mapping to SQLite | Mapping from SQLite |
|---|---|---|---|
| `bool` | `INTEGER` | `value ? 1 : 0` | `value == 1` |
| `DateTime` | `INTEGER` | `dateTime.millisecondsSinceEpoch` | `DateTime.fromMillisecondsSinceEpoch(value)` |
| `DateTime` | `TEXT` | `dateTime.toIso8601String()` | `DateTime.parse(value)` |
| `Enum` | `TEXT` | `status.name` | `Status.values.byName(value)` |
| `List<String>` / `Map` | `TEXT` | `jsonEncode(value)` | `jsonDecode(value)` |

### Entity Mapping Example:

```dart
extension MessageDbMapper on MessageEntity {
  Map<String, dynamic> toDbMap() {
    return {
      MessageTable.colId: id,
      MessageTable.colChatId: chatId,
      MessageTable.colSenderId: senderId,
      MessageTable.colContent: content,
      MessageTable.colStatus: status.name,
      MessageTable.colCreatedAt: createdAt.millisecondsSinceEpoch,
      MessageTable.colReplyToId: replyToId,
    };
  }

  static MessageEntity fromDbMap(Map<String, dynamic> map) {
    return MessageEntity(
      id: map[MessageTable.colId] as String,
      chatId: map[MessageTable.colChatId] as String,
      senderId: map[MessageTable.colSenderId] as String,
      content: map[MessageTable.colContent] as String,
      status: MessageStatus.values.byName(map[MessageTable.colStatus] as String),
      createdAt: DateTime.fromMillisecondsSinceEpoch(map[MessageTable.colCreatedAt] as int),
      replyToId: map[MessageTable.colReplyToId] as String?,
    );
  }
}
```

---

## 6. Atomic Transactions & Batch Operations

### Transactions for Atomic Multi-Step Logic
Ensure that either all operations succeed or none are committed:

```dart
Future<void> transferFunds({
  required String fromAccount,
  required String toAccount,
  required double amount,
}) async {
  final db = await _appDatabase.database;

  await db.transaction((txn) async {
    // 1. Deduct from sender
    await txn.rawUpdate(
      'UPDATE accounts SET balance = balance - ? WHERE id = ?',
      [amount, fromAccount],
    );

    // 2. Add to recipient
    await txn.rawUpdate(
      'UPDATE accounts SET balance = balance + ? WHERE id = ?',
      [amount, toAccount],
    );

    // 3. Insert audit log
    await txn.insert('transactions', {
      'from_id': fromAccount,
      'to_id': toAccount,
      'amount': amount,
      'timestamp': DateTime.now().millisecondsSinceEpoch,
    });
  });
}
```

---

## 7. Unit Testing with `sqflite_common_ffi`

Test your SQLite queries in unit tests without starting an Android/iOS emulator:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sqflite_common_ffi/sqflite_ffi.dart';

void main() {
  setUpAll(() {
    // Initialize FFI for in-memory testing
    sqfliteFfiInit();
    databaseFactory = databaseFactoryFfi;
  });

  test('insert and read message from in-memory database', () async {
    final db = await openDatabase(
      inMemoryDatabasePath,
      version: 1,
      onCreate: (db, version) async {
        await db.execute('CREATE TABLE messages (id TEXT PRIMARY KEY, content TEXT)');
      },
    );

    await db.insert('messages', {'id': 'msg_1', 'content': 'Hello test'});
    final result = await db.query('messages', where: 'id = ?', whereArgs: ['msg_1']);

    expect(result.first['content'], 'Hello test');
    await db.close();
  });
}
```

---

## 8. Best Practices Checklist

1. **Always enable Foreign Keys**: Run `PRAGMA foreign_keys = ON` in `onConfigure`.
2. **Always keep one open database instance**: Never call `openDatabase` and `close` on every single query. Keep the `Database` instance open for the app lifecycle.
3. **Use `batch` for bulk writes**: Calling `insert` 100 times individually executes 100 disk I/O operations; wrapping in `db.batch()` executes them in a single transaction.
4. **Use indexed columns**: Add `CREATE INDEX` on foreign keys and columns frequently used in `WHERE` or `ORDER BY` clauses.
5. **Sanitize parameters with `whereArgs`**: Never concatenate raw strings in SQL queries (prevents SQL injection):
   ```dart
   // Good
   db.query('users', where: 'email = ?', whereArgs: [userEmail]);
   
   // Bad
   db.rawQuery("SELECT * FROM users WHERE email = '$userEmail'");
   ```
6. **Use `ConflictAlgorithm.replace`**: Simplifies syncing remote API data into local storage without throwing duplicate key errors.
