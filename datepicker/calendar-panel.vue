<template>
  <div class="calendar">
    <div class="calendar-header">
      <a class="calendar__prev-icon" @click="changeYear(-1)">&laquo;</a>
      <a v-show="currentPanel === 'date'" class="calendar__prev-icon" @click="changeMonth(-1)">&lsaquo;</a>
      <a class="calendar__next-icon" @click="changeYear(1)">&raquo;</a>
      <a v-show="currentPanel === 'date'" class="calendar__next-icon" @click="changeMonth(1)">&rsaquo;</a>
      <a @click="showMonths">{{months[currentMonth]}}</a>
      <a @click="showYears">{{currentYear}}</a>
    </div>
    <div class="calendar-content">
      <table class="calendar-table" v-show="currentPanel === 'date'">
        <thead>
          <tr>
            <th v-for="day in days">{{day}}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in dates">
            <td v-for="cell in row" :title="cell.title" :class="getClasses(cell)" @click="selectDate(cell)">{{cell.day}}</td>
          </tr>
        </tbody>
      </table>
      <div class="calendar-year" v-show="currentPanel === 'years'">
        <a v-for="year in years" @click="selectYear(year)" :class="{'current': currentYear === year}">{{year}}</a>
      </div>
      <div class="calendar-month" v-show="currentPanel === 'months'">
        <a v-for="(month, index) in months" @click="selectMonth(index)" :class="{'current': currentMonth === index}">{{month}}</a>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    startAt: null,
    endAt: null,
    value: null,
    show: Boolean
  },
  data () {
    const translation = this.$parent.translation
    return {
      days: translation.days,
      months: translation.months,
      dates: [],
      now: new Date(), // calendar-header 显示的时间, 用于切换日历
      years: [], // 年代面板
      currentPanel: 'date'
    }
  },
  computed: {
    currentYear () {
      return this.now.getFullYear()
    },
    currentMonth () {
      return this.now.getMonth()
    }
  },
  created () {
    this.updateCalendar()
  },
  watch: {
    show (val) {
      if (val) {
        this.currentPanel = 'date'
        this.updateNow()
      }
    },
    value: {
      handler: 'updateNow',
      immediate: true
    },
    now: 'updateCalendar'
  },
  methods: {
    updateNow () {
      let now = this.value ? new Date(this.value) : new Date()
      now.setDate(1)
      this.now = now
    },
    // 更新面板选择时间
    updateCalendar () {
      function getCalendar (time, firstday, length, classes) {
        return Array.apply(null, { length }).map((v, i) => { // eslint-disable-line
          let day = firstday + i
          const date = new Date(time.getFullYear(), time.getMonth(), day, 0, 0, 0)
          return {
            title: date.toLocaleDateString(),
            date,
            day,
            classes
          }
        })
      }
      const time = new Date(this.now)
      time.setDate(0) // 把时间切换到上个月最后一天
      const lastMonthLength = time.getDay() + 1  // time.getDay() 0是星期天, 1是星期一 ...
      const lastMonthfirst = time.getDate() - (lastMonthLength - 1)
      const lastMonth = getCalendar(time, lastMonthfirst, lastMonthLength, 'lastMonth')

      time.setMonth(time.getMonth() + 2, 0) // 切换到这个月最后一天
      const curMonthLength = time.getDate()
      const curMonth = getCalendar(time, 1, curMonthLength, 'curMonth')

      time.setMonth(time.getMonth() + 1, 1)
      const nextMonthLength = 42 - (lastMonthLength + curMonthLength)
      const nextMonth = getCalendar(time, 1, nextMonthLength, 'nextMonth')

      // 分割数组
      let index = 0
      let resIndex = 0
      const arr = lastMonth.concat(curMonth, nextMonth)
      const result = new Array(6)
      while (index < 42) {
        result[resIndex++] = arr.slice(index, index += 7)
      }
      this.dates = result
    },
    getClasses (cell) {
      const classes = []
      const cellTime = cell.date.getTime()
      const curTime = this.value ? new Date(this.value).setHours(0, 0, 0, 0) : 0
      const startTime = this.startAt ? new Date(this.startAt).setHours(0, 0, 0, 0) : 0
      const endTime = this.endAt ? new Date(this.endAt).setHours(0, 0, 0, 0) : 0
      const today = new Date().setHours(0, 0, 0, 0)

      if (this.$parent.disabledDays.some(v => +new Date(v) === +cell.date) ||
        (this.$parent.notBefore !== '' && cell.date.getTime() < (new Date(this.$parent.notBefore)).getTime()) ||
        (this.$parent.notAfter !== '' && cell.date.getTime() > (new Date(this.$parent.notAfter)).getTime())) {
        return 'disabled'
      }

      classes.push(cell.classes)

      if (cellTime === today) {
        classes.push('today')
      }

      // range classes
      if (cellTime === curTime) {
        classes.push('current')
      } else if (startTime) {
        if (cellTime < startTime) {
          classes.push('disabled')
        } else if (curTime && cellTime <= curTime) {
          classes.push('inrange')
        }
      } else if (endTime) {
        if (cellTime > endTime) {
          classes.push('disabled')
        } else if (curTime && cellTime >= curTime) {
          classes.push('inrange')
        }
      }
      return classes.join(' ')
    },
    showMonths () {
      if (this.currentPanel === 'months') {
        this.currentPanel = 'date'
      } else {
        this.currentPanel = 'months'
      }
    },
    showYears () {
      // 当前年代
      if (this.currentPanel === 'years') {
        this.currentPanel = 'date'
      } else {
        let firstYear = Math.floor(this.now.getFullYear() / 10) * 10
        let years = []
        for (let i = 0; i < 10; i++) {
          years.push(firstYear + i)
        }
        this.years = years
        this.currentPanel = 'years'
      }
    },
    // 前进或后退一年
    changeYear (flag) {
      if (this.currentPanel === 'years') {
        this.years = this.years.map(v => v + flag * 10)
      } else {
        const now = new Date(this.now)
        now.setFullYear(now.getFullYear() + flag)
        this.now = now
      }
    },
    changeMonth (flag) {
      const now = new Date(this.now)
      now.setMonth(now.getMonth() + flag)
      this.now = now
    },
    selectDate (cell) {
      const classes = this.getClasses(cell)
      if (classes.indexOf('disabled') !== -1) {
        return
      }
      this.$emit('input', cell.date)
      this.$emit('select')
    },
    selectYear (year) {
      const now = new Date(this.now)
      now.setFullYear(year)
      this.now = now
      this.currentPanel = 'months'
    },
    selectMonth (month) {
      const now = new Date(this.now)
      now.setMonth(month)
      this.now = now
      this.currentPanel = 'date'
    }
  }
}

</script>
