---
title: js移动端单个拖拽功能的实现
author: sue
date: 2022-10-18 10:37:16
tags:
categories: vue
---

实现思路是使用js监听鼠标实现三个事件（鼠标按下mousedown，鼠标移动mousemove，鼠标放开mouseup），来绝对定位dom的位置
```html
<div class="drag"
  @mousedown="down"
  @mousemove="move"
  @mouseup="end"
  @touchstart.stop="down"
  @touchmove.stop="move"
  @touchend.stop="end"
  :style="{ top: position.y + 'px', left: position.x + 'px' }"
>
</div>
```
初始化默认值
```js
const dx, dy; // 鼠标位置和目标dom的左上角位置 差值
...
  position: {
    x: document.body.clientWidth,
    y: document.body.clientHeight
  }
...
```

还需要定义一个flags来标识鼠标是否被按下，只有按下才执行mousemove里面具体方法

```js
// 鼠标按下， 得到鼠标位置和目标dom的左上角位置 的差值
down (event) {
  this.flags = true
  const touch = event.touches ? event.touches[0] : event

  // console.log('鼠标点所在位置', touch.clientX, touch.clientY)
  // console.log('目标dom左上角位置', event.target.offsetTop, event.target.offsetLeft)

  dx = touch.clientX - event.target.offsetLeft
  dy = touch.clientY - event.target.offsetTop
},

// 鼠标移动
move (event) {
  if (this.flags) {
    const touch = event.touches ? event.touches[0] : event

    // 定位滑块的位置
    this.position.x = touch.clientX - dx
    this.position.y = touch.clientY - dy

    // console.log('屏幕大小', screenWidth, screenHeight)

    // 限制滑块超出页面
    if (this.position.x < 0) {
      this.position.x = 0
    } else if (this.position.x > screenWidth - touch.target.clientWidth) {
      this.position.x = screenWidth - touch.target.clientWidth
    }
    if (this.position.y < 0) {
      this.position.y = 0
    } else if (screenHeight - touch.target.clientHeight - touch.clientY < 0) {
      this.position.y = screenHeight - touch.target.clientHeight
    }

    // 阻止页面的滑动默认事件
    document.addEventListener(
      'touchmove',
      function () {
        event.preventDefault()
      },
      false
    )
  }
},

// 鼠标释放
end () {
  this.flags = false
}
```