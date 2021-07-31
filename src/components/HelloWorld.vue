<template>
  <p><button class="valera-button" @click="decideWinner">Poehali, nahui!</button></p>
  <p><label for="time">Время прокрутки в секундах: </label> <input v-model="animationDuration" type="number" /></p>

  <modal v-if="showModal" @close="showModal = false">
      <template #header>
        <h2>Ну вот ты и попался пидорок.</h2>
      </template>
      <template #body>
        Победила эта хуета: <a :href="winner.link">{{winner.name}}</a>
      </template>
      <template #footer>
        <button class="valera-button" @click="makeDecision('d')">Ебать, ну поехали...</button>
        <button class="valera-button valera-button--whack" @click="makeDecision('p')">Skippppppppp</button>
      </template>
   </modal>

  <div class="main-content" v-if="items.length">
    <div class="triangle"></div>
    <svg ref="chart" class="donut" :width="width" height="100%" :viewBox="viewBox">
      <circle class="donut-hole" :cx="centerOfTheViewBox" :cy="centerOfTheViewBox" :r="radius" fill="transparent"></circle>
      <circle
        v-for="(item, key) in items"
        :key="key"
        :cx="centerOfTheViewBox" :cy="centerOfTheViewBox" :r="radius"
        fill="transparent"
        :title="item.name"
        :ref="`items${key}`"
        :stroke-width="radius"
        stroke-dasharray="0 100"
        stroke-dashoffset="25"
        ></circle>
      <g v-for="(item, key) in items" :key="key" :style="`transform: rotate(${item.angle}deg) translateX(${radius / 1.5}px); text-align: left`">
      <text
        :x="centerOfTheViewBox"
        :y="centerOfTheViewBox"
        font-family="sans-serif"
        font-size="1px"
        fill="white"
        text-anchor="left"
        alignment-baseline="middle"
        >

        {{ truncateString(item.name, 20) }}
      </text>
      </g>
      <defs>
      <clipPath id="circleView">
      <circle :cx="centerOfTheViewBox" :cy="centerOfTheViewBox - 4" :r="radius / 2" />
      </clipPath>
      </defs>
      <g class="chart-text">
      <image
        :x="centerOfTheViewBox - diameter / 8" :y="centerOfTheViewBox - diameter / 8 - 5"
        class="lerich"
        href="@/assets/Lerich.jpg"
        alt="Lerich" :width="diameter / 4" :height="diameter / 4"
        clip-path="url(#circleView)" />
      </g>
    </svg>
  </div>
  <div v-else> Грузится блядь, падажжи</div>
</template>

<script>
import TWEEN from '@tweenjs/tween.js'
import { GoogleSpreadsheet } from 'google-spreadsheet'
import { Sound, truncateString } from '@/helpers'
import Modal from '@/components/Modal'
const API_GOOGLE_KEY = 'AIzaSyCb4nu3aSI5SJ9LH44OBX-D11V93O_uSsc'

export default {
  name: 'HelloWorld',
  components: { Modal },
  props: {
    msg: String
  },
  data () {
    return {
      showModal: false,
      animationDuration: 142,
      winner: null,
      value: 0,
      diameter: 100,
      offset: 25,
      width: 600,
      animationSpeed: '0.5s',
      itemsForDecision: [],
      itemsForPunishment: [],
      items: []
    }
  },
  async mounted () {
    await this.fetchExcelData()
    const animate = (time) => {
      requestAnimationFrame(animate)
      TWEEN.update(time)
    }
    requestAnimationFrame(animate)
  },
  computed: {
    viewBox () {
      return `-${this.diameter / 4} -${this.diameter / 4} ${this.diameter / 2} ${this.diameter / 2}`
    },
    radius () {
      return `${this.diameter / (Math.PI) / 2}`
    },
    centerOfTheViewBox () {
      return 0
    },
    itemsWithPercentage () {
      const totalValue = this.items.reduce((acc, { value }) => {
        acc += parseFloat(value)
        return acc
      }, 0)
      return Object.entries(this.items).map(([key, item]) => {
        item.percent = parseFloat(item.value) / totalValue * 100
        return item
      })
    }
  },
  methods: {
    async makeDecision (decision) {
      await this.calculate(decision)
      this.showModal = false
    },
    truncateString,
    getRandomInt: (min, max) => {
      min = Math.ceil(min)
      max = Math.floor(max)
      return Math.floor(Math.random() * (max - min + 1)) + min
    },
    async fetchExcelData () {
      const doc = new GoogleSpreadsheet('1WQUE6MLI4dFV_9DLop63OovmLIWGGEN7-18uO1JoWpk')
      doc.useApiKey(API_GOOGLE_KEY)
      await doc.loadInfo()
      await this.fetchDecisionData(doc)
      await this.fetchPunishmentData(doc)
      this.items = this.itemsForDecision
      await this.$nextTick(() => {
        this.calculate()
      })
    },
    async fetchDecisionData (doc) {
      const sheet = doc.sheetsByIndex[0]
      const rows = await sheet.getRows()
      this.itemsForDecision = rows
        .filter(row => {
          const item = row._rawData
          return !!item[0] && item[4] === 'FALSE' && item[3] === 'TRUE' && item[2] === 'TRUE'
        })
        .map(item => ({ name: item._rawData[0], link: item._rawData[1], value: 10 }))
    },
    async fetchPunishmentData (doc) {
      const sheet = doc.sheetsByIndex[2]
      const rows = await sheet.getRows()

      this.itemsForPunishment = rows
        .map(item => ({ name: item._rawData[0], value: 10 }))
    },
    getRandomColor () {
      const letters = '0123456789ABCDEF'
      let color = '#'
      for (let i = 0; i < 6; i++) {
        color += letters[Math.floor(Math.random() * 16)]
      }
      return color
    },
    getCorners (item) {
      const lowLine = item.leftCorner / 100 * 360
      const topLine = item.rightCorner / 100 * 360
      return [lowLine, topLine]
    },
    async calculate (type) {
      // TODO: no magic strings!
      if (type === 'p') {
        this.items = this.itemsForPunishment
      } else if (type === 'd') {
        this.items = this.itemsForDecision
      }
      this.$refs.chart.style.transform = 'rotate(0deg)'
      const { animationSpeed, offset } = this
      await this.$nextTick()
      for (const key in this.itemsWithPercentage) {
        const item = this.items[key]
        const itemPerc = item.percent
        const node = this.$refs[`items${key}`]
        node.style.transition = `stroke-dasharray ${animationSpeed} ease-in-out, stroke-dashoffset ${animationSpeed} ease-in-out`
        node.style.strokeDasharray = itemPerc + ' ' + (100 - itemPerc)
        node.setAttribute('stroke', this.getRandomColor())
        if (key === '0') {
          node.style.strokeDashoffset = offset
        } else {
          const leftOver = this.itemsWithPercentage.slice(0, key).reduce((acc, item) => {
            acc += item.percent
            return acc
          }, 0)

          const leftCorner = 100 - leftOver + offset
          node.style.strokeDashoffset = leftCorner
        }
        item.leftCorner = parseFloat(node.style.strokeDashoffset) - 25
        item.rightCorner = parseFloat(item.leftCorner) + item.percent
        const [leftCorner, rightCorner] = this.getCorners(item)
        item.angle = ((rightCorner - leftCorner) / 2 + leftCorner) - 90
      }
    },
    decideWinner () {
      const winnerIndx = this.getRandomInt(0, this.itemsWithPercentage.length - 1)
      const winner = this.itemsWithPercentage[winnerIndx]
      this.winner = winner
      const chartNode = this.$refs.chart

      const degObj = { value: 0 }
      const finishWith = 360 * this.animationDuration - this.getRandomInt(...this.getCorners(winner))
      const song = new Sound('/ZAKAT.mp3', 40, false)
      const abchihba = new Sound('/abchihba.mp3', 100, false)
      new TWEEN.Tween(degObj)
        .to({ value: finishWith }, parseFloat(this.animationDuration * 1000))
        .easing(TWEEN.Easing.Cubic.Out)
        .onStart(() => {
          song.start()
        })
        .onUpdate(() => {
          const { value } = degObj
          chartNode.style.transform = `rotate(${value}deg)`
        })
        .start()
        .onComplete(() => {
          song.stop()
          abchihba.start()
          setTimeout(() => {
            this.showModal = true
          }, 500)
        })
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
* {
  -webkit-box-sizing: border-box;
  -moz-box-sizing: border-box;
  box-sizing: border-box;
}
.donut-spin {
  animation: spin linear infinite;
  animation-duration: 30s;
}
.chart-text {
  /*font: 16px/1.4em "Montserrat", Arial, sans-serif;*/
  fill: #000;
  -moz-transform: translateY(0.25em);
  -ms-transform: translateY(0.25em);
  -webkit-transform: translateY(0.25em);
  transform: translateY(0.25em);
}

.chart-number {
  font-size: 0.5em;
  line-height: 1;
  text-anchor: middle;
  -moz-transform: translateY(-0.25em);
  -ms-transform: translateY(-0.25em);
  -webkit-transform: translateY(-0.25em);
  transform: translateY(-0.25em);
}

.chart-label {
  font-size: 0.2em;
  text-transform: uppercase;
  text-anchor: middle;
  -moz-transform: translateY(0.7em);
  -ms-transform: translateY(0.7em);
  -webkit-transform: translateY(0.7em);
  transform: translateY(0.7em);
}
.answer {
  transform: translateZ(30px);
}

@keyframes spin {
  100% {
    transform:rotate(360deg);
  }
}
.main-content {
  position: relative;
}
.triangle {
  position: absolute;
  right: calc(50% - 11px);
  top: 12px;
  z-index: 2;
  border-top: 20px solid red;
  border-right: 10px solid transparent;
  border-left: 10px solid transparent;
  border-bottom: 10px solid transparent;
  display: inline-block;
}

button.valera-button {
  z-index: 1;
  position: relative;
  font-size: inherit;
  font-family: inherit;
  color: white;
  padding: 0.5em 1em;
  outline: none;
  border: none;
  background-color: hsl(236, 32%, 26%);
  overflow: hidden;
  transition: color 0.4s ease-in-out;
}

button.valera-button::before {
  content: '';
  z-index: -1;
  position: absolute;
  top: 100%;
  left: 100%;
  width: 1em;
  height: 1em;
  border-radius: 50%;
  background-color: #3cefff;
  transform-origin: right;
  transform: translate3d(-50%, -50%, 0) scale3d(0, 0, 0);
  transition: transform 0.45s ease-in-out;
}

button.valera-button:hover {
  cursor: pointer;
  color: #161616;
}

button.valera-button:hover::before {
  transform: translate3d(-50%, -50%, 0) scale3d(15, 15, 15);
}
.valera-button.valera-button--whack {
  margin-top: 10px;
}
</style>
