# Nexira – Quiz Game Screen & Components (MVP Version 1)

## Overview
Complete implementation of the Quiz Game Screen - the core of Nexira MVP.
Includes game loop, timer, scoring, and answer validation.

---

## 📱 Quiz Screen: Main Game Interface

### File: `lib/screens/quiz/quiz_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:uuid/uuid.dart';
import 'package:nexira/providers/game_provider.dart';
import 'package:nexira/providers/difficulty_provider.dart';
import 'package:nexira/providers/auth_provider.dart';
import 'package:nexira/providers/user_provider.dart';
import 'package:nexira/models/question_model.dart';
import 'package:nexira/services/cloud_functions_service.dart';
import 'package:nexira/config/theme.dart';
import 'package:nexira/config/constants.dart';
import 'package:nexira/navigation/app_router.dart';
import 'package:nexira/widgets/quiz/timer_widget.dart';
import 'package:nexira/widgets/quiz/question_card.dart';
import 'package:nexira/widgets/quiz/options_widget.dart';
import 'package:nexira/widgets/quiz/score_display.dart';

class QuizScreen extends StatefulWidget {
  const QuizScreen({Key? key}) : super(key: key);

  @override
  State<QuizScreen> createState() => _QuizScreenState();
}

class _QuizScreenState extends State<QuizScreen> {
  late String _gameId;
  late DateTime _gameStartTime;
  int _currentQuestionIndex = 0;
  int _totalScore = 0;
  int _correctAnswers = 0;
  int _wrongAnswers = 0;
  Question? _currentQuestion;
  String? _selectedAnswer;
  bool _answered = false;
  bool _isLoading = true;
  String? _loadingError;
  final CloudFunctionsService _functionsService = CloudFunctionsService();
  late String _currentDifficulty;
  final List<QuestionAnswer> _answeredQuestions = [];

  @override
  void initState() {
    super.initState();
    _initializeGame();
  }

  Future<void> _initializeGame() async {
    try {
      _gameId = const Uuid().v4();
      _gameStartTime = DateTime.now();
      
      final authProvider = Provider.of<AuthProvider>(context, listen: false);
      final difficultyProvider = Provider.of<DifficultyProvider>(context, listen: false);
      
      // Get current difficulty for user
      _currentDifficulty = await _functionsService.calculateDifficulty(
        userId: authProvider.currentUser!.uid,
      );
      
      difficultyProvider.setCurrentDifficulty(_currentDifficulty);
      
      // Load first question
      await _loadNextQuestion();
    } catch (e) {
      setState(() {
        _loadingError = 'Failed to load game: ${e.toString()}';
        _isLoading = false;
      });
    }
  }

  Future<void> _loadNextQuestion() async {
    try {
      setState(() => _isLoading = true);
      
      final authProvider = Provider.of<AuthProvider>(context, listen: false);
      
      // Randomly select topic from the list
      final randomTopic = AppConstants.topics[
        DateTime.now().microsecond % AppConstants.topics.length
      ];
      
      final question = await _functionsService.generateQuestion(
        userId: authProvider.currentUser!.uid,
        difficulty: _currentDifficulty,
        topic: randomTopic,
      );
      
      setState(() {
        _currentQuestion = question;
        _selectedAnswer = null;
        _answered = false;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _loadingError = 'Failed to load question: ${e.toString()}';
        _isLoading = false;
      });
      
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Error: ${e.toString()}'),
          backgroundColor: AppTheme.errorColor,
        ),
      );
    }
  }

  void _onAnswerSelected(String answer) {
    if (_answered) return;
    
    setState(() {
      _selectedAnswer = answer;
      _answered = true;
    });
    
    // Delay to show selection before showing result
    Future.delayed(const Duration(milliseconds: 500), () {
      _processAnswer(answer);
    });
  }

  void _processAnswer(String answer) {
    if (!mounted) return;
    
    final isCorrect = answer == _currentQuestion!.correctAnswer;
    final timeTaken = DateTime.now().difference(_gameStartTime).inSeconds.toDouble();
    
    // Calculate score
    int score = 0;
    if (isCorrect) {
      score = AppConstants.baseScoreCorrect;
      _correctAnswers++;
    } else {
      score = -AppConstants.basePenaltyWrong;
      _wrongAnswers++;
    }
    
    // Speed bonus
    if (timeTaken < AppConstants.speedBonusThreshold) {
      score += AppConstants.speedBonusPoints;
    }
    
    // Difficulty multiplier
    final multiplier = AppConstants.difficultyMultipliers[_currentDifficulty] ?? 1.0;
    score = (score * multiplier).toInt();
    
    // Store answered question
    _answeredQuestions.add(
      QuestionAnswer(
        question: _currentQuestion!,
        userAnswer: answer,
        userAnswerIndex: _currentQuestion!.options.indexOf(answer),
        isCorrect: isCorrect,
        timeTaken: timeTaken,
        finalScore: score,
      ),
    );
    
    _totalScore += score;
    
    // Show answer feedback
    _showAnswerFeedback(isCorrect, score);
  }

  void _showAnswerFeedback(bool isCorrect, int score) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => AlertDialog(
        backgroundColor: isCorrect ? AppTheme.successColor : AppTheme.errorColor,
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              isCorrect ? Icons.check_circle : Icons.cancel,
              color: Colors.white,
              size: 48,
            ),
            const SizedBox(height: 16),
            Text(
              isCorrect ? 'Correct!' : 'Wrong!',
              style: const TextStyle(
                fontSize: 24,
                fontWeight: FontWeight.bold,
                color: Colors.white,
              ),
            ),
            const SizedBox(height: 8),
            Text(
              'Answer: ${_currentQuestion!.correctAnswer}',
              style: const TextStyle(
                fontSize: 16,
                color: Colors.white70,
              ),
            ),
            const SizedBox(height: 16),
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.white.withOpacity(0.2),
                borderRadius: BorderRadius.circular(8),
              ),
              child: Text(
                _currentQuestion!.explanation,
                style: const TextStyle(
                  fontSize: 14,
                  color: Colors.white,
                ),
                textAlign: TextAlign.center,
              ),
            ),
            const SizedBox(height: 16),
            Container(
              padding: const EdgeInsets.all(8),
              decoration: BoxDecoration(
                color: Colors.white.withOpacity(0.3),
                borderRadius: BorderRadius.circular(8),
              ),
              child: Text(
                '+$score points',
                style: const TextStyle(
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                  color: Colors.white,
                ),
              ),
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              _currentQuestionIndex++;
              
              if (_currentQuestionIndex >= 5) {
                // End game after 5 questions
                _endGame();
              } else {
                // Load next question
                _loadNextQuestion();
              }
            },
            child: const Text(
              'Next',
              style: TextStyle(color: Colors.white),
            ),
          ),
        ],
      ),
    );
  }

  Future<void> _endGame() async {
    try {
      final authProvider = Provider.of<AuthProvider>(context, listen: false);
      final userProvider = Provider.of<UserProvider>(context, listen: false);
      
      final gameEndTime = DateTime.now();
      final totalDuration = gameEndTime.difference(_gameStartTime).inSeconds;
      
      // Save game result to backend
      await _functionsService.saveGameResult(
        userId: authProvider.currentUser!.uid,
        gameId: _gameId,
        startedAt: _gameStartTime.toIso8601String(),
        completedAt: gameEndTime.toIso8601String(),
        questions: _answeredQuestions.map((q) => {
          'questionId': q.question.questionId,
          'question': q.question.question,
          'image': q.question.image,
          'topic': q.question.topic,
          'difficulty': q.question.difficulty,
          'options': q.question.options,
          'correctAnswer': q.question.correctAnswer,
          'userAnswer': q.userAnswer,
          'timeTaken': q.timeTaken,
          'isCorrect': q.isCorrect,
          'finalScore': q.finalScore,
        }).toList(),
      );
      
      // Refresh user data
      await userProvider.loadUserProfile();
      await userProvider.loadUserStats();
      
      if (!mounted) return;
      
      // Navigate to results screen
      Navigator.of(context).pushReplacementNamed(
        AppRouter.home,
        arguments: {
          'gameId': _gameId,
          'totalScore': _totalScore,
          'correctAnswers': _correctAnswers,
          'wrongAnswers': _wrongAnswers,
          'totalDuration': totalDuration,
        },
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Error saving game: ${e.toString()}'),
          backgroundColor: AppTheme.errorColor,
        ),
      );
    }
  }

  void _showQuitDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Quit Game?'),
        content: const Text('Are you sure you want to quit? Your progress will not be saved.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Continue'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              Navigator.of(context).pushReplacementNamed(AppRouter.home);
            },
            child: const Text('Quit'),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    if (_loadingError != null) {
      return Scaffold(
        backgroundColor: AppTheme.backgroundColor,
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(
                Icons.error_outline,
                color: AppTheme.errorColor,
                size: 48,
              ),
              const SizedBox(height: 16),
              Text(_loadingError!),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () {
                  Navigator.of(context).pushReplacementNamed(AppRouter.home);
                },
                child: const Text('Go Back'),
              ),
            ],
          ),
        ),
      );
    }

    return WillPopScope(
      onWillPop: () async {
        _showQuitDialog();
        return false;
      },
      child: Scaffold(
        backgroundColor: AppTheme.backgroundColor,
        appBar: AppBar(
          title: Text('Question ${_currentQuestionIndex + 1}/5'),
          centerTitle: true,
          leading: IconButton(
            icon: const Icon(Icons.close),
            onPressed: _showQuitDialog,
          ),
          actions: [
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16.0),
              child: Center(
                child: Text(
                  'Score: $_totalScore',
                  style: const TextStyle(
                    fontSize: 16,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            ),
          ],
        ),
        body: SafeArea(
          child: _isLoading || _currentQuestion == null
              ? const Center(child: CircularProgressIndicator())
              : Column(
                  children: [
                    // Timer
                    TimerWidget(
                      initialSeconds: AppConstants.questionTimerSeconds,
                      onTimeUp: () {
                        if (!_answered) {
                          setState(() => _answered = true);
                          _processAnswer('');
                        }
                      },
                    ),
                    const SizedBox(height: 16),
                    // Main Content
                    Expanded(
                      child: SingleChildScrollView(
                        padding: const EdgeInsets.symmetric(horizontal: 16.0),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            // Difficulty Badge
                            Row(
                              children: [
                                Container(
                                  padding: const EdgeInsets.symmetric(
                                    horizontal: 12,
                                    vertical: 6,
                                  ),
                                  decoration: BoxDecoration(
                                    color: _getDifficultyColor(_currentDifficulty),
                                    borderRadius: BorderRadius.circular(20),
                                  ),
                                  child: Text(
                                    _currentDifficulty,
                                    style: const TextStyle(
                                      color: Colors.white,
                                      fontWeight: FontWeight.bold,
                                      fontSize: 12,
                                    ),
                                  ),
                                ),
                                const SizedBox(width: 8),
                                Container(
                                  padding: const EdgeInsets.symmetric(
                                    horizontal: 12,
                                    vertical: 6,
                                  ),
                                  decoration: BoxDecoration(
                                    color: AppTheme.primaryColor.withOpacity(0.2),
                                    borderRadius: BorderRadius.circular(20),
                                  ),
                                  child: Text(
                                    _currentQuestion!.topic,
                                    style: const TextStyle(
                                      color: AppTheme.primaryColor,
                                      fontWeight: FontWeight.bold,
                                      fontSize: 12,
                                    ),
                                  ),
                                ),
                              ],
                            ),
                            const SizedBox(height: 20),
                            // Question Card
                            QuestionCard(question: _currentQuestion!),
                            const SizedBox(height: 24),
                            // Options
                            OptionsWidget(
                              options: _currentQuestion!.options,
                              onOptionSelected: _answered ? null : _onAnswerSelected,
                              selectedAnswer: _selectedAnswer,
                              correctAnswer: _answered ? _currentQuestion!.correctAnswer : null,
                            ),
                            const SizedBox(height: 32),
                          ],
                        ),
                      ),
                    ),
                  ],
                ),
        ),
      ),
    );
  }

  Color _getDifficultyColor(String difficulty) {
    switch (difficulty) {
      case 'Easy':
        return AppTheme.easyColor;
      case 'Medium':
        return AppTheme.mediumColor;
      case 'Hard':
        return AppTheme.hardColor;
      default:
        return AppTheme.primaryColor;
    }
  }

  @override
  void dispose() {
    super.dispose();
  }
}
```

---

## 📱 Widget: QuestionCard

### File: `lib/widgets/quiz/question_card.dart`

```dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:nexira/models/question_model.dart';
import 'package:nexira/config/theme.dart';

class QuestionCard extends StatelessWidget {
  final Question question;

  const QuestionCard({
    Key? key,
    required this.question,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // Question Image
        Container(
          width: double.infinity,
          height: 200,
          decoration: BoxDecoration(
            borderRadius: BorderRadius.circular(12),
            color: Colors.grey[300],
          ),
          child: ClipRRect(
            borderRadius: BorderRadius.circular(12),
            child: CachedNetworkImage(
              imageUrl: question.image,
              fit: BoxFit.cover,
              placeholder: (context, url) => const Center(
                child: CircularProgressIndicator(),
              ),
              errorWidget: (context, url, error) => Container(
                color: Colors.grey[300],
                child: const Icon(
                  Icons.image_not_supported,
                  color: Colors.grey,
                  size: 48,
                ),
              ),
            ),
          ),
        ),
        const SizedBox(height: 20),
        // Question Text
        Text(
          question.question,
          style: const TextStyle(
            fontSize: 18,
            fontWeight: FontWeight.bold,
            color: AppTheme.textColor,
            height: 1.4,
          ),
        ),
      ],
    );
  }
}
```

---

## 📱 Widget: OptionsWidget

### File: `lib/widgets/quiz/options_widget.dart`

```dart
import 'package:flutter/material.dart';
import 'package:nexira/config/theme.dart';

class OptionsWidget extends StatelessWidget {
  final List<String> options;
  final Function(String)? onOptionSelected;
  final String? selectedAnswer;
  final String? correctAnswer;

  const OptionsWidget({
    Key? key,
    required this.options,
    this.onOptionSelected,
    this.selectedAnswer,
    this.correctAnswer,
  }) : super(key: key);

  Color _getOptionColor(String option) {
    if (correctAnswer == null) {
      if (selectedAnswer == option) {
        return AppTheme.primaryColor;
      }
      return const Color(0xFFE5E7EB);
    }

    if (option == correctAnswer) {
      return AppTheme.successColor;
    }

    if (option == selectedAnswer && option != correctAnswer) {
      return AppTheme.errorColor;
    }

    return const Color(0xFFE5E7EB);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: List.generate(
        options.length,
        (index) => Padding(
          padding: const EdgeInsets.only(bottom: 12.0),
          child: GestureDetector(
            onTap: onOptionSelected != null
                ? () => onOptionSelected!(options[index])
                : null,
            child: Container(
              width: double.infinity,
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                border: Border.all(
                  color: _getOptionColor(options[index]),
                  width: 2,
                ),
                borderRadius: BorderRadius.circular(8),
                color: _getOptionColor(options[index]).withOpacity(0.1),
              ),
              child: Row(
                children: [
                  Container(
                    width: 32,
                    height: 32,
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      border: Border.all(
                        color: _getOptionColor(options[index]),
                        width: 2,
                      ),
                    ),
                    child: Center(
                      child: Text(
                        String.fromCharCode(65 + index), // A, B, C, D
                        style: TextStyle(
                          fontWeight: FontWeight.bold,
                          color: _getOptionColor(options[index]),
                        ),
                      ),
                    ),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    child: Text(
                      options[index],
                      style: TextStyle(
                        fontSize: 16,
                        fontWeight: FontWeight.w500,
                        color: AppTheme.textColor,
                      ),
                    ),
                  ),
                  if (options[index] == correctAnswer)
                    const Icon(
                      Icons.check_circle,
                      color: AppTheme.successColor,
                      size: 24,
                    ),
                  if (options[index] == selectedAnswer &&
                      options[index] != correctAnswer)
                    const Icon(
                      Icons.cancel,
                      color: AppTheme.errorColor,
                      size: 24,
                    ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## 📱 Widget: TimerWidget

### File: `lib/widgets/quiz/timer_widget.dart`

```dart
import 'package:flutter/material.dart';
import 'dart:async';
import 'package:nexira/config/theme.dart';

class TimerWidget extends StatefulWidget {
  final int initialSeconds;
  final VoidCallback onTimeUp;

  const TimerWidget({
    Key? key,
    required this.initialSeconds,
    required this.onTimeUp,
  }) : super(key: key);

  @override
  State<TimerWidget> createState() => _TimerWidgetState();
}

class _TimerWidgetState extends State<TimerWidget> {
  late int _secondsRemaining;
  late Timer _timer;

  @override
  void initState() {
    super.initState();
    _secondsRemaining = widget.initialSeconds;
    _startTimer();
  }

  void _startTimer() {
    _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
      setState(() {
        _secondsRemaining--;
      });

      if (_secondsRemaining <= 0) {
        _timer.cancel();
        widget.onTimeUp();
      }
    });
  }

  Color _getTimerColor() {
    if (_secondsRemaining <= 3) {
      return AppTheme.errorColor;
    } else if (_secondsRemaining <= 5) {
      return AppTheme.warningColor;
    }
    return AppTheme.primaryColor;
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      decoration: BoxDecoration(
        color: _getTimerColor().withOpacity(0.1),
        border: Border.all(color: _getTimerColor(), width: 2),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.timer,
            color: _getTimerColor(),
            size: 24,
          ),
          const SizedBox(width: 8),
          Text(
            '$_secondsRemaining s',
            style: TextStyle(
              fontSize: 20,
              fontWeight: FontWeight.bold,
              color: _getTimerColor(),
            ),
          ),
        ],
      ),
    );
  }

  @override
  void dispose() {
    _timer.cancel();
    super.dispose();
  }
}
```

---

## 📱 Widget: ScoreDisplay

### File: `lib/widgets/quiz/score_display.dart`

```dart
import 'package:flutter/material.dart';
import 'package:nexira/config/theme.dart';

class ScoreDisplay extends StatelessWidget {
  final int score;
  final int correctAnswers;
  final int wrongAnswers;
  final double accuracy;

  const ScoreDisplay({
    Key? key,
    required this.score,
    required this.correctAnswers,
    required this.wrongAnswers,
    required this.accuracy,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(20),
      decoration: BoxDecoration(
        gradient: LinearGradient(
          colors: [
            AppTheme.primaryColor,
            AppTheme.primaryColor.withOpacity(0.7),
          ],
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
        ),
        borderRadius: BorderRadius.circular(12),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'Game Results',
            style: TextStyle(
              fontSize: 20,
              fontWeight: FontWeight.bold,
              color: Colors.white,
            ),
          ),
          const SizedBox(height: 20),
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            children: [
              _buildResultItem(
                label: 'Total Score',
                value: '$score',
                icon: Icons.star,
              ),
              _buildResultItem(
                label: 'Correct',
                value: '$correctAnswers',
                icon: Icons.check_circle,
              ),
              _buildResultItem(
                label: 'Wrong',
                value: '$wrongAnswers',
                icon: Icons.cancel,
              ),
              _buildResultItem(
                label: 'Accuracy',
                value: '${accuracy.toStringAsFixed(1)}%',
                icon: Icons.trending_up,
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildResultItem({
    required String label,
    required String value,
    required IconData icon,
  }) {
    return Column(
      children: [
        Icon(icon, color: Colors.white, size: 28),
        const SizedBox(height: 8),
        Text(
          value,
          style: const TextStyle(
            fontSize: 18,
            fontWeight: FontWeight.bold,
            color: Colors.white,
          ),
        ),
        const SizedBox(height: 4),
        Text(
          label,
          style: const TextStyle(
            fontSize: 12,
            color: Colors.white70,
          ),
        ),
      ],
    );
  }
}
```

---

## 📱 Widget: DifficultyBadge

### File: `lib/widgets/quiz/difficulty_badge.dart`

```dart
import 'package:flutter/material.dart';
import 'package:nexira/config/theme.dart';

class DifficultyBadge extends StatelessWidget {
  final String difficulty;
  final Size? size;

  const DifficultyBadge({
    Key? key,
    required this.difficulty,
    this.size,
  }) : super(key: key);

  Color _getDifficultyColor() {
    switch (difficulty) {
      case 'Easy':
        return AppTheme.easyColor;
      case 'Medium':
        return AppTheme.mediumColor;
      case 'Hard':
        return AppTheme.hardColor;
      default:
        return AppTheme.primaryColor;
    }
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      decoration: BoxDecoration(
        color: _getDifficultyColor(),
        borderRadius: BorderRadius.circular(20),
      ),
      child: Text(
        difficulty,
        style: const TextStyle(
          color: Colors.white,
          fontWeight: FontWeight.bold,
          fontSize: 12,
        ),
      ),
    );
  }
}
```

---

## 🔄 Provider: GameProvider

### File: `lib/providers/game_provider.dart`

```dart
import 'package:flutter/material.dart';
import 'package:nexira/models/game_model.dart';
import 'package:nexira/models/question_model.dart';

class GameProvider extends ChangeNotifier {
  GameSession? _currentGame;
  List<QuestionAnswer> _answeredQuestions = [];
  int _totalScore = 0;
  bool _isGameActive = false;

  GameSession? get currentGame => _currentGame;
  List<QuestionAnswer> get answeredQuestions => _answeredQuestions;
  int get totalScore => _totalScore;
  bool get isGameActive => _isGameActive;

  void startGame({required String gameId, required String userId}) {
    _currentGame = GameSession(
      gameId: gameId,
      userId: userId,
      startedAt: DateTime.now(),
      totalDuration: 0,
      totalScore: 0,
      questionsAttempted: 0,
      correctAnswers: 0,
      wrongAnswers: 0,
      accuracy: 0,
      questions: [],
      status: 'in_progress',
    );
    _isGameActive = true;
    _totalScore = 0;
    _answeredQuestions = [];
    notifyListeners();
  }

  void addAnsweredQuestion(QuestionAnswer question) {
    _answeredQuestions.add(question);
    _totalScore += question.finalScore;
    notifyListeners();
  }

  void endGame({required DateTime endTime}) {
    if (_currentGame != null) {
      final duration = endTime.difference(_currentGame!.startedAt).inSeconds;
      final correctCount = _answeredQuestions.where((q) => q.isCorrect).length;
      final accuracy = (correctCount / _answeredQuestions.length) * 100;

      _currentGame = _currentGame!.copyWith(
        completedAt: endTime,
        totalDuration: duration,
        totalScore: _totalScore,
        questionsAttempted: _answeredQuestions.length,
        correctAnswers: correctCount,
        wrongAnswers: _answeredQuestions.length - correctCount,
        accuracy: accuracy,
        questions: _answeredQuestions,
        status: 'completed',
      );
    }
    _isGameActive = false;
    notifyListeners();
  }

  void resetGame() {
    _currentGame = null;
    _answeredQuestions = [];
    _totalScore = 0;
    _isGameActive = false;
    notifyListeners();
  }
}

extension GameSessionCopyWith on GameSession {
  GameSession copyWith({
    String? gameId,
    String? userId,
    DateTime? startedAt,
    DateTime? completedAt,
    int? totalDuration,
    int? totalScore,
    int? questionsAttempted,
    int? correctAnswers,
    int? wrongAnswers,
    double? accuracy,
    List<QuestionAnswer>? questions,
    String? status,
  }) {
    return GameSession(
      gameId: gameId ?? this.gameId,
      userId: userId ?? this.userId,
      startedAt: startedAt ?? this.startedAt,
      completedAt: completedAt ?? this.completedAt,
      totalDuration: totalDuration ?? this.totalDuration,
      totalScore: totalScore ?? this.totalScore,
      questionsAttempted: questionsAttempted ?? this.questionsAttempted,
      correctAnswers: correctAnswers ?? this.correctAnswers,
      wrongAnswers: wrongAnswers ?? this.wrongAnswers,
      accuracy: accuracy ?? this.accuracy,
      questions: questions ?? this.questions,
      status: status ?? this.status,
    );
  }
}
```

---

## 🔄 Provider: DifficultyProvider

### File: `lib/providers/difficulty_provider.dart`

```dart
import 'package:flutter/material.dart';

class DifficultyProvider extends ChangeNotifier {
  String _currentDifficulty = 'Easy';
  List<String> _difficultyHistory = [];

  String get currentDifficulty => _currentDifficulty;
  List<String> get difficultyHistory => _difficultyHistory;

  void setCurrentDifficulty(String difficulty) {
    _currentDifficulty = difficulty;
    notifyListeners();
  }

  void addToDifficultyHistory(String difficulty) {
    _difficultyHistory.add(difficulty);
    notifyListeners();
  }

  void resetDifficulty() {
    _currentDifficulty = 'Easy';
    _difficultyHistory = [];
    notifyListeners();
  }

  String getNextDifficulty({
    required int correctAnswers,
    required int wrongAnswers,
  }) {
    final performanceScore = correctAnswers - wrongAnswers;

    if (performanceScore > 2) {
      // Increase difficulty
      if (_currentDifficulty == 'Easy') {
        return 'Medium';
      } else if (_currentDifficulty == 'Medium') {
        return 'Hard';
      }
    } else if (performanceScore < -2) {
      // Decrease difficulty
      if (_currentDifficulty == 'Hard') {
        return 'Medium';
      } else if (_currentDifficulty == 'Medium') {
        return 'Easy';
      }
    }

    // Keep same difficulty
    return _currentDifficulty;
  }
}
```

---

## 🔄 Provider: UserProvider

### File: `lib/providers/user_provider.dart`

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:nexira/models/user_model.dart' as user_model;
import 'package:nexira/models/stats_model.dart';
import 'package:nexira/services/firestore_service.dart';

class UserProvider extends ChangeNotifier {
  final FirestoreService _firestoreService = FirestoreService();

  user_model.User? _user;
  UserStats? _stats;
  bool _isLoading = false;

  user_model.User? get user => _user;
  UserStats? get stats => _stats;
  bool get isLoading => _isLoading;

  Future<void> loadUserProfile() async {
    try {
      _isLoading = true;
      notifyListeners();

      final currentUser = FirebaseAuth.instance.currentUser;
      if (currentUser != null) {
        _user = await _firestoreService.getUserProfile(currentUser.uid);
      }

      _isLoading = false;
      notifyListeners();
    } catch (e) {
      _isLoading = false;
      notifyListeners();
    }
  }

  Future<void> loadUserStats() async {
    try {
      final currentUser = FirebaseAuth.instance.currentUser;
      if (currentUser != null) {
        _stats = await _firestoreService.getUserStats(currentUser.uid);
      }
      notifyListeners();
    } catch (e) {
      notifyListeners();
    }
  }

  void updateLocalUser(user_model.User updatedUser) {
    _user = updatedUser;
    notifyListeners();
  }
}
```

---

## 🔄 Service: FirestoreService

### File: `lib/services/firestore_service.dart`

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:nexira/models/user_model.dart' as user_model;
import 'package:nexira/models/stats_model.dart';

class FirestoreService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;

  // Create user profile
  Future<void> createUserProfile({
    required String uid,
    required String email,
    required String displayName,
  }) async {
    try {
      await _db.collection('users').doc(uid).set({
        'uid': uid,
        'email': email,
        'displayName': displayName,
        'avatar': null,
        'totalScore': 0,
        'gamesPlayed': 0,
        'currentDifficulty': 'Easy',
        'totalTimeSpent': 0,
        'isActive': true,
        'createdAt': DateTime.now().toIso8601String(),
        'lastPlayedAt': DateTime.now().toIso8601String(),
      });
    } catch (e) {
      rethrow;
    }
  }

  // Get user profile
  Future<user_model.User> getUserProfile(String uid) async {
    try {
      final doc = await _db.collection('users').doc(uid).get();
      if (doc.exists) {
        return user_model.User.fromJson(doc.data()!);
      }
      throw Exception('User not found');
    } catch (e) {
      rethrow;
    }
  }

  // Get user stats
  Future<UserStats> getUserStats(String uid) async {
    try {
      final doc = await _db.collection('userStats').doc(uid).get();
      if (doc.exists) {
        return UserStats.fromJson(doc.data()!);
      }
      // Return default stats if not found
      return UserStats(
        userId: uid,
        totalScore: 0,
        gamesPlayed: 0,
        totalCorrectAnswers: 0,
        totalWrongAnswers: 0,
        accuracyPercentage: 0,
        averageScorePerGame: 0,
        favoriteTopics: [],
        difficultyStats: {
          'Easy': DifficultyStats(attempted: 0, correct: 0, accuracy: 0),
          'Medium': DifficultyStats(attempted: 0, correct: 0, accuracy: 0),
          'Hard': DifficultyStats(attempted: 0, correct: 0, accuracy: 0),
        },
        lastUpdated: DateTime.now(),
      );
    } catch (e) {
      rethrow;
    }
  }

  // Get game history
  Future<List<dynamic>> getGameHistory({
    required String userId,
    int limit = 10,
    int offset = 0,
  }) async {
    try {
      final query = _db
          .collection('gameHistory')
          .where('userId', isEqualTo: userId)
          .orderBy('completedAt', descending: true)
          .limit(limit + 1)
          .offset(offset);

      final snapshot = await query.get();
      return snapshot.docs.map((doc) => doc.data()).toList();
    } catch (e) {
      rethrow;
    }
  }
}
```

---

## ✅ MVP Quiz Game Status

**Completed:**
- ✅ Quiz Screen (Main Game Loop)
- ✅ Timer Widget (10 seconds)
- ✅ Question Card (Image + Text)
- ✅ Options Widget (4 choices)
- ✅ Score Display
- ✅ Difficulty Badge
- ✅ Answer Validation & Feedback
- ✅ Hidden Learning (Explanation)
- ✅ Game Result Saving
- ✅ Dynamic Difficulty

**Architecture:**
- Game Flow: Question Generation → Answer → Score → Result → Next
- State Management: Provider Pattern
- Data Persistence: Firestore
- API Integration: Cloud Functions + Gemini

---

## 🚀 Final Steps for MVP

1. ✅ Database Schema
2. ✅ Cloud Functions
3. ✅ Flutter Structure
4. ✅ Screen Implementations
5. ✅ Quiz Game Screen (THIS FILE)
6. ⏳ Profile Screen
7. ⏳ Game History Screen
8. ⏳ Testing & Deployment
