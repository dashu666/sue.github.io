---
title: 小程序中canvas生成海报总结
author: sue
date: 2022-10-18 14:12:31
tags: canvas 小程序
categories: canvas
---

```html
<canvas class="canvas" canvas-id="secondCanvas"></canvas>
```
### 生成海报二维码有时无法识别
利用weapp-qrcode生成二维码再canvas画海报图片，二维码有时无法识别
原有问题的代码：
```js

// 生成二维码
import drawQrcode from '@/utils/weapp-qrcode.js'

drawQrcode({
  width: size.w,
  height: size.h,
  canvasId: 'mycanvas',
  text: '这里是二维码链接url',
  callback: () => {
    setTimeout(() => {
      uni.canvasToTempFilePath({
        fileType: "png",
        canvasId: "mycanvas",
        success: (res) => {
          console.log('生成的二维码本地图片', res.tempFilePath);
        },
        fail: function (err) {
          console.log(err, "二维码图片生成失败");
        },
      });
    }, 100)
  }
})

```
使用weapp-qrcode-base64，将链接url存为base64图片
修改后的代码：
```js
import drawQrcode from '@/utils/weapp-qrcode-base64.js'

var base64ImgData = drawQrcode.drawImg(
  '这里是二维码链接url', 
  {
    typeNumber: 4,
    errorCorrectLevel: 'M',
    size: 500
  }
)
const templateCodeImg = await this.Base64ToImage(base64ImgData)
console.log('生成的二维码本地图片', templateCodeImg)

```
```js
/**
* 将base64转成二进制保存在本地
* @param {Object} $data base64数据
 */
Base64ToImage($data) {
  return new Promise((resolve, reject) => {
    const fsm = wx.getFileSystemManager(); // 声明文件系统
    
    var times = new Date().getTime(); // 随机定义路径名称
    var codeImg = wx.env.USER_DATA_PATH + '/' + times + '.png';
    // 将base64图片写入
    fsm.writeFile({
      filePath: codeImg,
      data: $data.slice(22),
      encoding: 'base64',
      success: () => {
        resolve(codeImg)
      },
      fail: () => {
        console.log('base64存储失败')
        resolve('')
      }
    })
  })
}
```

### 微信小程序canvas绘制圆角图片
```js
/**
 * 绘制圆角图片
 * @param {Object} ctx
 * @param {Object} r 圆角的弧度大小
 * @param {Object} x 文本起始x坐标
 * @param {Object} y 文本起始y坐标
 * @param {Object} w 宽度
 * @param {Object} h 高度
 * @param {Object} img 图片
 */
drawRoundRect(ctx, r, x, y, w, h, img) {
  ctx.save();
  if (w < 2 * r) r = w / 2;
  if (h < 2 * r) r = h / 2;
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.arcTo(x + w, y, x + w, y + h, r);
  ctx.arcTo(x + w, y + h, x, y + h, r);
  ctx.arcTo(x, y + h, x, y, r);
  ctx.arcTo(x, y, x + w, y, r);
  ctx.closePath();
  ctx.clip();
  ctx.drawImage(img, x, y, w, h);
  ctx.restore()
}
```

### 微信小程序canvas绘制圆角矩形
```js
/**
   * 
   * @param {CanvasContext} ctx canvas上下文
   * @param {number} x 圆角矩形选区的左上角 x坐标
   * @param {number} y 圆角矩形选区的左上角 y坐标
   * @param {number} w 圆角矩形选区的宽度
   * @param {number} h 圆角矩形选区的高度
   
   // 此处有两个圆角半径是项目需要，只展示左上角 右上角的圆角或左下角 右下角的圆角
   * @param {number} r 圆角的半径   左上角 右上角
   * @param {number} r1 圆角的半径 左下角 右下角
   */
  roundRect(ctx, x, y, w, h, r, r1) {
    // 开始绘制
    ctx.save() // 先保存状态 已便于画完后面再用
    ctx.beginPath()
    // 因为边缘描边存在锯齿，最好指定使用 transparent 填充
    // 这里是使用 fill 还是 stroke都可以，二选一即可
    ctx.setFillStyle('rgba(0,0,0,.3)')
    // ctx.setStrokeStyle('transparent')

    // 左上角
    ctx.arc(x + r, y + r, r, Math.PI, Math.PI * 1.5)
    
    // border-top
    ctx.moveTo(x + r, y)
    ctx.lineTo(x + w - r, y)
    ctx.lineTo(x + w, y + r)
    // 右上角
    ctx.arc(x + w - r, y + r, r, Math.PI * 1.5, Math.PI * 2)
    
    // border-right
    ctx.lineTo(x + w, y + h - r1)
    ctx.lineTo(x + w - r1, y + h)
    // 右下角
    ctx.arc(x + w - r1, y + h - r1, r1, 0, Math.PI * 0.5)
    
    // border-bottom
    ctx.lineTo(x + r1, y + h)
    ctx.lineTo(x, y + h - r1)
    // 左下角
    ctx.arc(x + r1, y + h - r1, r1, Math.PI * 0.5, Math.PI)
    
    // border-left
    ctx.lineTo(x, y + r)
    ctx.lineTo(x + r, y)
    
    // 这里是使用 fill 还是 stroke都可以，二选一即可，但是需要与上面对应
    ctx.fill()
    // ctx.stroke()
    ctx.closePath()
    // 剪切
    ctx.clip()
  },
```

### canvas绘制自动换行的字符串（接口返回的文本）
```js
/**
 * 绘制自动换行的字符串
 * @param {Object} ctx
 * @param {Object} t 文本
 * @param {Object} x 文本起始x坐标
 * @param {Object} y 文本起始y坐标
 * @param {Object} w 文本最大宽度
 */
drawText(ctx, t,x,y,w){
  var tArr = t.split("\n");
  var result = []
  
  tArr.map((item ,index) => {
    var chr = item.split("");
    var temp = "";
    var row = [];
    
    chr.map((itm, idx) => {
      if (ctx.measureText(temp).width < w ) {
        ;
      } else{
        row.push(temp);
        temp = "";
      }
      temp += itm;
    })
    row.push(temp);
    row.map(itm => {
      result.push(itm)
    })
  })

  result.map((itm, idx) => {
    ctx.fillText(itm,x,y+(idx+1)*20);
  })
}
```