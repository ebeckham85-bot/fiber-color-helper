# Fiber Color Helper
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <!-- These meta settings prevent pinch-zooming and force proper scaling -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Fiber Optic Color Identifier</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        /* 100dvh locks the app to the exact mobile screen height without scrolling */
        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; 
            background: #000; 
            color: #fff; 
            overflow: hidden; 
            width: 100vw; 
            height: 100dvh; 
        }
        
        #viewport { 
            position: relative; 
            width: 100%; 
            height: 100%; 
            display: flex; 
            justify-content: center; 
            align-items: center; 
        }
        
        video { 
            width: 100%; 
            height: 100%; 
            object-fit: cover; 
        }
        
        canvas { display: none; }

        /* Reticle Box */
        #reticle {
            position: absolute; 
            width: 32px; 
            height: 32px;
            border: 3px solid #ffcc00; 
            box-shadow: 0 0 8px rgba(0,0,0,0.8);
            pointer-events: none;
        }

        /* Top Controls - adjusted for iPhone notch/island */
        #controls {
            position: absolute; 
            top: env(safe-area-inset-top, 20px); 
            right: 16px; 
            z-index: 10;
        }
        
        .btn {
            background: rgba(0, 0, 0, 0.7); 
            border: 1px solid rgba(255,255,255,0.3);
            color: #fff; 
            padding: 8px 14px; 
            border-radius: 20px; 
            font-size: 13px; 
            cursor: pointer;
        }

        /* Compact Result Overlay */
        #result-card {
            position: absolute; 
            bottom: calc(env(safe-area-inset-bottom, 20px) + 20px);
            background: rgba(0, 0, 0, 0.85); 
            border: 1px solid rgba(255, 255, 255, 0.2);
            padding: 12px 24px; 
            border-radius: 14px; 
            text-align: center;
            backdrop-filter: blur(8px);
            width: 80%;
            max-width: 300px;
        }
        
        #strand-num { 
            font-size: 14px; 
            color: #ffcc00; 
            font-family: monospace; 
            font-weight: bold; 
        }
        
        #color-name { 
            font-size: 24px; 
            font-weight: bold; 
            text-transform: uppercase; 
            margin-top: 2px; 
        }
    </style>
</head>
<body>

<div id="viewport">
    <video id="webcam" autoplay playsinline></video>
    <canvas id="analyzer"></canvas>
    <div id="reticle"></div>

    <div id="controls">
        <button id="torch-btn" class="btn" onclick="toggleTorch()">Flashlight: OFF</button>
    </div>

    <div id="result-card">
        <div id="strand-num">ALIGN RETICLE</div>
        <div id="color-name">SCANNING</div>
    </div>
</div>

<script>
const TIA598_PALETTE = [
    { name: "Blue", strand: 1, lab: [32, 79, -107] },
    { name: "Orange", strand: 2, lab: [67, 43, 74] },
    { name: "Green", strand: 3, lab: [46, -51, 49] },
    { name: "Brown", strand: 4, lab: [35, 13, 27] },
    { name: "Slate", strand: 5, lab: [53, 0, -2] },
    { name: "White", strand: 6, lab: [90, 0, 0] },
    { name: "Red", strand: 7, lab: [41, 62, 52] },
    { name: "Black", strand: 8, lab: [10, 0, 0] },
    { name: "Yellow", strand: 9, lab: [89, -10, 83] },
    { name: "Violet", strand: 10, lab: [30, 48, -58] },
    { name: "Rose", strand: 11, lab: [70, 48, 10] },
    { name: "Aqua", strand: 12, lab: [78, -31, -15] }
];

let videoTrack = null;
let torchOn = false;

async function initCamera() {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: { exact: "environment" }, width: { ideal: 1280 }, height: { ideal: 720 } }
        });
        const video = document.getElementById('webcam');
        video.srcObject = stream;
        videoTrack = stream.getVideoTracks()[0];
        requestAnimationFrame(processFrame);
    } catch (err) {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true });
        document.getElementById('webcam').srcObject = stream;
        videoTrack = stream.getVideoTracks()[0];
        requestAnimationFrame(processFrame);
    }
}

async function toggleTorch() {
    if (!videoTrack) return;
    const capabilities = videoTrack.getCapabilities();
    if (capabilities.torch) {
        torchOn = !torchOn;
        await videoTrack.applyConstraints({ advanced: [{ torch: torchOn }] });
        document.getElementById('torch-btn').innerText = `Flashlight: ${torchOn ? 'ON' : 'OFF'}`;
    } else {
        alert("Torch is not supported on this browser.");
    }
}

function rgbToLab(r, g, b) {
    let R = r / 255, G = g / 255, B = b / 255;
    R = (R > 0.04045) ? Math.pow((R + 0.055) / 1.055, 2.4) : (R / 12.92);
    G = (G > 0.04045) ? Math.pow((G + 0.055) / 1.055, 2.4) : (G / 12.92);
    B = (B > 0.04045) ? Math.pow((B + 0.055) / 1.055, 2.4) : (B / 12.92);

    const X = (R * 0.4124 + G * 0.3576 + B * 0.1805) * 100 / 95.047;
    const Y = (R * 0.2126 + G * 0.7152 + B * 0.0722) * 100 / 100.000;
    const Z = (R * 0.0193 + G * 0.1192 + B * 0.9505) * 100 / 108.883;

    const f = t => (t > 0.008856) ? Math.pow(t, 1/3) : (7.787 * t) + (16 / 116);

    const L = (116 * f(Y)) - 16;
    const aVal = 500 * (f(X) - f(Y));
    const bVal = 200 * (f(Y) - f(Z));

    return [L, aVal, bVal];
}

function deltaE(lab1, lab2) {
    const dL = lab1[0] - lab2[0];
    const da = lab1[1] - lab2[1];
    const db = lab1[2] - lab2[2];
    return Math.sqrt(dL * dL + da * da + db * db);
}

function processFrame() {
    const video = document.getElementById('webcam');
    const canvas = document.getElementById('analyzer');
    const ctx = canvas.getContext('2d');

    if (video.readyState === video.HAVE_ENOUGH_DATA) {
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

        const sampleSize = 20;
        const startX = Math.floor((canvas.width - sampleSize) / 2);
        const startY = Math.floor((canvas.height - sampleSize) / 2);
        const imgData = ctx.getImageData(startX, startY, sampleSize, sampleSize).data;

        let totalR = 0, totalG = 0, totalB = 0, count = 0;
        for (let i = 0; i < imgData.length; i += 4) {
            totalR += imgData[i];
            totalG += imgData[i + 1];
            totalB += imgData[i + 2];
            count++;
        }

        const avgLab = rgbToLab(totalR / count, totalG / count, totalB / count);

        let bestMatch = null;
        let minDistance = Infinity;

        for (const item of TIA598_PALETTE) {
            const dist = deltaE(item.lab, avgLab);
            if (dist < minDistance) {
                minDistance = dist;
                bestMatch = item;
            }
        }

        if (bestMatch) {
            document.getElementById('strand-num').innerText = `STRAND #${bestMatch.strand}`;
            document.getElementById('color-name').innerText = bestMatch.name;
        }
    }

    requestAnimationFrame(processFrame);
}

window.addEventListener('load', initCamera);
</script>
</body>
</html>
