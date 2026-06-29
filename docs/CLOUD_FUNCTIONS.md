# Nexira – Cloud Functions (Backend) - MVP Version 1

## Overview
This document contains all Cloud Functions for Nexira MVP (Light Version).
These functions handle AI question generation, scoring, difficulty calculation, and data management.

---

## 📁 Project Structure

```
firebase-functions/
├── functions/
│   ├── package.json
│   ├── .env.example
│   ├── index.js
│   ├── config/
│   │   └── firebaseConfig.js
│   ├── functions/
│   │   ├── generateQuestion.js
│   │   ├── saveGameResult.js
│   │   ├── calculateDifficulty.js
│   │   ├── getUserProfile.js
│   │   ├── getUserStats.js
│   │   └── getUserGameHistory.js
│   └── utils/
│       ├── scoringEngine.js
│       ├── difficultyEngine.js
│       └── geminiClient.js
```

---

## 🔧 Setup Instructions

### 1. Install Firebase CLI
```bash
npm install -g firebase-tools
firebase login
```

### 2. Initialize Functions Project
```bash
firebase init functions
# Choose Node.js
# Choose TypeScript: No
```

### 3. Install Dependencies
```bash
cd functions
npm install
npm install google-generative-ai
npm install dotenv
npm install cors
```

### 4. Create .env File
```
GEMINI_API_KEY=your_gemini_api_key_here
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_STORAGE_BUCKET=your_bucket
```

---

## 📝 Cloud Functions Code

### File: `functions/config/firebaseConfig.js`

```javascript
const admin = require('firebase-admin');

// Initialize Firebase Admin SDK
if (!admin.apps.length) {
  admin.initializeApp();
}

const db = admin.firestore();
const auth = admin.auth();

module.exports = {
  admin,
  db,
  auth
};
```

---

### File: `functions/utils/geminiClient.js`

```javascript
const { GoogleGenerativeAI } = require("google-generative-ai");
require('dotenv').config();

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

// Initialize Gemini model
const model = genAI.getGenerativeModel({
  model: "gemini-2.0-flash",
});

/**
 * Generate a quiz question using Gemini AI
 * @param {string} difficulty - Easy, Medium, Hard
 * @param {string} topic - Math, Science, History, English, General, IQ, Digital
 * @returns {Promise<Object>} Question object with text, options, image, explanation
 */
async function generateQuestion(difficulty, topic) {
  try {
    const difficultyGuide = {
      'Easy': 'Create a simple, beginner-level question',
      'Medium': 'Create a moderate-level question requiring some knowledge',
      'Hard': 'Create a challenging question requiring advanced knowledge'
    };

    const topics = {
      'Math': 'Mathematics, arithmetic, algebra, geometry',
      'Science': 'Physics, Chemistry, Biology, Natural Sciences',
      'History': 'World history, historical events, famous figures',
      'English': 'English language, grammar, vocabulary, literature',
      'General': 'General knowledge, culture, geography, diverse topics',
      'IQ': 'Logic puzzles, reasoning, problem-solving',
      'Digital': 'Technology, computers, internet, digital literacy'
    };

    const prompt = `
You are a quiz game AI. Generate a single quiz question in JSON format.

Requirements:
- Difficulty Level: ${difficulty}
- Topic: ${topics[topic] || 'General Knowledge'}
- ${difficultyGuide[difficulty]}
- Question must be SHORT (max 15 words)
- Question MUST be engaging and interesting
- Generate a relevant IMAGE URL (use descriptive image search terms)
- Provide exactly 4 multiple-choice options
- Mark the correct answer clearly
- Provide a SHORT explanation (max 30 words)
- Language: English

IMPORTANT: The question MUST be visual/image-based. Generate an image URL that represents the question.

Return ONLY valid JSON in this exact format (no markdown, no extra text):
{
  "question": "Short question text here?",
  "image": "https://images.example.com/descriptive-image-name.jpg",
  "options": {
    "A": "Option 1",
    "B": "Option 2",
    "C": "Option 3",
    "D": "Option 4"
  },
  "correctAnswer": "A",
  "correctAnswerText": "Option 1",
  "explanation": "Short explanation of the correct answer",
  "topic": "${topic}",
  "difficulty": "${difficulty}"
}`;

    const response = await model.generateContent(prompt);
    const text = response.response.text();
    
    // Parse JSON response
    let questionData = JSON.parse(text);
    
    // Validate response structure
    if (!questionData.question || !questionData.options || !questionData.correctAnswer) {
      throw new Error('Invalid response structure from Gemini API');
    }

    return {
      questionId: `q_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      question: questionData.question,
      image: questionData.image,
      options: [
        questionData.options.A,
        questionData.options.B,
        questionData.options.C,
        questionData.options.D
      ],
      correctAnswer: questionData.correctAnswerText,
      correctAnswerIndex: ['A', 'B', 'C', 'D'].indexOf(questionData.correctAnswer),
      explanation: questionData.explanation,
      topic: questionData.topic,
      difficulty: questionData.difficulty,
      generatedAt: new Date().toISOString(),
      timesUsed: 0
    };
  } catch (error) {
    console.error('Gemini API Error:', error);
    throw new Error(`Failed to generate question: ${error.message}`);
  }
}

module.exports = {
  generateQuestion
};
```

---

### File: `functions/utils/scoringEngine.js`

```javascript
/**
 * Calculate score for a single question answer
 * @param {Object} params - Scoring parameters
 * @returns {number} Final score
 */
function calculateScore(params) {
  const {
    isCorrect,
    timeTaken,
    difficulty,
    timeLimitSeconds = 10
  } = params;

  let score = 0;

  // Base score for correct answer
  if (isCorrect) {
    score = 10;
  } else {
    score = -5; // Penalty for wrong answer
    return score;
  }

  // Speed bonus (answered in < 3 seconds)
  if (timeTaken < 3) {
    score += 5;
  }

  // Difficulty multiplier
  const difficultyMultiplier = {
    'Easy': 1.0,
    'Medium': 1.5,
    'Hard': 2.0
  };

  const multiplier = difficultyMultiplier[difficulty] || 1.0;
  score = Math.floor(score * multiplier);

  return score;
}

/**
 * Calculate total game score
 * @param {Array} questions - Array of answered questions
 * @returns {number} Total score
 */
function calculateTotalScore(questions) {
  return questions.reduce((total, question) => {
    return total + question.finalScore;
  }, 0);
}

module.exports = {
  calculateScore,
  calculateTotalScore
};
```

---

### File: `functions/utils/difficultyEngine.js`

```javascript
/**
 * Determine next difficulty based on performance
 * @param {Object} params - Performance parameters
 * @returns {string} Next difficulty (Easy, Medium, Hard)
 */
function calculateNextDifficulty(params) {
  const {
    currentDifficulty,
    lastFiveQuestions,
    maxDifficulty = 'Hard',
    minDifficulty = 'Easy'
  } = params;

  // Score based on last 5 questions
  let performanceScore = 0;
  lastFiveQuestions.forEach(q => {
    if (q.isCorrect) {
      performanceScore += 1;
    } else {
      performanceScore -= 1;
    }
  });

  // Difficulty transitions
  const difficultyOrder = ['Easy', 'Medium', 'Hard'];
  const currentIndex = difficultyOrder.indexOf(currentDifficulty);

  if (performanceScore > 2) {
    // Very good performance - increase difficulty
    const nextIndex = Math.min(currentIndex + 1, difficultyOrder.length - 1);
    return difficultyOrder[nextIndex];
  } else if (performanceScore < -2) {
    // Poor performance - decrease difficulty
    const nextIndex = Math.max(currentIndex - 1, 0);
    return difficultyOrder[nextIndex];
  } else {
    // Stable performance - keep same difficulty
    return currentDifficulty;
  }
}

module.exports = {
  calculateNextDifficulty
};
```

---

### File: `functions/functions/generateQuestion.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { generateQuestion } = require('../utils/geminiClient');
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: generateQuestion
 * Generates a new quiz question using AI
 * 
 * Request body:
 * {
 *   "difficulty": "Easy|Medium|Hard",
 *   "topic": "Math|Science|History|English|General|IQ|Digital",
 *   "userId": "user_123"
 * }
 */
exports.generateQuestion = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { difficulty, topic, userId } = req.body;

      // Validate inputs
      if (!difficulty || !topic || !userId) {
        return res.status(400).json({
          success: false,
          error: 'Missing required fields: difficulty, topic, userId'
        });
      }

      const validDifficulties = ['Easy', 'Medium', 'Hard'];
      const validTopics = ['Math', 'Science', 'History', 'English', 'General', 'IQ', 'Digital'];

      if (!validDifficulties.includes(difficulty)) {
        return res.status(400).json({
          success: false,
          error: `Invalid difficulty. Must be one of: ${validDifficulties.join(', ')}`
        });
      }

      if (!validTopics.includes(topic)) {
        return res.status(400).json({
          success: false,
          error: `Invalid topic. Must be one of: ${validTopics.join(', ')}`
        });
      }

      // Generate question using Gemini AI
      const question = await generateQuestion(difficulty, topic);

      // Cache question in Firestore
      await db.collection('generatedQuestions').doc(question.questionId).set({
        ...question,
        cachedAt: new Date()
      });

      // Return question to client
      res.status(200).json({
        success: true,
        data: question
      });

    } catch (error) {
      console.error('Error in generateQuestion:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/functions/saveGameResult.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { calculateScore, calculateTotalScore } = require('../utils/scoringEngine');
const { calculateNextDifficulty } = require('../utils/difficultyEngine');
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: saveGameResult
 * Saves game session result and updates user stats
 * 
 * Request body:
 * {
 *   "userId": "user_123",
 *   "gameId": "game_abc123",
 *   "startedAt": "2026-06-29T10:00:00Z",
 *   "completedAt": "2026-06-29T10:05:00Z",
 *   "questions": [
 *     {
 *       "questionId": "q_001",
 *       "question": "What is 2+2?",
 *       "image": "url",
 *       "topic": "Math",
 *       "difficulty": "Easy",
 *       "options": ["3", "4", "5", "6"],
 *       "correctAnswer": "4",
 *       "userAnswer": "4",
 *       "timeTaken": 2.5
 *     }
 *   ]
 * }
 */
exports.saveGameResult = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { userId, gameId, startedAt, completedAt, questions } = req.body;

      if (!userId || !gameId || !questions || questions.length === 0) {
        return res.status(400).json({
          success: false,
          error: 'Missing required fields'
        });
      }

      // Calculate score for each question
      const processedQuestions = questions.map(q => {
        const isCorrect = q.userAnswer === q.correctAnswer;
        const finalScore = calculateScore({
          isCorrect,
          timeTaken: q.timeTaken,
          difficulty: q.difficulty
        });

        return {
          ...q,
          isCorrect,
          finalScore
        };
      });

      // Calculate game statistics
      const totalScore = calculateTotalScore(processedQuestions);
      const correctAnswers = processedQuestions.filter(q => q.isCorrect).length;
      const wrongAnswers = processedQuestions.filter(q => !q.isCorrect).length;
      const accuracy = (correctAnswers / processedQuestions.length) * 100;
      const totalDuration = (new Date(completedAt) - new Date(startedAt)) / 1000;

      // Save game history
      await db.collection('gameHistory').doc(gameId).set({
        userId,
        gameId,
        startedAt,
        completedAt,
        totalDuration,
        totalScore,
        questionsAttempted: processedQuestions.length,
        correctAnswers,
        wrongAnswers,
        accuracy,
        questions: processedQuestions,
        status: 'completed'
      });

      // Update user profile
      await db.collection('users').doc(userId).update({
        totalScore: functions.firestore.FieldValue.increment(totalScore),
        gamesPlayed: functions.firestore.FieldValue.increment(1),
        lastPlayedAt: new Date()
      });

      // Update user stats (denormalized)
      const userStatsRef = db.collection('userStats').doc(userId);
      const userStatsSnap = await userStatsRef.get();

      if (userStatsSnap.exists) {
        const stats = userStatsSnap.data();
        
        // Update stats
        await userStatsRef.update({
          totalScore: stats.totalScore + totalScore,
          gamesPlayed: stats.gamesPlayed + 1,
          totalCorrectAnswers: stats.totalCorrectAnswers + correctAnswers,
          totalWrongAnswers: stats.totalWrongAnswers + wrongAnswers,
          accuracyPercentage: ((stats.totalCorrectAnswers + correctAnswers) / (stats.totalCorrectAnswers + correctAnswers + stats.totalWrongAnswers + wrongAnswers)) * 100,
          averageScorePerGame: (stats.totalScore + totalScore) / (stats.gamesPlayed + 1),
          lastUpdated: new Date()
        });
      } else {
        // Create new user stats
        await userStatsRef.set({
          userId,
          totalScore,
          gamesPlayed: 1,
          totalCorrectAnswers: correctAnswers,
          totalWrongAnswers: wrongAnswers,
          accuracyPercentage: accuracy,
          averageScorePerGame: totalScore,
          favoriteTopics: [],
          difficultyStats: {
            Easy: { attempted: 0, correct: 0, accuracy: 0 },
            Medium: { attempted: 0, correct: 0, accuracy: 0 },
            Hard: { attempted: 0, correct: 0, accuracy: 0 }
          },
          lastUpdated: new Date()
        });
      }

      // Calculate next difficulty
      const difficultyProgressRef = db.collection('userDifficultyProgress').doc(userId);
      const difficultySnap = await difficultyProgressRef.get();

      if (difficultySnap.exists) {
        const difficultyData = difficultySnap.data();
        const lastFive = difficultyData.difficultyHistory.slice(-5);

        const nextDifficulty = calculateNextDifficulty({
          currentDifficulty: difficultyData.currentDifficulty,
          lastFiveQuestions: lastFive
        });

        await difficultyProgressRef.update({
          currentDifficulty: nextDifficulty,
          lastAdjustedAt: new Date(),
          difficultyChangeReason: `Based on last ${lastFive.length} questions`
        });
      }

      res.status(200).json({
        success: true,
        data: {
          gameId,
          totalScore,
          correctAnswers,
          wrongAnswers,
          accuracy: accuracy.toFixed(2),
          totalDuration
        }
      });

    } catch (error) {
      console.error('Error in saveGameResult:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/functions/calculateDifficulty.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { calculateNextDifficulty } = require('../utils/difficultyEngine');
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: calculateDifficulty
 * Gets current difficulty for user
 * 
 * Request body:
 * {
 *   "userId": "user_123"
 * }
 */
exports.calculateDifficulty = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { userId } = req.body;

      if (!userId) {
        return res.status(400).json({
          success: false,
          error: 'Missing userId'
        });
      }

      // Get user difficulty progress
      const difficultyRef = db.collection('userDifficultyProgress').doc(userId);
      const difficultySnap = await difficultyRef.get();

      let currentDifficulty = 'Easy'; // Default for new users

      if (difficultySnap.exists) {
        const difficultyData = difficultySnap.data();
        currentDifficulty = difficultyData.currentDifficulty;
      } else {
        // Initialize difficulty progress for new user
        await difficultyRef.set({
          userId,
          currentDifficulty: 'Easy',
          difficultyHistory: [],
          lastAdjustedAt: new Date(),
          difficultyChangeReason: 'New user - Default easy'
        });
      }

      res.status(200).json({
        success: true,
        data: {
          userId,
          currentDifficulty
        }
      });

    } catch (error) {
      console.error('Error in calculateDifficulty:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/functions/getUserProfile.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: getUserProfile
 * Fetches user profile data
 * 
 * Request body:
 * {
 *   "userId": "user_123"
 * }
 */
exports.getUserProfile = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { userId } = req.body;

      if (!userId) {
        return res.status(400).json({
          success: false,
          error: 'Missing userId'
        });
      }

      const userDoc = await db.collection('users').doc(userId).get();

      if (!userDoc.exists) {
        return res.status(404).json({
          success: false,
          error: 'User not found'
        });
      }

      const userData = userDoc.data();

      res.status(200).json({
        success: true,
        data: userData
      });

    } catch (error) {
      console.error('Error in getUserProfile:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/functions/getUserStats.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: getUserStats
 * Fetches user statistics
 * 
 * Request body:
 * {
 *   "userId": "user_123"
 * }
 */
exports.getUserStats = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { userId } = req.body;

      if (!userId) {
        return res.status(400).json({
          success: false,
          error: 'Missing userId'
        });
      }

      const statsDoc = await db.collection('userStats').doc(userId).get();

      if (!statsDoc.exists) {
        return res.status(404).json({
          success: false,
          error: 'User stats not found'
        });
      }

      const statsData = statsDoc.data();

      res.status(200).json({
        success: true,
        data: statsData
      });

    } catch (error) {
      console.error('Error in getUserStats:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/functions/getUserGameHistory.js`

```javascript
const functions = require('firebase-functions');
const cors = require('cors')({ origin: true });
const { db } = require('../config/firebaseConfig');

/**
 * Cloud Function: getUserGameHistory
 * Fetches user game history with pagination
 * 
 * Request body:
 * {
 *   "userId": "user_123",
 *   "limit": 10,
 *   "offset": 0
 * }
 */
exports.getUserGameHistory = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    try {
      const { userId, limit = 10, offset = 0 } = req.body;

      if (!userId) {
        return res.status(400).json({
          success: false,
          error: 'Missing userId'
        });
      }

      // Query game history
      const query = db.collection('gameHistory')
        .where('userId', '==', userId)
        .orderBy('completedAt', 'desc')
        .limit(limit + 1)
        .offset(offset);

      const snapshot = await query.get();
      const games = [];

      snapshot.forEach(doc => {
        games.push({
          gameId: doc.id,
          ...doc.data()
        });
      });

      res.status(200).json({
        success: true,
        data: {
          userId,
          games,
          count: games.length,
          limit,
          offset
        }
      });

    } catch (error) {
      console.error('Error in getUserGameHistory:', error);
      res.status(500).json({
        success: false,
        error: error.message
      });
    }
  });
});
```

---

### File: `functions/index.js`

```javascript
// Cloud Functions Index
const { generateQuestion } = require('./functions/generateQuestion');
const { saveGameResult } = require('./functions/saveGameResult');
const { calculateDifficulty } = require('./functions/calculateDifficulty');
const { getUserProfile } = require('./functions/getUserProfile');
const { getUserStats } = require('./functions/getUserStats');
const { getUserGameHistory } = require('./functions/getUserGameHistory');

module.exports = {
  generateQuestion,
  saveGameResult,
  calculateDifficulty,
  getUserProfile,
  getUserStats,
  getUserGameHistory
};
```

---

### File: `functions/package.json`

```json
{
  "name": "nexira-functions",
  "description": "Cloud Functions for Nexira Quiz Game",
  "version": "1.0.0",
  "private": true,
  "engines": {
    "node": "18"
  },
  "main": "index.js",
  "scripts": {
    "start": "firebase emulators:start --only functions",
    "deploy": "firebase deploy --only functions",
    "logs": "firebase functions:log",
    "test": "jest"
  },
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0",
    "google-generative-ai": "^0.3.0",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "firebase-functions-test": "^3.1.0"
  }
}
```

---

### File: `functions/.env.example`

```
GEMINI_API_KEY=your_gemini_api_key_here
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_STORAGE_BUCKET=your_storage_bucket
```

---

## 🚀 Deployment Instructions

### 1. Deploy to Firebase
```bash
cd functions
firebase deploy --only functions
```

### 2. Test Locally
```bash
cd functions
npm start
```

### 3. View Function Logs
```bash
firebase functions:log
```

---

## 📊 Function Endpoints (After Deployment)

| Function | Method | Endpoint |
|----------|--------|----------|
| generateQuestion | POST | `https://region-projectId.cloudfunctions.net/generateQuestion` |
| saveGameResult | POST | `https://region-projectId.cloudfunctions.net/saveGameResult` |
| calculateDifficulty | POST | `https://region-projectId.cloudfunctions.net/calculateDifficulty` |
| getUserProfile | POST | `https://region-projectId.cloudfunctions.net/getUserProfile` |
| getUserStats | POST | `https://region-projectId.cloudfunctions.net/getUserStats` |
| getUserGameHistory | POST | `https://region-projectId.cloudfunctions.net/getUserGameHistory` |

---

## ✅ MVP Constraints

✔️ **Included:**
- AI question generation
- Scoring calculation
- Dynamic difficulty
- User profile management
- Game history tracking
- User statistics

❌ **NOT Included (Version 2+):**
- 1vs1 functions
- Matchmaking
- Leaderboard calculation
- XP/Level system
- Real-time multiplayer
- Anti-cheat system

---

## 📝 Notes

1. **Gemini API Key:** Must be set in Firebase Environment Variables
2. **Error Handling:** All functions include proper error handling
3. **CORS:** Enabled for Flutter frontend communication
4. **Scalability:** Functions auto-scale based on demand
5. **Cost:** Free tier includes ~2M invocations/month

---

## 🔄 Next Steps

1. ✅ Firestore Schema (DONE)
2. ✅ Cloud Functions (THIS FILE)
3. ⏳ Flutter Project Structure (Next)
4. ⏳ Implementation & Testing
