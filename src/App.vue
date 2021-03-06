<template>
  <div id="app">
    <section class="hero is-fullheight">
      <div class="hero-head">
        <nav class="navbar">
          <div class="container">
            <div>
              <a href="/">
                <p class="title">Chess Guardian (beta)</p>
                <p class="subtitle">Answer Chess technical questions from your own positions to improve board vision.</p>
              </a>
              <p class="subtitle">Inspired by this article <a href="https://www.chess.com/blog/danheisman/the-amazing-power-of-board-vision">The Amazing Power of Board Vision</a> by Dan Heisman</p>

            </div>
            <div id="navbarMenu" class="navbar-menu">
              <div class="navbar-end"></div>
            </div>
          </div>
        </nav>
      </div>
      <div class="hero-body">
        <div class="container has-text-centered">
          <div class="columns">
            <div class="column is-5">
              <chessboard :fen="currentPosition.fen" :showThreats="showThreats" @onMove="showInfo"></chessboard>
              <div class="has-centered-text">
                {{currentPosition.white}} VS {{currentPosition.black}} <br><a :href="currentPosition.url">View game in chess.com</a>
              </div>
            </div>
            <div class="column is-7">
              <QuestionGame></QuestionGame>
            </div>
          </div>
        </div>
      </div>

      <div class="hero-foot">
        <div class="container">
          <div class="has-text-centered">
            Made by <a href="http://vitomd.com">@vitomd</a> with <a href="https://vuejs.org/">Vue.js</a> | Check all my <a href="http://vitomd.com/blog/projects/">chess related projects</a>
          </div>
        </div>
      </div>
    </section>
    <a href="https://github.com/vitogit/vue-chess-guardian"><img style="position: absolute; top: 0; right: 0; border: 0;" src="https://camo.githubusercontent.com/38ef81f8aca64bb9a64448d0d70f1308ef5341ab/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6461726b626c75655f3132313632312e706e67" alt="Fork me on GitHub" data-canonical-src="https://s3.amazonaws.com/github/ribbons/forkme_right_darkblue_121621.png"></a>              </div>
  </div>

</template>

<script>
import jQuery from 'jquery'
import Chess from 'chess.js'
import {chessboard} from 'vue-chessboard'
import 'vue-chessboard/dist/vue-chessboard.css'
import { shuffle } from './Util.js'
import {defaultPositions} from './PositionData.js'
import QuestionGame from './components/QuestionGame'
import StartModal from './components/StartModal'

export default {
  name: 'App',
  components: {
    chessboard,
    QuestionGame,
    StartModal
  },
  data () {
    return {
      currentPosition: {},
      showThreats: false,
      positionNumber: 0,
      started: false,
      isStartModalActive: true,
      positionInfo: {},
    }
  },
  methods: {
    nextQuestion() {
      let position = this.positions[this.positionNumber]
      this.currentPosition = position
      this.positionNumber++
    },
    start(data){ //TODO improve this method
      let positions = []
      if (data && data.username) {
        this.positions = this.getPositions(data)
      } else {
        this.positions = defaultPositions()
      }
      if (this.positions.length < 10 ) {
        this.promptAgain()
      } else {
        this.positionNumber = 0
        this.positions = shuffle(this.positions)
        this.nextQuestion()
      }
    },
    getPositions({username, year, month}){ //TODO refactor this big method and make it async

      month = ("00" + month).slice(-2) // API expect month like 2 digits MM

      username = username || 'hikaru'
      username = username.replace(/\s/g, '').toLowerCase()

      this.$toast.open(`Pulling positions from: ${username}`)
      let games = []
      jQuery.ajax({
        method: 'GET',
        url: `https://api.chess.com/pub/player/${username}/games/${year}/${month}`,
        async: false,
        success: function (data) {
          games = data.games.slice(0,50) //get 50 games
        },
        error: function (error) {
          console.log('Something wrong with ajax:', error);
        }
      });
      let positions = []
      let loadedGame = new Chess()
      games.forEach(game => {
        loadedGame.load_pgn(game.pgn)
        let middlegame = Math.round(loadedGame.history().length / 2)
        if (loadedGame.history().length < 40) {
          return // skip this and go to next game if is too short (40 halfmoves)
        }
        Array.from(Array(middlegame), () => loadedGame.undo()) //go back to middlegame
        let p = {fen: loadedGame.fen(), white: game.white.username, black: game.black.username, url: game.url}
        positions.push(p)
      })
      return positions
    },
    promptAgain() {
      this.genericPrompt({
        title: 'ERROR',
        message: `There are not enough games (${this.positions.length}) from the selected user and month. You need at least 10. Select another chess.com user or month.`,
        type:'is-danger'
      })
    },
    prompt() {
      this.genericPrompt({
        title: `What's your chess.com username?`,
        message: 'It will pull random positions from the games played in the selected month for the selected user '
      })
    },
    genericPrompt({title, message, type}) {
      this.$modal.open({
        parent: this,
        component: StartModal,
        hasModalCard: true,
        props: {
          title: title,
          message: message,
          inputAttrs: {
            placeholder: 'e.g. hikaru',
            maxlength: 20
          },
          type: type,
          onConfirm: (data) => this.start(data),
          onCancel: () => this.start()
        }
      })
    },
    showInfo(info){
      this.positionInfo = info
      this.$eventHub.$emit('game-changed', {color:this.positionInfo.turn, threats: this.positionInfo})
    }
  },
  mounted(){
    this.prompt()
  },
  created() {
    this.positions = []
    this.$eventHub.$on('paint-threats', () => {
      this.showThreats = true
    })

    this.$eventHub.$on('next-question', () => {
      this.showThreats = false
      this.nextQuestion()
    })
    this.$eventHub.$on('start-again', () => {
      this.showThreats = false
      this.positionNumber = 0
      this.positions = shuffle(this.positions)
      this.nextQuestion()
    })
  }
}
</script>

<style>
  #app {
    margin: 1vh;
  }
  .hero.is-fullheight {
    min-height: 95vh;
  }
  .cg-board-wrap {
    margin: 0 auto;
  }
  .hero-foot {
    padding-bottom: 20px;
  }
</style>
