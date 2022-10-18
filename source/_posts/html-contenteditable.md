---
title: 关于html contenteditable="true" 在移动端@功能的使用总结
author: sue
date: 2022-10-14 10:31:31
tags: contenteditable
categories: vue
---

给标签添加contenteditable属性可以实现富文本编辑框，随意定义内容的样式


```html
<div contenteditable="true"></div>
```
## 监听输入，获取内容
``` html
<div 
  ref="editorCon"
  contenteditable="true"
  @input="handleInput">
</div>
```
``` js
handleInput (e) {
  // 去除用户在输入时，主动输入的换行符
  const html = this.$refs.editorCon.innerHTML.toString().replace(/<br>/g, '')
  this.$emit('input', html)
}

```

## 获取光标，插入内容
```js
// 获取光标，插入html
pasteHtmlAtCaret ($html) {
  let sel, range
  // IE9 and non-IE
  if (window.getSelection) {
    sel = window.getSelection()

    if (sel && sel.rangeCount) range = sel.getRangeAt(0)
    if (['', null, undefined].includes(range)) {
      range = this.keepCursorEnd(true).getRangeAt(0)
    } else {
      const contentRange = document.createRange()
      contentRange.selectNode(this.$refs.editorCon)

      const compareStart = range.compareBoundaryPoints(
        Range.START_TO_START,
        contentRange
      )
      const compareEnd = range.compareBoundaryPoints(
        Range.END_TO_END,
        contentRange
      )
      const compare = compareStart !== -1 && compareEnd !== 1
      if (!compare) range = this.keepCursorEnd(true).getRangeAt(0)
    }
    let input = range.createContextualFragment($html)
    let lastNode = input.lastChild // 记录插入input之后的最后节点位置
    range.insertNode(input)
    if (lastNode) {
      // 如果有最后的节点
      range = range.cloneRange()
      range.setStartAfter(lastNode)
      range.collapse(true)
      sel.removeAllRanges()
      sel.addRange(range)
    }
  } else if (
    document['selection'] &&
    document['selection'].type !== 'Control'
  ) {
    // IE < 9
    document['selection'].createRange().pasteHTML($html)
  }
  // 解决最后一个是点击输入未传入的情况
  // 去除用户在输入时，主动输入的换行符
  const html = this.$refs.editorCon.innerHTML.toString().replace(/<br>/g, '')
  this.$emit('input', html)
},

/**
  * 将光标重新定位到内容最后
  * isReturn 是否要将range实例返回
  * */
keepCursorEnd ($isReturn) {
  if (window.getSelection) {
    // ie11 10 9 firefox safari
    this.$refs.editorCon.focus()
    let sel = window.getSelection() // 创建range
    sel.selectAllChildren(this.$refs.editorCon) // range 选择obj下所有子内容
    sel.collapseToEnd() // 光标移至最后
    if ($isReturn) return sel
  } else if (document['selection']) {
    // ie9以下
    let sel = document['selection'].createRange() // 创建选择对象
    sel.moveToElementText(this.$refs.editorCon) // range定位到编辑器
    sel.collapse(false) // 光标移至最后
    sel.select()
    if ($isReturn) return sel
  }
}
```

## 粘贴时会有原本样式，不要保留的话需要消除样式
``` html
<div 
  contenteditable="true"
  @paste="HandlePaste">
</div>
```
```js
  // 粘贴时消除样式
  HandlePaste (e) {
    e.stopPropagation()
    e.preventDefault()
    let text = ''
    const event = e.originalEvent || e
    if (event.clipboardData && event.clipboardData.getData) {
      text = event.clipboardData.getData('text/plain')
    } else if (window['clipboardData'] && window['clipboardData'].getData) {
      text = window['clipboardData'].getData('Text')
    }

    // 清除回车
    text = text.replace(/\[\d+\]|\n|\r/ig, '')

    // 检查浏览器是否支持指定的编辑器命令
    if (document.queryCommandSupported('insertText')) {
      document.execCommand('insertText', false, text)
    } else {
      document.execCommand('paste', false, text)
    }
  }
```

## 常见问题

#### ios下点击软键盘弹出但是无光标显示
出现的原因是：生成了默认样式： `-webkit-user-select:none;`（无法选中，导致出现问题）。需在该元素上添加以下css：
``` css
-webkit-user-select: text;
user-select: text;
```

#### 聚焦难，ios机型需要双击或者长按才会获取到焦点
由于项目中使用了Fastclick插件导致无法聚焦。
fastclick在ios条件下ontouchend方法中若needsClick（button、select、textarea、input、label、iframe、video 或类名包含needsclick）为不可点击防止了事件冒泡（preventDefault），所以出现无法点击情况。

##### 1.给该元素添加类needsclick，不阻止事件冒泡
``` html
<div 
  class="needsclick"
  contenteditable="true">
</div>
```

##### 2.重写FastClick

```js main.js
import fastclick from 'fastclick'
fastclick.attach(document.body)

// 用来解决fastclick在ios需要多次点击才能获取焦点的问题
var deviceIsWindowsPhone = navigator.userAgent.indexOf('Windows Phone') >= 0
var deviceIsIOS = /iP(ad|home|od)/.test(navigator.userAgent) && !deviceIsWindowsPhone
fastclick.prototype.focus = function (targetElement) {
  var length

  // Issue #160: on iOS 7, some input elements (e.g. date datetime month) throw a vague TypeError on setSelectionRange. These elements don't have an integer value for the selectionStart and selectionEnd properties, but unfortunately that can't be used for detection because accessing the properties also throws a TypeError. Just check the type instead. Filed as Apple bug #15122724.
  if (deviceIsIOS && targetElement.setSelectionRange && targetElement.type.indexOf('date') !== 0 && targetElement.type !== 'time' && targetElement.type !== 'month') {
    length = targetElement.value.length
    targetElement.setSelectionRange(length, length)
  } else {
    targetElement.focus()
  }
  // 新增一行：都获取焦点
  targetElement.focus()
}
```
#### 添加placeholder
``` html
<div 
  class="ss-editor needsclick"
  placeholder="请输入内容"
  contenteditable="true">
</div>
```
```less
.ss-editor {
  &:empty::before {
    content: attr(placeholder);
    color: #cccccc;
  }
}
```







