<template>
<div class="drag-container">
  <ul class="drag-list">
    <li v-for="stage in stages" class="drag-column" :class="{['drag-column-' + stage]: true}" :key="stage">
      <span class="drag-column-header">
        <slot :name="stage">
          <h2>{{ stage }}</h2>
        </slot>
      </span>
      <div class="drag-options"></div>
      <ul class="drag-inner-list" ref="list" :data-status="stage">
        <li @mousedown="grab" class="drag-item" v-for="block in getBlocks(stage)" :data-block-id="block.id" :key="block.id">
          <slot :name="block.id">
            <strong>{{ block.status }}</strong>
            <div>{{ block.id }}</div>
          </slot>
        </li>
      </ul>
      <div class="drag-column-footer">
          <slot :name="`footer-${stage}`"></slot>
      </div>
    </li>
  </ul>
</div>
</template>
<script type='text/ecmascript-6'>
import crossvent from 'crossvent'
import KanbanBoardMixin from '@/components/KanbanBoardMixin'
import classes from '@/common/utils/classes'
export default {
  mixins: [KanbanBoardMixin],
  data () {
    return {
      moveX: 0,
      moveY: 0,
      mirror: null,
      dragging: false
    }
  },
  props: {
    stages: {},
    blocks: {}
  },
  computed: {
    localBlocks () {
      return this.blocks
    }
  },
  methods: {
    invalidTarget () {
      return false
    },
    always () { return true },
    invalid () {
      return this.invalidTarget()
    },
    moves () {
      return this.always()
    },
    touchy (el, op, type, fn) {
      var touch = {
        mouseup: 'touchend',
        mousedown: 'touchstart',
        mousemove: 'touchmove'
      }
      var pointers = {
        mouseup: 'pointerup',
        mousedown: 'pointerdown',
        mousemove: 'pointermove'
      }
      var microsoft = {
        mouseup: 'MSPointerUp',
        mousedown: 'MSPointerDown',
        mousemove: 'MSPointerMove'
      }
      if (global.navigator.pointerEnabled) {
        crossvent[op](el, pointers[type], fn)
      } else if (global.navigator.msPointerEnabled) {
        crossvent[op](el, microsoft[type], fn)
      } else {
        crossvent[op](el, touch[type], fn)
        crossvent[op](el, type, fn)
      }
    },
    isContainer (el) {
      return el.classList.contains('drag-inner-list')
    },
    isList (el) {
      return el.classList.contains('drag-inner-list')
    },
    getBlocks (status) {
      return this.localBlocks.filter(block => block.status === status)
    },
    grab (e) {
      this.moveX = e.clientX
      this.moveY = e.clientY
      var ignore = this.whichMouseButton(e) !== 1 || e.metaKey || e.ctrlKey
      if (ignore) {
        return // we only care about honest-to-god left clicks and touch events
      }
      var item = e.target
      var context = this.canStart(item)
      if (!context) {
        return
      }
      this.grabbed = context
      this.eventualMovements()
    },
    eventualMovements (remove) {
      var op = remove ? 'remove' : 'add'
      this.touchy(this.documentElement, op, 'mousemove', this.startBecauseMouseMoved)
    },
    whichMouseButton (e) {
      if (e.touches !== void 0) { return e.touches.length }
      if (e.which !== void 0 && e.which !== 0) { return e.which } // see https://github.com/bevacqua/dragula/issues/261
      if (e.buttons !== void 0) { return e.buttons }
      var button = e.button
      if (button !== void 0) { // see https://github.com/jquery/jquery/blob/99e8ff1baa7ae341e94bb89c3e84570c7c3ad9ea/src/event.js#L573-L575
        return button & 1 ? 1 : button & 2 ? 3 : (button & 4 ? 2 : 0)
      }
    },
    canStart (item) {
      if (this.dragging && this.mirror) {
        return
      }
      if (this.isContainer(item)) {
        return // don't drag container itself
      }
      var handle = item
      while (this.getParent(item) && this.isList(this.getParent(item)) === false) {
        if (this.invalid(item, handle)) { // 无效的
          return
        }
        item = this.getParent(item) // drag target should be a top element
        if (!item) {
          return
        }
      }
      var source = this.getParent(item)
      if (!source) {
        return
      }
      if (this.invalid(item, handle)) {
        return
      }

      var movable = this.moves(item, source, handle, this.nextEl(item))
      if (!movable) {
        return
      }
      return {
        item: item,
        source: source
      }
    },
    release () {
    },
    startBecauseMouseMoved (e) {
      if (!this.grabbed) {
        return
      }
      if (this.whichMouseButton(e) === 0) {
        this.release({})
        return // when text is selected on an input and then dragged, mouseup doesn't fire. this is our only hope
      }
      if (e.clientX !== void 0 && e.clientX === this.moveX && e.clientY !== void 0 && e.clientY === this.moveY) {
        return
      }
      if (this.ignoreInputTextSelection) {
        var clientX = this.getCoord('clientX', e)// 触摸点的clientX
        var clientY = this.getCoord('clientY', e)// 触摸点的clientX
        var elementBehindCursor = document.elementFromPoint(clientX, clientY) // 鼠标点击的落点元素
        if (this.isInput(elementBehindCursor)) { // 判断鼠标点击的落点元素是否是表单
          return
        }
      }
      var grabbed = this.grabbed
      this.eventualMovements(true)
      this.movements()
      this.end()

      this.start(grabbed)

      var offset = this.getOffset(this.item)
      this.offsetX = this.getCoord('pageX', e) - offset.left
      this.offsetY = this.getCoord('pageY', e) - offset.top
      classes.add(this.copy || this.item, 'gu-transit')
      this.renderMirrorImage()
    },
    renderMirrorImage () {
      if (this.mirror) {
        return
      }
      var rect = this.item.getBoundingClientRect()
      this.mirror = this.item.cloneNode(true)
      this.mirror.style.width = this.getRectWidth(rect) + 'px'
      this.mirror.style.height = this.getRectHeight(rect) + 'px'
      classes.rm(this.mirror, 'gu-transit')
      classes.add(this.mirror, 'gu-mirror')
      this.o.mirrorContainer.appendChild(this.mirror)
      this.touchy(this.documentElement, 'add', 'mousemove', this.drag)
      classes.add(this.o.mirrorContainer, 'gu-unselectable')
    },
    start (context) {
      if (this.isCopy(context.item, context.source)) {
        this.copy = context.item.cloneNode(true) // true  深克隆
      }
      this.source = context.source
      this.item = context.item
      this.initialSibling = this.currentSibling = this.nextEl(context.item) // 拿到被拖动元素的下个元素节点
      this.dragging = true
    },
    drag (e) {
      if (!this.mirror) {
        return
      }
      e.preventDefault()

      var clientX = this.getCoord('clientX', e)
      var clientY = this.getCoord('clientY', e)
      var x = clientX - this.offsetX
      var y = clientY - this.offsetY

      this.mirror.style.left = x + 'px'
      this.mirror.style.top = y + 'px'

      // var item = _copy || _item
      // var elementBehindCursor = getElementBehindPoint(_mirror, clientX, clientY) // 鼠标下的元素
      // console.log('elementBehindCursor', elementBehindCursor)
      // var dropTarget = findDropTarget(elementBehindCursor, clientX, clientY) // drop目标
      // console.log('isContainer(dropTarget)', isContainer(dropTarget))
      // if (isContainer(dropTarget)) {
      //   console.log('test', dropTarget)
      //   dropTarget = getChildren(dropTarget)[0]
      //   console.log('test1', dropTarget)
      //   while (!isList(dropTarget)) {
      //     debugger
      //     dropTarget = getChildren(dropTarget)[0]
      //   }
      //   var changed = dropTarget !== null && dropTarget !== _lastDropTarget
      //   if (changed || dropTarget === null) {
      //     out()
      //     _lastDropTarget = dropTarget
      //     over()
      //   }
      //   var reference = null
      //   if (
      //     (reference === null && changed) ||
      //     reference !== item &&
      //     reference !== nextEl(item) // 拿到元素的下个元素节点
      //   ) {
      //     console.log('(reference === null && changed) ||reference !== item &&reference !== nextEl(item)')
      //     console.log(reference)
      //     _currentSibling = reference

      //     console.log('dropTarget.insertBefore(item, reference)', reference)
      //     dropTarget.insertBefore(item, reference)
      //     drake.emit('shadow', item, dropTarget, _source)
      //   }
      //   return
      // }
      // var changed = dropTarget !== null && dropTarget !== _lastDropTarget
      // console.log('_lastDropTarget', _lastDropTarget)
      // console.log('dropTarget', dropTarget)
      // console.log('changed', changed)
      // if (changed || dropTarget === null) {
      //   out()
      //   _lastDropTarget = dropTarget
      //   over()
      // }
      // var parent = getParent(item) // 删除原来的元素
      // if (dropTarget === _source && _copy && !o.copySortSource) {
      //   if (parent) {
      //     parent.removeChild(item)
      //   }
      //   console.log('-==-=-=-=-=-=--=1')
      //   return
      // }
      // var reference
      // var immediate = getImmediateChild(dropTarget, elementBehindCursor) // 拿到dropTarget下级元素
      // console.log('immediate', immediate)
      // if (immediate !== null) {
      //   console.log('immediate !== null')
      //   reference = getReference(dropTarget, immediate, clientX, clientY)
      //   console.log('reference', reference)
      // } else if (o.revertOnSpill === true && !_copy) {
      //   console.log('o.revertOnSpill === true && !_copy')
      //   reference = _initialSibling
      //   dropTarget = _source
      // } else {
      //   if (_copy && parent) {
      //     parent.removeChild(item)
      //   }
      //   console.log('-==-=-=-=-=-=--=2')
      //   return
      // }
      // if (
      //   (reference === null && changed) ||
      //   reference !== item &&
      //   reference !== nextEl(item) // 拿到元素的下个元素节点
      // ) {
      //   console.log('(reference === null && changed) ||reference !== item &&reference !== nextEl(item)')
      //   console.log(reference)
      //   _currentSibling = reference

      //   console.log('dropTarget.insertBefore(item, reference)', reference)
      //   dropTarget.insertBefore(item, reference)
      //   drake.emit('shadow', item, dropTarget, _source)
      // }
      // function moved (type) { drake.emit(type, item, _lastDropTarget, _source) }
      // function over () { if (changed) { moved('over') } }
      // function out () { if (_lastDropTarget) { moved('out') } }
      // console.log('---', 'dragend')
    }
  }
}
</script>
<style scoped lang='stylus' rel='stylesheet/stylus'>
$ease-out = all .3s cubic-bezier(0.23, 1, 0.32, 1)

ul.drag-list, ul.drag-inner-list {
  list-style-type: none
  margin: 0
  padding: 0
}

.drag-container {
  max-width: 1000px
  margin: 20px auto
}

.drag-list {
  display: flex
  align-items: flex-start

  @media (max-width: 690px) {
      display: block
  }
}

.drag-column {
  flex: 1
  margin: 0 10px
  position: relative
  background: rgba(black, 0.2)
  overflow: hidden

  @media (max-width: 690px) {
    margin-bottom: 30px
  }

  h2 {
    font-size: 0.8rem
    margin: 0
    text-transform: uppercase
    font-weight: 600
  }
}

.drag-column-header {
  display: flex
  align-items: center
  justify-content: space-between
  padding: 10px
}

.drag-inner-list {
  min-height: 50px
  color: white
}

.drag-item {
  padding: 10px
  margin: 10px
  height: 100px
  background: rgba(black, 0.4)
  transition: $ease-out

  &.is-moving {
    transform: scale(1.5)
    background: rgba(black, 0.8)
  }
}

.drag-header-more {
  cursor: pointer
}

.drag-options {
  position: absolute
  top: 44px
  left: 0
  width: 100%
  height: 100%
  padding: 10px
  transform: translateX(100%)
  opacity: 0
  transition: $ease-out

  &.active {
    transform: translateX(0)
    opacity: 1
  }

  &-label {
    display: block
    margin: 0 0 5px 0

    input {
      opacity: 0.6
    }

    span {
      display: inline-block
      font-size: 0.9rem
      font-weight: 400
      margin-left: 5px
    }
  }
}
</style>
