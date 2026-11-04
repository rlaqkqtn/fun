<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>AI 텍스트 아쿠아리움: 급식 에디션</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000205; color: white; font-family: 'Pretendard', sans-serif; }
        canvas { display: block; }
        
        const HARDCODED_API_KEY = "7fa09fa2743e4c78844dfc42faef8a41";
        #api-panel {
            position: absolute; top: 15px; right: 20px; z-index: 10;
            background: rgba(0, 0, 0, 0.75); padding: 12px 16px; border-radius: 16px;
            border: 1px solid rgba(255, 255, 255, 0.1); backdrop-filter: blur(10px);
            display: flex; flex-direction: column; gap: 8px; font-size: 13px; width: 280px;
        }
        #api-panel input {
            background: rgba(255, 255, 255, 0.08); border: 1px solid rgba(255, 255, 255, 0.15);
            color: white; border-radius: 6px; padding: 6px 10px; font-size: 12px; outline: none;
        }
        .btn-group { display: flex; gap: 6px; }
        .btn-group button { flex: 1; padding: 8px; font-size: 12px; }

        /* 하단 단어 투하 UI */
        #ui { position: absolute; bottom: 30px; left: 50%; transform: translateX(-50%); z-index: 10; width: 100%; max-width: 450px; }
        #input-box { display: flex; gap: 10px; background: rgba(255, 255, 255, 0.03); padding: 12px; border-radius: 20px; backdrop-filter: blur(20px); border: 1px solid rgba(255,255,255,0.08); box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        input[type="text"] { flex: 1; background: transparent; border: none; color: white; outline: none; font-size: 16px; padding-left: 10px; }
        input::placeholder { color: rgba(255,255,255,0.3); }
        button { background: linear-gradient(45deg, #00f2fe, #4facfe); border: none; padding: 10px 20px; border-radius: 12px; font-weight: bold; color: white; cursor: pointer; transition: 0.2s; white-space: nowrap; }
        button:hover { transform: scale(1.03); box-shadow: 0 0 15px rgba(0,242,254,0.6); }
        
        #leaderboard { position: absolute; top: 20px; left: 20px; background: rgba(0,0,0,0.7); padding: 15px; border-radius: 15px; border: 1px solid rgba(255,255,255,0.08); width: 220px; z-index: 5; pointer-events: none; }
    </style>
</head>
<body>

<div id="leaderboard">
    <h3 style="margin:0 0 10px 0; font-size:14px; color:#aaa; text-align:center; text-transform:uppercase; letter-spacing:1px;">👑 생태계 순위</h3>
    <div id="list"></div>
</div>

<!-- NEIS API 제어 패널 -->
<div id="api-panel">
    <div style="font-weight: bold; color: #00f2fe; margin-bottom: 2px;">🍱 나이스 급식 API 연동</div>
    <input type="text" id="api-key" placeholder="API KEY" value="">
    <div style="display: flex; gap: 5px;">
        <input type="text" id="office-code" placeholder="시도교육청코드(예: B10)" value="B10" style="width: 50%;">
        <input type="text" id="school-code" placeholder="학교코드(7자리)" value="7010537" style="width: 50%;">
    </div>
    <input type="date" id="meal-date">
    <div class="btn-group">
        <button id="fetch-meal-btn">급식 불러오기</button>
        <button id="sample-meal-btn" style="background: linear-gradient(45deg, #ff0844, #ffb199);">샘플 투하</button>
    </div>
</div>

<div id="ui">
    <form id="spawn-form">
        <div id="input-box">
            <input type="text" id="target-text" placeholder="물고기 이름 입력..." required maxlength="15" autocomplete="off">
            <button type="submit">물고기 생성</button>
        </div>
    </form>
</div>

<canvas id="world"></canvas>

<script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.19.0/matter.min.js"></script>
<script>
    // 오늘 날짜 기본 세팅
    document.getElementById('meal-date').valueAsDate = new Date();

    const { Engine, Runner, Bodies, Composite, Events, Body, Vector } = Matter;
    const engine = Engine.create({ enableSleeping: false });
    
    // 밥이 아래로 떨어지도록 중력 설정
    engine.world.gravity.y = 0.3;

    const canvas = document.getElementById('world');
    const ctx = canvas.getContext('2d');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    const boundarySize = { width: canvas.width * 0.95, height: canvas.height * 0.95 }; 
    const autoWords = ["고래상어", "대왕오징어", "청새치", "가오리", "피라쿠루", "참치", "방어", "메기"]; 

    // ==========================================
    // [1] 물고기 생성
    // ==========================================
    const spawn = (text, x, y, sizeOverride = null) => {
        let isJackpot = sizeOverride ? false : Math.random() < 0.05;
        let baseSize = sizeOverride || (35 + (text.length * 6)); 
        if (isJackpot) baseSize *= 2.5;

        const spawnX = x || (canvas.width / 2 + (Math.random() - 0.5) * (boundarySize.width * 0.7));
        const spawnY = y || (canvas.height / 2 + (Math.random() - 0.5) * (boundarySize.height * 0.7));

        const fish = Bodies.circle(spawnX, spawnY, baseSize, {
            frictionAir: 0.08,
            isSensor: true,
            label: 'fish',
            gravityScale: 0
        });

        fish.word = text;
        fish.size = baseSize;
        fish.isJackpot = isJackpot;
        
        const hue = isJackpot ? 300 : Math.random() * 360; 
        fish.color = `hsl(${hue}, 100%, 65%)`;
        fish.currentAngle = Math.random() * Math.PI * 2;
        fish.tailAngle = 0;
        fish.eatEffect = 0;

        Body.setVelocity(fish, { x: (Math.random()-0.5)*4, y: (Math.random()-0.5)*4 });
        Composite.add(engine.world, fish);
        return fish;
    };

    // ==========================================
    // [2] 급식 밥 (Food) 투하
    // ==========================================
    const dropFood = (menuText) => {
        const foodWidth = Math.min(Math.max(menuText.length * 14 + 20, 45), 140);
        const foodHeight = 26;

        const spawnX = Math.random() * (canvas.width - 200) + 100;
        const food = Bodies.rectangle(spawnX, -30, foodWidth, foodHeight, {
            frictionAir: 0.05,
            restitution: 0.3,
            label: 'food'
        });

        food.menuText = menuText;
        food.width = foodWidth;
        food.height = foodHeight;
        food.color = '#fffb00';

        Composite.add(engine.world, food);
    };

    const checkPopulation = () => {
        const bodies = Composite.allBodies(engine.world).filter(b => b.label === 'fish');
        if (bodies.length < 8) {
            const randomWord = autoWords[Math.floor(Math.random() * autoWords.length)];
            spawn(randomWord);
        }
    };
    setInterval(checkPopulation, 1500); 

    // ==========================================
    // [3] 물리 및 섭취 로직 업데이트
    // ==========================================
    Events.on(engine, 'afterUpdate', () => {
        const allBodies = Composite.allBodies(engine.world);
        const fishes = allBodies.filter(b => b.label === 'fish');
        const foods = allBodies.filter(b => b.label === 'food');
        const center = { x: canvas.width / 2, y: canvas.height / 2 }; 

        foods.forEach(food => {
            if (food.position.y > canvas.height + 50) {
                Composite.remove(engine.world, food);
            }
        });

        fishes.forEach(me => {
            Body.setVelocity(me, { x: me.velocity.x, y: me.velocity.y - engine.world.gravity.y * 0.9 });

            if (me.size > 450) {
                for (let i = 0; i < 3; i++) {
                    spawn(me.word, me.position.x, me.position.y, 40);
                }
                Composite.remove(engine.world, me);
                return;
            }

            const dx = me.position.x - center.x;
            const dy = me.position.y - center.y;
            if (Math.abs(dx) > boundarySize.width / 2 || Math.abs(dy) > boundarySize.height / 2) {
                const toCenter = Vector.normalise(Vector.sub(center, me.position));
                Body.applyForce(me, me.position, Vector.mult(toCenter, 0.008 * me.size));
            }

            let targetFood = null;
            let minFoodDist = Infinity;
            foods.forEach(food => {
                const dist = Vector.magnitude(Vector.sub(me.position, food.position));
                if (dist < 500 && dist < minFoodDist) {
                    minFoodDist = dist;
                    targetFood = food;
                }
            });

            if (targetFood) {
                const diff = Vector.sub(targetFood.position, me.position);
                const norm = Vector.normalise(diff);
                Body.applyForce(me, me.position, Vector.mult(norm, 0.0008 * me.size));

                if (minFoodDist < me.size + 15) {
                    const oldSize = me.size;
                    me.size += Math.max(targetFood.menuText.length * 1.5, 5);
                    Body.scale(me, me.size / oldSize, me.size / oldSize);
                    me.eatEffect = 20;

                    Composite.remove(engine.world, targetFood);
                }
            } else {
                let nearest = null;
                let minDist = Infinity;
                const visionRange = me.size * 8; 

                fishes.forEach(other => {
                    if (me === other) return;
                    const dist = Vector.magnitude(Vector.sub(me.position, other.position));
                    if (dist < visionRange && dist < minDist) {
                        minDist = dist;
                        nearest = other;
                    }
                });

                if (nearest) {
                    const diff = Vector.sub(nearest.position, me.position);
                    const norm = Vector.normalise(diff);
                    const forceMag = me.size > nearest.size ? 0.0005 : -0.0003;
                    Body.applyForce(me, me.position, Vector.mult(norm, forceMag * me.size));
                }
            }

            fishes.forEach(other => {
                if (me === other || other.removed) return;
                const dist = Vector.magnitude(Vector.sub(me.position, other.position));
                if (dist < me.size * 0.8 && me.size > other.size * 1.15) {
                    const oldSize = me.size;
                    me.size += other.size * 0.2;
                    Body.scale(me, me.size / oldSize, me.size / oldSize);
                    me.eatEffect = 25;
                    other.removed = true;
                    Composite.remove(engine.world, other);
                }
            });

            const speed = Vector.magnitude(me.velocity);
            const limit = me.size > 200 ? 2.5 : 4.0; 
            if (speed > limit) {
                Body.setVelocity(me, Vector.mult(Vector.normalise(me.velocity), limit));
            }
        });
    });

    // ==========================================
    // [4] 렌더링
    // ==========================================
    function drawFish(ctx, f) {
        const { x, y } = f.position;
        const speed = Vector.magnitude(f.velocity);
        if (speed > 0.1) {
            const targetAngle = Math.atan2(f.velocity.y, f.velocity.x);
            let angleDiff = targetAngle - f.currentAngle;
            while (angleDiff < -Math.PI) angleDiff += Math.PI * 2;
            while (angleDiff > Math.PI) angleDiff -= Math.PI * 2;
            f.currentAngle += angleDiff * 0.07;
        }

        f.tailAngle += 0.15;
        if (f.eatEffect > 0) f.eatEffect--;

        ctx.save();
        ctx.translate(x, y);
        ctx.rotate(f.currentAngle);

        if (f.isJackpot || f.eatEffect > 0) {
            ctx.shadowBlur = (f.isJackpot ? 60 : 35);
            ctx.shadowColor = f.color;
        }

        ctx.fillStyle = f.color.replace('hsl', 'hsla').replace(')', `, ${f.eatEffect > 0 ? 0.8 : 0.25})`); 
        ctx.beginPath();
        ctx.ellipse(0, 0, f.size, f.size/2.2, 0, 0, Math.PI * 2);
        ctx.fill();
        
        ctx.strokeStyle = f.color;
        ctx.lineWidth = f.isJackpot ? 3 : 2;
        ctx.stroke();

        ctx.fillStyle = f.color;
        const tailWag = Math.sin(f.tailAngle) * (f.size / 5);
        ctx.beginPath();
        ctx.moveTo(-f.size, 0);
        ctx.lineTo(-f.size * 1.4, -f.size / 2.5 + tailWag);
        ctx.lineTo(-f.size * 1.4, f.size / 2.5 + tailWag);
        ctx.closePath();
        ctx.fill();

        ctx.shadowBlur = 0; 
        ctx.fillStyle = "white";
        ctx.font = `bold ${Math.max(f.size * 0.35, 12)}px Arial`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(f.word, 0, 0);
        ctx.restore();
    }

    function drawFood(ctx, food) {
        const { x, y } = food.position;
        ctx.save();
        ctx.translate(x, y);
        ctx.rotate(food.angle);

        ctx.shadowBlur = 15;
        ctx.shadowColor = food.color;

        ctx.fillStyle = "rgba(255, 251, 0, 0.2)";
        ctx.strokeStyle = food.color;
        ctx.lineWidth = 1.5;

        ctx.beginPath();
        ctx.roundRect(-food.width/2, -food.height/2, food.width, food.height, 8);
        ctx.fill();
        ctx.stroke();

        ctx.shadowBlur = 0;
        ctx.fillStyle = "#ffffff";
        ctx.font = `bold 12px Arial`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(food.menuText, 0, 0);

        ctx.restore();
    }

    (function render() {
        ctx.fillStyle = 'rgba(0, 1, 3, 0.4)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        const bodies = Composite.allBodies(engine.world);
        bodies.filter(b => b.label === 'food').forEach(food => drawFood(ctx, food));
        
        const fishes = bodies.filter(b => b.label === 'fish');
        fishes.forEach(f => drawFish(ctx, f));

        const sorted = fishes.sort((a,b) => b.size - a.size).slice(0, 5);
        document.getElementById('list').innerHTML = sorted.map((f, i) => `
            <div style="margin-bottom:8px; font-size:12px; display:flex; justify-content:space-between; align-items:center;">
                <span style="color:#aaa;">${i+1}.</span>
                <span style="color:${f.color}; font-weight:bold; flex:1; text-align:left; padding:0 10px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">
                    ${f.word}
                </span>
                <span style="color:#aaa; font-size:11px;">${Math.floor(f.size)}</span>
            </div>
        `).join('');
        requestAnimationFrame(render);
    })();

    // ==========================================
    // [5] 나이스(NEIS) 급식 API 호출 파트
    // ==========================================

    // 🔑 제공해주신 API 키 자동 적용 완료
    const HARDCODED_API_KEY = "7fa09fa2743e4c78844dfc42faef8a41"; 

    if (HARDCODED_API_KEY) {
        document.getElementById('api-key').value = HARDCODED_API_KEY;
    }

    async function fetchNeisMeal() {
        const apiKey = document.getElementById('api-key').value.trim() || HARDCODED_API_KEY;
        const officeCode = document.getElementById('office-code').value.trim();
        const schoolCode = document.getElementById('school-code').value.trim();
        const dateInput = document.getElementById('meal-date').value.replace(/-/g, '');

        if (!officeCode || !schoolCode || !dateInput) {
            alert('교육청코드, 학교코드, 날짜를 정확히 입력해주세요!');
            return;
        }

        let url = `https://open.neis.go.kr/hub/mealServiceDietInfo?Type=json&ATPT_OFCDC_SC_CODE=${officeCode}&SD_SCHUL_CODE=${schoolCode}&MLSV_YMD=${dateInput}`;
        
        if (apiKey) {
            url += `&KEY=${apiKey}`;
        }

        try {
            const response = await fetch(url);
            const data = await response.json();

            if (data.mealServiceDietInfo) {
                const rawDDISH = data.mealServiceDietInfo[1].row[0].DDISH_NM;
                
                const cleanMenu = rawDDISH
                    .replace(/<br\/>/g, '\n')
                    .replace(/\([0-9.]+\)/g, '')
                    .replace(/[*&]/g, '')
                    .trim();

                const menuItems = cleanMenu.split('\n').filter(item => item.trim() !== '');

                menuItems.forEach((item, index) => {
                    setTimeout(() => {
                        dropFood(item.trim());
                    }, index * 600);
                });
            } else {
                alert('해당 날짜의 급식 정보가 없거나 코드가 올바르지 않습니다. (주말/방학 여부 확인)');
            }
        } catch (error) {
            console.error('API Fetch Error:', error);
            alert('급식 데이터를 불러오는데 실패했습니다. CORS 문제 또는 네트워크 상태를 확인해주세요.');
        }
    }

    function dropSampleMeal() {
        const samples = ["제육볶음", "돈까스", "배추김치", "잔치국수", "초코케이크", "바나나우유", "해물파전", "된장찌개"];
        samples.forEach((item, index) => {
            setTimeout(() => {
                dropFood(item);
            }, index * 400);
        });
    }

    document.getElementById('fetch-meal-btn').onclick = fetchNeisMeal;
    document.getElementById('sample-meal-btn').onclick = dropSampleMeal;

    document.getElementById('spawn-form').onsubmit = (e) => {
        e.preventDefault();
        const input = document.getElementById('target-text');
        spawn(input.value);
        input.value = '';
    };

    Runner.run(Runner.create(), engine);
    window.onresize = () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; };
</script>
</body>
</html>
