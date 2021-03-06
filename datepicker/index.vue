<template>
  <div class="datepicker"
       :style="{'width': width + 'px','min-width':range ? '210px' : '140px'}"
       v-clickoutside="closePopup">
    <input readonly
          class="input"
          :value="text"
          :placeholder="innerPlaceholder"
          ref="input"
          @click="togglePopup"
          @mousedown="$event.preventDefault()">
    <i class="input-icon" 
      :class="showCloseIcon ? 'input-icon__close' : 'input-icon__calendar'" 
      @mouseenter="hoverIcon"
      @mouseleave="hoverIcon"
      @click="clickIcon" ></i>
    <div class="datepicker-popup"
         :class="{'range':range}"
         :style="position"
         ref="calendar"
         v-show="showPopup">
      <template v-if="!range">
        <calendar-panel @select="showPopup = false"
                        v-model="currentValue"
                        :show="showPopup"></calendar-panel>
      </template>
      <template v-else>
        <div class="datepicker-top" v-if="ranges.length">
          <span v-for="range in ranges" @click="selectRange(range)">{{range.text}}</span>
        </div>
        <calendar-panel style="width:50%;box-shadow:1px 0 rgba(0, 0, 0, .1)"
                        v-model="currentValue[0]"
                        :end-at="currentValue[1]"
                        :show="showPopup"></calendar-panel>
        <calendar-panel style="width:50%;"
                        v-model="currentValue[1]"
                        :start-at="currentValue[0]"
                        :show="showPopup"></calendar-panel>
      </template>
    </div>
  </div>
</template>

<script>
import CalendarPanel from './calendar-panel.vue'
import Languages from './languages.js'
var  dateFormat = require('dateformat');

export default {
  components: { CalendarPanel },
  props: {
    format: {
      type: String,
      default: 'yyyy-MM-dd'
    },
    range: {
      type: Boolean,
      default: false
    },
    width: {
      type: [String, Number],
      default: 210
    },
    placeholder: String,
    lang: {
      type: String,
      default: 'en'
    },
    value: null,
    shortcuts: {
      type: [Boolean, Array],
      default: true
    },
    disabledDays: {
      type: Array,
      default: function () { return [] }
    },
    notBefore: {
      type: String,
      default: ''
    },
    notAfter: {
      type: String,
      default: ''
    }
  },
  data () {
    return {
      showPopup: false,
      showCloseIcon: false,
      currentValue: this.value,
      position: null,
      ranges: []  // 快捷选项
    }
  },
  watch: {
    value: {
      handler (val) {
        if (!this.range) {
          this.currentValue = this.isValidDate(val) ? val : undefined
        } else {
          this.currentValue = this.isValidRange(val) ? val : [undefined, undefined]
        }
      },
      immediate: true
    },
    currentValue (val) {
      if ((!this.range && val) || (this.range && val[0] && val[1])) {
        this.$emit('input', val)
      }
    },
    showPopup (val) {
      if (val) {
        this.$nextTick(this.displayPopup)
      }
    }
  },
  computed: {
    translation () {
      return Languages[this.lang] || Languages['en']
    },
    innerPlaceholder () {
      return this.placeholder || (this.range ? this.translation.placeholder.dateRange : this.translation.placeholder.date)
    },
    text () {
      if (!this.range && this.currentValue) {
        return this.stringify(this.currentValue)
      }
      if (this.range && this.currentValue[0] && this.currentValue[1]) {
        return this.stringify(this.currentValue[0]) + ' ~ ' + this.stringify(this.currentValue[1])
      }
      return ''
    }
  },
  methods: {
    closePopup () {
      this.showPopup = false
    },
    togglePopup () {
      if (this.showPopup) {
        this.$refs.input.blur()
        this.showPopup = false
      } else {
        this.$refs.input.focus()
        this.showPopup = true
      }
    },
    hoverIcon (e) {
      if (e.type === 'mouseenter' && this.text) {
        this.showCloseIcon = true
      }
      if (e.type === 'mouseleave') {
        this.showCloseIcon = false
      }
    },
    clickIcon () {
      if (this.showCloseIcon) {
        this.$emit('input', '')
      } else {
        this.togglePopup()
      }
    },
    stringify (date) {
      return dateFormat(new Date(date), this.format)
    },
    isValidDate (date) {
      return !!new Date(date).getTime()
    },
    isValidRange (date) {
      return Array.isArray(date) &&
        date.length === 2 &&
        this.isValidDate(date[0]) &&
        this.isValidDate(date[1])
    },
    selectRange (range) {
      this.$emit('input', [range.start, range.end])
    },
    initRanges () {
      if (Array.isArray(this.shortcuts)) {
        this.ranges = this.shortcuts
      } else if (this.shortcuts) {
        this.ranges = [{
          text: '未来7天',
          start: new Date(),
          end: new Date(Date.now() + 3600 * 1000 * 24 * 7)
        }, {
          text: '未来30天',
          start: new Date(),
          end: new Date(Date.now() + 3600 * 1000 * 24 * 30)
        }, {
          text: '最近7天',
          start: new Date(Date.now() - 3600 * 1000 * 24 * 7),
          end: new Date()
        }, {
          text: '最近30天',
          start: new Date(Date.now() - 3600 * 1000 * 24 * 30),
          end: new Date()
        }]
        this.ranges.forEach((v, i) => {
          v.text = this.translation.pickers[i]
        })       
      } else {
        this.ranges = []
      }
    },
    displayPopup () {
      const dw = document.documentElement.clientWidth
      const dh = document.documentElement.clientHeight
      const InputRect = this.$el.getBoundingClientRect()
      const PopupRect = this.$refs.calendar.getBoundingClientRect()
      this.position = {}
      if (dw - InputRect.left < PopupRect.width && InputRect.right < PopupRect.width) {
        this.position.left = 1 - InputRect.left + 'px'
      } else if (InputRect.left + InputRect.width / 2 <= dw / 2) {
        this.position.left = 0
      } else {
        this.position.right = 0
      }
      if (InputRect.top <= PopupRect.height + 1 && dh - InputRect.bottom <= PopupRect.height + 1) {
        this.position.top = dh - InputRect.top - PopupRect.height - 1 + 'px'
      } else if (InputRect.top + InputRect.height / 2 <= dh / 2) {
        this.position.top = '100%'
      } else {
        this.position.bottom = '100%'
      }
    }
  },
  created () {
    this.initRanges()
  },
  directives: {
    clickoutside: {
      bind (el, binding, vnode) {
        el['@clickoutside'] = (e) => {
          if (!el.contains(e.target) && binding.expression && vnode.context[binding.expression]) {
            binding.value()
          }
        }
        document.addEventListener('click', el['@clickoutside'], true)
      },
      unbind (el) {
        document.removeEventListener('click', el['@clickoutside'], true)
      }
    }
  }
}
</script>



