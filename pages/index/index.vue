<template>
  <view class="app-camera">
    <!-- 状态栏 电池那行 -->
    <view class="status-bar" :style="`height: ${ statusBarHeight }px;`"></view>
    <!-- 导航栏 标题那行 -->
    <view
      class="header" 
      :style="`height: ${ headerHeight }rpx; top: ${ statusBarHeight }px;`"
    >
      <view class="title-text">APP端LivePusher模拟非全屏相机</view>
    </view>
    <!--  #ifdef  APP-PLUS -->
    <!-- 相机 -->
    <template v-if="!snapped">
      <web-view :webview-styles="{ width: width, height: height }"></web-view>
    </template>
    <!-- 照片预览 -->
    <template v-else>
      <view
        class="img-preview"
        :style="`
          top: calc(${ statusBarHeight }px + ${ headerHeight }rpx);
          width: ${ width }px;
          height: ${ height }px;
        `"
      >
        <canvas
          canvas-id="rotate-img-canvas"
          :style="`width: ${ width }px; height: ${ height }px;`"
        ></canvas>
      </view>
    </template>
    <!--  #endif -->
    <!-- 底部 -->
    <view class="footer" :style="`height: ${ footerHeight }rpx;`">
      <template v-if="!snapped">
        <!-- 相册 -->
        <view class="title-text" @click="getImageFromAlbum">相册</view>
        <!-- 快门 -->
        <view class="outer-white-circle" @click="snapshot">
          <view class="black-circle"></view>
          <view class="inner-white-circle"></view>
        </view>
        <!-- 提示 -->
        <view class="title-text" @click="handleClickShowGuide">提示</view>
      </template>
      <template v-else>
        <view class="go-back title-text" @click="handleClickBack">返回</view>
      </template>
    </view>
  </view>
</template>
<script>
export default {
  data() {
    return {
      width: 0, // 相机容器宽度
      height: 0, // 相机容器高度
      top: 0, // 相机容器距离顶部高度
      containView: null, // 当前页面的webView
      livePusher: null, // 模拟相机的LivePusher
      snapped: false, // 是否已经拍照
      statusBarHeight: 0, // 状态栏高度
      headerHeight: 180, // 顶部高度 rpx
      footerHeight: 364, // 底部高度 rpx
      guideView: null, // 拍照指南view
      windowHeight: 0, // 屏幕高度
      maskView: null, // 盖在LivePusher上的view
    };
  },
  created() {
    const sysInfo = uni.getSystemInfoSync();
    this.statusBarHeight = sysInfo.statusBarHeight;
    this.windowHeight = sysInfo.windowHeight;
    this.width = uni.upx2px(750); // rpx转px
    this.height = this.windowHeight - uni.upx2px(this.headerHeight + this.footerHeight);
    this.top = uni.upx2px(this.headerHeight) + this.statusBarHeight;
  },
  mounted() {
    this.initCamera();
  },
  methods: {
    /*
      初始化相机
    */
    initCamera() {
      // #ifdef APP-PLUS
      // 用livePusher模拟相机
      let currentWebview = this.$scope.$getAppWebview();
      this.containView = currentWebview.children()[0];
      this.containView.setStyle({ top: this.top, left: 0 });
      this.livePusher = plus.video.createLivePusher("simulatedCamera", {
        url: "",
        mode: "SD",
        muted: true,
        top: "0px",
        left: "0px",
        orientation: "vertical",
      });
      this.containView.append(this.livePusher);
      // 创建蒙在上面的view
      const maskViewStyles = {
        top: this.top + "px",
        left: 0 + "px",
        width: this.width + "px",
        height: this.height + "px",
      };
      const leftMaskTag = {
        tag: "rect",
        id: "left-mask-tag",
        color: "rgba(0, 0, 0, 0.5)",
        position: {
          top: "0px",
          left: "0px",
          height: "100%",
          width: uni.upx2px(75) + "px",
        },
      };
      const rightMaskTag = {
        tag: "rect",
        id: "right-mask-tag",
        color: "rgba(0, 0, 0, 0.5)",
        position: {
          top: "0px",
          right: "0px",
          height: "100%",
          width: uni.upx2px(75) + "px",
        },
      };
      const tags = [leftMaskTag, rightMaskTag];
      const diff = this.height - uni.upx2px(600);
      if (diff > 0) {
        const topMaskTag = {
          tag: "rect",
          id: "top-mask-tag",
          color: "rgba(0, 0, 0, 0.5)",
          position: {
            top: "0px",
            left: uni.upx2px(75) + "px",
            height: diff / 2 + "px",
            width: (uni.upx2px(600) + 1) + "px",
          },
        };
        const bottomMaskTag = {
          tag: "rect",
          id: "bottom-mask-tag",
          color: "rgba(0, 0, 0, 0.5)",
          position: {
            bottom: "0px",
            left: uni.upx2px(75) + "px",
            height: diff / 2 + "px",
            width: (uni.upx2px(600) + 1) + "px",
          },
        };
        tags.push(topMaskTag);
        tags.push(bottomMaskTag);
      }
      this.maskView = new plus.nativeObj.View("mask", maskViewStyles, tags);
      this.maskView.show();
      // #endif
    },
    /*
      点击拍照键的处理函数
    */
    snapshot() {
      // #ifdef APP-PLUS
      if (!this.livePusher) return;
      this.livePusher.snapshot(
        (event) => {
          // 获取图片地址
          const rawImg = event.tempImagePath;
          this.snapped = true;
          this.maskView.close();
          // snapshot拍出来的照片莫名奇妙就是顺时针转90度的
          // 下面 逆时针转90度 相当于顺时针270度
          this.$nextTick(() => {
            const ctx = uni.createCanvasContext("rotate-img-canvas", this);
            ctx.translate(this.width / 2, this.height / 2);
            ctx.rotate(3 * Math.PI / 2);
            ctx.drawImage(rawImg, this.height / -2, this.width / -2, this.height, this.width);
            ctx.draw();
            uni.canvasToTempFilePath({
              canvasId: "rotate-img-canvas",
              success: (res) => {
                console.log("拍照获取的URL: ", res.tempFilePath);
              },
              fail: (err) => {
                console.log("canvas转URL失败: ", err);
              }
            });
          });
        },
        (err) => {
          console.log("快照报错: ", err);
          this.snapped = false;
          this.$nextTick(() => {
            this.initCamera();
          });
        }
      );
      // #endif
    },
    /*
      返回上一页
    */
    routerBack() {
      if (this.snapped) {
        this.snapped = false;
        this.$nextTick(() => {
          this.initCamera();
        });
      } else {
        uni.navigateBack({ delta: 1 });
      }
    },
    /*
      从相册中选择照片
    */
    getImageFromAlbum() {
      this.snapped = true;
      this.maskView.close();
      // 选择相册照片
      uni.chooseImage({
        count: 1,
        sizeType: ["compressed"],
        sourceType: ["album"],
        success: (res) => {
          // 获取照片信息
          uni.getImageInfo({
            src: res.tempFilePaths[0],
            success: (img) => {
              // 绘制照片
              this.$nextTick(() => {
                const ctx = uni.createCanvasContext("rotate-img-canvas", this);
                ctx.drawImage(img.path, 0, 0, this.width, this.height);
                ctx.draw();
                uni.canvasToTempFilePath({
                  canvasId: "rotate-img-canvas",
                  success: (res) => {
                    console.log("拍照获取的URL: ", res.tempFilePath);
                  },
                  fail: (err) => {
                    console.log("canvas转URL失败: ", err);
                  }
                });
              });
            },
            fail: (err) => {
              console.log("绘制照片报错: ", err);
              this.snapped = false;
              this.$nextTick(() => {
                this.initCamera();
              });
            }
          });
        },
        fail: (err) => {
          console.log("选择照片报错: ", err);
          this.snapped = false;
          this.$nextTick(() => {
            this.initCamera();
          });
        }
      });
    },
    /*
      点击显示拍照指南
    */
    handleClickShowGuide() {
      // guideView的样式
      const guideViewStyles = {
        top: "0px",
        left: "0px",
        height: "100%",
        width: "100%"
      };
      // 背景
      const backgroundTag = {
        tag: "rect",
        id: "guide-background",
        color: "rgba(0, 0, 0, 0.5)",
        position: {
          top: "0px",
          left: "0px",
          height: "100%",
          width: "100%"
        },
      };
      // 提示
      const guideTagWidth = uni.upx2px(640);
      const guideTagHeight = uni.upx2px(530);
      const guideTag = {
        tag: "rect",
        id: "guide-container",
        color: "#FFFFFF",
        position: {
          width: guideTagWidth + "px",
          height: guideTagHeight + "px",
          left: uni.upx2px(55) + "px",
          top: (this.windowHeight - guideTagHeight) / 2 + "px",
        },
        rectStyles: {
          radius: uni.upx2px(16) + "px"
        },
      }
      // 标题
      const titleTag = {
        tag: "font",
        id: "guide-title",
        position: {
          width: uni.upx2px(192) + "px",
          height: uni.upx2px(45) + "px",
          left: uni.upx2px(279) + "px",
          top: ((this.windowHeight - guideTagHeight) / 2) + uni.upx2px(38) + "px",
        },
        text: "如何正确拍照",
        textStyles: {
          color: "#333333",
          size: uni.upx2px(32) + "px",
          weight: "bold",
        },
      };
      // 提示文本
      const tipTag = {
        tag: "font",
        id: "guide-tip",
        position: {
          width: uni.upx2px(522) + "px",
          height: uni.upx2px(74) + "px",
          left: uni.upx2px(125) + "px",
          top: ((this.windowHeight - guideTagHeight) / 2) + uni.upx2px(120) + "px",
        },
        text: "请保证图片清晰，目标物体清晰可见，并拍摄完整目标物体情况。",
        textStyles: {
          align: "left",
          color: "#333333",
          size: uni.upx2px(26) + "px",
          whiteSpace: "normal"
        },
      };
      // 提示图片
      const imgTag = {
        tag: "img",
        id: "guide-img",
        src: "static/logo.png", // 不支持svg
        position: {
          width: uni.upx2px(218) + "px",
          height: uni.upx2px(218) + "px",
          left: uni.upx2px(276) + "px",
          top: ((this.windowHeight - guideTagHeight) / 2) + uni.upx2px(235) + "px",
        }
      };
      const tags = [backgroundTag, guideTag, titleTag, tipTag, imgTag];
      // 创建guideView并绘制
      this.guideView = new plus.nativeObj.View("guide", guideViewStyles, tags);
      this.guideView.addEventListener("click", () => {
        this.guideView.close();
      });
      this.guideView.show();
    },
    /*
      点击返回按钮
    */
    handleClickBack() {
      this.snapped = false;
      this.$nextTick(() => {
        this.initCamera();
      });
    },
  },
}
</script>
<style scoped>
.app-camera {
  width: 100vw;
  height: 100vh;
  position: relative;
  background-color: black;
}
.status-bar {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
}
.header {
  position: absolute;
  left: 0;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
.title-text {
  font-weight: 500;
  font-size: 36rpx;
  color: #FFFFFF;
}
.footer {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 35rpx;
  box-sizing: border-box;
}
.outer-white-circle {
  width: 126rpx;
  height: 126rpx;
  background-color: white;
  border-radius: 50%;
  position: relative;
}
.black-circle {
  width: 116rpx;
  height: 116rpx;
  background-color: black;
  border-radius: 50%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.inner-white-circle {
  width: 104rpx;
  height: 104rpx;
  background-color: white;
  border-radius: 50%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1;
}
.img-preview {
  position: absolute;
  left: 0;
}
.go-back {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>