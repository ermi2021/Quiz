<template>
  <v-dialog
    v-model="endofQuiz"
    persistent
    class="rounded-lg flex items-center justify-center"
  >
  <ConfettiExplosion
          :particleCount="500"
          :force="0.7"
          :stageHeight="800"
        
        />
    <quiz-completed-overlay
      :percent="percentageScore"
      @restartQuiz="onQuizStart"
      @goHome="goHome"
    ></quiz-completed-overlay>
  </v-dialog>
  <!-- Quiz overlay -->
  <v-app class="relative">
    <div class="text-right text-gray-800 score_container">
      <p class="text-sm leading-1 score_text">Score</p>
      <p class="font-bold score">{{ score }}</p>
    </div>
    <div class="text-right text-gray-800 timer_container">
      <p class="text-sm leading-1 timer_text">Time Left:</p>
      <p class="font-bold time_left">{{ timer }}</p>
    </div>
    <v-main
      class="px-auto Questions_wrapper w-full d-flex items-center justify-around"
    >
      <!-- contents -->
      <!-- <div class="relative z-20">
         -->

      <!-- question container -->

      <v-row class="w-full mr-30 ml-20 d-flex items-center justify-center">
        <v-col class="d-flex items-center justify-center" cols="12">
          <h3 class="text-center question w-2/3">
            {{ formattedQuestion }}
          </h3>
        </v-col>

        <v-col
          class="mr-3 mt-2 h-32 d-flex items-center justify-center choice_container"
          v-for="(choice, item) in currentQuestion.choices"
          :key="choice"
          cols="5"
        >
          <div class="option-default" @click="onOptionClicked($event, item)">
            <div class="">+10</div>
            {{ choice }}
          </div>
        </v-col>

        <v-col cols="12">
          <!-- progress indicator container-->
          <div class="mt-8 text-center">
            <div
              class="h-1 w-12 rounded-full mx-auto"
              style="
                background-color: #85ffbd;
                background-image: linear-gradient(
                  45deg,
                  #85ffbd 0%,
                  #fffb7d 100%
                );
              "
            ></div>
            <p class="font-bold text-white h-10" style="vertical-align: middle">
              <span class="questionCounter">{{ questionCounter }}</span> /{{
                questions.length
              }}
            </p>
          </div>
        </v-col>
      </v-row>

      <!-- options container -->
      <div>
        <!-- option container -->
      </div>

      <!-- score container -->
      <!-- </div> -->
    </v-main>
  </v-app>
  <!-- quiz container -->
</template>
<style scoped>
.neumorph-1 {
  box-shadow: 6px 6px 18px rgba(0, 0, 0, 0.09), -6px -6px 18px #ffffff;
}
.container {
  max-width: 400px;
  border-radius: 25px;
}
</style>

<script>
import { onMounted, ref, toRefs } from 'vue'
import { useRoute } from 'vue-router'
import { useCatagoryStore } from '../stores/catagoryStore'
import QuizCompletedOverlay from '../components/QuizCompletedOverlay.vue'
import router from '../router'
import { defineProps, reactive } from 'vue'
export default {
  components: { QuizCompletedOverlay },
  setup() {
    //data
    const catagoryStore = useCatagoryStore()
    const route = useRoute()
    let canClick = true
    let timer = ref(30)
    let endofQuiz = ref(false)
    let questionCounter = ref(0)
    let score = ref(0)

    const currentQuestion = ref({
      answer: 1,
      choices: [],
      question: '',
    })

    let percentageScore = ref(0)

    const questions = ref([])

    const loadQuestion = () => {
      canClick = true
      //check if there are more questions to load
      if (questions.value.length > questionCounter.value) {
        // load questions
        timer.value = 30

        currentQuestion.value.choices = formatOptions(
          questions.value[questionCounter.value].choices
        )
        currentQuestion.value.question =
          questions.value[questionCounter.value].question
        currentQuestion.value.answer =
          questions.value[questionCounter.value].answer
        console.log('The Question ', currentQuestion.value)
        questionCounter.value++
      } else {
        //no more questions
        endofQuiz.value = true
        onQuizEnd()
      }
    }

    //methods
    let itemRef = []
    const optionChosen = (element) => {
      if (element) {
        itemRef.push(element)
      }
    }

    const formatOptions = (choices) => {
      console.log(choices)
      let formatedChoices = []
      let entities = {
        amp: '&',
        apos: "'",
        '#x27': "'",
        '#x2F': '/',
        '#39': "'",
        '#47': '/',
        lt: '<',
        gt: '>',
        nbsp: ' ',
        quot: '"',
        '#039': "'",
        cent: '¢',
        pound: '£',
        copy: '©',
        laquo: '«',
        raquo: '»',
        ldquo: '“',
        rdquo: '”',
        bdquo: '„',
        lt: '<',
        gt: '>',
      }

      choices.forEach((choice) => {
        console.log('before', choice)
        let formatedChoice = choice.replace(
          /&([^;]+);/gm,
          function (match, entity) {
            return entities[entity] || match
          }
        )
        formatedChoices.push(formatedChoice)
        console.log('after', choice)
      })
      console.log('formated choices', formatedChoices)
      return formatedChoices
    }

    const clearSelected = (divSelected) => {
      setTimeout(() => {
        divSelected.classList.add('option-default')
        divSelected.classList.remove('option-wrong')
        divSelected.classList.remove('option-correct')
        loadQuestion()
      }, 2000)
    }
    const onOptionClicked = (event, item) => {
      if (canClick) {
        const divContainer = event.target
        const optionId = item
        if (currentQuestion.value.answer == optionId) {
          score.value += 10
          event.target.classList.add('option-correct')
        } else {
          event.target.classList.add('option-wrong')
          let correctAnswer =
            document.getElementsByClassName('option-default')[
              currentQuestion.value.answer
            ]
          correctAnswer.classList.remove('option-default')
          correctAnswer.classList.add('correction')
        }

        timer.value = 30
        canClick = false
        //Goto next question

        clearSelected(divContainer)
      } else {
        // Cant select option
        console.log('cant select an option')
      }
    }

    const countDownTimer = function () {
      let interval = setInterval(() => {
        if (timer.value > 0) {
          timer.value--
        } else {
          onQuizEnd()
          clearInterval(interval)
        }
      }, 1000)
    }

    const fetchQuestionsFromApi = async function () {
      fetch(catagoryStore.selectedCatagory.url)
        .then((res) => {
          return res.json()
        })
        .then((data) => {
          // map json
          const newQuestions = data.results.map((apiQuestion) => {
            const arrangedQuestion = {
              question: apiQuestion.question,
              choices: [],
              answer: '',
            }

            const choices = apiQuestion.incorrect_answers

            arrangedQuestion.answer = Math.floor(Math.random() * 3 + 1)

            choices.splice(
              arrangedQuestion.answer,
              0,
              apiQuestion.correct_answer
            )

            arrangedQuestion.choices = choices

            return arrangedQuestion
          })

          questions.value = newQuestions

          loadQuestion()
          countDownTimer()
        })
    }

    const onQuizEnd = function () {
      // calculate the score
      percentageScore.value = (score.value / 100) * 100

      //stop timer
      timer.value = 0

      //show overlay
      endofQuiz.value = true
    }

    const onQuizStart = function () {
      //set default values
      canClick = true
      timer.value = 30
      endofQuiz.value = false
      questionCounter.value = 0
      score.value = 0
      currentQuestion.value = {
        answer: 1,
        choices: [],
        question: '',
      }

      percentageScore.value = 0

      questions.value = []

      //fetch questions from server

      fetchQuestionsFromApi()
    }

    const goHome = function () {
      router.push({ name: 'Home' })
    }

    //lifecycle hooks
    onMounted(() => {
      fetchQuestionsFromApi()
    })

    // return
    return {
      timer,
      currentQuestion,
      questions,
      questionCounter,
      loadQuestion,
      score,
      endofQuiz,
      onQuizEnd,
      percentageScore,
      onQuizStart,
      catagoryStore,
      goHome,
    }
  },

  computed: {
    formattedQuestion() {
      let entities = {
        amp: '&',
        apos: "'",
        '#x27': "'",
        '#x2F': '/',
        '#39': "'",
        '#47': '/',
        lt: '<',
        gt: '>',
        nbsp: ' ',
        quot: '"',
        '#039': "'",
        cent: '¢',
        pound: '£',
        copy: '©',
        laquo: '«',
        raquo: '»',
        ldquo: '“',
        rdquo: '”',
        bdquo: '„',
        lt: '<',
        gt: '>',
      }
      return this.currentQuestion.question.replace(
        /&([^;]+);/gm,
        function (match, entity) {
          return entities[entity] || match
        }
      )
    },
    formattedOption(choice) {
      let entities = {
        amp: '&',
        apos: "'",
        '#x27': "'",
        '#x2F': '/',
        '#39': "'",
        '#47': '/',
        lt: '<',
        gt: '>',
        nbsp: ' ',
        quot: '"',
        '#039': "'",
        cent: '¢',
        pound: '£',
        copy: '©',
        laquo: '«',
        raquo: '»',
        ldquo: '“',
        rdquo: '”',
        bdquo: '„',
        lt: '<',
        gt: '>',
      }
      return choice.replace(/&([^;]+);/gm, function (match, entity) {
        return entities[entity] || match
      })
    },
  },

  components: {
    QuizCompletedOverlay,
  },
}
</script>
