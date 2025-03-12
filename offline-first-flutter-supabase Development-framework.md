# Offline-First Flutter + Supabase Development Framework

## Overview
This framework provides a comprehensive architecture for building offline-first mobile applications using Flutter, Supabase, and Brick ORM. It enables applications to function seamlessly regardless of network connectivity, with reliable data synchronization between local storage and Supabase backend.

## Core Technologies
- **Frontend**: Flutter/Dart (latest version)
- **Backend Services**: Supabase (PostgreSQL, Auth, Storage, Functions)
- **Offline Data Management**: Brick ORM + SQLite
- **State Management**: Riverpod (code generation version)
- **Immutable Data**: freezed + fast_immutable_collections

## Why Offline-First?
- **Enhanced User Experience**: Applications remain fully functional even without network connectivity
- **Improved Performance**: Reduced latency by accessing local data first
- **Reduced Data Usage**: Only changes are synchronized, minimizing bandwidth consumption
- **Reliability**: Critical operations succeed regardless of network status

## Project Structure
```
app/
├── .vscode/                    # VS Code configuration
├── android/                    # Android platform code
├── ios/                        # iOS platform code
├── lib/
│   ├── app/                    # App core
│   │   ├── app.dart            # App entry point
│   │   ├── routes.dart         # Route definitions
│   │   └── theme.dart          # App theme
│   ├── brick/                  # Brick ORM configuration
│   │   ├── models/             # Model definitions
│   │   ├── adapters/           # Generated adapters (auto-generated)
│   │   ├── db/                 # Database migrations (auto-generated)
│   │   └── brick.g.dart        # Generated Brick code
│   ├── core/                   # Core components
│   │   ├── constants/          # Constant definitions
│   │   ├── errors/             # Error handling
│   │   ├── services/           # Core services
│   │   │   ├── supabase/       # Supabase service wrappers
│   │   │   └── connectivity/   # Network connectivity service
│   │   └── utils/              # Common utility functions
│   ├── features/               # Feature modules
│   │   ├── auth/               # Authentication feature
│   │   ├── home/               # Home page feature
│   │   ├── profile/            # User profile feature 
│   │   └── [other features]/
│   ├── shared/                 # Shared components
│   │   ├── widgets/            # Reusable UI components
│   │   └── extensions/         # Dart extension methods
│   └── main.dart               # Application entry point
├── assets/                     # Static assets
├── test/                       # Test folder
│   ├── unit/                   # Unit tests
│   ├── widget/                 # Widget tests
│   └── integration/            # Integration tests
├── supabase/                   # Supabase configuration
├── pubspec.yaml                # Dependency configuration
└── README.md                   # Project documentation
```

## Feature Module Structure
Each feature module follows a clean architecture approach with offline-first considerations:

```
features/users/                 # Users feature module
├── data/                       # Data layer
│   └── models/                 # Data models (for non-Brick models)
├── domain/                     # Domain layer
│   ├── entities/               # Business entities
│   ├── repositories/           # Repository interfaces
│   └── usecases/               # Use cases (business logic)
└── presentation/               # Presentation layer (UI)
    ├── providers/              # Riverpod providers
    ├── pages/                  # Pages
    └── widgets/                # Feature-specific components
```

## Core Dependencies

### Foundation
- `supabase_flutter`: Supabase client for Flutter
- `brick_offline_first_with_supabase`: Main Brick package for Supabase integration
- `brick_sqlite`: Brick ORM for SQLite persistence
- `sqflite`: SQLite database implementation
- `connectivity_plus`: Network connectivity detection
- `uuid`: Unique ID generation

### State Management & UI
- `flutter_riverpod`: Reactive state management
- `riverpod_generator`: Riverpod code generation
- `freezed`: Immutable data model generation
- `go_router`: Declarative routing
- `flutter_hooks`: UI state hooks

### Development Tools
- `build_runner`: Code generation tool
- `brick_offline_first_with_supabase_build`: Brick ORM code generation
- `flutter_lints`: Code style checker
- `mocktail`: Unit testing mock framework

## Setting Up Brick Models

### Creating a Model
```dart
// lib/brick/models/user.model.dart
import 'package:brick_offline_first_with_supabase/brick_offline_first_with_supabase.dart';
import 'package:brick_sqlite/brick_sqlite.dart';
import 'package:brick_supabase/brick_supabase.dart';
import 'package:uuid/uuid.dart';

@ConnectOfflineFirstWithSupabase(
  supabaseConfig: SupabaseSerializable(tableName: 'users'),
)
class User extends OfflineFirstWithSupabaseModel {
  final String name;
  final String? email;
  final bool isActive;

  // Using UUID for distributed ID generation
  @Supabase(unique: true)
  @Sqlite(index: true, unique: true)
  final String id;

  User({
    String? id,
    required this.name,
    this.email,
    this.isActive = true,
  }) : this.id = id ?? const Uuid().v4();
}
```

### Model with Associations
```dart
// lib/brick/models/task.model.dart
import 'package:brick_offline_first_with_supabase/brick_offline_first_with_supabase.dart';
import 'package:brick_sqlite/brick_sqlite.dart';
import 'package:brick_supabase/brick_supabase.dart';
import 'package:uuid/uuid.dart';
import 'user.model.dart';

@ConnectOfflineFirstWithSupabase(
  supabaseConfig: SupabaseSerializable(tableName: 'tasks'),
)
class Task extends OfflineFirstWithSupabaseModel {
  final String title;
  final String? description;
  final bool isCompleted;
  final DateTime createdAt;
  final User assignee;

  @Supabase(unique: true)
  @Sqlite(index: true, unique: true)
  final String id;

  Task({
    String? id,
    required this.title,
    this.description,
    this.isCompleted = false,
    required this.assignee,
    DateTime? createdAt,
  }) : 
    this.id = id ?? const Uuid().v4(),
    this.createdAt = createdAt ?? DateTime.now();
}
```

## Setting Up the Repository

```dart
// lib/brick/repository.dart
import 'package:brick_offline_first_with_supabase/brick_offline_first_with_supabase.dart';
import 'package:brick_sqlite/brick_sqlite.dart';
import 'package:brick_supabase/brick_supabase.dart' hide Supabase;
import 'package:sqflite_common/sqlite_api.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import 'brick.g.dart';

class AppRepository extends OfflineFirstWithSupabaseRepository {
  static AppRepository? _instance;

  AppRepository._({
    required super.supabaseProvider,
    required super.sqliteProvider,
    required super.migrations,
    required super.offlineRequestQueue,
    super.memoryCacheProvider,
  });

  factory AppRepository() => _instance!;

  static Future<void> configure(
    DatabaseFactory databaseFactory, {
    required String supabaseUrl,
    required String supabaseAnonKey,
  }) async {
    final (client, queue) = OfflineFirstWithSupabaseRepository.clientQueue(
      databaseFactory: databaseFactory,
    );

    await Supabase.initialize(
      url: supabaseUrl,
      anonKey: supabaseAnonKey,
      httpClient: client,
    );

    final provider = SupabaseProvider(
      Supabase.instance.client,
      modelDictionary: supabaseModelDictionary,
    );

    _instance = AppRepository._(
      supabaseProvider: provider,
      sqliteProvider: SqliteProvider(
        'app_repository.sqlite',
        databaseFactory: databaseFactory,
        modelDictionary: sqliteModelDictionary,
      ),
      migrations: migrations,
      offlineRequestQueue: queue,
      memoryCacheProvider: MemoryCacheProvider(),
    );
  }
}
```

## Application Initialization

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sqflite/sqflite.dart' show databaseFactory;
import 'package:supabase_flutter/supabase_flutter.dart';
import 'package:your_app/brick/repository.dart';
import 'package:your_app/app/app.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Repository with Brick + Supabase
  await AppRepository.configure(
    databaseFactory,
    supabaseUrl: 'YOUR_SUPABASE_URL',
    supabaseAnonKey: 'YOUR_SUPABASE_ANON_KEY',
  );
  
  // Initialize the repository (can also be done in a state manager)
  await AppRepository().initialize();
  
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
```

## Data Access with Riverpod (Code Gen)

```dart
// lib/features/tasks/presentation/providers/task_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:brick_offline_first_with_supabase/brick_offline_first_with_supabase.dart';
import 'package:your_app/brick/models/task.model.dart';
import 'package:your_app/brick/repository.dart';

// Generate the .g.dart file with: dart run build_runner build
part 'task_providers.g.dart';

// Provider for task stream (reactive)
@riverpod
Stream<List<Task>> tasks(TasksRef ref) {
  return AppRepository().subscribe<Task>();
}

// Provider for filtering tasks by assignee
@riverpod
Stream<List<Task>> filteredTasks(FilteredTasksRef ref, String assigneeId) {
  return AppRepository().subscribe<Task>(
    query: Query.where('assignee', Where.exact('id', assigneeId)),
  );
}

// Provider for task by ID
@riverpod
Future<Task?> task(TaskRef ref, String id) {
  return AppRepository().get<Task>(
    query: Query.where('id', id),
  ).then((tasks) => tasks.isNotEmpty ? tasks.first : null);
}

// Task operations provider
@riverpod
class TaskOperations extends _$TaskOperations {
  @override
  AsyncValue<void> build() {
    return const AsyncValue.data(null);
  }

  Future<void> addTask(Task task) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      await AppRepository().upsert<Task>(task);
      return null;
    });
  }

  Future<void> updateTask(Task task) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      await AppRepository().upsert<Task>(task);
      return null;
    });
  }

  Future<void> deleteTask(Task task) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      await AppRepository().delete<Task>(task);
      return null;
    });
  }

  Future<void> toggleTaskCompletion(Task task) async {
    final updatedTask = Task(
      id: task.id,
      title: task.title,
      description: task.description,
      isCompleted: !task.isCompleted,
      assignee: task.assignee,
      createdAt: task.createdAt,
    );
    await updateTask(updatedTask);
  }
}
```

## UI Implementation with Connectivity Status

```dart
// lib/features/tasks/presentation/pages/task_list_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:your_app/brick/models/task.model.dart';
import 'package:your_app/features/tasks/presentation/providers/task_providers.dart';

part 'task_list_page.g.dart';

// Connectivity provider
@riverpod
Stream<ConnectivityResult> connectivity(ConnectivityRef ref) {
  return Connectivity().onConnectivityChanged;
}

class TaskListPage extends ConsumerWidget {
  const TaskListPage({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tasksAsync = ref.watch(tasksProvider);
    final connectivity = ref.watch(connectivityProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Tasks'),
        actions: [
          // Connectivity indicator
          connectivity.when(
            data: (status) => Icon(
              status != ConnectivityResult.none 
                ? Icons.wifi 
                : Icons.wifi_off,
              color: status != ConnectivityResult.none 
                ? Colors.green 
                : Colors.red,
            ),
            loading: () => const SizedBox.shrink(),
            error: (_, __) => const Icon(Icons.error),
          ),
        ],
      ),
      body: tasksAsync.when(
        data: (tasks) {
          if (tasks.isEmpty) {
            return const Center(child: Text('No tasks yet'));
          }
          
          return ListView.builder(
            itemCount: tasks.length,
            itemBuilder: (context, index) {
              final task = tasks[index];
              return TaskListTile(task: task);
            },
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Navigate to create task screen
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}

class TaskListTile extends ConsumerWidget {
  final Task task;
  
  const TaskListTile({required this.task, Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ListTile(
      title: Text(task.title),
      subtitle: Text(task.description ?? ''),
      trailing: Checkbox(
        value: task.isCompleted,
        onChanged: (value) {
          if (value != null) {
            final updatedTask = Task(
              id: task.id,
              title: task.title,
              description: task.description,
              isCompleted: value,
              assignee: task.assignee,
              createdAt: task.createdAt,
            );
            
            ref.read(taskOperationsProvider.notifier).updateTask(updatedTask);
          }
        },
      ),
      onTap: () {
        // Navigate to task details
      },
    );
  }
}
```

## Testing with Brick and Supabase Mock

```dart
// test/unit/tasks_repository_test.dart
import 'package:brick_offline_first_with_supabase/brick_offline_first_with_supabase.dart';
import 'package:brick_supabase/testing.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sqflite_common_ffi/sqflite_ffi.dart';
import 'package:your_app/brick/brick.g.dart';
import 'package:your_app/brick/models/task.model.dart';
import 'package:your_app/brick/models/user.model.dart';

void main() {
  // Initialize SQLite for testing
  sqfliteFfiInit();
  
  // Set up mock Supabase server
  final mock = SupabaseMockServer(modelDictionary: supabaseModelDictionary);
  
  late OfflineFirstWithSupabaseRepository repository;
  late User testUser;
  
  setUp(() async {
    // Set up mock server
    await mock.setUp();
    
    // Create test repository
    final provider = SupabaseProvider(
      mock.client,
      modelDictionary: supabaseModelDictionary,
    );
    
    repository = OfflineFirstWithSupabaseRepository(
      supabaseProvider: provider,
      sqliteProvider: SqliteProvider(
        'test_db.sqlite',
        databaseFactory: databaseFactoryFfi,
        modelDictionary: sqliteModelDictionary,
      ),
      migrations: migrations,
      offlineRequestQueue: <OfflineFirstWithSupabaseRequest>[],
    );
    
    await repository.initialize();
    
    // Create test user
    testUser = User(name: 'Test User');
    await repository.upsert<User>(testUser);
  });
  
  tearDown(() async {
    await repository.close();
    await mock.tearDown();
  });
  
  group('Tasks Repository', () {
    test('should create and retrieve a task', () async {
      // Set up mock response
      final task = Task(
        title: 'Test Task',
        assignee: testUser,
      );
      
      final req = SupabaseRequest<Task>();
      final resp = SupabaseResponse([
        await mock.serialize(task),
      ]);
      
      mock.handle({req: resp});
      
      // Create task
      await repository.upsert<Task>(task);
      
      // Retrieve task
      final tasks = await repository.get<Task>();
      
      expect(tasks, isNotEmpty);
      expect(tasks.first.title, equals('Test Task'));
      expect(tasks.first.assignee.id, equals(testUser.id));
    });
    
    test('should update a task', () async {
      // Create initial task
      final task = Task(
        title: 'Initial Title',
        assignee: testUser,
      );
      
      final initialReq = SupabaseRequest<Task>();
      final initialResp = SupabaseResponse([
        await mock.serialize(task),
      ]);
      
      mock.handle({initialReq: initialResp});
      
      await repository.upsert<Task>(task);
      
      // Update task
      final updatedTask = Task(
        id: task.id,
        title: 'Updated Title',
        assignee: testUser,
        createdAt: task.createdAt,
      );
      
      final updateReq = SupabaseRequest<Task>(
        where: {'id': task.id},
      );
      final updateResp = SupabaseResponse([
        await mock.serialize(updatedTask),
      ]);
      
      mock.handle({updateReq: updateResp});
      
      await repository.upsert<Task>(updatedTask);
      
      // Retrieve updated task
      final tasks = await repository.get<Task>(
        query: Query.where('id', task.id),
      );
      
      expect(tasks, isNotEmpty);
      expect(tasks.first.title, equals('Updated Title'));
    });
  });
}
```

## Best Practices

### 1. Offline Data Management
- Use UUIDs for all primary keys to prevent conflicts
- Include `isSynced` flags in UI to indicate sync status
- Implement retry mechanisms for failed sync operations
- Consider data pruning strategies for local storage

### 2. Connectivity Handling
- Implement graceful degradation of features when offline
- Use Connectivity Plus package to monitor network status
- Display user-friendly indicators for sync/offline status
- Queue operations for later synchronization

### 3. User Experience
- Design UI to work well in both online and offline modes
- Provide feedback on sync status (pending, completed, failed)
- Handle conflicts with a clear resolution strategy
- Minimize UI freezes during sync operations

### 4. Data Modeling
- Keep models simple and focused
- Use annotations properly for Brick configuration
- Document field constraints and relationships
- Follow naming conventions consistent with Supabase tables

### 5. Testing
- Mock Supabase responses for unit tests
- Test offline functionality specifically
- Verify synchronization flows with integration tests
- Test edge cases like network interruptions during sync

## Development Workflow

1. **Setup Phase**:
   - Configure Supabase tables and relationships
   - Set up Brick models and generate code
   - Create repository and initialize in app

2. **Development Phase**:
   - Start with core models and repositories
   - Implement offline-first data access patterns
   - Build UI with sync status indicators
   - Add comprehensive error handling

3. **Testing Phase**:
   - Test offline operations extensively
   - Verify sync mechanisms with different connectivity patterns
   - Test edge cases like conflicts and partial connectivity
   - Performance test with large datasets

4. **Refinement Phase**:
   - Optimize query performance
   - Implement intelligent batch synchronization
   - Add retry mechanisms and conflict resolution
   - Fine-tune offline experience

## Conclusion

This offline-first architecture with Brick and Supabase provides a robust foundation for building applications that work seamlessly regardless of network connectivity. By leveraging Brick's ORM capabilities and Supabase's flexible backend, developers can focus on creating great user experiences without worrying about the complexities of data synchronization.
