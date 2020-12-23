<template>

  <!-- 掃描框 -->
  <div id="scanner">
    <div class="model">
      <div class="scanner-view">
        <div class="scanner-view-arrow arrow1"></div>
        <div class="scanner-view-arrow arrow2"></div>
        <div class="scanner-view-arrow arrow3"></div>
        <div class="scanner-view-arrow arrow4"></div>
        <div class="scanner-line"></div>
      </div>
    </div>

    <!-- 攝像頭開啟後，將畫面顯示於此 -->
    <video class="video-view" ref="video" autoplay playsinline="true" webkit-playsinline="true"></video>

    <!-- 儲存由攝像頭影片截圖取得的相片-->
    <canvas ref="canvas" width="478" height="850" style="display: none"></canvas>

  </div>
</template>

<script>
  import jsQR from "jsqr";
  import Quagga from "quagga";
  export default {
    name: '',
    data() {
      return {
        cameraWidth: 0,
        cameraHeight: 0
      }
    },
    methods: {
      initVideo(constrains){

        let _this = this;

        // 關於navigator.mediaDevices:
        // 1.就是要開啟用戶攝像頭
        // 2.該API需要https
        // 3.開啟成功會將影片流回傳，詳細介紹: https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
        // 4.其他舊的API詳細介紹: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getUserMedia
        if(navigator.mediaDevices.getUserMedia){
          //最新標準API
          navigator.mediaDevices.getUserMedia(constrains).then(_this.videoSuccess).catch(_this.videoError);
        } else if (navigator.webkitGetUserMedia){
          //webkit內核瀏覽器
          navigator.webkitGetUserMedia(constrains).then(_this.videoSuccess).catch(_this.videoError);
        } else if (navigator.mozGetUserMedia){
          //Firefox瀏覽器
          navigator.mozGetUserMedia(constrains).then(_this.videoSuccess).catch(_this.videoError);
        } else if (navigator.getUserMedia){
          //舊版API
          navigator.getUserMedia(constrains).then(_this.videoSuccess).catch(_this.videoError);
        }
      },

      // 成功開啟攝像頭後，會取得回傳的畫面，
      videoSuccess(stream){
        // 設置多個變數
        let video = this.$refs.video,
            _this = this;

        // 將視頻流設置為 video 元素的源
        video.srcObject = stream;

        // 播放視頻
        video.play();

        // 影片偵測可以撥放時觸發
        video.oncanplay = function () {
          // 攝像頭分辨率,手機480x640
          console.log('攝像頭分辨率');
          console.log(video.videoWidth,video.videoHeight);
          _this.cameraWidth = video.videoWidth;
          _this.cameraHeight = video.videoHeight;

          // 發送圖片進行辨識
          _this.readImg();
        };
      },
      videoError(error){
        console.log("訪問用戶媒體設備失敗：",error.name,error.message);
        alert("訪問用戶媒體設備失敗：",error.name,error.message);
      },
      readImg(){

        // 宣告變數初始化
        let video = this.$refs.video,
            canvas = this.$refs.canvas,
            context = canvas.getContext("2d"), // The getContext() function returns the drawing context - which is an object that has all the drawing properties and functions you use to draw on the canvas.
            _this = this;

        // 目前每秒掃一次條碼/二維碼
        let timer = setInterval(function () {
          // 掃描條碼
          let imgUri = canvas.toDataURL(); // 轉base64
          _this.readBarcode(imgUri,timer); // 反解取得條碼內容

          // 掃描二維碼
          // 將影片畫出來，詳見: https://www.w3schools.com/tags/tryit.asp?filename=tryhtml5_canvas_drawimage_video
          context.drawImage(video,0,0,_this.cameraWidth,_this.cameraHeight,0,0,478,850);
          let imageData = context.getImageData(0, 0, 478, 850); // 轉為 ImageData Object型別, 又為underlying pixel data(基礎像素數據)
          _this.readQrcode(imageData.data,timer); // 反解取得二維碼內容
        },1000)

      },
      readBarcode(imgBase64,timer){
        let _this = this;
        Quagga.decodeSingle({
          inputStream: {
            size: 1920
          },
          locator: {
            patchSize: "medium",
            halfSample: false
          },
          decoder: {
            // readers: [{
            //   format: "code_128_reader",
            //   config: {}
            // }]
            readers : [
              "code_128_reader",
              "ean_reader",
              "ean_8_reader",
              "code_39_reader",
              "code_39_vin_reader",
              "codabar_reader",
              "upc_reader",
              "upc_e_reader",
              "i2of5_reader",
              "2of5_reader",
              "code_93_reader"
            ]
          },
          locate: true,
          src: imgBase64
        }, function(result){
          if (result){
            if(result.codeResult) {
              console.log(result.codeResult);
              clearInterval(timer);
              _this.$emit('ondata',result.codeResult.code);
              // alert("掃描成功，結果是..."+result.codeResult.code);

            } else {
              console.log("正在掃條碼...not detected");
              // alert("正在掃條碼...not detected");
            }
          }else {
            console.log("正在掃條碼...not detected");
            // alert("正在掃條碼...not detected");
          }

        });
      },
      readQrcode(data,timer){
        let _this = this;
        let code = jsQR(data, 478, 850, {
          inversionAttempts: "dontInvert",
        });

        if (code){
          clearInterval(timer);
          _this.$emit('ondata',code.data);
          alert('掃二維碼成功，結果是...' + code.data);
        }else {
          console.log('正在掃二維碼...');
          // alert("正在二維碼...not detected");
        }
      }
    },

    // 頁面初始化後，載入資料階段
    mounted(){
      if (navigator.mediaDevices.getUserMedia || navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia){
        //調用用戶媒體設備，訪問攝像頭
        this.initVideo({
          video:{
            height: 800,
            facingMode: {
              // 強制後置攝像頭
              exact: "environment"
            }
          }
        });
      } else {
        alert("您的瀏覽器不支持訪問用戶媒體設備");
      }
    }
  }
</script>

<style scoped>
  #scanner {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    position: relative;
  }
  .model{
    box-sizing: border-box;
    width: 100vw;
    height: 100vh;
    position: relative;
    z-index: 88;
    border-top: calc((100vh - 60vw)/2) solid rgba(0,0,0,.2);
    border-bottom: calc((100vh - 60vw)/2) solid rgba(0,0,0,.2);
    border-right: 20vw solid rgba(0,0,0,.2);
    border-left: 20vw solid rgba(0,0,0,.2);
  }
  .scanner-view{
    width: 100%;
    height: 100%;
    position: relative;
    border: 1px solid rgba(255,255,255,.3);
    z-index: 89;
  }
  .scanner-line{
    position: absolute;
    width: 100%;
    height: 1px;
    background: #49FF46;
    border-radius: 20px;
    z-index: 90;
    animation: myScan 1s infinite alternate;
  }
  @keyframes  myScan{
    from {
      top: 0;
    }
    to {
      top: 34vh;
    }
  }
  .scanner-view-arrow{
    position: absolute;
    width: 5vw;
    height: 5vw;
    border: 2px solid #09bb07;
  }
  .scanner-view-arrow.arrow1{
    top: -1px;
    left: 0px;
    z-index: 99;
    border-right: none;
    border-bottom: none;
  }
  .scanner-view-arrow.arrow2{
    top: -1px;
    right: 0px;
    z-index: 99;
    border-left: none;
    border-bottom: none;
  }
  .scanner-view-arrow.arrow3{
    bottom: -1px;
    left: 0px;
    z-index: 99;
    border-right: none;
    border-top: none;
  }
  .scanner-view-arrow.arrow4{
    bottom: -1px;
    right: 0px;
    z-index: 99;
    border-left: none;
    border-top: none;
  }
  .video-view{
    position: absolute;
    width: 100vw;
    height: 100vh;
    object-fit: cover;
    top: 0px;
    left: 0px;
    z-index: 80;
  }
</style>
