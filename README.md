
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sheriff Robot View</title>

<style>
* { box-sizing: border-box; }

body {
    margin: 0;
    background: #0b1118;
    color: #eaf6ff;
    font-family: Arial, sans-serif;
    text-align: center;
}

header { padding: 18px 10px; }

h1 {
    margin: 0;
    color: #36e6ff;
    font-size: 25px;
}

#status {
    color: #a8bdcc;
    margin: 12px;
    font-size: 14px;
}

main {
    width: min(960px, 96%);
    margin: auto;
}

.screen {
    position: relative;
    width: 100%;
    background: #000;
    border: 1px solid #294b5b;
    border-radius: 12px;
    overflow: hidden;
}

canvas {
    display: block;
    width: 100%;
    height: auto;
}

#hud {
    position: absolute;
    top: 12px;
    right: 12px;
    background: rgba(0, 0, 0, 0.8);
    color: #36e6ff;
    text-align: left;
    padding: 12px;
    border: 1px solid #36e6ff;
    border-radius: 8px;
    font: bold 15px monospace;
    line-height: 1.8;
}

#warning {
    color: #42ff9e;
    font-weight: bold;
    margin: 15px;
}

button {
    border: 0;
    border-radius: 8px;
    padding: 13px 20px;
    margin: 8px 4px;
    font-size: 15px;
    font-weight: bold;
    cursor: pointer;
}

#start { background: #36e6ff; color: #071119; }
#stop { background: #293946; color: white; }

.note {
    color: #a8bdcc;
    font-size: 13px;
    line-height: 1.7;
    padding: 8px 12px 24px;
}
</style>
</head>

<body>
<header>
    <h1>SHERIFF ROBOT VIEW</h1>
    <div id="status">시작 버튼을 눌러 카메라를 연결하세요.</div>
</header>

<main>
    <div class="screen">
        <canvas id="view" width="640" height="480"></canvas>
        <div id="hud">
            X: ---<br>
            Y: ---<br>
            Z: ---
        </div>
    </div>

    <div id="warning">SYSTEM STANDBY</div>

    <button id="start">카메라 시작</button>
    <button id="stop">카메라 종료</button>

    <p class="note">
        X/Y: 화면 중앙 기준 좌표(픽셀)<br>
        Z: 얼굴 크기로 계산한 거리 추정치(cm)<br>
        카메라 권한이 필요하며 HTTPS 웹사이트 또는 localhost에서 실행해야 합니다.
    </p>
</main>

<video id="camera" autoplay playsinline muted
       style="display:none"></video>

<script type="module">
import {
    FaceDetector,
    FilesetResolver
} from "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14";

const video = document.getElementById("camera");
const canvas = document.getElementById("view");
const ctx = canvas.getContext("2d");
const hud = document.getElementById("hud");
const statusText = document.getElementById("status");
const warning = document.getElementById("warning");
const startButton = document.getElementById("start");
const stopButton = document.getElementById("stop");

let detector = null;
let stream = null;
let running = false;
let busy = false;
let animationId = null;
let lastVideoTime = -1;

// 얼굴 너비 약 15cm라는 가정과 임시 초점 거리.
// 정확한 실제 거리를 얻으려면 카메라 보정이 필요합니다.
const KNOWN_FACE_WIDTH_CM = 15;
const FOCAL_LENGTH_PX = 600;
const WARN_RATIO = 0.35;

async function loadDetector() {
    if (detector) return;

    statusText.textContent = "얼굴 인식 모델을 불러오는 중...";

    const vision = await FilesetResolver.forVisionTasks(
        "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm"
    );

    detector = await FaceDetector.createFromOptions(vision, {
        baseOptions: {
            modelAssetPath:
                "https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite",
            delegate: "CPU"
        },
        runningMode: "VIDEO",
        minDetectionConfidence: 0.5
    });
}

async function startCamera() {
    if (running) return;

    startButton.disabled = true;

    try {
        await loadDetector();

        stream = await navigator.mediaDevices.getUserMedia({
            audio: false,
            video: {
                facingMode: "user",
                width: { ideal: 640 },
                height: { ideal: 480 }
            }
        });

        video.srcObject = stream;
        await video.play();

        canvas.width = video.videoWidth || 640;
        canvas.height = video.videoHeight || 480;

        running = true;
        lastVideoTime = -1;
        statusText.textContent = "카메라 연결 완료";
        warning.textContent = "얼굴을 찾는 중...";
        animationId = requestAnimationFrame(processFrame);

    } catch (error) {
        statusText.textContent =
            "실행 실패: " + error.message;

        warning.textContent =
            "카메라 권한과 HTTPS 연결을 확인하세요.";

        stopCamera();
    } finally {
        startButton.disabled = false;
    }
}

function drawFrame() {
    const W = canvas.width;
    const H = canvas.height;

    // 거울처럼 좌우 반전된 화면 표시
    ctx.save();
    ctx.translate(W, 0);
    ctx.scale(-1, 1);
    ctx.drawImage(video, 0, 0, W, H);
    ctx.restore();

    // 화면 중앙 십자선
    ctx.strokeStyle = "#ffffff";
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(W / 2 - 12, H / 2);
    ctx.lineTo(W / 2 + 12, H / 2);
    ctx.moveTo(W / 2, H / 2 - 12);
    ctx.lineTo(W / 2, H / 2 + 12);
    ctx.stroke();
}

function processFrame() {
    if (!running) return;

    if (video.readyState < 2 || busy) {
        animationId = requestAnimationFrame(processFrame);
        return;
    }

    drawFrame();

    if (video.currentTime !== lastVideoTime) {
        lastVideoTime = video.currentTime;
        busy = true;

        try {
            const result = detector.detectForVideo(
                video,
                performance.now()
            );

            updateOverlay(result.detections);
        } catch (error) {
            console.error(error);
        } finally {
            busy = false;
        }
    }

    animationId = requestAnimationFrame(processFrame);
}

function updateOverlay(detections) {
    const W = canvas.width;
    const H = canvas.height;

    hud.innerHTML = "X: ---<br>Y: ---<br>Z: ---";

    if (!detections.length) {
        warning.textContent = "얼굴을 찾는 중...";
        warning.style.color = "#42ff9e";
        return;
    }

    // 가장 큰 얼굴 선택
    const face = detections.reduce((best, current) =>
        current.boundingBox.width * current.boundingBox.height >
        best.boundingBox.width * best.boundingBox.height
            ? current : best
    );

    const box = face.boundingBox;

    // 원본 영상의 얼굴 상자를 거울 화면 좌표로 변환
    const x = W - box.originX - box.width;
    const y = box.originY;
    const w = box.width;
    const h = box.height;

    const cx = x + w / 2;
    const cy = y + h / 2;

    // 화면 중앙 기준 X/Y 좌표
    const X = Math.round(cx - W / 2);
    const Y = Math.round(H / 2 - cy);

    // 얼굴 폭으로 추정한 Z 거리
    const Z = KNOWN_FACE_WIDTH_CM * FOCAL_LENGTH_PX / w;

    const tooClose = w / W > WARN_RATIO;
    const color = tooClose ? "#ff3030" : "#42ff9e";

    // 얼굴 테두리
    ctx.strokeStyle = color;
    ctx.lineWidth = 3;
    ctx.strokeRect(x, y, w, h);

    // 얼굴 중심점
    ctx.fillStyle = "#36e6ff";
    ctx.beginPath();
    ctx.arc(cx, cy, 5, 0, Math.PI * 2);
    ctx.fill();

    // 중앙과 얼굴을 연결하는 선
    ctx.strokeStyle = "#36e6ff";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(W / 2, H / 2);
    ctx.lineTo(cx, cy);
    ctx.stroke();

    hud.innerHTML =
        `X: ${X}<br>` +
        `Y: ${Y}<br>` +
        `Z: ${Z.toFixed(1)} cm`;

    warning.textContent = tooClose
        ? "WARNING! TOO CLOSE!"
        : "TARGET LOCKED";

    warning.style.color = color;
}

function stopCamera() {
    running = false;

    if (animationId !== null) {
        cancelAnimationFrame(animationId);
        animationId = null;
    }

    if (stream) {
        stream.getTracks().forEach(track => track.stop());
        stream = null;
    }

    video.srcObject = null;
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    hud.innerHTML = "X: ---<br>Y: ---<br>Z: ---";
    statusText.textContent = "카메라가 종료되었습니다.";
    warning.textContent = "SYSTEM STANDBY";
    warning.style.color = "#a8bdcc";
}

startButton.addEventListener("click", startCamera);
stopButton.addEventListener("click", stopCamera);

window.addEventListener("pagehide", stopCamera);
</script>
</body>
</html>
