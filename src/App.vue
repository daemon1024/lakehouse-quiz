<template>
  <div class="app">
    <!-- Team Onboarding Screen -->
    <div v-if="!gameStarted" class="onboarding-screen">
      <h1 class="onboarding-title">Team Onboarding</h1>

      <div class="teams-section">
        <div class="add-team-form">
          <input
            v-model="newTeamName"
            type="text"
            placeholder="Enter team name"
            class="team-input"
            @keyup.enter="addTeam"
          />
          <button class="add-team-btn" @click="addTeam" :disabled="!newTeamName.trim()">
            Add a team
          </button>
        </div>

        <div v-if="teams.length > 0" class="teams-list">
          <h3 class="teams-list-title">Teams Added:</h3>
          <div class="team-item" v-for="(team, index) in teams" :key="index">
            <span class="team-name">{{ team }}</span>
            <button class="remove-team-btn" @click="removeTeam(index)">×</button>
          </div>
        </div>
      </div>

      <button
        v-if="teams.length > 0"
        class="start-btn"
        @click="startGame"
      >
        Start
      </button>
    </div>

    <!-- Jeopardy Game Screen -->
    <div v-else class="game-screen">
      <h1 class="title">Lakehouse Quiz</h1>

      <!-- Scoreboard -->
      <div v-if="teams.length" class="scoreboard">
        <div
          class="score-card"
          v-for="(team, index) in teams"
          :key="`score-${index}`"
        >
          <span class="score-team-name">{{ team }}</span>
          <span class="score-team-points">{{ scores[team] || 0 }}</span>
        </div>
      </div>

      <div class="grid-container">
      <!-- Category Row -->
      <div
        v-for="(category, colIndex) in categories"
        :key="`cat-${colIndex}`"
        class="grid-cell category-cell"
      >
        {{ category.name }}
      </div>

      <!-- Question Cells (5 rows × 5 columns) -->
      <template v-for="(points, rowIndex) in pointValues" :key="`row-${rowIndex}`">
        <div
          v-for="(category, colIndex) in categories"
          :key="`q-${rowIndex}-${colIndex}`"
          :class="['grid-cell', 'question-cell', { 'question-answered': isQuestionAnswered(category, points) }]"
          @click="openPopup(category, points)"
        >
          {{ points }}<span v-if="points === 20" class="shot-emoji"> 🥃</span><span v-if="category.name === 'Boll-emoji'" class="dance-emoji"> 💃</span>
        </div>
      </template>
    </div>

    <!-- Popup Modal -->
    <div v-if="showPopup" class="popup-overlay" @click="closePopup">
      <div class="popup-content" @click.stop>
        <button class="close-btn" @click="closePopup">&times;</button>

        <!-- Slide 1: Safety Slide (Category + Points) -->
        <div v-if="currentSlide === 1" class="modal-slide">
          <div class="safety-slide">
            <h2 class="popup-category">{{ selectedCategory?.name }}</h2>
            <div class="popup-points">{{ selectedPoints }} Points</div>
          </div>
          <div class="nav-buttons">
            <button class="nav-btn next-btn" @click="nextSlide">→</button>
          </div>
        </div>

        <!-- Slide 2: Question Slide -->
        <div v-if="currentSlide === 2" class="modal-slide">
          <div class="question-slide">
            <h3 class="slide-label">Question</h3>
            <div class="question-text" v-html="getCurrentQuestion()"></div>
          </div>
          <div class="nav-buttons">
            <button class="nav-btn prev-btn" @click="prevSlide">←</button>
            <button class="nav-btn next-btn" @click="nextSlide">→</button>
          </div>
        </div>

        <!-- Slide 3: Blank Safety Slide -->
        <div v-if="currentSlide === 3" class="modal-slide">
          <div class="blank-slide">
            <!-- Completely blank -->
          </div>
          <div class="nav-buttons">
            <button class="nav-btn prev-btn" @click="prevSlide">←</button>
            <button class="nav-btn next-btn" @click="nextSlide">→</button>
          </div>
        </div>

        <!-- Slide 4: Answer Slide -->
        <div v-if="currentSlide === 4" class="modal-slide">
          <div class="answer-slide">
            <h3 class="slide-label">Answer</h3>
            <div class="answer-content">
              <div class="answer-text-wrapper">
                <div class="answer-text">{{ getCurrentAnswer() }}</div>
              </div>
              <div class="answer-image-wrapper" v-if="getAnswerImage()">
                <img
                  :src="getAnswerImage()"
                  :alt="getCurrentAnswer()"
                  class="answer-image"
                />
              </div>
            </div>
          </div>
          <div class="nav-buttons">
            <button class="nav-btn prev-btn" @click="prevSlide">←</button>
            <button class="nav-btn scoring-btn" @click="openScoring">Scoring</button>
            <button class="nav-btn done-btn" @click="markQuestionDone">Done</button>
            <button class="nav-btn close-btn-bottom" @click="closePopup">Close</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Scoring Modal -->
    <div v-if="showScoring" class="popup-overlay" @click="closeScoring">
      <div class="scoring-content" @click.stop>
        <h2 class="scoring-title">Award Points</h2>

        <div class="scoring-form" v-if="teams.length">
          <div class="form-group">
            <label for="team-select" class="form-label">Select Team:</label>
            <select
              id="team-select"
              v-model="selectedTeamForScoring"
              class="team-select"
            >
              <option value="">-- Select a team --</option>
              <option v-for="(team, index) in teams" :key="`team-${index}`" :value="team">
                {{ team }}
              </option>
            </select>
          </div>

          <div class="form-group">
            <label for="points-input" class="form-label">Points:</label>
            <input
              id="points-input"
              type="number"
              v-model.number="customPoints"
              class="points-input"
              placeholder="Enter points"
            />
            <p class="points-hint">Default: {{ selectedPoints || 0 }} points (can be negative for incorrect answers). Enter custom points below.</p>
          </div>

          <div class="scoring-actions">
            <button
              class="award-btn"
              @click="awardPoints"
              :disabled="!selectedTeamForScoring || customPoints === null || customPoints === ''"
            >
              Award Points
            </button>
            <button class="close-scoring-btn" @click="closeScoring">
              Close
            </button>
          </div>
        </div>

        <p v-else class="no-teams-text">
          No teams available. Add teams on the onboarding screen.
        </p>
      </div>
    </div>

    <!-- Scoring and Reset Buttons -->
    <button class="scoring-quick-btn" @click="openQuickScoring">
      Quick Scoring
    </button>
    <button class="reset-btn" @click="resetTeamsAndScores">
      Reset Teams & Scores
    </button>
    </div>
  </div>
</template>

<script>
// No answer images yet. Drop images into ./answer_images/ and add the
// imports + answerImages map entries below to show them on the answer slide.

const MAY_27_CALENDAR = '<svg viewBox="0 0 64 64" style="display:inline-block;width:1.3em;height:1.3em;vertical-align:-0.25em;"><rect x="4" y="8" width="56" height="52" rx="6" fill="#FAFAFA"/><rect x="4" y="8" width="56" height="14" rx="6" fill="#E64B3D"/><rect x="4" y="14" width="56" height="8" fill="#E64B3D"/><text x="32" y="20" font-family="system-ui,sans-serif" font-size="8" font-weight="700" fill="white" text-anchor="middle">MAY</text><text x="32" y="52" font-family="system-ui,sans-serif" font-size="26" font-weight="800" fill="#333" text-anchor="middle">27</text><circle cx="20" cy="6" r="2" fill="#999"/><circle cx="44" cy="6" r="2" fill="#999"/></svg>'

export default {
  name: 'App',
  data() {
    return {
      gameStarted: false,
      teams: [],
      newTeamName: '',
      scores: {},
      answeredQuestions: {},
      showScoring: false,
      selectedTeamForScoring: '',
      customPoints: null,
      storageKey: 'birthdayQuizState',
      showPopup: false,
      currentSlide: 1,
      selectedCategory: null,
      selectedPoints: null,
      pointValues: [10, 20, 30, 40, 50],
      // Keys are lowercase answer text; values are imported image variables.
      answerImages: {},
      categories: [
        {
          name: 'Fifa Fufa',
          questions: {
            10: { question: 'Napoli\'s retired number 10 jersey honours this Argentine.', answer: 'Diego Maradona' },
            20: { question: 'Holds the record for the second-most Ballon d\'Or wins with 5, behind only Lionel Messi\'s 8.', answer: 'Cristiano Ronaldo' },
            30: { question: 'Lionel Messi\'s MLS club post-PSG; he won the 2023 Leagues Cup with them in his debut tournament.', answer: 'Inter Miami CF' },
            40: { question: 'The Spanish midfielder whose 116th-minute goal won Spain the 2010 World Cup final.', answer: 'Andrés Iniesta' },
            50: { question: 'Youngest player to ever score in a FIFA World Cup final — did it at age 17, scoring twice in the match.', answer: 'Pelé' }
          }
        },
        {
          name: 'Plot Twist',
          questions: {
            10: { question: 'Launched in 2005 as a video dating site, before becoming a video sharing platform.', answer: 'YouTube' },
            20: { question: 'Began as Odeo, a podcast platform that became obsolete when Apple launched iTunes podcasts.', answer: 'Twitter / X' },
            30: { question: 'Started as Glitch, a multiplayer browser game that flopped, before becoming this workplace messaging tool.', answer: 'Slack' },
            40: { question: 'Founded in 2004 as Snowdevil, a snowboarding gear shop, before pivoting to e-commerce tooling.', answer: 'Shopify' },
            50: { question: 'Founded in 1889 as a hanafuda playing-card maker in Kyoto.', answer: 'Nintendo' }
          }
        },
        {
          name: 'On The Rocks',
          questions: {
            10: { question: 'This 007\'s signature drink in nearly every film since 1962 — "shaken, not stirred."', answer: 'James Bond / Vodka Martini' },
            20: { question: 'Buried in 1998 with a bottle of Jack Daniel\'s, a Zippo, a pack of Camels, and ten dimes.', answer: 'Frank Sinatra' },
            30: { question: 'The Sex and the City protagonist who turned this pink cocktail into the 2000s\' most-ordered drink.', answer: 'Carrie Bradshaw / Cosmopolitan' },
            40: { question: 'In 1998 cult film "The Big Lebowski," his character The Dude drinks only White Russians.', answer: 'Jeff Bridges / White Russian' },
            50: { question: 'Bought a minority stake in Aviation American Gin in 2018, sold it to Diageo for up to $610M in 2020.', answer: 'Ryan Reynolds / Aviation Gin' }
          }
        },
        {
          name: 'Bhukkad Kukkad',
          questions: {
            10: { question: 'PUMA', answer: 'Upma' },
            20: { question: 'AIRY BIN', answer: 'Biryani' },
            30: { question: 'OFFER FELT ICE', answer: 'Filter Coffee' },
            40: { question: 'SAMOA SALAD', answer: 'Masala Dosa' },
            50: { question: 'GAZE ZAP HAIR TRIM', answer: 'Margherita Pizza' }
          }
        },
        {
          name: 'Boll-emoji',
          questions: {
            10: { question: '💨🍻', answer: 'Hookah Bar (Khiladi 786, 2012)' },
            20: { question: '🙅✌️ 🙅✌️', answer: 'Naatu Naatu (RRR, 2022)' },
            30: { question: '😮 🍶🍶', answer: 'O Saki Saki (Batla House, 2019)' },
            40: { question: '❓👂💍', answer: 'What Jhumka? (Rocky Aur Rani Kii Prem Kahaani, 2023)' },
            50: { question: MAY_27_CALENDAR + ' 🔑 🌃', answer: 'Aaj Ki Raat (Don, 2006)' }
          }
        }
      ]
    }
  },
  created() {
    this.loadState()
  },
  mounted() {
    this.handleKeydown = this.handleKeydown.bind(this)
  },
  beforeUnmount() {
    document.removeEventListener('keydown', this.handleKeydown)
  },
  watch: {
    showPopup(newVal) {
      if (newVal) {
        document.addEventListener('keydown', this.handleKeydown)
      } else {
        document.removeEventListener('keydown', this.handleKeydown)
      }
    }
  },
  methods: {
    handleKeydown(event) {
      if (!this.showPopup) return

      if (event.key === 'ArrowLeft') {
        event.preventDefault()
        this.prevSlide()
      } else if (event.key === 'ArrowRight') {
        event.preventDefault()
        this.nextSlide()
      }
    },
    openPopup(category, points) {
      this.selectedCategory = category
      this.selectedPoints = points
      this.currentSlide = 1
      this.showPopup = true
    },
    closePopup() {
      this.showPopup = false
      this.currentSlide = 1
      this.selectedCategory = null
      this.selectedPoints = null
    },
    nextSlide() {
      if (this.currentSlide < 4) {
        this.currentSlide++
      }
    },
    prevSlide() {
      if (this.currentSlide > 1) {
        this.currentSlide--
      }
    },
    getCurrentQuestion() {
      if (this.selectedCategory && this.selectedPoints) {
        const questionData = this.selectedCategory.questions[this.selectedPoints]
        return questionData ? questionData.question : 'Question not available'
      }
      return ''
    },
    getCurrentAnswer() {
      if (this.selectedCategory && this.selectedPoints) {
        const questionData = this.selectedCategory.questions[this.selectedPoints]
        return questionData ? questionData.answer : 'Answer not available'
      }
      return ''
    },
    getAnswerImage() {
      const answer = this.getCurrentAnswer()
      if (!answer) return null

      const answerLower = answer.toLowerCase().trim()

      // First try exact match
      if (this.answerImages[answerLower]) {
        return this.answerImages[answerLower]
      }

      // Then try partial match - check if answer contains any key
      for (const [key, image] of Object.entries(this.answerImages)) {
        if (answerLower.includes(key) || key.includes(answerLower)) {
          return image
        }
      }

      return null
    },
    isQuestionAnswered(category, points) {
      const name = category && category.name ? category.name : category
      return !!(this.answeredQuestions[name] && this.answeredQuestions[name][points])
    },
    markQuestionDone() {
      if (this.selectedCategory && this.selectedPoints != null) {
        const name = this.selectedCategory.name
        if (!this.answeredQuestions[name]) {
          this.answeredQuestions[name] = {}
        }
        this.answeredQuestions[name][this.selectedPoints] = true
        this.saveState()
      }
      // Close question modal after marking as done
      this.closePopup()
    },
    openScoring() {
      if (!this.selectedPoints) return
      this.customPoints = this.selectedPoints
      this.selectedTeamForScoring = ''
      this.showScoring = true
    },
    openQuickScoring() {
      this.customPoints = 0
      this.selectedTeamForScoring = ''
      this.showScoring = true
    },
    closeScoring() {
      this.showScoring = false
      this.selectedTeamForScoring = ''
      this.customPoints = null
    },
    awardPoints() {
      const team = this.selectedTeamForScoring
      const pts = this.customPoints

      if (!team || pts === null || pts === '') return

      const current = this.scores[team] || 0
      this.scores[team] = current + pts
      this.saveState()

      // Close scoring modal after awarding points
      this.closeScoring()
    },
    addTeam() {
      const trimmedName = this.newTeamName.trim()
      if (trimmedName && !this.teams.includes(trimmedName)) {
        this.teams.push(trimmedName)
        if (this.scores[trimmedName] == null) {
          this.scores[trimmedName] = 0
        }
        this.newTeamName = ''
        this.saveState()
      }
    },
    removeTeam(index) {
      const [removed] = this.teams.splice(index, 1)
      if (removed && this.scores[removed] != null) {
        delete this.scores[removed]
      }
      this.saveState()
    },
    startGame() {
      if (this.teams.length > 0) {
        this.gameStarted = true
        this.saveState()
      }
    },
    resetTeamsAndScores() {
      if (confirm('Are you sure you want to reset all teams and scores? This cannot be undone.')) {
        this.teams = []
        this.scores = {}
        this.answeredQuestions = {}
        this.gameStarted = false
        this.saveState()
      }
    },
    saveState() {
      if (typeof window === 'undefined' || !window.localStorage) return
      const state = {
        gameStarted: this.gameStarted,
        teams: this.teams,
        scores: this.scores,
        answeredQuestions: this.answeredQuestions
      }
      window.localStorage.setItem(this.storageKey, JSON.stringify(state))
    },
    loadState() {
      if (typeof window === 'undefined' || !window.localStorage) return
      const raw = window.localStorage.getItem(this.storageKey)
      if (!raw) return
      try {
        const state = JSON.parse(raw)
        if (Array.isArray(state.teams)) {
          this.teams = state.teams
        }
        if (state.scores && typeof state.scores === 'object') {
          this.scores = state.scores
        }
        if (state.answeredQuestions && typeof state.answeredQuestions === 'object') {
          this.answeredQuestions = state.answeredQuestions
        }
        if (state.gameStarted) {
          this.gameStarted = true
        }
      } catch (e) {
        // ignore parse errors
      }
    }
  }
}
</script>

<style scoped>
.app {
  padding: 20px;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  overflow: hidden;
}

.title {
  text-align: center;
  color: #FBE9C4; /* warm cream like the poster's QUIZ NIGHT */
  font-family: 'Norwester', 'Arial Narrow', sans-serif;
  font-size: 3rem;
  font-weight: bold;
  letter-spacing: 3px;
  text-transform: uppercase;
  margin-bottom: 20px;
  text-shadow:
    0 0 8px rgba(251, 191, 36, 0.6),
    0 0 24px rgba(225, 29, 116, 0.55),
    0 0 48px rgba(249, 115, 22, 0.35);
}

/* Onboarding Screen Styles */
.onboarding-screen {
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  padding: 40px 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
}

.onboarding-title {
  text-align: center;
  color: #FBE9C4;
  font-family: 'Norwester', 'Arial Narrow', sans-serif;
  font-size: 3.2rem;
  font-weight: bold;
  letter-spacing: 3px;
  text-transform: uppercase;
  margin-bottom: 40px;
  text-shadow:
    0 0 8px rgba(251, 191, 36, 0.6),
    0 0 24px rgba(225, 29, 116, 0.55),
    0 0 48px rgba(249, 115, 22, 0.35);
}

.teams-section {
  width: 100%;
  margin-bottom: 30px;
}

.add-team-form {
  display: flex;
  gap: 10px;
  margin-bottom: 30px;
}

.team-input {
  flex: 1;
  padding: 12px 16px;
  font-size: 1rem;
  border: 2px solid #B45309;
  border-radius: 6px;
  outline: none;
  transition: border-color 0.2s;
}

.team-input:focus {
  border-color: #F97316;
}

.add-team-btn {
  padding: 12px 24px;
  font-size: 1rem;
  font-weight: bold;
  background: #B45309;
  color: white;
  border: 2px solid #B45309;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}

.add-team-btn:hover:not(:disabled) {
  background: white;
  color: #B45309;
}

.add-team-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.teams-list {
  background: white;
  border: 2px solid #F97316;
  border-radius: 8px;
  padding: 20px;
  max-height: 300px;
  overflow-y: auto;
}

.teams-list-title {
  font-size: 1.2rem;
  color: #333;
  margin-bottom: 15px;
  font-weight: bold;
}

.team-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  margin-bottom: 8px;
  background: #f5f5f5;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.team-name {
  font-size: 1rem;
  color: #333;
  font-weight: 500;
}

.remove-team-btn {
  background: #E11D74;
  color: white;
  border: none;
  border-radius: 50%;
  width: 24px;
  height: 24px;
  font-size: 1.2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
  line-height: 1;
}

.remove-team-btn:hover {
  background: #9F1239;
}

.start-btn {
  padding: 16px 48px;
  font-size: 1.3rem;
  font-weight: bold;
  background: #F97316;
  color: white;
  border: 3px solid #E11D74;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

.start-btn:hover {
  background: #C2410C;
  transform: translateY(-2px);
  box-shadow: 0 6px 14px rgba(0, 0, 0, 0.4);
}

/* Wrapper for the playing screen — flex column so .grid-container can flex: 1 against it */
.game-screen {
  flex: 1;
  min-height: 0;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  grid-template-rows: repeat(6, 1fr);
  gap: 10px;
  width: 80vw;
  max-width: 1200px;
  /* Take up remaining vertical space so the 50-row never crops. */
  flex: 1;
  min-height: 0;
  padding-bottom: 12px;
}

.grid-cell {
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  font-weight: 800;
  font-family: 'Norwester', 'Arial Narrow', sans-serif;
  letter-spacing: 1px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.35);
  text-align: center;
}

.category-cell {
  background: linear-gradient(180deg, #E11D74 0%, #F97316 100%);
  color: #FBE9C4; /* warm cream */
  font-size: 1.4rem;
  padding: 6px 4px;
  text-shadow: 0 0 8px rgba(251, 191, 36, 0.5), 0 1px 2px rgba(0, 0, 0, 0.4);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.35), 0 0 20px rgba(225, 29, 116, 0.4);
}

.question-cell {
  /* Translucent dark fill — lets the wine-purple body show through, no white. */
  background: rgba(15, 5, 20, 0.55);
  color: #FBE9C4;
  font-size: 2.6rem;
  cursor: pointer;
  border: 2px solid rgba(249, 115, 22, 0.55);
  /* Subtle baseline neon glow so it always reads as "lit" */
  text-shadow:
    0 0 6px rgba(249, 115, 22, 0.45),
    0 0 16px rgba(225, 29, 116, 0.25);
  box-shadow:
    0 0 8px rgba(249, 115, 22, 0.15),
    inset 0 0 12px rgba(225, 29, 116, 0.08);
  transition: transform 0.15s ease-out, box-shadow 0.2s ease-out,
              background 0.2s, border-color 0.2s, text-shadow 0.2s, color 0.2s;
}

.question-cell:hover {
  transform: translateY(-3px);
  background: rgba(58, 15, 38, 0.65);
  border-color: #F97316;
  color: #FFF3D0;
  /* Strong neon glow on the number itself */
  text-shadow:
    0 0 10px rgba(251, 191, 36, 0.85),
    0 0 24px rgba(249, 115, 22, 0.75),
    0 0 48px rgba(225, 29, 116, 0.55);
  /* Multilayer outer + inner glow on the box */
  box-shadow:
    0 0 16px rgba(225, 29, 116, 0.55),
    0 0 32px rgba(249, 115, 22, 0.40),
    0 6px 18px rgba(0, 0, 0, 0.55),
    inset 0 0 16px rgba(249, 115, 22, 0.18);
}

.question-answered {
  opacity: 0.3;
  filter: saturate(0.4);
}

.scoreboard {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 12px;
  margin-bottom: 16px;
}

.score-card {
  min-width: 140px;
  padding: 8px 16px;
  border-radius: 999px;
  border: 2px solid #FFFFFF;
  background: linear-gradient(135deg, #B45309 0%, #F97316 100%);
  color: #FFFFFF;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.score-team-name {
  margin-right: 8px;
}

.score-team-points {
  font-size: 1.2rem;
}

.popup-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.popup-content {
  background: white;
  border-radius: 8px;
  padding: 0;
  width: 75vw;
  height: 75vh;
  position: relative;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5);
  border: 4px solid #E11D74;
  border-top: 4px solid #F97316;
  border-bottom: 4px solid #B45309;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.close-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  background: white;
  border: 2px solid #E11D74;
  font-size: 1.5rem;
  cursor: pointer;
  color: #E11D74;
  width: 35px;
  height: 35px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  transition: background 0.2s, color 0.2s;
  z-index: 10;
}

.close-btn:hover {
  background: #E11D74;
  color: white;
}

.modal-slide {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 40px;
  justify-content: flex-start;
  align-items: center;
  min-height: 0;
  overflow: hidden;
}

.safety-slide {
  text-align: center;
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.popup-category {
  font-size: 3rem;
  color: #E11D74;
  margin-bottom: 30px;
  font-weight: bold;
}

.popup-points {
  font-size: 4rem;
  color: #F97316;
  font-weight: bold;
}

.question-slide {
  text-align: center;
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: 100%;
}

.answer-slide {
  text-align: center;
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  width: 100%;
  min-height: 0;
  max-height: 100%;
  overflow: hidden;
}

.slide-label {
  font-size: 1.5rem;
  color: #666;
  margin-bottom: 20px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 2px;
  flex-shrink: 0;
}

.question-text {
  font-size: 2.2rem;
  color: #333;
  font-weight: 500;
  line-height: 1.6;
  max-width: 90%;
  margin-bottom: 20px;
}

.answer-content {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  gap: 30px;
  padding: 0 40px;
  flex: 1;
  min-height: 0;
  max-height: 100%;
  overflow: hidden;
}

.answer-text-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
}

.answer-text {
  font-size: 2.2rem;
  color: #333;
  font-weight: 500;
  line-height: 1.6;
}

.answer-image-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  max-height: calc(75vh - 250px);
  min-height: 0;
  max-width: 50%;
  overflow: hidden;
}

.answer-image {
  max-width: 100%;
  max-height: calc(75vh - 250px);
  width: auto;
  height: auto;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  object-fit: contain;
  display: block;
}

.blank-slide {
  flex: 1;
  width: 100%;
}

.nav-buttons {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  padding: 12px 0 0 0;
  border-top: 2px solid #e0e0e0;
  margin-top: 12px;
  flex-shrink: 0;
}

.nav-btn {
  background: #B45309;
  color: white;
  border: 2px solid #B45309;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 12px 24px;
  border-radius: 6px;
  transition: background 0.2s, color 0.2s;
  font-weight: bold;
  min-width: 60px;
}

.nav-btn:hover {
  background: white;
  color: #B45309;
}

.prev-btn {
  margin-right: auto;
}

.next-btn {
  margin-left: auto;
}

.scoring-btn {
  background: #F97316 !important;
  border-color: #F97316 !important;
}

.scoring-btn:hover {
  background: white !important;
  color: #F97316 !important;
}

.done-btn {
  background: #E11D74 !important;
  border-color: #E11D74 !important;
}

.done-btn:hover {
  background: white !important;
  color: #E11D74 !important;
}

.close-btn-bottom {
  background: #E11D74;
  color: white;
  border: 2px solid #E11D74;
  font-size: 1rem;
  cursor: pointer;
  padding: 12px 24px;
  border-radius: 6px;
  transition: background 0.2s, color 0.2s;
  font-weight: bold;
  margin-left: auto;
}

.close-btn-bottom:hover {
  background: white;
  color: #E11D74;
}

.scoring-content {
  background: white;
  border-radius: 8px;
  padding: 30px 30px 24px;
  width: 40vw;
  max-width: 600px;
  min-width: 320px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5);
  border: 3px solid #F97316;
  border-top: 3px solid #E11D74;
  border-bottom: 3px solid #B45309;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.scoring-title {
  font-size: 1.8rem;
  margin-bottom: 24px;
  color: #B45309;
  text-align: center;
}

.scoring-form {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.form-label {
  font-size: 1rem;
  font-weight: 600;
  color: #333;
}

.team-select {
  padding: 12px 16px;
  font-size: 1rem;
  border: 2px solid #B45309;
  border-radius: 6px;
  background: white;
  color: #333;
  cursor: pointer;
  outline: none;
  transition: border-color 0.2s;
}

.team-select:focus {
  border-color: #B45309;
}

.points-input {
  padding: 12px 16px;
  font-size: 1rem;
  border: 2px solid #F97316;
  border-radius: 6px;
  outline: none;
  transition: border-color 0.2s;
}

.points-input:focus {
  border-color: #B45309;
}

.points-hint {
  font-size: 0.85rem;
  color: #666;
  margin: 0;
  font-style: italic;
}

.scoring-actions {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin-top: 10px;
}

.award-btn {
  padding: 12px 24px;
  font-size: 1rem;
  font-weight: 600;
  border-radius: 6px;
  border: 2px solid #F97316;
  background: #F97316;
  color: white;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}

.award-btn:hover:not(:disabled) {
  background: white;
  color: #F97316;
}

.award-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.no-teams-text {
  font-size: 0.95rem;
  color: #777;
  margin-bottom: 20px;
  text-align: center;
}

.close-scoring-btn {
  padding: 12px 24px;
  font-size: 1rem;
  font-weight: 600;
  border-radius: 6px;
  border: 2px solid #E11D74;
  background: #E11D74;
  color: white;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}

.close-scoring-btn:hover {
  background: white;
  color: #E11D74;
}

.scoring-quick-btn {
  position: fixed;
  bottom: 80px;
  left: 20px;
  padding: 10px 20px;
  font-size: 0.9rem;
  font-weight: 600;
  border-radius: 6px;
  border: 2px solid #F97316;
  background: #F97316;
  color: white;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
  z-index: 100;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

.scoring-quick-btn:hover {
  background: white;
  color: #F97316;
}

.reset-btn {
  position: fixed;
  bottom: 20px;
  left: 20px;
  padding: 10px 20px;
  font-size: 0.9rem;
  font-weight: 600;
  border-radius: 6px;
  border: 2px solid #E11D74;
  background: #E11D74;
  color: white;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
  z-index: 100;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

.reset-btn:hover {
  background: white;
  color: #E11D74;
}
</style>
