# Nexira – Firestore Database Schema (MVP Version 1)

## Overview
This document defines the Firestore collections and document structure for Nexira MVP (Light Version).

---

## 📊 Collections Structure

### 1. **Collection: `users`**
Stores user profile information.

```json
{
  "uid": "user_123",
  "email": "player@example.com",
  "displayName": "Player Name",
  "avatar": "https://url-to-avatar.jpg",
  "totalScore": 5420,
  "gamesPlayed": 15,
  "createdAt": "2026-06-29T10:30:00Z",
  "lastPlayedAt": "2026-06-29T15:45:00Z",
  "currentDifficulty": "Medium",
  "totalTimeSpent": 1350,
  "isActive": true
}
```

**Index Requirements:**
- Primary: `uid` (auto)
- No composite indexes needed for MVP

---

### 2. **Collection: `gameHistory`**
Stores complete game session data.

```json
{
  "gameId": "game_abc123def456",
  "userId": "user_123",
  "startedAt": "2026-06-29T10:00:00Z",
  "completedAt": "2026-06-29T10:05:00Z",
  "totalDuration": 300,
  "totalScore": 145,
  "questionsAttempted": 5,
  "correctAnswers": 4,
  "wrongAnswers": 1,
  "averageDifficulty": "Medium",
  "questions": [
    {
      "questionId": "q_001",
      "question": "What is the capital of France?",
      "image": "https://url-to-image.jpg",
      "topic": "History",
      "difficulty": "Easy",
      "options": ["Paris", "London", "Berlin", "Madrid"],
      "correctAnswer": "Paris",
      "userAnswer": "Paris",
      "isCorrect": true,
      "timeTaken": 3.2,
      "speedBonus": 5,
      "baseScore": 10,
      "difficultyMultiplier": 1.0,
      "finalScore": 15,
      "explanation": "Paris is the capital and most populous city of France."
    }
  ],
  "status": "completed"
}
```

**Index Requirements:**
- Composite: `userId` + `completedAt` (for fetching user history)

---

### 3. **Collection: `userStats`**
Aggregated statistics for quick access (denormalized).

```json
{
  "userId": "user_123",
  "totalScore": 5420,
  "gamesPlayed": 15,
  "totalCorrectAnswers": 62,
  "totalWrongAnswers": 13,
  "accuracyPercentage": 82.67,
  "averageScorePerGame": 361.33,
  "favoriteTopics": ["Math", "History", "Science"],
  "difficultyStats": {
    "Easy": {
      "attempted": 45,
      "correct": 44,
      "accuracy": 97.78
    },
    "Medium": {
      "attempted": 22,
      "correct": 16,
      "accuracy": 72.73
    },
    "Hard": {
      "attempted": 8,
      "correct": 2,
      "accuracy": 25.0
    }
  },
  "lastUpdated": "2026-06-29T15:45:00Z"
}
```

**Index Requirements:**
- Primary: `userId` (auto)

---

### 4. **Collection: `generatedQuestions`**
Cache of AI-generated questions (for reuse & tracking).

```json
{
  "questionId": "q_001",
  "question": "What is the capital of France?",
  "image": "https://url-to-image.jpg",
  "topic": "History",
  "subtopic": "European Capitals",
  "difficulty": "Easy",
  "options": {
    "option_1": "Paris",
    "option_2": "London",
    "option_3": "Berlin",
    "option_4": "Madrid"
  },
  "correctAnswerKey": "option_1",
  "correctAnswer": "Paris",
  "explanation": "Paris is the capital and most populous city of France.",
  "generatedAt": "2026-06-29T10:00:00Z",
  "timesUsed": 2,
  "source": "gemini_api",
  "metadata": {
    "generationModel": "gemini-2.0-flash",
    "processingTime": 2.5,
    "imageGenerationModel": "image-generation-001",
    "language": "en"
  }
}
```

**Index Requirements:**
- Composite: `topic` + `difficulty` (for question selection)
- Composite: `difficulty` + `generatedAt` (for recent questions)

---

### 5. **Collection: `userDifficultyProgress`**
Tracks difficulty adjustment per user (for dynamic difficulty).

```json
{
  "userId": "user_123",
  "currentDifficulty": "Medium",
  "difficultyHistory": [
    {
      "difficulty": "Easy",
      "startedAt": "2026-06-29T10:00:00Z",
      "endedAt": "2026-06-29T10:15:00Z",
      "questionsAttempted": 5,
      "correctAnswers": 5,
      "lastScore": 3
    },
    {
      "difficulty": "Medium",
      "startedAt": "2026-06-29T10:15:00Z",
      "endedAt": "2026-06-29T10:25:00Z",
      "questionsAttempted": 4,
      "correctAnswers": 3,
      "lastScore": 0
    }
  ],
  "lastAdjustedAt": "2026-06-29T10:25:00Z",
  "difficultyChangeReason": "Stable performance"
}
```

**Index Requirements:**
- Primary: `userId` (auto)

---

## 🔐 Firestore Security Rules (MVP)

```javascript
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    
    // Users can only read/write their own profile
    match /users/{userId} {
      allow read, write: if request.auth.uid == userId;
    }
    
    // Users can only read/write their own game history
    match /gameHistory/{gameId} {
      allow read, write: if resource.data.userId == request.auth.uid;
      allow create: if request.auth.uid != null;
    }
    
    // Users can only read/write their own stats
    match /userStats/{userId} {
      allow read, write: if request.auth.uid == userId;
    }
    
    // Everyone can read generated questions
    match /generatedQuestions/{questionId} {
      allow read: if request.auth.uid != null;
      allow write: if false; // Only Cloud Functions can write
    }
    
    // Users can only read/write their own difficulty progress
    match /userDifficultyProgress/{userId} {
      allow read, write: if request.auth.uid == userId;
    }
    
  }
}
```

---

## 📈 Firestore Scaling & Optimization (MVP)

### Collection Sizes (Estimated after 1 year):
| Collection | Est. Documents | Est. Size |
|------------|----------------|-----------|
| users | 10,000 | ~20 MB |
| gameHistory | 500,000 | ~1 GB |
| userStats | 10,000 | ~50 MB |
| generatedQuestions | 50,000 | ~500 MB |
| userDifficultyProgress | 10,000 | ~30 MB |

### Read/Write Operations (Daily):
| Operation | Count | Cost (reads/writes) |
|-----------|-------|-------------------|
| User Sign In | 5,000 | 5,000 reads |
| Start Game | 5,000 | 10,000 writes |
| Answer Question | 50,000 | 50,000 writes |
| End Game | 5,000 | 15,000 writes |
| View History | 3,000 | 3,000 reads |

**Total Daily: ~83,000 read/write operations**
**Cost at Spark Plan: FREE (up to 50,000 free reads/writes)**

---

## 🔄 Data Flow (MVP)

```
1. USER SIGNS UP
   → Create document in /users/{uid}
   → Initialize /userStats/{uid}
   → Initialize /userDifficultyProgress/{uid}

2. USER STARTS GAME
   → Read /userDifficultyProgress/{uid} (get current difficulty)
   → Create new document in /gameHistory/{gameId}

3. AI GENERATES QUESTION
   → Cloud Function calls Gemini API
   → Stores question in /generatedQuestions/{questionId}
   → Returns question to Flutter app

4. USER ANSWERS QUESTION
   → Calculate score
   → Update /gameHistory/{gameId} with question data

5. GAME ENDS
   → Finalize /gameHistory/{gameId}
   → Update /userStats/{uid} (aggregate stats)
   → Calculate new difficulty
   → Update /userDifficultyProgress/{uid}
   → Update /users/{uid} (lastPlayedAt, totalScore)
```

---

## ✅ MVP Constraints

✔️ **Included:**
- User profiles
- Game history
- User stats
- Generated questions cache
- Difficulty tracking
- Simple security rules

❌ **NOT Included (Version 2+):**
- 1vs1 match data
- Matchmaking queues
- Global leaderboards
- XP/Level system
- Friends system
- Real-time sync for multiplayer
- Tournament brackets
- Season data

---

## 📝 Notes

1. **Question Cache:** Questions are cached in `generatedQuestions` to:
   - Reduce API calls to Gemini
   - Track question usage
   - Enable offline fallback (future)

2. **Denormalized Stats:** `userStats` is denormalized for fast reads. Update via Cloud Function after each game.

3. **Difficulty Tracking:** Kept separate for easy adjustment in Version 2 when matchmaking is added.

4. **Firebase Plan:** MVP works perfectly on Spark Plan (free). Upgrade to Blaze as scale increases.

5. **Image Storage:** Images are hosted on external CDN or Firebase Storage. Firestore only stores URLs.

---

## 🚀 Next Steps

1. ✅ Firestore Schema (THIS FILE)
2. ⏳ Cloud Functions (Backend)
3. ⏳ Flutter Project Structure (Frontend)
4. ⏳ Implementation & Testing
