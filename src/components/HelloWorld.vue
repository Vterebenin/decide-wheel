<template>
  <input v-model="value" type="text">
  <p><button @click="addItem">AddItem</button></p>
  <p><button @click="decideWinner">decideWinner</button></p>
  <p><button @click="calculate">Calculate</button></p>

  <svg ref="chart" class="donut" width="300" height="100%" :viewBox="viewBox">
    <circle class="donut-hole" :cx="centerOfTheViewBox" :cy="centerOfTheViewBox" :r="radius" fill="grey"></circle>
      <circle
        v-for="(item, key) in items"
        :key="key"
        :cx="centerOfTheViewBox" :cy="centerOfTheViewBox" :r="radius"
        fill="transparent"
        stroke="#b1c94e"
        :title="item.name"
        :ref="`items${key}`"
        @mouseover="log"
        stroke-width="12"
        stroke-dasharray="0 100"
        stroke-dashoffset="25"
      ></circle>
    <g v-for="(item, key) in items" :key="key">
      <text
          :x="centerOfTheViewBox"
          :y="centerOfTheViewBox"
          :style="`transform: rotate(${item.angle}deg) translateX(16px)`"
          font-family="sans-serif"
          font-size="2px"
          fill="white"
          text-anchor="middle"
          alignment-baseline="middle"
      >
        {{item.name}}
      </text>
    </g>
    <defs>
        <clipPath id="circleView">
            <circle :cx="centerOfTheViewBox" :cy="centerOfTheViewBox - 4" :r="radius - 6" fill="#FFFFFF" />
        </clipPath>
    </defs>
   <g class="chart-text">
      <image
        :x="centerOfTheViewBox - 12.5" :y="centerOfTheViewBox - 17.5" class="lerich"
        href="@/assets/Lerich.jpg" alt="Lerich" width="25" height="25"
        clip-path="url(#circleView)" />
    </g>
  </svg>
  <br>
  {{ items }}
</template>

<script>
import TWEEN from '@tweenjs/tween.js'

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data () {
    return {
      value: 0,
      diameter: 50,
      offset: 25,
      animationSpeed: '0.5s',
      items: [
        { name: 'В жопу', value: 10 },
        { name: 'В рот', value: 10 },
        { name: 'В пизду', value: 10 }
      ]
    }
  },
  async mounted () {
    await this.startAnim()
    const animate = (time) => {
      requestAnimationFrame(animate)
      TWEEN.update(time)
    }
    requestAnimationFrame(animate)
  },
  computed: {
    viewBox () {
      return `-25 -25 ${this.diameter} ${this.diameter}`
    },
    radius () {
      return `${this.diameter / (Math.PI)}`
    },
    centerOfTheViewBox () {
      // return `${this.diameter / 2}`
      return 0
    },
    itemsWithPercentage () {
      const totalValue = this.items.reduce((acc, { value }) => {
        acc += parseFloat(value)
        return acc
      }, 0)
      console.log('total:', totalValue)
      return Object.entries(this.items).map(([key, item]) => {
        item.percent = parseFloat(item.value) / totalValue * 100
        return item
      })
    }
  },
  methods: {
    log (event) {
      console.log(event)
    },
    addItem () {
      this.items.push({ name: 'Test', value: 10 })
      console.log(this.items)
    },
    getRandomInt: (min, max) => {
      min = Math.ceil(min)
      max = Math.floor(max)
      return Math.floor(Math.random() * (max - min + 1)) + min
    },
    async startAnim () {
      const { animationSpeed } = this
      await setTimeout(() => {
        this.$refs.items1.style.transition = `stroke-dasharray ${animationSpeed} ease, stroke-dashoffset ${animationSpeed} ease`
        this.$refs.items1.style.strokeDasharray = '100 0'
      }, 20)
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
    calculate () {
      this.$refs.chart.style.transform = 'rotate(0deg)'
      const { animationSpeed, offset } = this
      for (const key in this.itemsWithPercentage) {
        const item = this.items[key]
        const itemPerc = item.percent
        const node = this.$refs[`items${key}`]
        node.style.transition = `stroke-dasharray ${animationSpeed} ease-in-out, stroke-dashoffset ${animationSpeed} ease-in-out`
        node.style.strokeDasharray = itemPerc + ' ' + (100 - itemPerc)
        node.setAttribute('stroke', this.getRandomColor())
        if (key === '0') {
          node.setAttribute('stroke', 'black')
          node.style.strokeDashoffset = offset
        } else {
          const leftOver = this.itemsWithPercentage.slice(0, key).reduce((acc, item) => {
            acc += item.percent
            return acc
          }, 0)

          const leftCorner = 100 - leftOver + offset
          node.style.strokeDashoffset = leftCorner
          console.log(node.style.strokeDashoffset)
        }
        item.leftCorner = parseFloat(node.style.strokeDashoffset) - 25
        item.rightCorner = parseFloat(item.leftCorner) + item.percent
        const [leftCorner, rightCorner] = this.getCorners(item)
        item.angle = ((rightCorner - leftCorner) / 2 + leftCorner) - 90
        console.log(item.angle, '333')
      }
    },
    decideWinner () {
      const winnerIndx = this.getRandomInt(0, this.itemsWithPercentage.length - 1)
      const winner = this.itemsWithPercentage[winnerIndx]
      const chartNode = this.$refs.chart
      // chartNode.style.transitionDuration = '10s'
      // chartNode.style.transform = 'rotate(199deg)'
      // chartNode.classList.toggle('donut-spin')

      const degObj = { value: 0 }
      const finishWith = 360 * 20 - this.getRandomInt(...this.getCorners(winner))
      console.log(this.getCorners(winner))
      new TWEEN.Tween(degObj)
        .to({ value: finishWith }, 2000)
        .easing(TWEEN.Easing.Quadratic.Out)
        .onUpdate(() => {
          const { value } = degObj
          chartNode.style.transform = `rotate(${value}deg)`
        })
        .start()
        .onComplete(() => {
          alert(winner.name)
          alert(finishWith - 360 * 20)
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
@keyframes spin {
  100% {
    transform:rotate(360deg);
  }
}
</style>
