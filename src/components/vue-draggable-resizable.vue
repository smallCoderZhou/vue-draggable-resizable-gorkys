<template>
  <div
    :style="style"
    :class="[{
      [classNameActive]: enabled,
      [classNameDragging]: dragging,
      [classNameResizing]: resizing,
      [classNameDraggable]: draggable,
      [classNameResizable]: resizable
    }, className]"
    @mousedown="elementDown"
    @touchstart="elementTouchDown"
  >
    <div
      v-for="handle in actualHandles"
      :key="handle"
      :class="[classNameHandle, classNameHandle + '-' + handle]"
      :style="{display: enabled ? 'block' : 'none'}"
      @mousedown.stop.prevent="handleDown(handle, $event)"
      @touchstart.stop.prevent="handleTouchDown(handle, $event)"
    >
      <slot :name="handle"></slot>
    </div>
    <slot></slot>
  </div>
</template>

<script>
import { matchesSelectorToParentElements, addEvent, removeEvent } from '../utils/dom'

const events = {
  mouse: {
    start: 'mousedown',
    move: 'mousemove',
    stop: 'mouseup'
  },
  touch: {
    start: 'touchstart',
    move: 'touchmove',
    stop: 'touchend'
  }
}

// 禁止用户选取
const userSelectNone = {
  userSelect: 'none',
  MozUserSelect: 'none',
  WebkitUserSelect: 'none',
  MsUserSelect: 'none'
}
// 用户选中自动
const userSelectAuto = {
  userSelect: 'auto',
  MozUserSelect: 'auto',
  WebkitUserSelect: 'auto',
  MsUserSelect: 'auto'
}

let eventsFor = events.mouse

export default {
  replace: true,
  name: 'vue-draggable-resizable',
  props: {
    className: {
      type: String,
      default: 'vdr'
    },
    classNameDraggable: {
      type: String,
      default: 'draggable'
    },
    classNameResizable: {
      type: String,
      default: 'resizable'
    },
    classNameDragging: {
      type: String,
      default: 'dragging'
    },
    classNameResizing: {
      type: String,
      default: 'resizing'
    },
    classNameActive: {
      type: String,
      default: 'active'
    },
    classNameHandle: {
      type: String,
      default: 'handle'
    },
    disableUserSelect: {
      type: Boolean,
      default: true
    },
    enableNativeDrag: {
      type: Boolean,
      default: false
    },
    preventDeactivation: {
      type: Boolean,
      default: false
    },
    active: {
      type: Boolean,
      default: false
    },
    draggable: {
      type: Boolean,
      default: true
    },
    resizable: {
      type: Boolean,
      default: true
    },
    // 锁定宽高比
    lockAspectRatio: {
      type: Boolean,
      default: false
    },
    w: {
      type: Number,
      default: 200,
      validator: (val) => val > 0
    },
    h: {
      type: Number,
      default: 200,
      validator: (val) => val > 0
    },
    minWidth: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },
    minHeight: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },
    maxWidth: {
      type: Number,
      default: null,
      validator: (val) => val >= 0
    },
    maxHeight: {
      type: Number,
      default: null,
      validator: (val) => val >= 0
    },
    x: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === 'number'
    },
    y: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === 'number'
    },
    z: {
      type: [String, Number],
      default: 'auto',
      validator: (val) => (typeof val === 'string' ? val === 'auto' : val >= 0)
    },
    handles: {
      type: Array,
      default: () => ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'],
      validator: (val) => {
        const s = new Set(['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'])

        return new Set(val.filter(h => s.has(h))).size === val.length
      }
    },
    dragHandle: {
      type: String,
      default: null
    },
    dragCancel: {
      type: String,
      default: null
    },
    axis: {
      type: String,
      default: 'both',
      validator: (val) => ['x', 'y', 'both'].includes(val)
    },
    grid: {
      type: Array,
      default: () => [1, 1]
    },
    parent: {
      type: [Boolean, String],
      default: false
    },
    onDragStart: {
      type: Function,
      default: null
    },
    onResizeStart: {
      type: Function,
      default: null
    },
    // 冲突检测
    isConflictCheck: {
      type: Boolean, default: false
    },
    // 元素对齐
    snap: {
      type: Boolean, default: false
    },
    // 当调用对齐时，用来设置组件与组件之间的对齐距离，以像素为单位
    snapTolerance: {
      type: Number,
      default: 5,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    // 缩放比例
    scaleRatio: {
      type: Number,
      default: 1,
      validator: (val) => typeof val === 'number'
    }
  },

  data: function () {
    return {
      rawWidth: this.w,
      rawHeight: this.h,
      rawLeft: this.x,
      rawTop: this.y,
      rawRight: null,
      rawBottom: null,

      left: this.x,
      top: this.y,
      right: null,
      bottom: null,

      aspectFactor: this.w / this.h,

      parentWidth: null,
      parentHeight: null,

      minW: this.minWidth,
      minH: this.minHeight,

      maxW: this.maxWidth,
      maxH: this.maxHeight,

      handle: null,
      enabled: this.active,
      resizing: false,
      dragging: false,
      zIndex: this.z
    }
  },

  created: function () {
    // eslint-disable-next-line 无效的prop：minWidth不能大于maxWidth
    if (this.maxWidth && this.minWidth > this.maxWidth) console.warn('[Vdr warn]: Invalid prop: minWidth cannot be greater than maxWidth')
    // eslint-disable-next-line 无效prop：minHeight不能大于maxHeight'
    if (this.maxWidth && this.minHeight > this.maxHeight) console.warn('[Vdr warn]: Invalid prop: minHeight cannot be greater than maxHeight')

    this.resetBoundsAndMouseState()
  },
  mounted: function () {
    if (!this.enableNativeDrag) {
      this.$el.ondragstart = () => false
    }

    [this.parentWidth, this.parentHeight] = this.getParentSize()

    this.rawRight = this.parentWidth - this.rawWidth - this.rawLeft
    this.rawBottom = this.parentHeight - this.rawHeight - this.rawTop

    this.settingAttribute()

    addEvent(document.documentElement, 'mousedown', this.deselect)
    addEvent(document.documentElement, 'touchend touchcancel', this.deselect)

    addEvent(window, 'resize', this.checkParentSize)
  },
  beforeDestroy: function () {
    removeEvent(document.documentElement, 'mousedown', this.deselect)
    removeEvent(document.documentElement, 'touchstart', this.handleUp)
    removeEvent(document.documentElement, 'mousemove', this.move)
    removeEvent(document.documentElement, 'touchmove', this.move)
    removeEvent(document.documentElement, 'mouseup', this.handleUp)
    removeEvent(document.documentElement, 'touchend touchcancel', this.deselect)

    removeEvent(window, 'resize', this.checkParentSize)
  },

  methods: {
    // 重置边界和鼠标状态
    resetBoundsAndMouseState () {
      this.mouseClickPosition = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 }

      this.bounds = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      }
    },
    // 检查父元素大小
    checkParentSize () {
      if (this.parent) {
        const [newParentWidth, newParentHeight] = this.getParentSize()

        const deltaX = this.parentWidth - newParentWidth
        const deltaY = this.parentHeight - newParentHeight

        this.rawRight -= deltaX
        this.rawBottom -= deltaY

        this.parentWidth = newParentWidth
        this.parentHeight = newParentHeight
      }
    },
    // 获取父元素大小
    getParentSize () {
      if (this.parent === true) {
        const style = window.getComputedStyle(this.$el.parentNode, null)
        return [
          parseInt(style.getPropertyValue('width'), 10),
          parseInt(style.getPropertyValue('height'), 10)
        ]
      }
      if (typeof this.parent === 'string') {
        const parentNode = document.querySelector(this.parent)
        if (!(parentNode instanceof HTMLElement)) {
          throw new Error(`The selector ${this.parent} does not match any element`)
        }
        return [parentNode.offsetWidth, parentNode.offsetHeight]
      }

      return [null, null]
    },
    // 元素触摸按下
    elementTouchDown (e) {
      eventsFor = events.touch

      this.elementDown(e)
    },
    // 元素按下
    elementDown (e) {
      const target = e.target || e.srcElement

      if (this.$el.contains(target)) {
        if (this.onDragStart && this.onDragStart(e) === false) {
          return
        }

        if (
          (this.dragHandle && !matchesSelectorToParentElements(target, this.dragHandle, this.$el)) ||
          (this.dragCancel && matchesSelectorToParentElements(target, this.dragCancel, this.$el))
        ) {
          return
        }

        if (!this.enabled) {
          this.enabled = true

          this.$emit('activated')
          this.$emit('update:active', true)
        }

        if (this.draggable) {
          this.dragging = true
        }

        this.mouseClickPosition.mouseX = e.touches ? e.touches[0].pageX : e.pageX
        this.mouseClickPosition.mouseY = e.touches ? e.touches[0].pageY : e.pageY

        this.mouseClickPosition.left = this.left
        this.mouseClickPosition.right = this.right
        this.mouseClickPosition.top = this.top
        this.mouseClickPosition.bottom = this.bottom

        if (this.parent) {
          this.bounds = this.calcDragLimits()
        }

        addEvent(document.documentElement, eventsFor.move, this.move)
        addEvent(document.documentElement, eventsFor.stop, this.handleUp)
      }
    },
    // 计算移动范围
    calcDragLimits () {
      return {
        minLeft: (this.parentWidth + this.left) % this.grid[0],
        maxLeft: Math.floor((this.parentWidth - this.width - this.left) / this.grid[0]) * this.grid[0] + this.left,
        minRight: (this.parentWidth + this.right) % this.grid[0],
        maxRight: Math.floor((this.parentWidth - this.width - this.right) / this.grid[0]) * this.grid[0] + this.right,
        minTop: (this.parentHeight + this.top) % this.grid[1],
        maxTop: Math.floor((this.parentHeight - this.height - this.top) / this.grid[1]) * this.grid[1] + this.top,
        minBottom: (this.parentHeight + this.bottom) % this.grid[1],
        maxBottom: Math.floor((this.parentHeight - this.height - this.bottom) / this.grid[1]) * this.grid[1] + this.bottom
      }
    },
    // 取消
    deselect (e) {
      const target = e.target || e.srcElement
      const regex = new RegExp(this.className + '-([trmbl]{2})', '')

      if (!this.$el.contains(target) && !regex.test(target.className)) {
        if (this.enabled && !this.preventDeactivation) {
          this.enabled = false

          this.$emit('deactivated')
          this.$emit('update:active', false)
        }

        removeEvent(document.documentElement, eventsFor.move, this.handleMove)
      }

      this.resetBoundsAndMouseState()
    },
    // 控制柄触摸按下
    handleTouchDown (handle, e) {
      eventsFor = events.touch

      this.handleDown(handle, e)
    },
    // 控制柄按下
    handleDown (handle, e) {
      if (this.onResizeStart && this.onResizeStart(handle, e) === false) {
        return
      }

      if (e.stopPropagation) e.stopPropagation()

      // Here we avoid a dangerous recursion by faking
      // corner handles as middle handles
      if (this.lockAspectRatio && !handle.includes('m')) {
        this.handle = 'm' + handle.substring(1)
      } else {
        this.handle = handle
      }

      this.resizing = true

      this.mouseClickPosition.mouseX = e.touches ? e.touches[0].pageX : e.pageX
      this.mouseClickPosition.mouseY = e.touches ? e.touches[0].pageY : e.pageY
      this.mouseClickPosition.left = this.left
      this.mouseClickPosition.right = this.right
      this.mouseClickPosition.top = this.top
      this.mouseClickPosition.bottom = this.bottom

      this.bounds = this.calcResizeLimits()

      addEvent(document.documentElement, eventsFor.move, this.handleMove)
      addEvent(document.documentElement, eventsFor.stop, this.handleUp)
    },
    // 计算调整大小范围
    calcResizeLimits () {
      let minW = this.minW
      let minH = this.minH
      let maxW = this.maxW
      let maxH = this.maxH

      const aspectFactor = this.aspectFactor
      const [gridX, gridY] = this.grid
      const width = this.width
      const height = this.height
      const left = this.left
      const top = this.top
      const right = this.right
      const bottom = this.bottom

      if (this.lockAspectRatio) {
        if (minW / minH > aspectFactor) {
          minH = minW / aspectFactor
        } else {
          minW = aspectFactor * minH
        }

        if (maxW && maxH) {
          maxW = Math.min(maxW, aspectFactor * maxH)
          maxH = Math.min(maxH, maxW / aspectFactor)
        } else if (maxW) {
          maxH = maxW / aspectFactor
        } else if (maxH) {
          maxW = aspectFactor * maxH
        }
      }

      maxW = maxW - (maxW % gridX)
      maxH = maxH - (maxH % gridY)

      const limits = {
        minLeft: null,
        maxLeft: null,
        minTop: null,
        maxTop: null,
        minRight: null,
        maxRight: null,
        minBottom: null,
        maxBottom: null
      }

      if (this.parent) {
        limits.minLeft = (this.parentWidth + left) % gridX
        limits.maxLeft = left + Math.floor((width - minW) / gridX) * gridX
        limits.minTop = (this.parentHeight + top) % gridY
        limits.maxTop = top + Math.floor((height - minH) / gridY) * gridY
        limits.minRight = (this.parentWidth + right) % gridX
        limits.maxRight = right + Math.floor((width - minW) / gridX) * gridX
        limits.minBottom = (this.parentHeight + bottom) % gridY
        limits.maxBottom = bottom + Math.floor((height - minH) / gridY) * gridY

        if (maxW) {
          limits.minLeft = Math.max(limits.minLeft, this.parentWidth - right - maxW)
          limits.minRight = Math.max(limits.minRight, this.parentWidth - left - maxW)
        }

        if (maxH) {
          limits.minTop = Math.max(limits.minTop, this.parentHeight - bottom - maxH)
          limits.minBottom = Math.max(limits.minBottom, this.parentHeight - top - maxH)
        }

        if (this.lockAspectRatio) {
          limits.minLeft = Math.max(limits.minLeft, left - top * aspectFactor)
          limits.minTop = Math.max(limits.minTop, top - left / aspectFactor)
          limits.minRight = Math.max(limits.minRight, right - bottom * aspectFactor)
          limits.minBottom = Math.max(limits.minBottom, bottom - right / aspectFactor)
        }
      } else {
        limits.minLeft = null
        limits.maxLeft = left + Math.floor((width - minW) / gridX) * gridX
        limits.minTop = null
        limits.maxTop = top + Math.floor((height - minH) / gridY) * gridY
        limits.minRight = null
        limits.maxRight = right + Math.floor((width - minW) / gridX) * gridX
        limits.minBottom = null
        limits.maxBottom = bottom + Math.floor((height - minH) / gridY) * gridY

        if (maxW) {
          limits.minLeft = -(right + maxW)
          limits.minRight = -(left + maxW)
        }

        if (maxH) {
          limits.minTop = -(bottom + maxH)
          limits.minBottom = -(top + maxH)
        }

        if (this.lockAspectRatio && (maxW && maxH)) {
          limits.minLeft = Math.min(limits.minLeft, -(right + maxW))
          limits.minTop = Math.min(limits.minTop, -(maxH + bottom))
          limits.minRight = Math.min(limits.minRight, -left - maxW)
          limits.minBottom = Math.min(limits.minBottom, -top - maxH)
        }
      }

      return limits
    },
    // 移动
    move (e) {
      if (this.resizing) {
        this.handleMove(e)
      } else if (this.dragging) {
        this.elementMove(e)
      }
    },
    // 元素移动
    elementMove (e) {
      const axis = this.axis
      const grid = this.grid
      const mouseClickPosition = this.mouseClickPosition

      const tmpDeltaX = axis && axis !== 'y' ? mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX) : 0
      const tmpDeltaY = axis && axis !== 'x' ? mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY) : 0

      const [deltaX, deltaY] = this.snapToGrid(this.grid, tmpDeltaX, tmpDeltaY)

      this.rawTop = mouseClickPosition.top - deltaY
      this.rawBottom = mouseClickPosition.bottom + deltaY
      this.rawLeft = mouseClickPosition.left - deltaX
      this.rawRight = mouseClickPosition.right + deltaX

      this.snapCheck()

      this.$emit('dragging', this.left, this.top)
    },
    // 控制柄移动
    handleMove (e) {
      const handle = this.handle
      const mouseClickPosition = this.mouseClickPosition

      const tmpDeltaX = mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX)
      const tmpDeltaY = mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY)

      const [deltaX, deltaY] = this.snapToGrid(this.grid, tmpDeltaX, tmpDeltaY)

      if (handle.includes('b')) {
        this.rawBottom = mouseClickPosition.bottom + deltaY
      } else if (handle.includes('t')) {
        this.rawTop = mouseClickPosition.top - deltaY
      }

      if (handle.includes('r')) {
        this.rawRight = mouseClickPosition.right + deltaX
      } else if (handle.includes('l')) {
        this.rawLeft = mouseClickPosition.left - deltaX
      }

      this.$emit('resizing', this.left, this.top, this.width, this.height)
    },
    // 从控制柄松开
    async handleUp (e) {
      this.handle = null

      this.rawTop = this.top
      this.rawBottom = this.bottom
      this.rawLeft = this.left
      this.rawRight = this.right
      const refLine = {
        vLine: [
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          },
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          },
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          }
        ],
        hLine: [
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          },
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          },
          {
            display: false,
            position: '',
            origin: '',
            lineLength: ''
          }
        ]
      }

      if (this.resizing) {
        this.resizing = false
        await this.conflictCheck()
        this.$emit('refLineParams', refLine)
        this.$emit('resizestop', this.left, this.top, this.width, this.height)
      }
      if (this.dragging) {
        this.dragging = false
        await this.conflictCheck()
        this.$emit('refLineParams', refLine)
        this.$emit('dragstop', this.left, this.top)
      }
      this.resetBoundsAndMouseState()
      removeEvent(document.documentElement, eventsFor.move, this.handleMove)
    },
    // 对齐网格
    snapToGrid (grid, pendingX, pendingY) {
      const x = Math.round(pendingX / grid[0]) * grid[0]
      const y = Math.round(pendingY / grid[1]) * grid[1]

      // return [x, y]
      return [Math.floor(x / this.scaleRatio), Math.floor(y / this.scaleRatio)]
    },
    // 新增方法 ↓↓↓
    // 设置属性
    settingAttribute () {
      // 设置冲突检测
      this.isConflictCheck ? this.$el.setAttribute('data-is-check', 'true') : this.$el.setAttribute('data-is-check', 'false')
      // 设置对齐元素
      this.snap ? this.$el.setAttribute('data-is-snap', 'true') : this.$el.setAttribute('data-is-snap', 'false')
    },
    // 冲突检测
    conflictCheck () {
      const top = this.rawTop
      const left = this.rawLeft
      const width = this.width
      const height = this.height

      if (this.isConflictCheck) {
        const nodes = this.$el.parentNode.childNodes // 获取当前父节点下所有子节点
        for (let item of nodes){
          if (item !== this.$el && item.className !== undefined && item.getAttribute('data-is-check') !== null && item.getAttribute('data-is-check') !== 'false') {
            const tw = item.offsetWidth
            const th = item.offsetHeight
            const tl = item.offsetLeft
            const tt = item.offsetTop
            // 左上角与右下角重叠
            const tfAndBr = (top >= tt && left >= tl && tt + th > top && tl + tw > left) || (top <= tt && left < tl && top + height > tt && left + width > tl)
            // 右上角与左下角重叠
            const brAndTf = (left <= tl && top >= tt && left + width > tl && top < tt + th) || (top < tt && left > tl && top + height > tt && left < tl + tw)
            // 下边与上边重叠
            const bAndT = (top <= tt && left >= tl && top + height > tt && left < tl + tw) || (top >= tt && left <= tl && top < tt + th && left > tl + tw)
            // 上边与下边重叠（宽度不一样）
            const tAndB = (top <= tt && left >= tl && top + height > tt && left < tl + tw) || (top >= tt && left <= tl && top < tt + th && left > tl + tw)
            // 左边与右边重叠
            const lAndR = (left >= tl && top >= tt && left < tl + tw && top < tt + th) || (top > tt && left <= tl && left + width > tl && top < tt + th)
            // 左边与右边重叠（高度不一样）
            const rAndL = (top <= tt && left >= tl && top + height > tt && left < tl + tw) || (top >= tt && left <= tl && top < tt + th && left + width > tl)

            // 如果冲突，就将回退到移动前的位置
            if (tfAndBr || brAndTf || bAndT || tAndB || lAndR || rAndL) {
              this.rawTop = this.mouseClickPosition.top
              this.rawLeft = this.mouseClickPosition.left
              this.rawRight = this.mouseClickPosition.right
              this.rawBottom = this.mouseClickPosition.bottom
            }
          }
        }
      }
    },
    // 检测对齐元素
    async snapCheck () {
      const width = this.width
      const height = this.height
      if (this.snap) {
        const activeLeft = this.rawLeft
        const activeRight = this.rawLeft + width
        const activeTop = this.rawTop
        const activeBottom = this.rawTop + height

        // let horizontalArray = []
        // let verticalArray = []

        const refLine = {
          vLine: [
              {
                display: false,
                position: '',
                origin: '',
                lineLength: ''
              },
              {
                display: false,
                position: '',
                origin: '',
                lineLength: ''
              },
              {
                display: false,
                position: '',
                origin: '',
                lineLength: ''
              }
          ],
          hLine: [
            {
              display: false,
              position: '',
              origin: '',
              lineLength: ''
            },
            {
              display: false,
              position: '',
              origin: '',
              lineLength: ''
            },
            {
              display: false,
              position: '',
              origin: '',
              lineLength: ''
            }
          ]
        }

        const nodes = this.$el.parentNode.childNodes // 获取当前父节点下所有子节点
        // 获取距离达到阔值的所有组件结果
        let { XArray, YArray } = await this.getResult(nodes,{ width, height, activeLeft, activeRight, activeTop, activeBottom })

        for (let item of nodes){
          if (item !== this.$el && item.className !== undefined && item.getAttribute('data-is-snap') !== null && item.getAttribute('data-is-snap') !== 'false') {
            const w = item.offsetWidth
            const h = item.offsetHeight
            const l = item.offsetLeft // 对齐目标的left
            const r = l + w // 对齐目标right
            const t = item.offsetTop// 对齐目标的top
            const b = t + h // 对齐目标的bottom

            const xMin = Math.min(...XArray) + 'px'
            const xSpacing = Math.max(...XArray) - Math.min(...XArray) + 'px'

            const yMin = Math.min(...YArray) + 'px'
            const ySpacing = Math.max(...YArray) - Math.min(...YArray) + 'px'

            const hc = Math.abs((activeTop + height / 2) - (t + h / 2)) <= this.snapTolerance // 水平中线
            const vc = Math.abs((activeLeft + width / 2) - (l + w / 2)) <= this.snapTolerance // 垂直中线

            const ts = Math.abs(t - activeBottom) <= this.snapTolerance // 从上到下
            const TS = Math.abs(b - activeBottom) <= this.snapTolerance // 从上到下
            const bs = Math.abs(t - activeTop) <= this.snapTolerance // 从下到上
            const BS = Math.abs(b - activeTop) <= this.snapTolerance // 从下到上

            const ls = Math.abs(l - activeRight) <= this.snapTolerance // 外左
            const LS = Math.abs(r - activeRight) <= this.snapTolerance // 外左
            const rs = Math.abs(l - activeLeft) <= this.snapTolerance // 外右
            const RS = Math.abs(r - activeLeft) <= this.snapTolerance // 外右

            if (ts) {
              this.rawTop = t - height
              this.rawBottom = this.parentHeight - this.rawTop - height
              refLine.hLine[0].display = true
              refLine.hLine[0].position = t + 'px'
              refLine.hLine[0].origin = yMin
              refLine.hLine[0].lineLength = ySpacing
            }
            if (bs) {
              this.rawTop = t
              this.rawBottom = this.parentHeight - this.rawTop - height
              refLine.hLine[2].display = true
              refLine.hLine[2].position = t + 'px'
              refLine.hLine[2].origin = yMin
              refLine.hLine[2].lineLength = ySpacing
            }
            if (TS) {
              this.rawTop = b - height
              this.rawBottom = this.parentHeight - this.rawTop - height
              refLine.hLine[0].display = true
              refLine.hLine[0].position = b + 'px'
              refLine.hLine[0].origin = yMin
              refLine.hLine[0].lineLength = ySpacing
            }
            if (BS) {
              this.rawTop = b
              this.rawBottom = this.parentHeight - this.rawTop - height
              refLine.hLine[2].display = true
              refLine.hLine[2].position = b + 'px'
              refLine.hLine[2].origin = yMin
              refLine.hLine[2].lineLength = ySpacing
            }

            if (ls) {
              this.rawLeft = l - width
              this.rawRight = this.parentWidth - this.rawLeft - width
              refLine.vLine[0].display = true
              refLine.vLine[0].position = l + 'px'
              refLine.vLine[0].origin = xMin
              refLine.vLine[0].lineLength = xSpacing
            }
            if (rs) {
              this.rawLeft = l
              this.rawRight = this.parentWidth - this.rawLeft - width
              refLine.vLine[2].display = true
              refLine.vLine[2].position = l + 'px'
              refLine.vLine[2].origin = xMin
              refLine.vLine[2].lineLength = xSpacing
            }
            if (LS) {
              this.rawLeft = r - width
              this.rawRight = this.parentWidth - this.rawLeft - width
              refLine.vLine[0].display = true
              refLine.vLine[0].position = r + 'px'
              refLine.vLine[0].origin = xMin
              refLine.vLine[0].lineLength = xSpacing
            }
            if (RS) {
              this.rawLeft = r
              this.rawRight = this.parentWidth - this.rawLeft - width
              refLine.vLine[2].display = true
              refLine.vLine[2].position = r + 'px'
              refLine.vLine[2].origin = xMin
              refLine.vLine[2].lineLength = xSpacing
            }

            if (hc) {
              this.rawTop = t + h / 2 - height / 2
              this.rawBottom = this.parentHeight - this.rawTop - height

              refLine.hLine[1].display = true
              refLine.hLine[1].position = t + h / 2 + 'px'
              refLine.hLine[1].origin = yMin
              refLine.hLine[1].lineLength = ySpacing
            }
            if (vc) {
              this.rawLeft = l + w / 2 - width / 2
              this.rawRight = this.parentWidth - this.rawLeft - width

              refLine.vLine[1].display = true
              refLine.vLine[1].position = l + w / 2 + 'px'
              refLine.vLine[1].origin = xMin
              refLine.vLine[1].lineLength = xSpacing
            }
          }
        }
        this.$emit('refLineParams', refLine)
      }
    },
    getResult(nodes, actives) {
      return new Promise(resolve => {
        let { width, height, activeLeft, activeRight, activeTop, activeBottom } = actives
        const TEM = {
          XArray: [],
          YArray: []
        }
        for (let item of nodes){
          if (item !== this.$el && item.className !== undefined && item.getAttribute('data-is-snap') !== null && item.getAttribute('data-is-snap') !== 'false') {
            const w = item.offsetWidth
            const h = item.offsetHeight
            const l = item.offsetLeft
            const r = l + w
            const t = item.offsetTop
            const b = t + h

            const hc = Math.abs((activeTop + height / 2) - (t + h / 2)) <= this.snapTolerance // 水平中线
            const vc = Math.abs((activeLeft + width / 2) - (l + w / 2)) <= this.snapTolerance // 垂直中线

            const ts = Math.abs(t - activeBottom) <= this.snapTolerance // 从上到下
            const TS = Math.abs(b - activeBottom) <= this.snapTolerance // 从上到下
            const bs = Math.abs(t - activeTop) <= this.snapTolerance // 从下到上
            const BS = Math.abs(b - activeTop) <= this.snapTolerance // 从下到上

            const ls = Math.abs(l - activeRight) <= this.snapTolerance // 外左
            const LS = Math.abs(r - activeRight) <= this.snapTolerance // 外左
            const rs = Math.abs(l - activeLeft) <= this.snapTolerance // 外右
            const RS = Math.abs(r - activeLeft) <= this.snapTolerance // 外右

            if (hc || vc || ts || TS || bs || BS || ls || LS || rs|| RS){
              TEM.XArray.push(t, b, activeTop, activeBottom)
              TEM.YArray.push(l, r, activeLeft, activeRight)
            }
          }
        }
        resolve(TEM)
      })
    }
  },
  computed: {
    style () {
      return {
        position: 'absolute',
        top: this.top + 'px',
        left: this.left + 'px',
        width: this.width + 'px',
        height: this.height + 'px',
        zIndex: this.zIndex,
        ...(this.dragging && this.disableUserSelect ? userSelectNone : userSelectAuto)
      }
    },
    // 控制柄显示与否
    actualHandles () {
      if (!this.resizable) return []

      return this.handles
    },
    width () {
      return this.parentWidth - this.left - this.right
    },
    height () {
      return this.parentHeight - this.top - this.bottom
    },
    resizingOnX () {
      return (Boolean(this.handle) && (this.handle.includes('l') || this.handle.includes('r')))
    },
    resizingOnY () {
      return (Boolean(this.handle) && (this.handle.includes('t') || this.handle.includes('b')))
    },
    isCornerHandle () {
      return (Boolean(this.handle) && ['tl', 'tr', 'br', 'bl'].includes(this.handle))
    }
  },

  watch: {
    active (val) {
      this.enabled = val

      if (val) {
        this.$emit('activated')
      } else {
        this.$emit('deactivated')
      }
    },
    z (val) {
      if (val >= 0 || val === 'auto') {
        this.zIndex = val
      }
    },
    rawLeft (newLeft) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const left = this.left
      const top = this.top

      if (bounds.minLeft !== null && newLeft < bounds.minLeft) {
        newLeft = bounds.minLeft
      } else if (bounds.maxLeft !== null && bounds.maxLeft < newLeft) {
        newLeft = bounds.maxLeft
      }

      if (lockAspectRatio && this.resizingOnX) {
        this.rawTop = top - (left - newLeft) / aspectFactor
      }

      this.left = newLeft
    },
    rawRight (newRight) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const right = this.right
      const bottom = this.bottom

      if (bounds.minRight !== null && newRight < bounds.minRight) {
        newRight = bounds.minRight
      } else if (bounds.maxRight !== null && bounds.maxRight < newRight) {
        newRight = bounds.maxRight
      }

      if (lockAspectRatio && this.resizingOnX) {
        this.rawBottom = bottom - (right - newRight) / aspectFactor
      }

      this.right = newRight
    },
    rawTop (newTop) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const left = this.left
      const top = this.top

      if (bounds.minTop !== null && newTop < bounds.minTop) {
        newTop = bounds.minTop
      } else if (bounds.maxTop !== null && bounds.maxTop < newTop) {
        newTop = bounds.maxTop
      }

      if (lockAspectRatio && this.resizingOnY) {
        this.rawLeft = left - (top - newTop) * aspectFactor
      }

      this.top = newTop
    },
    rawBottom (newBottom) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const right = this.right
      const bottom = this.bottom

      if (bounds.minBottom !== null && newBottom < bounds.minBottom) {
        newBottom = bounds.minBottom
      } else if (bounds.maxBottom !== null && bounds.maxBottom < newBottom) {
        newBottom = bounds.maxBottom
      }

      if (lockAspectRatio && this.resizingOnY) {
        this.rawRight = right - (bottom - newBottom) * aspectFactor
      }

      this.bottom = newBottom
    },
    x () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcDragLimits()
      }

      const delta = this.x - this.left

      if (delta % this.grid[0] === 0) {
        this.rawLeft = this.x
        this.rawRight = this.right - delta
      }
    },
    y () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcDragLimits()
      }

      const delta = this.y - this.top

      if (delta % this.grid[1] === 0) {
        this.rawTop = this.y
        this.rawBottom = this.bottom - delta
      }
    },
    lockAspectRatio (val) {
      if (val) {
        this.aspectFactor = this.width / this.height
      } else {
        this.aspectFactor = undefined
      }
    },
    minWidth (val) {
      if (val > 0 && val <= this.width) {
        this.minW = val
      }
    },
    minHeight (val) {
      if (val > 0 && val <= this.height) {
        this.minH = val
      }
    },
    maxWidth (val) {
      this.maxW = val
    },
    maxHeight (val) {
      this.maxH = val
    },
    w () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcResizeLimits()
      }

      const delta = this.width - this.w

      if (delta % this.grid[0] === 0) {
        this.rawRight = this.right + delta
      }
    },
    h () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcResizeLimits()
      }

      const delta = this.height - this.h

      if (delta % this.grid[1] === 0) {
        this.rawBottom = this.bottom + delta
      }
    }
  }
}
</script>
