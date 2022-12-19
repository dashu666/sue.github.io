---
title: 小程序canvas彩色二维码生成
author: sue
date: 2022-12-06 15:34:50
tags:
categories:
---
参考：https://www.gxlcms.com/JavaScript-237527.html

利用canvas drawImage方法把生成的黑白二维码画到canvas上，canvasGetImageData方法获得一个矩形区域所有像素点的信息，每四个一组，分别代表r,g,b和透明度。修改颜色后，利用canvasPutImageData方法，把更改过的像素信息数组重新扔回画布上。

这里用到了深度优先搜索去遍历图形，然后给码点进行对应的颜色赋值

##### 关于深度优先搜索(DFS)
深度优先搜索是常见的图搜索方法之一。会沿着一条路径一直搜索下去，在无法搜索时，回退到刚刚访问过的节点。深度优先遍历按照深度优先搜索的方式对图进行遍历。并且每个节点只能访问一次。

深搜优先搜索的本质上就是持续搜索，遍历了所有可能的情况，必然能得到解。DFS搜索的流程是一个树的形式，每次一条路走到黑。

##### 深度优先搜索算法步骤
* 步骤1：初始化图中的所有节点为均未被访问。
* 步骤2：从图中的某个节点v出发，访问v并标记其已被访问。
* 步骤3：依次检查v的所有邻接点w，如果w未被访问，则从w出发进行深度优先遍历（递归调用，重复步骤2和3）。


代码：[https://github.com/suesoft/color-qrcode](https://github.com/suesoft/uniapp-common-components/blob/master/src/pages/component/color-qrcode/color-qrcode.vue)

```html
  <canvas
    canvas-id="secondCanvas"
    :style="{
      width: width + 'px',
      height: height + 'px',
    }"
  >
  </canvas>
```
```js
<script>
export default {
  props: {
    // 必传。原二维码图片（临时路径）
    codeImg: {
      type: String,
      default: "",
    },
    width: {
      type: Number,
      default: 200,
    },
    height: {
      type: Number,
      default: 200,
    },
    // 渲染的颜色
    // 不传默认#333， 传单个颜色默认全部渲染， 多个颜色需要传方向，不传就默认随机
    colors: {
      type: Array,
      default: ["#333333"],
    },
    // 颜色渲染方向。
    // toRight: 从左往右 toBottom: 从上往下 toRightBottom: 从左上到右下
    direction: {
      type: String,
      default: "",
    },
  },
  data() {
    return {
      book: [], // 标记数组
      imgD: null, // 预留给像素信息数组
    };
  },
  created() {
    this.drawCanvas();
  },

  methods: {
    // 画图
    drawCanvas() {
      const that = this;
      const bg = this.colorRgb("#fff"); // 忽略的背景色

      this.book = [];
      for (var i = 0; i < this.height; i++) {
        this.book[i] = [];
        for (var j = 0; j < this.width; j++) {
          this.book[i][j] = 0;
        }
      }

      // 随机colors数组的一个序号
      var ranNum = (function () {
        const len = that.colors.length;
        return function () {
          return Math.floor(Math.random() * len);
        };
      })();

      // canvas 部分
      const { width, height } = this;
      const ctx = uni.createCanvasContext("secondCanvas", this);
      ctx.drawImage(this.codeImg, 0, 0, width, height);
      ctx.draw(
        setTimeout(() => {
          uni.canvasGetImageData(
            {
              canvasId: "secondCanvas",
              x: 0,
              y: 0,
              width,
              height,
              success: (res) => {
                this.imgD = res;

                for (let i = 0; i < height; i++) {
                  for (let j = 0; j < width; j++) {
                    if (
                      this.book[i][j] == 0 &&
                      this.checkColor(i, j, width, bg)
                    ) {
                      // 没标记过 且是非背景色
                      this.book[i][j] = 1;

                      var color = that.colorRgb(this.colors[ranNum()]);
                      this.dfs(i, j, color, bg); // 深度优先搜索
                    }
                  }
                }

                setTimeout(() => {
                  uni.canvasPutImageData(
                    {
                      canvasId: "secondCanvas",
                      x: 0,
                      y: 0,
                      width: width,
                      height: height,
                      data: this.imgD.data,
                      success(res) {
                        console.log("=============2===============", res);
                      },
                      fail(err) {
                        console.log("=============2=====err", err);
                      },
                    },
                    this
                  );
                }, 500);
              },
            },
            this
          );
        }, 500)
      );
    },

    // 深度优先搜索
    async dfs(x, y, color, bg) {
      // 必须执行完成数据更新后 再执行后面的递归，不然会导致死循环，堆栈溢出
      await this.changeColor(x, y, color);

      // 方向数组
      const next = [
        [0, 1], // 右
        [1, 0], // 下
        [0, -1], // 左
        [-1, 0], // 上
      ];
      for (var k = 0; k <= 3; k++) {
        // 下一个坐标
        var tx = x + next[k][0];
        var ty = y + next[k][1];

        // 判断越界
        if (tx < 0 || tx >= this.height || ty < 0 || ty >= this.width) {
          continue;
        }

        if (this.book[tx][ty] == 0 && this.checkColor(tx, ty, this.width, bg)) {
          // 判断位置
          this.book[tx][ty] = 1;
          this.dfs(tx, ty, color, bg);
        }
      }
      return;
    },

    // 验证该位置的像素 不是背景色为true
    checkColor(i, j, width, bg) {
      var x = this.calc(width, i, j);

      if (
        this.imgD.data[x] != bg[0] &&
        this.imgD.data[x + 1] != bg[1] &&
        this.imgD.data[x + 2] != bg[2]
      ) {
        return true;
      } else {
        return false;
      }
    },

    // 改变颜色值
    changeColor(i, j, colorArr) {
      var x = this.calc(this.width, i, j);
      this.imgD.data[x] = colorArr[0];
      this.imgD.data[x + 1] = colorArr[1];
      this.imgD.data[x + 2] = colorArr[2];
    },

    // 返回对应像素点的序号
    calc(width, i, j) {
      if (j < 0) {
        j = 0;
      }
      return 4 * (i * width + j);
    },

    // 分离颜色参数 返回一个数组
    colorRgb(str) {
      const reg = /^#([0-9a-fA-f]{3}|[0-9a-fA-f]{6})$/;
      var sColor = str.toLowerCase();
      if (sColor && reg.test(sColor)) {
        if (sColor.length === 4) {
          var sColorNew = "#";
          for (var i = 1; i < 4; i += 1) {
            sColorNew += sColor.slice(i, i + 1).concat(sColor.slice(i, i + 1));
          }
          sColor = sColorNew;
        }
        // 处理六位的颜色值
        var sColorChange = [];
        for (var i = 1; i < 7; i += 2) {
          sColorChange.push(parseInt("0x" + sColor.slice(i, i + 2)));
        }
        return sColorChange;
      } else {
        var sColorChange = sColor
          .replace(/(rgb\()|(\))/g, "")
          .split(",")
          .map(function (a) {
            return parseInt(a);
          });
        return sColorChange;
      }
    },
  },
};
</script>
```