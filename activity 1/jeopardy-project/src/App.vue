<template>
  <div id="app">
    <h1>Jeopardy Game</h1>
    <div class="scores">
      <div v-for="(player, index) in players" :key="index">
        Player {{ index + 1 }}: ${{ player.score }}
      </div>
    </div>

    <!-- Game Board -->
    <table v-if="categories.length > 0" class="game-board">
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
    <p v-else>Loading game board...</p>

    <!-- Display the question and answer options if one is selected -->
    <div v-if="currentQuestion">
      <p>Category: {{ currentQuestion.category.name }} - ${{ currentQuestion.value }}</p>
      <p>{{ currentQuestion.question }}</p>
      <button @click="answerQuestion('True')">True</button>
      <button @click="answerQuestion('False')">False</button>
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
      players: [
        { score: 0 },
        { score: 0 },
        { score: 0 },
      ],
      currentPlayer: 0,
      categories: [],
      selectedCells: Array.from({ length: 4 }, () => Array(3).fill(null)),
      currentQuestion: null,
      playerAnswer: null,
      message: "",
      isGameOver: false,
      winnerMessage: "",
    };
  },
  methods: {
    async fetchCategories() {
      // Fetch available categories
      const response = await fetch("https://opentdb.com/api_category.php");
      const data = await response.json();

      // Randomly select 4 unique categories
      const allCategories = data.trivia_categories;
      const selectedCategories = [];
      while (selectedCategories.length < 4) {
        const randomCategory = allCategories[Math.floor(Math.random() * allCategories.length)];
        if (!selectedCategories.some(cat => cat.id === randomCategory.id)) {
          selectedCategories.push(randomCategory);
        }
      }

      // Fetch questions of varying difficulties for each selected category
      this.categories = [];
      for (const category of selectedCategories) {
        const questions = await this.fetchCategoryQuestions(category.id);
        if (questions) {
          this.categories.push({ name: category.name, questions });
        }
      }
    },

    async fetchCategoryQuestions(categoryId) {
      const questions = [];
      for (const [index, difficulty] of ['easy', 'medium', 'hard'].entries()) {
        const value = (index + 1) * 100;
        const apiUrl = `https://opentdb.com/api.php?amount=1&category=${categoryId}&difficulty=${difficulty}&type=boolean`;
        const response = await fetch(apiUrl);
        const data = await response.json();

        if (data.response_code === 0 && data.results.length > 0) {
          questions.push({
            question: data.results[0].question,
            correctAnswer: data.results[0].correct_answer,
            value,
          });
        } else {
          console.error(`No ${difficulty} question found for category ${categoryId}`);
          return null; // If any difficulty is missing, retry with a different category
        }
      }
      return questions;
    },

    selectQuestion(category, value, colIndex) {
      const rowIndex = value / 100 - 1;
      if (this.selectedCells[colIndex][rowIndex]) return; // Prevent re-selection

      const question = category.questions.find(q => q.value === value);
      this.currentQuestion = { ...question, value, category };
      this.message = `Player ${this.currentPlayer + 1} selected ${category.name} for $${value}`;
      this.playerAnswer = null; // Reset player answer display
    },

    answerQuestion(answer) {
      if (!this.currentQuestion) return;

      const isCorrect = this.currentQuestion.correctAnswer === answer;
      const rowIndex = this.currentQuestion.value / 100 - 1;
      const colIndex = this.categories.indexOf(this.currentQuestion.category);

      this.playerAnswer = answer;
      if (isCorrect) {
        this.message = "Correct!";
        this.players[this.currentPlayer].score += this.currentQuestion.value;
        this.selectedCells[colIndex][rowIndex] = `P${this.currentPlayer + 1} correct`;
      } else {
        this.message = "Incorrect!";
        this.players[this.currentPlayer].score -= this.currentQuestion.value;
        this.selectedCells[colIndex][rowIndex] = `P${this.currentPlayer + 1} incorrect`;
        this.currentPlayer = (this.currentPlayer + 1) % this.players.length;
      }

      this.checkGameOver();
      this.currentQuestion = null;
    },

    getCellClass(colIndex, rowIndex) {
      const cellStatus = this.selectedCells[colIndex][rowIndex];
      if (cellStatus && cellStatus.includes("correct")) return "correct";
      if (cellStatus && cellStatus.includes("incorrect")) return "incorrect";
      return "";
    },

    getCellStyle(colIndex, rowIndex) {
      const cellStatus = this.selectedCells[colIndex][rowIndex];
      if (cellStatus && cellStatus.includes("incorrect")) {
        return { color: "red" };
      }
      return {};
    },

    checkGameOver() {
      const allQuestionsAnswered = this.selectedCells.flat().every(cell => cell !== null);
      if (allQuestionsAnswered) {
        this.isGameOver = true;
        const highestScore = Math.max(...this.players.map(player => player.score));
        const winnerIndex = this.players.findIndex(player => player.score === highestScore) + 1;
        this.winnerMessage = `Player ${winnerIndex} has won the game!`;
      }
    },
  },

  async mounted() {
    await this.fetchCategories();
  },
};
</script>

<style scoped>
#app {
  text-align: center;
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
  color: red !important;
  font-weight: bold;
}
.winner-message {
  font-size: 1.2em;
  color: blue;
  font-weight: bold;
  margin-top: 20px;
}
</style>
