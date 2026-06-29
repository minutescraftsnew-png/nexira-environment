# Nexira – Flutter Project Structure (MVP Version 1)

## Overview
Complete Flutter project structure for Nexira MVP (Light Version).
Supports Android, iOS, and Web platforms with a single codebase.

---

## 📁 Project Directory Structure

```
nexira-flutter/
├── android/                          # Android native files
├── ios/                              # iOS native files
├── web/                              # Web files
├── lib/
│   ├── main.dart                     # App entry point
│   ├── config/
│   │   ├── firebase_config.dart      # Firebase initialization
│   │   ├── theme.dart                # App theme & colors
│   │   └── constants.dart            # App constants
│   ├── models/
│   │   ├── user_model.dart           # User data model
│   │   ├── game_model.dart           # Game session model
│   │   ├── question_model.dart       # Question model
│   │   └── stats_model.dart          # User statistics model
│   ├── services/
│   │   ├── firebase_auth_service.dart    # Auth service
│   │   ├── firestore_service.dart       # Firestore operations
│   │   ├── cloud_functions_service.dart # Cloud Functions API
│   │   └── storage_service.dart         # Local storage
│   ├── providers/
│   │   ├── auth_provider.dart        # Auth state management
│   │   ├── game_provider.dart        # Game state management
│   │   ├── user_provider.dart        # User data management
│   │   └── difficulty_provider.dart  # Difficulty management
│   ├── screens/
│   │   ├── auth/
│   │   │   ├── splash_screen.dart
│   │   │   ├── sign_in_screen.dart
│   │   │   ├── sign_up_screen.dart
│   │   │   └── forgot_password_screen.dart
│   │   ├── home/
│   │   │   ├── home_screen.dart
│   │   │   ├── profile_screen.dart
│   │   │   └── stats_screen.dart
│   │   ├── quiz/
│   │   │   ├── quiz_screen.dart
│   │   │   ├── question_screen.dart
│   │   │   ├── result_screen.dart
│   │   │   └── game_over_screen.dart
│   │   └── history/
│   │       └── game_history_screen.dart
│   ├── widgets/
│   │   ├── common/
│   │   │   ├── custom_button.dart
│   │   │   ├── custom_text_field.dart
│   │   │   ├── loading_widget.dart
│   │   │   └── error_widget.dart
│   │   ├── quiz/
│   │   │   ├── question_card.dart
│   │   │   ├── options_widget.dart
│   │   │   ├── timer_widget.dart
│   │   │   ├── score_display.dart
│   │   │   └── difficulty_badge.dart
│   │   └── profile/
│   │       ├── profile_header.dart
│   │       ├── stats_card.dart
│   │       └── game_history_card.dart
│   ├── utils/
│   │   ├── validators.dart           # Form validators
│   │   ├── helpers.dart              # Helper functions
│   │   ├── formatters.dart           # Data formatters
│   │   └── logger.dart               # Logging utility
│   └── navigation/
│       ├── app_router.dart           # Route management
│       └── route_paths.dart          # Route constants
├── pubspec.yaml                      # Dependencies
├── pubspec.lock                      # Lock file
├── .env.example                      # Environment variables example
├── .gitignore
├── README.md
└── analysis_options.yaml             # Lint rules
```

---

## 📄 Core Files

### File: `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:provider/provider.dart';
import 'package:nexira/config/firebase_config.dart';
import 'package:nexira/config/theme.dart';
import 'package:nexira/navigation/app_router.dart';
import 'package:nexira/providers/auth_provider.dart';
import 'package:nexira/providers/game_provider.dart';
import 'package:nexira/providers/user_provider.dart';
import 'package:nexira/providers/difficulty_provider.dart';
import 'package:nexira/screens/auth/splash_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  runApp(const NexiraApp());
}

class NexiraApp extends StatelessWidget {
  const NexiraApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AuthProvider()),
        ChangeNotifierProvider(create: (_) => UserProvider()),
        ChangeNotifierProvider(create: (_) => GameProvider()),
        ChangeNotifierProvider(create: (_) => DifficultyProvider()),
      ],
      child: MaterialApp(
        title: 'Nexira',
        theme: AppTheme.lightTheme,
        darkTheme: AppTheme.darkTheme,
        themeMode: ThemeMode.light,
        home: const SplashScreen(),
        routes: AppRouter.routes,
        onGenerateRoute: AppRouter.onGenerateRoute,
      ),
    );
  }
}
```

---

### File: `lib/config/theme.dart`

```dart
import 'package:flutter/material.dart';

class AppTheme {
  // Colors
  static const Color primaryColor = Color(0xFF6366F1);      // Indigo
  static const Color secondaryColor = Color(0xFF10B981);    // Emerald
  static const Color accentColor = Color(0xFFF59E0B);       // Amber
  static const Color errorColor = Color(0xFFEF4444);        // Red
  static const Color successColor = Color(0xFF10B981);      // Green
  static const Color warningColor = Color(0xFFF59E0B);      // Amber

  static const Color backgroundColor = Color(0xFFF9FAFB);   // Light gray
  static const Color surfaceColor = Color(0xFFFFFFFF);      // White
  static const Color textColor = Color(0xFF1F2937);         // Dark gray
  static const Color subtextColor = Color(0xFF6B7280);      // Medium gray

  // Difficulty Colors
  static const Color easyColor = Color(0xFF10B981);         // Green
  static const Color mediumColor = Color(0xFFF59E0B);       // Amber
  static const Color hardColor = Color(0xFFEF4444);         // Red

  // Light Theme
  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: primaryColor,
        brightness: Brightness.light,
      ),
      scaffoldBackgroundColor: backgroundColor,
      appBarTheme: AppBarTheme(
        backgroundColor: surfaceColor,
        elevation: 0,
        centerTitle: true,
        titleTextStyle: const TextStyle(
          color: textColor,
          fontSize: 20,
          fontWeight: FontWeight.bold,
        ),
        iconTheme: const IconThemeData(color: textColor),
      ),
      textTheme: const TextTheme(
        displayLarge: TextStyle(
          fontSize: 32,
          fontWeight: FontWeight.bold,
          color: textColor,
        ),
        displayMedium: TextStyle(
          fontSize: 28,
          fontWeight: FontWeight.bold,
          color: textColor,
        ),
        titleLarge: TextStyle(
          fontSize: 20,
          fontWeight: FontWeight.w600,
          color: textColor,
        ),
        bodyLarge: TextStyle(
          fontSize: 16,
          fontWeight: FontWeight.w500,
          color: textColor,
        ),
        bodyMedium: TextStyle(
          fontSize: 14,
          fontWeight: FontWeight.w400,
          color: subtextColor,
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: backgroundColor,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
          borderSide: const BorderSide(color: Color(0xFFE5E7EB)),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
          borderSide: const BorderSide(color: Color(0xFFE5E7EB)),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
          borderSide: const BorderSide(color: primaryColor, width: 2),
        ),
        hintStyle: const TextStyle(color: subtextColor),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: primaryColor,
          foregroundColor: Colors.white,
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8),
          ),
          elevation: 2,
        ),
      ),
      textButtonTheme: TextButtonThemeData(
        style: TextButton.styleFrom(
          foregroundColor: primaryColor,
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        ),
      ),
      outlinedButtonTheme: OutlinedButtonThemeData(
        style: OutlinedButton.styleFrom(
          foregroundColor: primaryColor,
          side: const BorderSide(color: primaryColor),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8),
          ),
        ),
      ),
    );
  }

  // Dark Theme
  static ThemeData get darkTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: primaryColor,
        brightness: Brightness.dark,
      ),
      scaffoldBackgroundColor: const Color(0xFF111827),
      appBarTheme: const AppBarTheme(
        backgroundColor: Color(0xFF1F2937),
        elevation: 0,
        centerTitle: true,
        titleTextStyle: TextStyle(
          color: Colors.white,
          fontSize: 20,
          fontWeight: FontWeight.bold,
        ),
      ),
    );
  }
}
```

---

### File: `lib/models/question_model.dart`

```dart
class Question {
  final String questionId;
  final String question;
  final String image;
  final List<String> options;
  final String correctAnswer;
  final int correctAnswerIndex;
  final String explanation;
  final String topic;
  final String difficulty;
  final DateTime generatedAt;

  Question({
    required this.questionId,
    required this.question,
    required this.image,
    required this.options,
    required this.correctAnswer,
    required this.correctAnswerIndex,
    required this.explanation,
    required this.topic,
    required this.difficulty,
    required this.generatedAt,
  });

  factory Question.fromJson(Map<String, dynamic> json) {
    return Question(
      questionId: json['questionId'] as String,
      question: json['question'] as String,
      image: json['image'] as String,
      options: List<String>.from(json['options'] as List),
      correctAnswer: json['correctAnswer'] as String,
      correctAnswerIndex: json['correctAnswerIndex'] as int,
      explanation: json['explanation'] as String,
      topic: json['topic'] as String,
      difficulty: json['difficulty'] as String,
      generatedAt: DateTime.parse(json['generatedAt'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'questionId': questionId,
      'question': question,
      'image': image,
      'options': options,
      'correctAnswer': correctAnswer,
      'correctAnswerIndex': correctAnswerIndex,
      'explanation': explanation,
      'topic': topic,
      'difficulty': difficulty,
      'generatedAt': generatedAt.toIso8601String(),
    };
  }
}

class QuestionAnswer {
  final Question question;
  final String? userAnswer;
  final int? userAnswerIndex;
  final bool isCorrect;
  final double timeTaken;
  final int finalScore;

  QuestionAnswer({
    required this.question,
    this.userAnswer,
    this.userAnswerIndex,
    required this.isCorrect,
    required this.timeTaken,
    required this.finalScore,
  });

  factory QuestionAnswer.fromJson(Map<String, dynamic> json) {
    return QuestionAnswer(
      question: Question.fromJson(json['question'] as Map<String, dynamic>),
      userAnswer: json['userAnswer'] as String?,
      userAnswerIndex: json['userAnswerIndex'] as int?,
      isCorrect: json['isCorrect'] as bool,
      timeTaken: (json['timeTaken'] as num).toDouble(),
      finalScore: json['finalScore'] as int,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'question': question.toJson(),
      'userAnswer': userAnswer,
      'userAnswerIndex': userAnswerIndex,
      'isCorrect': isCorrect,
      'timeTaken': timeTaken,
      'finalScore': finalScore,
    };
  }
}
```

---

### File: `lib/models/user_model.dart`

```dart
class User {
  final String uid;
  final String email;
  final String displayName;
  final String? avatar;
  final int totalScore;
  final int gamesPlayed;
  final String currentDifficulty;
  final int totalTimeSpent;
  final bool isActive;
  final DateTime createdAt;
  final DateTime lastPlayedAt;

  User({
    required this.uid,
    required this.email,
    required this.displayName,
    this.avatar,
    required this.totalScore,
    required this.gamesPlayed,
    required this.currentDifficulty,
    required this.totalTimeSpent,
    required this.isActive,
    required this.createdAt,
    required this.lastPlayedAt,
  });

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      uid: json['uid'] as String,
      email: json['email'] as String,
      displayName: json['displayName'] as String,
      avatar: json['avatar'] as String?,
      totalScore: json['totalScore'] as int,
      gamesPlayed: json['gamesPlayed'] as int,
      currentDifficulty: json['currentDifficulty'] as String? ?? 'Easy',
      totalTimeSpent: json['totalTimeSpent'] as int? ?? 0,
      isActive: json['isActive'] as bool? ?? true,
      createdAt: DateTime.parse(json['createdAt'] as String),
      lastPlayedAt: DateTime.parse(json['lastPlayedAt'] as String? ?? DateTime.now().toIso8601String()),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'uid': uid,
      'email': email,
      'displayName': displayName,
      'avatar': avatar,
      'totalScore': totalScore,
      'gamesPlayed': gamesPlayed,
      'currentDifficulty': currentDifficulty,
      'totalTimeSpent': totalTimeSpent,
      'isActive': isActive,
      'createdAt': createdAt.toIso8601String(),
      'lastPlayedAt': lastPlayedAt.toIso8601String(),
    };
  }

  User copyWith({
    String? uid,
    String? email,
    String? displayName,
    String? avatar,
    int? totalScore,
    int? gamesPlayed,
    String? currentDifficulty,
    int? totalTimeSpent,
    bool? isActive,
    DateTime? createdAt,
    DateTime? lastPlayedAt,
  }) {
    return User(
      uid: uid ?? this.uid,
      email: email ?? this.email,
      displayName: displayName ?? this.displayName,
      avatar: avatar ?? this.avatar,
      totalScore: totalScore ?? this.totalScore,
      gamesPlayed: gamesPlayed ?? this.gamesPlayed,
      currentDifficulty: currentDifficulty ?? this.currentDifficulty,
      totalTimeSpent: totalTimeSpent ?? this.totalTimeSpent,
      isActive: isActive ?? this.isActive,
      createdAt: createdAt ?? this.createdAt,
      lastPlayedAt: lastPlayedAt ?? this.lastPlayedAt,
    );
  }
}
```

---

### File: `lib/models/stats_model.dart`

```dart
class UserStats {
  final String userId;
  final int totalScore;
  final int gamesPlayed;
  final int totalCorrectAnswers;
  final int totalWrongAnswers;
  final double accuracyPercentage;
  final double averageScorePerGame;
  final List<String> favoriteTopics;
  final Map<String, DifficultyStats> difficultyStats;
  final DateTime lastUpdated;

  UserStats({
    required this.userId,
    required this.totalScore,
    required this.gamesPlayed,
    required this.totalCorrectAnswers,
    required this.totalWrongAnswers,
    required this.accuracyPercentage,
    required this.averageScorePerGame,
    required this.favoriteTopics,
    required this.difficultyStats,
    required this.lastUpdated,
  });

  factory UserStats.fromJson(Map<String, dynamic> json) {
    return UserStats(
      userId: json['userId'] as String,
      totalScore: json['totalScore'] as int,
      gamesPlayed: json['gamesPlayed'] as int,
      totalCorrectAnswers: json['totalCorrectAnswers'] as int,
      totalWrongAnswers: json['totalWrongAnswers'] as int,
      accuracyPercentage: (json['accuracyPercentage'] as num).toDouble(),
      averageScorePerGame: (json['averageScorePerGame'] as num).toDouble(),
      favoriteTopics: List<String>.from(json['favoriteTopics'] as List? ?? []),
      difficultyStats: (json['difficultyStats'] as Map<String, dynamic>? ?? {}).map(
        (key, value) => MapEntry(
          key,
          DifficultyStats.fromJson(value as Map<String, dynamic>),
        ),
      ),
      lastUpdated: DateTime.parse(json['lastUpdated'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'userId': userId,
      'totalScore': totalScore,
      'gamesPlayed': gamesPlayed,
      'totalCorrectAnswers': totalCorrectAnswers,
      'totalWrongAnswers': totalWrongAnswers,
      'accuracyPercentage': accuracyPercentage,
      'averageScorePerGame': averageScorePerGame,
      'favoriteTopics': favoriteTopics,
      'difficultyStats': difficultyStats.map((key, value) => MapEntry(key, value.toJson())),
      'lastUpdated': lastUpdated.toIso8601String(),
    };
  }
}

class DifficultyStats {
  final int attempted;
  final int correct;
  final double accuracy;

  DifficultyStats({
    required this.attempted,
    required this.correct,
    required this.accuracy,
  });

  factory DifficultyStats.fromJson(Map<String, dynamic> json) {
    return DifficultyStats(
      attempted: json['attempted'] as int? ?? 0,
      correct: json['correct'] as int? ?? 0,
      accuracy: (json['accuracy'] as num?)?.toDouble() ?? 0.0,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'attempted': attempted,
      'correct': correct,
      'accuracy': accuracy,
    };
  }
}
```

---

### File: `lib/models/game_model.dart`

```dart
import 'package:nexira/models/question_model.dart';

class GameSession {
  final String gameId;
  final String userId;
  final DateTime startedAt;
  final DateTime? completedAt;
  final int totalDuration;
  final int totalScore;
  final int questionsAttempted;
  final int correctAnswers;
  final int wrongAnswers;
  final double accuracy;
  final List<QuestionAnswer> questions;
  final String status;

  GameSession({
    required this.gameId,
    required this.userId,
    required this.startedAt,
    this.completedAt,
    required this.totalDuration,
    required this.totalScore,
    required this.questionsAttempted,
    required this.correctAnswers,
    required this.wrongAnswers,
    required this.accuracy,
    required this.questions,
    required this.status,
  });

  factory GameSession.fromJson(Map<String, dynamic> json) {
    return GameSession(
      gameId: json['gameId'] as String,
      userId: json['userId'] as String,
      startedAt: DateTime.parse(json['startedAt'] as String),
      completedAt: json['completedAt'] != null ? DateTime.parse(json['completedAt'] as String) : null,
      totalDuration: json['totalDuration'] as int,
      totalScore: json['totalScore'] as int,
      questionsAttempted: json['questionsAttempted'] as int,
      correctAnswers: json['correctAnswers'] as int,
      wrongAnswers: json['wrongAnswers'] as int,
      accuracy: (json['accuracy'] as num).toDouble(),
      questions: (json['questions'] as List? ?? [])
          .map((q) => QuestionAnswer.fromJson(q as Map<String, dynamic>))
          .toList(),
      status: json['status'] as String? ?? 'completed',
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'gameId': gameId,
      'userId': userId,
      'startedAt': startedAt.toIso8601String(),
      'completedAt': completedAt?.toIso8601String(),
      'totalDuration': totalDuration,
      'totalScore': totalScore,
      'questionsAttempted': questionsAttempted,
      'correctAnswers': correctAnswers,
      'wrongAnswers': wrongAnswers,
      'accuracy': accuracy,
      'questions': questions.map((q) => q.toJson()).toList(),
      'status': status,
    };
  }
}
```

---

### File: `lib/config/constants.dart`

```dart
class AppConstants {
  // App Info
  static const String appName = 'Nexira';
  static const String appVersion = '1.0.0';
  static const String appDescription = 'Competitive Quiz Game with Hidden Learning';

  // Game Constants
  static const int questionTimerSeconds = 10;
  static const int speedBonusThreshold = 3;
  static const int baseScoreCorrect = 10;
  static const int basePenaltyWrong = 5;
  static const int speedBonusPoints = 5;

  // Difficulty Multipliers
  static const Map<String, double> difficultyMultipliers = {
    'Easy': 1.0,
    'Medium': 1.5,
    'Hard': 2.0,
  };

  // Difficulty Distribution
  static const Map<String, int> difficultyDistribution = {
    'Easy': 40,
    'Medium': 40,
    'Hard': 20,
  };

  // Topics
  static const List<String> topics = [
    'Math',
    'Science',
    'History',
    'English',
    'General',
    'IQ',
    'Digital',
  ];

  // API Endpoints
  static const String cloudFunctionsBaseUrl = 'https://us-central1-project-id.cloudfunctions.net';
  static const String generateQuestionEndpoint = '/generateQuestion';
  static const String saveGameResultEndpoint = '/saveGameResult';
  static const String calculateDifficultyEndpoint = '/calculateDifficulty';
  static const String getUserProfileEndpoint = '/getUserProfile';
  static const String getUserStatsEndpoint = '/getUserStats';
  static const String getUserGameHistoryEndpoint = '/getUserGameHistory';

  // Storage Keys
  static const String keyAccessToken = 'access_token';
  static const String keyRefreshToken = 'refresh_token';
  static const String keyUserData = 'user_data';
  static const String keyGameHistory = 'game_history';

  // Network
  static const int connectTimeoutMs = 30000;
  static const int receiveTimeoutMs = 30000;

  // Pagination
  static const int defaultPageSize = 10;
  static const int maxGameHistoryItems = 100;
}
```

---

### File: `pubspec.yaml`

```yaml
name: nexira
description: "Nexira - Competitive Quiz Game with Hidden Learning"
publish_to: 'none'

version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  # Firebase
  firebase_core: ^2.24.0
  firebase_auth: ^4.14.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0

  # State Management
  provider: ^6.1.0

  # HTTP & Networking
  http: ^1.1.0
  dio: ^5.3.0

  # Local Storage
  shared_preferences: ^2.2.2
  hive: ^2.2.3
  hive_flutter: ^1.1.0

  # Image & Media
  cached_network_image: ^3.3.0
  image_picker: ^1.0.4
  flutter_image_compress: ^1.1.0

  # UI & UX
  flutter_svg: ^2.0.7
  intl: ^0.19.0
  animated_flip_counter: ^1.1.0
  lottie: ^2.6.0

  # Validation
  form_validator: ^1.1.0

  # Logging & Debugging
  logger: ^2.1.0
  firebase_crashlytics: ^3.4.0

  # Utilities
  uuid: ^4.1.0
  get_it: ^7.6.0
  equatable: ^2.0.5

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  build_runner: ^2.4.6
  hive_generator: ^2.0.1

flutter:
  uses-material-design: true
  
  assets:
    - assets/images/
    - assets/icons/
    - assets/animations/
    - assets/
  
  fonts:
    - family: Poppins
      fonts:
        - asset: assets/fonts/Poppins-Regular.ttf
        - asset: assets/fonts/Poppins-Bold.ttf
          weight: 700
        - asset: assets/fonts/Poppins-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Poppins-Medium.ttf
          weight: 500
```

---

### File: `lib/navigation/app_router.dart`

```dart
import 'package:flutter/material.dart';
import 'package:nexira/screens/auth/sign_in_screen.dart';
import 'package:nexira/screens/auth/sign_up_screen.dart';
import 'package:nexira/screens/home/home_screen.dart';
import 'package:nexira/screens/home/profile_screen.dart';
import 'package:nexira/screens/quiz/quiz_screen.dart';
import 'package:nexira/screens/history/game_history_screen.dart';

class AppRouter {
  static const String splash = '/splash';
  static const String signIn = '/sign-in';
  static const String signUp = '/sign-up';
  static const String home = '/home';
  static const String quiz = '/quiz';
  static const String profile = '/profile';
  static const String history = '/history';

  static Map<String, WidgetBuilder> routes = {
    signIn: (context) => const SignInScreen(),
    signUp: (context) => const SignUpScreen(),
    home: (context) => const HomeScreen(),
    quiz: (context) => const QuizScreen(),
    profile: (context) => const ProfileScreen(),
    history: (context) => const GameHistoryScreen(),
  };

  static Route<dynamic> onGenerateRoute(RouteSettings settings) {
    switch (settings.name) {
      case signIn:
        return MaterialPageRoute(builder: (_) => const SignInScreen());
      case signUp:
        return MaterialPageRoute(builder: (_) => const SignUpScreen());
      case home:
        return MaterialPageRoute(builder: (_) => const HomeScreen());
      case quiz:
        return MaterialPageRoute(builder: (_) => const QuizScreen());
      case profile:
        return MaterialPageRoute(builder: (_) => const ProfileScreen());
      case history:
        return MaterialPageRoute(builder: (_) => const GameHistoryScreen());
      default:
        return MaterialPageRoute(
          builder: (_) => Scaffold(
            body: Center(
              child: Text('No route defined for ${settings.name}'),
            ),
          ),
        );
    }
  }
}
```

---

## 📋 Services Layer

### File: `lib/services/firebase_auth_service.dart`

```dart
import 'package:firebase_auth/firebase_auth.dart';

class FirebaseAuthService {
  final FirebaseAuth _firebaseAuth = FirebaseAuth.instance;

  Stream<User?> get authStateChanges => _firebaseAuth.authStateChanges();

  User? get currentUser => _firebaseAuth.currentUser;

  Future<UserCredential> signUp({
    required String email,
    required String password,
  }) async {
    return await _firebaseAuth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<UserCredential> signIn({
    required String email,
    required String password,
  }) async {
    return await _firebaseAuth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<void> signOut() async {
    await _firebaseAuth.signOut();
  }

  Future<void> updateUserProfile({
    String? displayName,
    String? photoURL,
  }) async {
    await _firebaseAuth.currentUser?.updateDisplayName(displayName);
    await _firebaseAuth.currentUser?.updatePhotoURL(photoURL);
  }

  Future<void> resetPassword({required String email}) async {
    await _firebaseAuth.sendPasswordResetEmail(email: email);
  }
}
```

---

### File: `lib/services/cloud_functions_service.dart`

```dart
import 'package:http/http.dart' as http;
import 'dart:convert';
import 'package:nexira/config/constants.dart';
import 'package:nexira/models/question_model.dart';

class CloudFunctionsService {
  final String _baseUrl = AppConstants.cloudFunctionsBaseUrl;

  Future<Question> generateQuestion({
    required String userId,
    required String difficulty,
    required String topic,
  }) async {
    try {
      final response = await http.post(
        Uri.parse('$_baseUrl${AppConstants.generateQuestionEndpoint}'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'userId': userId,
          'difficulty': difficulty,
          'topic': topic,
        }),
      ).timeout(
        const Duration(milliseconds: AppConstants.connectTimeoutMs),
        onTimeout: () => throw Exception('Request timeout'),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        return Question.fromJson(data['data']);
      } else {
        throw Exception('Failed to generate question');
      }
    } catch (e) {
      rethrow;
    }
  }

  Future<Map<String, dynamic>> saveGameResult({
    required String userId,
    required String gameId,
    required String startedAt,
    required String completedAt,
    required List<Map<String, dynamic>> questions,
  }) async {
    try {
      final response = await http.post(
        Uri.parse('$_baseUrl${AppConstants.saveGameResultEndpoint}'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'userId': userId,
          'gameId': gameId,
          'startedAt': startedAt,
          'completedAt': completedAt,
          'questions': questions,
        }),
      ).timeout(
        const Duration(milliseconds: AppConstants.connectTimeoutMs),
        onTimeout: () => throw Exception('Request timeout'),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        return data['data'];
      } else {
        throw Exception('Failed to save game result');
      }
    } catch (e) {
      rethrow;
    }
  }

  Future<String> calculateDifficulty({required String userId}) async {
    try {
      final response = await http.post(
        Uri.parse('$_baseUrl${AppConstants.calculateDifficultyEndpoint}'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({'userId': userId}),
      ).timeout(
        const Duration(milliseconds: AppConstants.connectTimeoutMs),
        onTimeout: () => throw Exception('Request timeout'),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        return data['data']['currentDifficulty'];
      } else {
        return 'Easy'; // Default difficulty
      }
    } catch (e) {
      return 'Easy'; // Default difficulty on error
    }
  }
}
```

---

## 🔄 State Management (Providers)

### File: `lib/providers/auth_provider.dart`

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:nexira/services/firebase_auth_service.dart';
import 'package:nexira/services/firestore_service.dart';

class AuthProvider extends ChangeNotifier {
  final FirebaseAuthService _authService = FirebaseAuthService();
  final FirestoreService _firestoreService = FirestoreService();

  User? _currentUser;
  bool _isLoading = false;
  String? _errorMessage;

  User? get currentUser => _currentUser;
  bool get isLoading => _isLoading;
  String? get errorMessage => _errorMessage;
  bool get isAuthenticated => _currentUser != null;

  AuthProvider() {
    _initializeAuth();
  }

  void _initializeAuth() {
    _authService.authStateChanges.listen((user) {
      _currentUser = user;
      notifyListeners();
    });
  }

  Future<bool> signUp({
    required String email,
    required String password,
    required String displayName,
  }) async {
    try {
      _isLoading = true;
      _errorMessage = null;
      notifyListeners();

      final userCredential = await _authService.signUp(
        email: email,
        password: password,
      );

      // Update profile
      await _authService.updateUserProfile(displayName: displayName);

      // Create user document in Firestore
      await _firestoreService.createUserProfile(
        uid: userCredential.user!.uid,
        email: email,
        displayName: displayName,
      );

      _isLoading = false;
      notifyListeners();
      return true;
    } catch (e) {
      _errorMessage = e.toString();
      _isLoading = false;
      notifyListeners();
      return false;
    }
  }

  Future<bool> signIn({
    required String email,
    required String password,
  }) async {
    try {
      _isLoading = true;
      _errorMessage = null;
      notifyListeners();

      await _authService.signIn(
        email: email,
        password: password,
      );

      _isLoading = false;
      notifyListeners();
      return true;
    } catch (e) {
      _errorMessage = e.toString();
      _isLoading = false;
      notifyListeners();
      return false;
    }
  }

  Future<void> signOut() async {
    try {
      await _authService.signOut();
      _currentUser = null;
      notifyListeners();
    } catch (e) {
      _errorMessage = e.toString();
      notifyListeners();
    }
  }
}
```

---

## 📝 Notes

1. **State Management:** Using Provider for simplicity in MVP
2. **Firebase Integration:** All services integrated
3. **Error Handling:** Comprehensive error handling throughout
4. **Scalability:** Structure ready for future features (Version 2+)
5. **Platform Support:** Supports Android, iOS, and Web

---

## 🚀 Next Steps

1. ✅ Firestore Schema (DONE)
2. ✅ Cloud Functions (DONE)
3. ✅ Flutter Project Structure (THIS FILE)
4. ⏳ Screen Implementations (Quiz, Home, Profile, etc.)
5. ⏳ Widget Components (Timer, Score, Options, etc.)
6. ⏳ Testing & Optimization

---

## 🔧 Setup Instructions

```bash
# Clone repository
git clone https://github.com/minutescraftsnew-png/nexira-flutter.git
cd nexira-flutter

# Install dependencies
flutter pub get

# Run app (choose platform)
flutter run -d android
flutter run -d ios
flutter run -d chrome
```

---

## ⚠️ Important Notes for MVP

✔️ **Included:**
- Auth system (Sign Up/Sign In)
- Solo Quiz Mode
- Question generation (AI)
- Scoring system
- Dynamic difficulty
- User profile
- Game history
- Simple UI

❌ **NOT Included (Version 2+):**
- 1vs1 multiplayer
- Real-time sync
- Matchmaking
- Global leaderboards
- Advanced analytics
- Social features
