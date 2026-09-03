# Fiber Color Helper
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Fiber Optic Color Identifier</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        html, body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; 
            background: #000; 
            color: #fff; 
            overflow: hidden; 
            width: 100vw; 
            height: 100vh;
            margin: 0;
            padding: 0;
        }
        
        #viewport { 
            position: absolute;
            top: 0;
            left: 0;
            width: 100%; 
            height: 100%; 
        }
        
        /* Video element must exist in DOM with explicit attributes for iOS live playback */
        video { 
            position: absolute;
            width: 1px;
            height: 1px;
            opacity: 0.01;
        } 
        
        canvas { 
            width: 100%; 
            height: 100%; 
            object-fit: cover; 
            display: block;
        }

        /* Controls Pinned at Top Right */
        #torch-btn {
            position: fixed;
            top: env(safe-area-inset-top, 20px);
            right: 20px;
            z-index: 9999;
            background: rgba(0, 0, 0, 0.8); 
            border: 1px solid rgba(255,255,255,0.4);
            color: #fff; 
            padding: 8px 16px; 
            border-radius: 20px; 
            font-size: 12px; 
            font-weight: 600;
        }

        /* Start Button Overlay for iOS Safari autoplay policies */
        #start-overlay {
            position: fixed;
            top: 0; left: 0; width: 100%; height: 100%;
            background: #000;
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 10000;
        }

        #start-btn {
            background: #ffcc00;
            color: #000;
            border: none;
            padding: 16px 32px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 30px;
            cursor: pointer;
        }
    </style>
</head>
<body>

<div id="start-overlay">
    <button id="start-btn" onclick="startApp()">START CAMERA</button>
</div>

<div id="viewport">
    <!-- Required attributes for iOS Safari live streaming: autoplay, loop, muted, playsinline -->
    <video id="webcam" autoplay loop muted playsinline></video>
    <canvas id="analyzer"></canvas>
    <button id="torch-btn" onclick="toggleTorch()">Flashlight: OFF</button>
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

async function startApp() {
    document.getElementById('start-overlay').style.display = 'none';
    const video = document.getElementById('webcam');

    try {
        const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: { exact: "environment" }, width: { ideal: 1280 }, height: { ideal: 720 } }
        });
        video.srcObject = stream;
        videoTrack = stream.getVideoTracks()[0];
    } catch (err) {
        // Fallback to default camera if rear environment camera is unavailable
        const stream = await navigator.mediaDevices.getUserMedia({ video: true });
        video.srcObject = stream;
        videoTrack = stream.getVideoTracks()[0];
    }

    // Explicitly play video stream to prevent freezing
    await video.play();
    requestAnimationFrame(processFrame);
}

async function toggleTorch() {
    if (!videoTrack) return;
    const capabilities = videoTrack.getCapabilities();
    if (capabilities.torch) {
        torchOn = !torchOn;
        await videoTrack.applyConstraints({ advanced: [{ torch: torchOn }] });
        document.getElementById('torch-btn').innerText = `Flashlight: ${torchOn ? 'ON' : 'OFF'}`;
    } else {
        alert("Torch is not supported on this device.");
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

    if (video.readyState >= video.HAVE_CURRENT_DATA) {
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        
        // Draw live camera frame continuously
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

        // Center Pixel Sampling Box
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

        // Draw Reticle Box
        ctx.strokeStyle = "#ffcc00";
        ctx.lineWidth = 4;
        ctx.strokeRect(startX, startY, sampleSize, sampleSize);

        // Draw Canvas Overlay Box at Top
        const cardWidth = Math.min(canvas.width * 0.8, 400);
        const cardHeight = 100;
        const cardX = (canvas.width - cardWidth) / 2;
        const cardY = 40;

        ctx.fillStyle = "rgba(0, 0, 0, 0.85)";
        ctx.strokeStyle = "rgba(255, 255, 255, 0.3)";
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.roundRect(cardX, cardY, cardWidth, cardHeight, 16);
        ctx.fill();
        ctx.stroke();

        ctx.textAlign = "center";
        
        ctx.fillStyle = "#ffcc00";
        ctx.font = "bold 20px monospace";
        const strandText = bestMatch ? `STRAND #${bestMatch.strand}` : "ALIGN RETICLE";
        ctx.fillText(strandText, canvas.width / 2, cardY + 35);

        ctx.fillStyle = "#ffffff";
        ctx.font = "bold 36px -apple-system, sans-serif";
        const colorText = bestMatch ? bestMatch.name.toUpperCase() : "SCANNING";
        ctx.fillText(colorText, canvas.width / 2, cardY + 75);
    }

    // Keep loop active
    requestAnimationFrame(processFrame);
}
</script>
</body>
</html>


