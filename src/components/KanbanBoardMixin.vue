<template>
<div></div>
</template>
<script type='text/ecmascript-6'>
import crossvent from 'crossvent'
export default {
  props: {
    options: {
      default () {
        return {
          copy: false,
          mirrorContainer: document.body
        }
      }
    }
  },
  data () {
    return {
      o: this.options,
      copy: false,
      offsetX: 0,
      offsetY: 0,
      dragging: false,
      ignoreInputTextSelection: true,
      grabbed: null,
      item: null,
      copySortSource: false,
      source: null,
      initialSibling: null,
      currentSibling: null,
      documentElement: document.documentElement,
      mirror: false
    }
  },
  methods: {
    getParent (el) { return el.parentNode === document ? null : el.parentNode },
    isInput (el) { return el.tagName === 'INPUT' || el.tagName === 'TEXTAREA' || el.tagName === 'SELECT' || this.isEditable(el) },
    isEditable (el) {
      if (!el) { return false } // no parents were editable
      if (el.contentEditable === 'false') { return false } // stop the lookup
      if (el.contentEditable === 'true') { return true } // found a contentEditable element in the chain
      return this.isEditable(this.getParent(el)) // contentEditable is set to 'inherit'
    },
    getEventHost (e) {
      // on touchend event, we have to use `e.changedTouches`
      // see http://stackoverflow.com/questions/7192563/touchend-event-properties
      // see https://github.com/bevacqua/dragula/issues/34
      if (e.targetTouches && e.targetTouches.length) { // targetTouches当前对象上所有触摸点的列表
        return e.targetTouches[0]
      }
      if (e.changedTouches && e.changedTouches.length) {
        return e.changedTouches[0]
      }
      return e
    },
    getCoord (coord, e) {
      var host = this.getEventHost(e)
      var missMap = {
        pageX: 'clientX', // IE8
        pageY: 'clientY' // IE8
      }
      if (coord in missMap && !(coord in host) && missMap[coord] in host) {
        coord = missMap[coord]
      }
      return host[coord]
    },
    movements (remove) {
      var op = remove ? 'remove' : 'add'
      crossvent[op](this.documentElement, 'selectstart', this.preventGrabbed) // IE8
      crossvent[op](this.documentElement, 'click', this.preventGrabbed)
    },
    preventGrabbed (e) {
      if (this.grabbed) {
        e.preventDefault()
      }
    },
    end () {
      if (!this.dragging) {
        return
      }
      var item = this.copy || this.item
      console.log(this.item)
      this.drop(item, this.getParent(item))
    },
    drop (item, target) {
      var parent = this.getParent(item)
      if (this.copy && this.copySortSource && target === this.source) {
        parent.removeChild(this.item)
      }
      // if (isInitialPlacement(target)) {
      //   drake.emit('cancel', item, _source, _source)
      // } else {
      //   drake.emit('drop', item, target, _source, _currentSibling)
      // }
      // cleanup()
    },
    getOffset (el) {
      var rect = el.getBoundingClientRect() // getBoundingClientRect用于获取某个元素相对于视窗的位置集合
      return { // 滚动条 + x y距离
        left: rect.left + this.getScroll('scrollLeft', 'pageXOffset'), // scrollLeft //此属性可以获取或者设置对象的最左边到对象在当前窗口显示的范围内的左边的距离，也就是元素被滚动条向左拉动的距离。
        top: rect.top + this.getScroll('scrollTop', 'pageYOffset') // pageXOffset//置或返回当前页面相对于窗口显示区左上角的 X 位置
      }
    },
    getScroll (scrollProp, offsetProp) {
      if (typeof global[offsetProp] !== 'undefined') {
        return global[offsetProp]
      }
      if (this.documentElement.clientHeight) {
        return this.documentElement[scrollProp]
      }
      return document.body[scrollProp]
    },
    nextEl (el) {
      return el.nextElementSibling || manually()
      function manually () {
        var sibling = el
        do {
          sibling = sibling.nextSibling
        } while (sibling && sibling.nodeType !== 1)
        return sibling
      }
    },
    isCopy (item, container) {
      return typeof this.o.copy === 'boolean' ? this.o.copy : this.o.copy(item, container)
    },
    getRectWidth (rect) { return rect.width || (rect.right - rect.left) },
    getRectHeight (rect) { return rect.height || (rect.bottom - rect.top) }
  }
}
</script>
<style scoped lang='stylus' rel='stylesheet/stylus'>
</style>
