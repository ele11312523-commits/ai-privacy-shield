```html
<!DOCTYPE html>
<html lang="zh-TW">

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width, initial-scale=1.0">

<title>
AI 生物識別隱私盾
</title>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<style>

*{
    margin:0;
    padding:0;
    box-sizing:border-box;
}

body{

    background:
    radial-gradient(circle at top,
    #0f172a,
    #020617);

    color:white;

    font-family:
    "Segoe UI",
    sans-serif;

    overflow:hidden;
}

.title{

    text-align:center;

    margin-top:18px;

    font-size:42px;

    font-weight:bold;

    color:cyan;

    text-shadow:
    0 0 12px cyan,
    0 0 30px cyan;
}

.subtitle{

    text-align:center;

    margin-top:8px;

    color:#94a3b8;

    font-size:18px;
}

.main{

    display:flex;

    justify-content:center;

    margin-top:25px;
}

.container{

    position:relative;

    width:960px;

    height:720px;

    border:3px solid cyan;

    border-radius:25px;

    overflow:hidden;

    box-shadow:
    0 0 40px rgba(0,255,255,0.4);
}

.input_video{

    position:absolute;

    width:100%;
    height:100%;

    object-fit:cover;

    transform:scaleX(-1);
}

canvas{

    position:absolute;

    width:100%;
    height:100%;
}

/* ===== 掃描線 ===== */

.scan-line{

    position:absolute;

    width:100%;

    height:4px;

    background:
    linear-gradient(
    to right,
    transparent,
    cyan,
    transparent
    );

    animation:
    scan 2s linear infinite;

    z-index:5;
}

@keyframes scan{

    0%{
        top:0%;
    }

    100%{
        top:100%;
    }
}

/* ===== HUD ===== */

.hud{

    position:absolute;

    top:15px;
    left:15px;

    background:
    rgba(0,0,0,0.45);

    border:1px solid cyan;

    border-radius:14px;

    padding:15px;

    backdrop-filter:blur(5px);

    z-index:10;
}

.hud p{

    margin:6px 0;

    font-size:16px;
}

.safe{
    color:#00ff99;
}

.danger{
    color:#ff5555;
}

/* ===== LOCK ===== */

.lockText{

    position:absolute;

    top:20px;
    right:20px;

    background:
    rgba(0,255,255,0.15);

    border:1px solid cyan;

    padding:12px 18px;

    border-radius:12px;

    color:cyan;

    font-weight:bold;

    z-index:10;
}

/* ===== 控制 ===== */

.controls{

    margin-top:20px;

    text-align:center;
}

button{

    padding:16px 34px;

    border:none;

    border-radius:18px;

    background:cyan;

    color:black;

    font-size:20px;

    font-weight:bold;

    cursor:pointer;

    transition:0.3s;
}

button:hover{

    transform:scale(1.05);

    background:white;

    box-shadow:
    0 0 25px cyan;
}

/* ===== Flash ===== */

#flash{

    position:fixed;

    top:0;
    left:0;

    width:100%;
    height:100%;

    background:white;

    opacity:0;

    pointer-events:none;

    z-index:99999;

    transition:0.2s;
}

/* ===== 預覽 ===== */

#previewScreen{

    position:fixed;

    top:0;
    left:0;

    width:100%;
    height:100%;

    background:black;

    display:none;

    justify-content:center;

    align-items:center;

    z-index:50000;
}

#previewImage{

    width:100%;
    height:100%;

    object-fit:contain;
}

#closePreview{

    position:absolute;

    top:20px;
    left:20px;

    width:60px;
    height:60px;

    border-radius:50%;

    font-size:30px;

    z-index:60000;
}

#downloadBtn{

    position:absolute;

    bottom:30px;
}

.footer{

    position:absolute;

    bottom:10px;

    width:100%;

    text-align:center;

    color:#64748b;

    font-size:14px;
}

</style>

</head>

<body>

<div class="title">
🛡 AI 生物識別隱私盾
</div>

<div class="subtitle">
拍攝後自動進行 AI 指紋去識別化
</div>

<div class="main">

<div class="container">

<div class="scan-line"></div>

<video
class="input_video"
autoplay
playsinline
muted>
</video>

<canvas
class="output_canvas"
width="960"
height="720">
</canvas>

<div class="hud">

<p>
AI Status :
<span class="safe">
ACTIVE
</span>
</p>

<p id="fingerCount">
Detected Fingers : 0
</p>

<p id="protectStatus">
Protection :
<span class="safe">
READY
</span>
</p>

</div>

<div class="lockText">
AI TARGET LOCKED
</div>

</div>

</div>

<div class="controls">

<button id="captureBtn">
📸 拍攝安全照片
</button>

</div>

<div id="flash"></div>

<!-- 預覽 -->

<div id="previewScreen">

<button id="closePreview">
←
</button>

<img id="previewImage">

<a
id="downloadBtn"
download="safe_photo.png">

<button>
⬇ 下載照片
</button>

</a>

</div>

<div class="footer">
Edge AI Privacy Shield System v4.2
</div>

<script>

const videoElement =
document.querySelector(".input_video");

const canvasElement =
document.querySelector(".output_canvas");

const canvasCtx =
canvasElement.getContext("2d");

let latestResults = null;

/* ===== MediaPipe ===== */

const hands = new Hands({

    locateFile:(file)=>{

        return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
    }
});

hands.setOptions({

    maxNumHands:2,

    modelComplexity:1,

    minDetectionConfidence:0.7,

    minTrackingConfidence:0.7
});

hands.onResults(onResults);

/* ===== 即時畫面 ===== */

function onResults(results){

    latestResults = results;

    canvasCtx.save();

    canvasCtx.clearRect(
        0,
        0,
        canvasElement.width,
        canvasElement.height
    );

    canvasCtx.scale(-1,1);

    canvasCtx.drawImage(
        results.image,
        -canvasElement.width,
        0,
        canvasElement.width,
        canvasElement.height
    );

    canvasCtx.restore();

    let fingerCount = 0;

    if(results.multiHandLandmarks){

        for(const hand of results.multiHandLandmarks){

            fingerCount += 5;
        }
    }

    document
    .getElementById("fingerCount")
    .innerHTML =
    `Detected Fingers : ${fingerCount}`;
}

/* ===== 快門 ===== */

function flashEffect(){

    const flash =
    document.getElementById("flash");

    flash.style.opacity = "1";

    setTimeout(()=>{

        flash.style.opacity = "0";

    },150);
}

/* ===== 真實透明馬賽克 ===== */

function applyTransparentMosaic(
ctx,
x,
y,
size
){

    const protectedSize =
    size * 0.5;

    const mosaicSize = 10;

    const startX =
    x - protectedSize / 2;

    const startY =
    y - protectedSize / 2;

    for(
        let py = 0;
        py < protectedSize;
        py += mosaicSize
    ){

        for(
            let px = 0;
            px < protectedSize;
            px += mosaicSize
        ){

            const sampleX =
            Math.floor(startX + px);

            const sampleY =
            Math.floor(startY + py);

            if(
                sampleX < 0 ||
                sampleY < 0 ||
                sampleX >= 960 ||
                sampleY >= 720
            ){
                continue;
            }

            const pixel =
            ctx.getImageData(
                sampleX,
                sampleY,
                1,
                1
            ).data;

            ctx.fillStyle =
            `rgba(
                ${pixel[0]},
                ${pixel[1]},
                ${pixel[2]},
                0.72
            )`;

            ctx.fillRect(
                startX + px,
                startY + py,
                mosaicSize,
                mosaicSize
            );
        }
    }

    /* 柔化邊緣 */

    const gradient =
    ctx.createRadialGradient(
        x,
        y,
        protectedSize * 0.15,
        x,
        y,
        protectedSize * 0.55
    );

    gradient.addColorStop(
        0,
        "rgba(255,255,255,0)"
    );

    gradient.addColorStop(
        1,
        "rgba(255,255,255,0.12)"
    );

    ctx.fillStyle = gradient;

    ctx.beginPath();

    ctx.arc(
        x,
        y,
        protectedSize / 2,
        0,
        Math.PI * 2
    );

    ctx.fill();
}

/* ===== 拍攝後處理 ===== */

document
.getElementById("captureBtn")
.addEventListener("click",()=>{

    if(!latestResults) return;

    flashEffect();

    const captureCanvas =
    document.createElement("canvas");

    captureCanvas.width = 960;
    captureCanvas.height = 720;

    const ctx =
    captureCanvas.getContext("2d");

    /* 原始畫面 */

    ctx.save();

    ctx.scale(-1,1);

    ctx.drawImage(
        videoElement,
        -960,
        0,
        960,
        720
    );

    ctx.restore();

    /* AI 指紋去識別化 */

    if(latestResults.multiHandLandmarks){

        for(const landmarks of latestResults.multiHandLandmarks){

            const fingers = [

                [3,4],
                [7,8],
                [11,12],
                [15,16],
                [19,20]
            ];

            for(const finger of fingers){

                const joint =
                landmarks[finger[0]];

                const tip =
                landmarks[finger[1]];

                const dx =
                tip.x - joint.x;

                const dy =
                tip.y - joint.y;

                const fx =
                (1 - (tip.x + dx*0.25))
                * 960;

                const fy =
                (tip.y + dy*0.25)
                * 720;

                const distance =
                Math.sqrt(dx*dx + dy*dy);

                const dynamicSize =
                Math.max(
                    30,
                    distance*320
                );

                applyTransparentMosaic(
                    ctx,
                    fx,
                    fy,
                    dynamicSize
                );
            }
        }
    }

    const finalImage =
    captureCanvas.toDataURL(
        "image/png"
    );

    document
    .getElementById("previewScreen")
    .style.display =
    "flex";

    document
    .getElementById("previewImage")
    .src =
    finalImage;

    document
    .getElementById("downloadBtn")
    .href =
    finalImage;

    document
    .getElementById("protectStatus")
    .innerHTML =
    `Protection :
    <span class="danger">
    APPLIED
    </span>`;
});

/* ===== 關閉預覽 ===== */

document
.getElementById("closePreview")
.addEventListener("click",()=>{

    document
    .getElementById("previewScreen")
    .style.display =
    "none";
});

/* ===== Camera ===== */

const camera = new Camera(
videoElement,
{

    onFrame: async ()=>{

        await hands.send({
            image:videoElement
        });
    },

    width:960,
    height:720
});

/* ===== 啟動鏡頭 ===== */

window.addEventListener("click",()=>{

    camera.start();

},{
    once:true
});

</script>

</body>

</html>
```
