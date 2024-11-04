In activity 1, the current url used is: https://opentdb.com/api.php?amount=5&category=18&difficulty=medium&type=boolean. Based on this API, other requirements are implemented except R1.a.

Due to 429 error code, I couldn't test the api based on different difficulty for easy, medium, hard.
Method fetchCategories which has the new API is :
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
