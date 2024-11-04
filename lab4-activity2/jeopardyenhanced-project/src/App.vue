<template>
  <div id="app">
    <h1>Jeopardy Game</h1>
    
    <!-- Configurable Players and Categories -->
    <div class="config">
      <label>
        Number of players:
        <input type="number" v-model="numPlayers" min="2" max="4" @change="initializeGame" />
      </label>
      <label>
        Number of categories:
        <input type="number" v-model="numCategories" min="3" max="6" @change="initializeGame" />
      </label>
    </div>

    <!-- Display Scores -->
    <div class="scores">
      <div v-for="(player, index) in players" :key="index">
        Player {{ index + 1 }}: ${{ player.score }}
      </div>
    </div>

    <!-- Game Board -->
    <table v-if="categories.length > 0 && !isFinalJeopardy" class="game-board">
      <thead>
        <tr>
          <th v-for="(category, index) in categories" :key="index">{{ category.name }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(value, rowIndex) in [100, 200, 300]" :key="rowIndex">
          <td
            v-for="(category, colIndex) in categories"
            :key="colIndex"
            @click="selectQuestion(category, value, colIndex)"
            :class="getCellClass(colIndex, rowIndex)"
            :style="getCellStyle(colIndex, rowIndex)"
          >
            {{ selectedCells[colIndex][rowIndex] || `$${value}` }}
          </td>
        </tr>
      </tbody>
    </table>
    <p v-else-if="!isFinalJeopardy">Loading game board...</p>

    <!-- Display the question and answer options if one is selected -->
    <div v-if="currentQuestion">
      <p>Category: {{ currentQuestion.category.name }} - ${{ currentQuestion.value }}</p>
      <p>{{ currentQuestion.question }}</p>
      
      <!-- Double Jeopardy wager input -->
      <div v-if="isDoubleJeopardy">
        <label>
          Double Jeopardy wager:
          <input type="number" v-model="wager" :max="getMaxWager()" />
        </label>
      </div>

      <button @click="answerQuestion('True')">True</button>
      <button @click="answerQuestion('False')">False</button>
    </div>

    <!-- Final Jeopardy Section -->
    <div v-if="isFinalJeopardy">
      <h2>Final Jeopardy</h2>
      <p>Category: {{ finalJeopardyCategory }}</p>
      <div v-for="(player, index) in players" :key="index">
        <p v-if="player.score > 0">Player {{ index + 1 }}: Wager up to ${{ player.score }}</p>
        <input v-if="player.score > 0" type="number" v-model="player.finalWager" :max="player.score" />
      </div>
      <button @click="startFinalJeopardyQuestion">Start Final Jeopardy Question</button>
    </div>

    <!-- Display Player's Answer -->
    <div v-if="playerAnswer">
      <p>Player {{ currentPlayer + 1 }} answered: {{ playerAnswer }} - {{ message }}</p>
    </div>

    <!-- Winner Message -->
    <div v-if="isGameOver" class="winner-message">
      {{ winnerMessage }}
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      numPlayers: 3,
      numCategories: 4,
      players: [],
      currentPlayer: 0,
      categories: [],
      selectedCells: [],
      currentQuestion: null,
      playerAnswer: null,
      message: "",
      isDoubleJeopardy: false,
      doubleJeopardyCells: [],
      wager: 0, // Amount wagered in Double Jeopardy
      isFinalJeopardy: false,
      finalJeopardyCategory: "",
      finalJeopardyQuestion: null,
      isGameOver: false,
      winnerMessage: "",
    };
  },
  methods: {
    initializeGame() {
      this.players = Array.from({ length: this.numPlayers }, () => ({ score: 0, finalWager: 0 }));
      this.selectedCells = Array.from({ length: this.numCategories }, () => Array(3).fill(null));
      this.fetchCategories();
      this.assignDoubleJeopardyCells();
    },
    async fetchCategories() {
      // Fetch categories and assign questions per difficulty level for each category
      this.categories = [];
      for (let i = 0; i < this.numCategories; i++) {
        const category = await this.fetchCategoryQuestions(i + 1);
        if (category) this.categories.push(category);
      }
    },
    delay(ms) {
      return new Promise(resolve => setTimeout(resolve, ms));
    },
    async fetchCategoryQuestions(categoryIndex) {
      const questions = [];
      for (let difficulty of ['easy', 'medium', 'hard']) {
        const apiUrl =  `https://opentdb.com/api.php?amount=1&category=18&difficulty=${difficulty}&type=boolean`;
        const response = await fetch(apiUrl);
        const data = await response.json();

        if (data.response_code === 0 && data.results.length > 0) {
          questions.push({
            question: data.results[0].question,
            correctAnswer: data.results[0].correct_answer,
            value: questions.length * 100 + 100,
          });
        }
        // Add a delay of 500ms between requests
        await this.delay(5000);
      }
      return questions.length === 3 ? { name: `Category ${categoryIndex}`, questions } : null;
    },

    async fetchCategories() {
      // Call fetchCategoryQuestions for each category, which uses the delay
      this.categories = [];
      for (let i = 0; i < this.numCategories; i++) {
        const category = await this.fetchCategoryQuestions(i + 1);
        if (category) this.categories.push(category);
      }
    },
    assignDoubleJeopardyCells() {
      this.doubleJeopardyCells = [];
      while (this.doubleJeopardyCells.length < 2) {
        const colIndex = Math.floor(Math.random() * this.numCategories);
        const rowIndex = Math.floor(Math.random() * 3);
        const cell = `${colIndex}-${rowIndex}`;
        if (!this.doubleJeopardyCells.includes(cell)) {
          this.doubleJeopardyCells.push(cell);
        }
      }
    },
    selectQuestion(category, value, colIndex) {
      const rowIndex = value / 100 - 1;
      if (this.selectedCells[colIndex][rowIndex]) return;

      this.isDoubleJeopardy = this.doubleJeopardyCells.includes(`${colIndex}-${rowIndex}`);
      if (this.isDoubleJeopardy) {
        this.wager = Math.min(this.players[this.currentPlayer].score, value);
      } else {
        this.wager = value;
      }

      const question = category.questions.find((q) => q.value === value);
      this.currentQuestion = { ...question, value, category };
      this.message = `Player ${this.currentPlayer + 1} selected ${category.name} for $${value}`;
      this.playerAnswer = null;
    },
    getMaxWager() {
      return Math.max(this.players[this.currentPlayer].score, this.currentQuestion.value);
    },
    answerQuestion(answer) {
      if (!this.currentQuestion) return;

      const isCorrect = this.currentQuestion.correctAnswer === answer;
      const rowIndex = this.currentQuestion.value / 100 - 1;
      const colIndex = this.categories.indexOf(this.currentQuestion.category);

      this.playerAnswer = answer;
      let wagerValue = this.isDoubleJeopardy ? this.wager : this.currentQuestion.value;

      if (isCorrect) {
        this.message = "Correct!";
        this.players[this.currentPlayer].score += wagerValue;
        this.selectedCells[colIndex][rowIndex] = `P${this.currentPlayer + 1} correct`;
      } else {
        this.message = "Incorrect!";
        this.players[this.currentPlayer].score -= wagerValue;
        this.selectedCells[colIndex][rowIndex] = `P${this.currentPlayer + 1} incorrect`;
        this.currentPlayer = (this.currentPlayer + 1) % this.players.length;
      }

      this.checkGameOver();
      this.currentQuestion = null;
      this.isDoubleJeopardy = false;
    },
    checkGameOver() {
      const allQuestionsAnswered = this.selectedCells.flat().every((cell) => cell !== null);
      if (allQuestionsAnswered) {
        this.startFinalJeopardy();
      }
    },
    startFinalJeopardy() {
      this.isFinalJeopardy = true;
      this.finalJeopardyCategory = "Random Final Jeopardy Category";
    },
    startFinalJeopardyQuestion() {
      this.finalJeopardyQuestion = {
        question: "This is a hard Final Jeopardy question.",
        correctAnswer: "Correct Answer",
        options: ["Correct Answer", "Wrong Answer 1", "Wrong Answer 2", "Wrong Answer 3"].sort(() => Math.random() - 0.5),
      };
      this.isGameOver = true;
      this.winnerMessage = "Final Jeopardy completed.";
    },
  },
  async mounted() {
    this.initializeGame();
  },
};
</script>

<style scoped>
#app {
  text-align: center;
}
.config {
  margin-bottom: 1em;
}
.scores {
  display: flex;
  justify-content: space-around;
  margin-bottom: 1em;
}
.game-board {
  margin: 0 auto;
  border-collapse: collapse;
}
.game-board th, .game-board td {
  border: 1px solid #ccc;
  padding: 10px;
  width: 100px;
  text-align: center;
  cursor: pointer;
}
.correct {
  background-color: lightgreen;
  font-weight: bold;
}
.incorrect {
  color: red;
  font-weight: bold;
}
.winner-message {
  font-size: 1.2em;
  color: blue;
  font-weight: bold;
  margin-top: 20px;
}
</style>
