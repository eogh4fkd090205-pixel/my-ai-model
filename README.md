[hello.html](https://github.com/user-attachments/files/28578673/hello.html)
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 인사 인식 거울</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: radial-gradient(circle, #1e2022 0%, #0f1011 100%);
            color: #fff;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }

        .mirror-container {
            background: rgba(255, 255, 255, 0.03);
            border: 4px solid #4a5568;
            border-radius: 24px;
            padding: 30px;
            box-shadow: 0 20px 50px rgba(0, 0, 0, 0.5), inset 0 0 20px rgba(255, 255, 255, 0.05);
            backdrop-filter: blur(10px);
            text-align: center;
            max-width: 450px;
            width: 90%;
        }

        h1 {
            font-size: 20px;
            font-weight: 400;
            letter-spacing: 2px;
            color: #cbd5e1;
            margin-bottom: 25px;
            text-shadow: 0 0 8px rgba(255,255,255,0.2);
        }

        /* 버튼 기본 스타일 */
        .toggle-btn {
            color: white;
            border: none;
            padding: 14px 28px;
            font-size: 16px;
            font-weight: bold;
            border-radius: 12px;
            cursor: pointer;
            transition: all 0.3s ease;
            width: 100%;
            margin-bottom: 25px;
        }

        /* 거울이 꺼져있을 때 (켜기 버튼) */
        .btn-off {
            background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
            box-shadow: 0 4px 15px rgba(59, 130, 246, 0.4);
        }
        .btn-off:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(59, 130, 246, 0.6);
        }

        /* 거울이 켜져있을 때 (끄기 버튼) */
        .btn-on {
            background: linear-gradient(135deg, #ef4444 0%, #b91c1c 100%);
            box-shadow: 0 4px 15px rgba(239, 68, 68, 0.4);
        }
        .btn-on:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(239, 68, 68, 0.6);
        }

        #webcam-container {
            margin: 0 auto 25px;
            border-radius: 16px;
            overflow: hidden;
            width: 300px;
            height: 300px;
            box-shadow: 0 0 0 4px #2d3748, 0 8px 24px rgba(0,0,0,0.4);
            background-color: #1a202c;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        .status-display {
            background: rgba(0, 0, 0, 0.3);
            border-radius: 16px;
            padding: 20px;
            min-height: 120px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            border: 1px solid rgba(255,255,255,0.05);
        }

        .emoji-box {
            font-size: 48px;
            margin-bottom: 10px;
            height: 60px;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: transform 0.2s ease;
        }

        .text-box {
            font-size: 18px;
            font-weight: 500;
            color: #94a3b8;
            letter-spacing: 1px;
        }

        .active-hello {
            text-shadow: 0 0 20px #38bdf8;
            animation: wave 0.5s infinite alternate;
        }

        @keyframes wave {
            from { transform: rotate(0deg); }
            to { transform: rotate(15deg); }
        }
    </style>
</head>
<body>

<div class="mirror-container">
    <h1>스마트 AI 거울</h1>
    <button type="button" id="power-btn" class="toggle-btn btn-off" onclick="toggleMirror()">거울 켜기</button>
    
    <div id="webcam-container">
        <span style="color: #4a5568;">카메라가 꺼져 있습니다.</span>
    </div>
    
    <div class="status-display">
        <div id="emoji-target" class="emoji-box">🪞</div>
        <div id="text-target" class="text-box">버튼을 눌러 거울을 켜주세요</div>
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest/dist/teachablemachine-image.min.js"></script>
<script type="text/javascript">
    const URL = "https://teachablemachine.withgoogle.com/models/4PQHOXeHS/";

    let model, webcam, maxPredictions;
    let isMirrorOn = false; // 거울 상태를 추적하는 변수 (false: 꺼짐, true: 켜짐)

    // 하나의 버튼으로 켜고 끄는 것을 제어하는 토글 함수
    async function toggleMirror() {
        const btn = document.getElementById("power-btn");

        if (!isMirrorOn) {
            // [1] 거울 켜기 로직
            isMirrorOn = true;
            btn.innerText = "거울 끄기";
            btn.className = "toggle-btn btn-on"; // 빨간색 버튼으로 변경
            
            await startMirror();
        } else {
            // [2] 거울 끄기 로직
            isMirrorOn = false;
            btn.innerText = "거울 켜기";
            btn.className = "toggle-btn btn-off"; // 파란색 버튼으로 변경
            
            stopMirror();
        }
    }

    // 거울(웹캠 및 AI) 시작
    async function startMirror() {
        const textTarget = document.getElementById("text-target");
        const emojiTarget = document.getElementById("emoji-target");
        const webcamContainer = document.getElementById("webcam-container");

        textTarget.innerText = "AI 모델 로딩 중...";

        // 모델이 아직 로드되지 않은 경우에만 로드 (중복 로드 방지)
        if (!model) {
            const modelURL = URL + "model.json";
            const metadataURL = URL + "metadata.json";
            model = await tmImage.load(modelURL, metadataURL);
            maxPredictions = model.getTotalClasses();
        }

        const flip = true; 
        webcam = new tmImage.Webcam(300, 300, flip); 
        await webcam.setup(); 
        await webcam.play();
        
        // 반복 루프 시작
        window.requestAnimationFrame(loop);

        webcamContainer.innerHTML = "";
        webcamContainer.appendChild(webcam.canvas);
        
        textTarget.innerText = "거울 앞에 서서 손을 흔들어보세요!";
        emojiTarget.innerText = "✨";
    }

    // 거울(웹캠 및 AI) 완전히 끄기
    function stopMirror() {
        if (webcam) {
            webcam.stop(); // Teachable Machine 웹캠 일시정지
            
            // 핵심: 웹캠 내부에 연결된 하드웨어 미디어 트랙을 완전히 종료 (카메라 불 끄기)
            if (webcam.stream) {
                webcam.stream.getTracks().forEach(track => track.stop());
            }
        }

        // 화면 UI 초기화
        document.getElementById("webcam-container").innerHTML = '<span style="color: #4a5568;">카메라가 꺼져 있습니다.</span>';
        
        const emojiTarget = document.getElementById("emoji-target");
        emojiTarget.innerText = "🪞";
        emojiTarget.classList.remove("active-hello");
        
        document.getElementById("text-target").innerText = "버튼을 눌러 거울을 켜주세요";
    }

    // 반복 루프 함수
    async function loop() {
        // 거울이 꺼지면(isMirrorOn === false) 루프를 즉시 중단하여 CPU 낭비를 막음
        if (!isMirrorOn) return; 

        webcam.update(); 
        await predict();
        window.requestAnimationFrame(loop);
    }

    // 예측 함수 (기존과 동일)
    async function predict() {
        if (!model || !webcam) return;
        
        const prediction = await model.predict(webcam.canvas);
        let helloProb = 0;
        let nothingProb = 0;

        for (let i = 0; i < maxPredictions; i++) {
            if (prediction[i].className.toLowerCase() === "hello") {
                helloProb = prediction[i].probability;
            } else if (prediction[i].className.toLowerCase() === "nothing") {
                nothingProb = prediction[i].probability;
            }
        }

        const emojiTarget = document.getElementById("emoji-target");
        const textTarget = document.getElementById("text-target");

        if (helloProb >= 0.8) {
            emojiTarget.innerText = "👋";
            emojiTarget.classList.add("active-hello");
            textTarget.innerHTML = `HELLO <span style="color:#38bdf8;">(${(helloProb * 100).toFixed(0)}%)</span>`;
        } 
        else if (nothingProb >= 0.8) {
            emojiTarget.innerText = "❓";
            emojiTarget.classList.remove("active-hello");
            textTarget.innerHTML = `nothing <span style="color:#e2e8f0;">(${(nothingProb * 100).toFixed(0)}%)</span>`;
        } 
        else {
            emojiTarget.innerText = "🔍";
            emojiTarget.classList.remove("active-hello");
            textTarget.innerText = "자세를 명확히 해주세요...";
        }
    }
</script>
</body>
</html>
