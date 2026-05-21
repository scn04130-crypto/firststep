<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>용해와 석출 시뮬레이션</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body {
            background-color: #f3f4f6;
            font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
        }
        canvas.sim-canvas {
            display: block;
            margin: 0 auto;
            border-radius: 0.5rem;
            background: linear-gradient(to bottom, #ffffff, #f1f5f9);
            box-shadow: inset 0 2px 4px 0 rgba(0, 0, 0, 0.06);
        }
        .btn {
            transition: all 0.2s ease;
        }
        .btn:active {
            transform: scale(0.95);
        }
        input[type=range] {
            accent-color: #3b82f6;
        }
    </style>
</head>
<body class="min-h-screen flex flex-col items-center py-8 px-4">

    <div class="max-w-7xl w-full bg-white rounded-2xl shadow-xl overflow-hidden p-6">
        <h1 class="text-3xl font-bold text-gray-800 text-center mb-2">정량적 용해와 석출 시뮬레이션</h1>
        <p class="text-center text-gray-500 mb-8">온도에 따른 용해도 곡선을 바탕으로 용해와 석출 과정을 관찰하세요. (질산 칼륨 $KNO_3$ 모델)</p>

        <!-- 상단: 시뮬레이션(좌) + 그래프(우) 가로 병렬 배치 (강제 병렬 유지) -->
        <div class="flex flex-row gap-4 mb-8 w-full">
            
            <!-- 왼쪽: 시뮬레이션 Canvas -->
            <div class="w-1/2 flex flex-col items-center bg-gray-50 p-4 rounded-xl border border-gray-200">
                <h3 class="text-lg font-bold text-gray-700 mb-4">비커 시뮬레이션</h3>
                <div class="relative w-full flex justify-center">
                    <canvas id="simCanvas" width="340" height="360" class="sim-canvas border border-gray-300 w-full max-w-[340px] h-auto"></canvas>
                    
                    <!-- 애니메이션 효과용 텍스트 -->
                    <div id="toastMessage" class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-sm font-bold opacity-0 transition-opacity duration-500 pointer-events-none drop-shadow-md"></div>
                </div>
                
                <!-- 온도 조절 슬라이더 -->
                <div class="w-full max-w-[340px] mt-6 bg-white p-4 rounded-xl border border-gray-200 shadow-sm">
                    <div class="flex justify-between items-center mb-2">
                        <label class="text-sm font-semibold text-gray-700 flex items-center">
                            <span class="mr-1">🌡️</span> 용액의 온도 조절
                        </label>
                        <span id="tempValueDisplay" class="font-bold text-red-500">20°C</span>
                    </div>
                    <input type="range" id="tempSlider" min="0" max="100" value="20" class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer">
                    <div class="flex justify-between text-xs text-gray-400 mt-1">
                        <span>0°C</span>
                        <span>100°C</span>
                    </div>
                </div>
            </div>

            <!-- 오른쪽: 용해도 그래프 -->
            <div class="w-1/2 flex flex-col bg-white border border-gray-200 p-4 rounded-xl shadow-sm">
                <h3 class="text-lg font-bold text-gray-700 mb-2 text-center">용해도 곡선 (용매 100g 기준)</h3>
                <p class="text-xs lg:text-sm text-gray-500 mb-4 text-center">
                    <span class="inline-block w-3 h-3 bg-blue-500 rounded-full mr-1"></span>다 녹음 &nbsp; 
                    <span class="inline-block w-3 h-3 bg-red-500 rounded-full mr-1"></span>초과분 석출됨
                </p>
                <!-- 그래프 캔버스 컨테이너 (유동적 높이) -->
                <div class="relative w-full flex-1 min-h-[250px]">
                    <canvas id="solubilityChart"></canvas>
                </div>
            </div>
        </div>

        <!-- 하단: 정보 및 제어 패널 -->
        <div class="grid md:grid-cols-2 gap-8">
            <!-- 물리량 상태 패널 -->
            <div class="bg-blue-50 p-5 rounded-xl border border-blue-200 shadow-sm">
                <h3 class="font-bold text-blue-800 border-b-2 border-blue-200 pb-2 mb-4 text-lg">현재 상태 정보</h3>
                <div class="space-y-4 text-base text-gray-800">
                    
                    <div class="flex justify-between items-center bg-white p-2 rounded-lg">
                        <span class="text-gray-600 font-medium">용매 (물) 부피 / 질량:</span>
                        <span class="font-mono">100 mL / 0.1 kg</span>
                    </div>
                    
                    <div class="flex justify-between items-center bg-purple-50 p-2 rounded-lg border border-purple-100">
                        <span class="text-purple-800 font-medium">투입된 총 용질 질량:</span>
                        <span class="font-mono font-bold text-purple-700 text-lg"><span id="statTotalSolute">0</span> g</span>
                    </div>
                    
                    <div class="flex justify-between items-center bg-white p-2 rounded-lg">
                        <span class="text-gray-600 font-medium">현재 온도 기준 용해도:</span>
                        <span class="font-mono"><span id="statSolubility">31.6</span> g / 물 100g</span>
                    </div>

                    <div class="grid grid-cols-2 gap-4 mt-2">
                        <div class="bg-green-50 p-3 rounded-lg border border-green-200 text-center">
                            <div class="text-sm text-green-700 mb-1">녹은 용질 (용해)</div>
                            <div class="font-mono font-bold text-green-600 text-xl"><span id="statDissolvedMass">0.0</span> g</div>
                        </div>
                        <div class="bg-gray-100 p-3 rounded-lg border border-gray-300 text-center">
                            <div class="text-sm text-gray-700 mb-1">가라앉은 용질 (석출)</div>
                            <div class="font-mono font-bold text-gray-700 text-xl"><span id="statSolidMass">0.0</span> g</div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- 컨트롤 버튼 -->
            <div class="flex flex-col justify-center space-y-4">
                <div class="grid grid-cols-2 gap-4">
                    <button id="btnAdd5g" class="btn py-4 px-4 bg-purple-100 hover:bg-purple-200 text-purple-800 border border-purple-300 rounded-xl font-bold shadow-sm flex items-center justify-center text-lg">
                        <span class="mr-2">🧂</span> + 5g 추가
                    </button>
                    <button id="btnAdd20g" class="btn py-4 px-4 bg-purple-500 hover:bg-purple-600 text-white rounded-xl font-bold shadow-sm flex items-center justify-center text-lg">
                        <span class="mr-2">🧂</span> + 20g 추가
                    </button>
                </div>
                
                <button id="btnStir" class="btn w-full py-4 px-4 bg-blue-500 hover:bg-blue-600 text-white rounded-xl font-bold shadow-sm flex items-center justify-center text-lg mt-2">
                    <span class="mr-2">🥄</span> 저어주기 (용해 촉진)
                </button>
                
                <button id="btnReset" class="btn w-full mt-4 py-3 px-4 bg-white border border-gray-300 hover:bg-gray-50 text-gray-700 rounded-xl font-medium">
                    실험 초기화
                </button>
            </div>
        </div>
    </div>

    <script>
        // 화학 상수 (KNO3 온도별 용해도 데이터)
        const solubilityDataPoints = [
            {x: 0, y: 13.3}, {x: 10, y: 20.9}, {x: 20, y: 31.6},
            {x: 30, y: 45.8}, {x: 40, y: 63.9}, {x: 50, y: 85.5},
            {x: 60, y: 110.0}, {x: 70, y: 138.0}, {x: 80, y: 169.0},
            {x: 90, y: 202.0}, {x: 100, y: 246.0}
        ];

        function getSolubilityAt(temp) {
            if (temp <= 0) return solubilityDataPoints[0].y;
            if (temp >= 100) return solubilityDataPoints[solubilityDataPoints.length-1].y;
            for (let i = 0; i < solubilityDataPoints.length - 1; i++) {
                let p1 = solubilityDataPoints[i];
                let p2 = solubilityDataPoints[i+1];
                if (temp >= p1.x && temp <= p2.x) {
                    let ratio = (temp - p1.x) / (p2.x - p1.x);
                    return p1.y + ratio * (p2.y - p1.y);
                }
            }
            return 31.6;
        }

        // 시뮬레이션 상태
        let state = {
            temperature: 20,
            totalSoluteMass_g: 0,
            dissolvedMass_g: 0,
            solidMass_g: 0,
            isStirring: false,
            stirTimer: 0
        };

        const PARTICLES_PER_GRAM = 2; // 시각적 구분을 위한 입자 배수

        // 비커 기하학적 정의 (명확한 직선)
        const beaker = {
            x: 45,
            y: 30,
            width: 250,
            height: 300,
            waterLevelY: 130, // 물 표면 Y 좌표
            get left() { return this.x; },
            get right() { return this.x + this.width; },
            get bottom() { return this.y + this.height; }
        };

        const canvas = document.getElementById('simCanvas');
        const ctx = canvas.getContext('2d');
        let particles = [];

        const ctxChart = document.getElementById('solubilityChart').getContext('2d');
        const solubilityChart = new Chart(ctxChart, {
            type: 'line',
            data: {
                datasets: [{
                    label: '용해도 곡선 (g/물100g)',
                    data: solubilityDataPoints,
                    borderColor: 'rgb(75, 192, 192)',
                    backgroundColor: 'rgba(75, 192, 192, 0.1)',
                    borderWidth: 3,
                    tension: 0.4,
                    fill: true,
                    pointRadius: 0,
                    pointHitRadius: 10
                }, {
                    label: '현재 상태',
                    data: [{x: state.temperature, y: state.totalSoluteMass_g}],
                    backgroundColor: 'rgb(59, 130, 246)',
                    borderColor: 'white',
                    borderWidth: 2,
                    pointRadius: 8,
                    pointHoverRadius: 10,
                    type: 'scatter'
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    x: {
                        type: 'linear',
                        title: { display: true, text: '온도 (°C)', font: {size: 14, weight: 'bold'} },
                        min: 0, max: 100
                    },
                    y: {
                        title: { display: true, text: '용질 질량 (g)', font: {size: 14, weight: 'bold'} },
                        min: 0, max: 260
                    }
                },
                plugins: {
                    legend: { display: false }
                },
                animation: { duration: 300 }
            }
        });

        function updateChart() {
            solubilityChart.data.datasets[1].data = [{x: state.temperature, y: state.totalSoluteMass_g}];
            let currentSolubility = getSolubilityAt(state.temperature);
            // 한계 초과 시 붉은색 점
            if (state.totalSoluteMass_g > currentSolubility + 0.1) {
                solubilityChart.data.datasets[1].backgroundColor = 'rgb(239, 68, 68)'; 
            } else {
                solubilityChart.data.datasets[1].backgroundColor = 'rgb(59, 130, 246)'; 
            }
            solubilityChart.update();
        }

        class Particle {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.vx = (Math.random() - 0.5) * 4;
                this.vy = Math.random() * 2 + 2; 
                this.radius = 4;
                this.state = 'solid'; // 'solid' or 'dissolved'
                this.color = '#8b5cf6';
            }

            update(dissolvedMass, maxSolubilityMass, allParticles) {
                if (this.state === 'solid') {
                    // 고체 상태 낙하 로직
                    this.vy += 0.3; // 중력

                    // 물 속에 들어가면 저항 발생
                    if (this.y > beaker.waterLevelY) {
                        this.vy *= 0.85; 
                        this.vx *= 0.90;
                    }

                    this.x += this.vx;
                    this.y += this.vy;

                    // 용해 로직
                    let dissolveChance = state.isStirring ? 0.08 : 0.01; 
                    let tempFactor = Math.max(0.5, state.temperature / 20);
                    dissolveChance *= tempFactor;

                    if (this.y > beaker.waterLevelY && dissolvedMass < maxSolubilityMass && Math.random() < dissolveChance) {
                        this.state = 'dissolved';
                        this.vy = -Math.random() * 3 - 1; 
                        this.vx = (Math.random() - 0.5) * 4;
                        return;
                    }

                    // 비커 경계면 직관적인 충돌 처리 (직선 바운더리)
                    if (this.x < beaker.left + this.radius) { 
                        this.x = beaker.left + this.radius; 
                        this.vx *= -0.5; 
                    }
                    if (this.x > beaker.right - this.radius) { 
                        this.x = beaker.right - this.radius; 
                        this.vx *= -0.5; 
                    }
                    if (this.y > beaker.bottom - this.radius) { 
                        this.y = beaker.bottom - this.radius; 
                        this.vy *= -0.3; // 바닥 충돌 후 에너지 감소
                        this.vx *= 0.8; // 마찰력
                    }

                    // 뭉치지 않게 하는 입자 간 밀어내기 (고체끼리만 적용)
                    for (let i = 0; i < allParticles.length; i++) {
                        let other = allParticles[i];
                        if (other !== this && other.state === 'solid') {
                            let dx = this.x - other.x;
                            let dy = this.y - other.y;
                            let dist = Math.hypot(dx, dy);
                            let minDist = this.radius + other.radius + 0.5; // 살짝 여백
                            
                            if (dist < minDist && dist > 0) {
                                let angle = Math.atan2(dy, dx);
                                let overlap = (minDist - dist) * 0.5;
                                
                                // 서로 밀어내기
                                this.x += Math.cos(angle) * overlap;
                                this.y += Math.sin(angle) * overlap;
                                other.x -= Math.cos(angle) * overlap;
                                other.y -= Math.sin(angle) * overlap;
                                
                                // 속도 감쇠 (안정적으로 쌓이도록)
                                this.vx *= 0.6;
                                this.vy *= 0.6;
                                other.vx *= 0.6;
                                other.vy *= 0.6;
                            }
                        }
                    }
                } 
                else if (this.state === 'dissolved') {
                    // 석출 로직
                    let precipitateChance = state.isStirring ? 0.02 : 0.05;
                    if (dissolvedMass > maxSolubilityMass && Math.random() < precipitateChance) {
                        this.state = 'solid'; 
                    } else {
                        // 액체 내부 부유 운동
                        let speed = state.isStirring ? 2.0 : (0.2 + (state.temperature / 100));
                        this.vx += (Math.random() - 0.5) * speed;
                        this.vy += (Math.random() - 0.5) * speed;

                        let maxSpeed = state.isStirring ? 5 : 2;
                        this.vx = Math.max(-maxSpeed, Math.min(maxSpeed, this.vx));
                        this.vy = Math.max(-maxSpeed, Math.min(maxSpeed, this.vy));

                        this.x += this.vx;
                        this.y += this.vy;

                        // 물 영역 안에서만 튕기기
                        if (this.x < beaker.left + this.radius) { this.x = beaker.left + this.radius; this.vx *= -1; }
                        if (this.x > beaker.right - this.radius) { this.x = beaker.right - this.radius; this.vx *= -1; }
                        if (this.y < beaker.waterLevelY + this.radius) { this.y = beaker.waterLevelY + this.radius; this.vy *= -1; }
                        if (this.y > beaker.bottom - this.radius) { this.y = beaker.bottom - this.radius; this.vy *= -1; }
                    }
                }
            }

            draw(ctx) {
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                
                if (this.state === 'dissolved') {
                    ctx.fillStyle = 'rgba(139, 92, 246, 0.4)'; 
                    ctx.strokeStyle = 'rgba(139, 92, 246, 0.2)';
                    ctx.lineWidth = 1;
                    ctx.fill();
                    ctx.stroke();
                } else {
                    let grd = ctx.createRadialGradient(this.x - 1, this.y - 1, 0, this.x, this.y, this.radius);
                    grd.addColorStop(0, '#c4b5fd'); // 밝은 보라
                    grd.addColorStop(1, '#6d28d9'); // 짙은 보라
                    ctx.fillStyle = grd;
                    ctx.fill();
                }
                ctx.closePath();
            }
        }

        function drawEnvironment() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // 물 그리기 (온도에 따른 색상 변화)
            let tempRatio = state.temperature / 100;
            let r = Math.floor(191 + (254 - 191) * tempRatio);
            let b = Math.floor(254 - (254 - 202) * tempRatio);
            ctx.fillStyle = `rgba(${r}, 210, ${b}, 0.3)`;
            ctx.fillRect(beaker.left, beaker.waterLevelY, beaker.width, beaker.height - (beaker.waterLevelY - beaker.y));
            
            // 젓기 이펙트
            if (state.isStirring) {
                ctx.beginPath();
                let cy = beaker.waterLevelY + (beaker.bottom - beaker.waterLevelY)/2;
                ctx.ellipse(beaker.x + beaker.width/2, cy, 60, 15, 0, 0, Math.PI * 2);
                ctx.strokeStyle = 'rgba(255,255,255,0.7)';
                ctx.lineWidth = 2;
                ctx.stroke();
            }

            // 비커 외곽선 (직선으로 뚜렷하게)
            ctx.beginPath();
            ctx.moveTo(beaker.left, beaker.y);
            ctx.lineTo(beaker.left, beaker.bottom);
            ctx.lineTo(beaker.right, beaker.bottom);
            ctx.lineTo(beaker.right, beaker.y);
            
            ctx.strokeStyle = '#475569'; // 짙은 회색
            ctx.lineWidth = 6;
            ctx.lineCap = 'round';
            ctx.lineJoin = 'round';
            ctx.stroke();

            // 물결/수면 선
            ctx.beginPath();
            ctx.moveTo(beaker.left, beaker.waterLevelY);
            ctx.lineTo(beaker.right, beaker.waterLevelY);
            ctx.strokeStyle = '#93c5fd';
            ctx.lineWidth = 2;
            ctx.stroke();
            
            // 100mL 눈금선
            ctx.fillStyle = '#64748b';
            ctx.font = '12px sans-serif';
            ctx.fillText('100mL', beaker.left - 45, beaker.waterLevelY + 5);
            ctx.beginPath();
            ctx.moveTo(beaker.left - 5, beaker.waterLevelY);
            ctx.lineTo(beaker.left, beaker.waterLevelY);
            ctx.strokeStyle = '#64748b';
            ctx.lineWidth = 2;
            ctx.stroke();
        }

        function showToast(msg) {
            const toast = document.getElementById('toastMessage');
            toast.innerText = msg;
            toast.className = `absolute top-1/3 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-sm font-bold bg-white/90 px-3 py-1 rounded-full shadow-md transition-opacity duration-300 pointer-events-none z-10 text-purple-700`;
            toast.style.opacity = 1;
            setTimeout(() => { toast.style.opacity = 0; }, 1000);
        }

        function animate() {
            drawEnvironment();

            let currentSolubility = getSolubilityAt(state.temperature);

            let dissolvedParticles = particles.filter(p => p.state === 'dissolved');
            let solidParticles = particles.filter(p => p.state === 'solid');
            
            state.dissolvedMass_g = dissolvedParticles.length / PARTICLES_PER_GRAM;
            state.solidMass_g = solidParticles.length / PARTICLES_PER_GRAM;

            particles.forEach(p => {
                p.update(state.dissolvedMass_g, currentSolubility, particles);
                p.draw(ctx);
            });

            if (state.isStirring) {
                state.stirTimer--;
                if (state.stirTimer <= 0) state.isStirring = false;
            }

            // UI 업데이트
            document.getElementById('statTotalSolute').innerText = state.totalSoluteMass_g.toFixed(1);
            document.getElementById('statSolubility').innerText = currentSolubility.toFixed(1);
            document.getElementById('statDissolvedMass').innerText = state.dissolvedMass_g.toFixed(1);
            document.getElementById('statSolidMass').innerText = state.solidMass_g.toFixed(1);

            requestAnimationFrame(animate);
        }

        function addParticlesForMass(mass_g) {
            state.totalSoluteMass_g += mass_g;
            let numParticles = mass_g * PARTICLES_PER_GRAM;
            
            for (let i = 0; i < numParticles; i++) {
                let p = new Particle(
                    beaker.left + 20 + Math.random() * (beaker.width - 40),
                    beaker.y - 10 - Math.random() * 50
                );
                particles.push(p);
            }
            updateChart();
            showToast(`+${mass_g}g 추가됨`);
        }

        document.getElementById('btnAdd5g').addEventListener('click', () => addParticlesForMass(5));
        document.getElementById('btnAdd20g').addEventListener('click', () => addParticlesForMass(20));
        document.getElementById('btnStir').addEventListener('click', () => {
            state.isStirring = true;
            state.stirTimer = 60;
        });

        const tempSlider = document.getElementById('tempSlider');
        const tempValueDisplay = document.getElementById('tempValueDisplay');
        tempSlider.addEventListener('input', (e) => {
            let newTemp = parseInt(e.target.value);
            state.temperature = newTemp;
            
            if (newTemp < 30) tempValueDisplay.className = 'font-bold text-blue-500';
            else if (newTemp < 60) tempValueDisplay.className = 'font-bold text-green-500';
            else tempValueDisplay.className = 'font-bold text-red-500';
            
            tempValueDisplay.innerText = `${newTemp}°C`;
            updateChart();
        });

        document.getElementById('btnReset').addEventListener('click', () => {
            particles = [];
            state.totalSoluteMass_g = 0;
            state.temperature = 20;
            tempSlider.value = 20;
            tempValueDisplay.innerText = `20°C`;
            tempValueDisplay.className = 'font-bold text-blue-500';
            updateChart();
        });

        // 애니메이션 시작
        updateChart();
        animate();
    </script>
</body>
</html>
