<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>AI 텍스트 아쿠아리움: 급식 에디션</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000205; color: white; font-family: 'Pretendard', sans-serif; }
        canvas { display: block; }

        /* [수정 3] 오른쪽 상단 급식표 UI */
        #meal-board {
            position: absolute; top: 20px; right: 20px;
            background: rgba(0, 0, 0, 0.75); padding: 15px; border-radius: 15px;
            border: 1px solid rgba(255,255,255,0.1); backdrop-filter: blur(10px);
            width: 220px; z-index: 10; font-size: 13px;
        }
        #meal-board h3 { margin: 0 0 10px 0; font-size: 14px; color: #00f2fe; text-align: center; }
        #meal-list { list-style: none; padding: 0; margin: 0; max-height: 250px; overflow-y: auto; }
        #meal-list li { padding: 4px 0; border-bottom: 1px solid rgba(255,255,255,0.05); color: #ddd; font-size: 12px; }

        /* 좌측 상단 리더보드 (기존 유지) */
        #leaderboard { position: absolute; top: 20px; left: 20px; background: rgba(0,0,0,0.7); padding: 15px; border-radius: 15px; border: 1px solid rgba(255,255,255,0.08); width: 220px; z-index: 5; pointer-events: none; }

        /* 하단 투하 UI (기존 유지) */
        #ui { position: absolute; bottom: 30px; left: 50%; transform: translateX(-50%); z-index: 10; width: 100%; max-width: 450px; }
        #input-box { display: flex; gap: 10px; background: rgba(255,255,255,0.03); padding: 12px; border-radius: 20px; backdrop-filter: blur(20px); border: 1px solid rgba(255,255,255,0.08); box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        input { flex: 1; background: transparent; border: none; color: white; outline: none; font-size: 16px; padding-left: 10px; }
        input::placeholder { color: rgba(255,255,255,0.3); }
        button { background: linear-gradient(45deg, #00f2fe, #4facfe); border: none; padding: 10px 20px; border-radius: 12px; font-weight: bold; color: white; cursor: pointer; transition: 0.2s; white-space: nowrap; }
        button:hover { transform: scale(1.05); box-shadow: 0 0 15px rgba(0,242,254,0.6); }
    </style>
</head>
<body>

<div id="leaderboard">
    <h3 style="margin:0 0 10px 0; font-size:14px; color:#aaa; text-align:center; text-transform:uppercase; letter-spacing:1px;">👑 생태계 순위</h3>
    <div id="list"></div>
</div>

<!-- 오른쪽 상단 급식표 -->
<div id="meal-board">
    <h3>🍱오늘의 급식표</h3>
    <ul id="meal-list">
        <li style="color:#888; text-align:center;">급식 불러오는 중...</li>
    </ul>
    <button id="retry-btn" style="width:100%; margin-top:10px; font-size:11px; padding:6px;">급식 다시 투하</button>
</div>

<div id="ui">
    <form id="spawn-form">
        <div id="input-box">
            <input type="text" id="target-text" placeholder="단어를 입력해 생명체 투하..." required maxlength="15" autocomplete="off">
            <button type="submit">투하</button>
        </div>
    </form>
</div>

<canvas id="world"></canvas>

<script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.19.0/matter.min.js"></script>
<script>
    const { Engine, Runner, Bodies, Composite, Events, Body, Vector } = Matter;
    const engine = Engine.create({ enableSleeping: false });
    
    // 급식 밥이천천히 떨어지도록 중력 적용
    engine.world.gravity.y = 0.2;

    const canvas = document.getElementById('world');
    const ctx = canvas.getContext('2d');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    const boundarySize = { width: 800, height: 700 }; 
    const autoWords = ["새우", "치어", "플랑크톤", "조개", "해파리", "멸치", "해조류", "송사리", "구피","상어","오징어"]; 

    // 물고기 스폰 (기존 크기보다 확대 요청 반영)
    const spawn = (text, x, y, sizeOverride = null) => {
        let isJackpot = sizeOverride ? false : Math.random() < 0.05;
        let baseSize = sizeOverride || (30 + (text.length * 6)); 
        if (isJackpot) baseSize *= 3;

        const spawnX = x || (canvas.width / 2 + (Math.random() - 0.5) * (boundarySize.width * 0.8));
        const spawnY = y || (canvas.height / 2 + (Math.random() - 0.5) * (boundarySize.height * 0.8));

        const fish = Bodies.circle(spawnX, spawnY, baseSize, {
            frictionAir: 0.1,
            isSensor: true,
            label: 'fish'
        });

        fish.word = text;
        fish.size = baseSize;
        fish.isJackpot = isJackpot;
        
        const hue = isJackpot ? 300 : Math.random() * 360; 
        fish.color = `hsl(${hue}, 100%, 65%)`; 
        fish.currentAngle = Math.random() * Math.PI * 2;
        fish.tailAngle = 0;
        fish.eatEffect = 0;

        Body.setVelocity(fish, { x: (Math.random()-0.5)*3, y: (Math.random()-0.5)*3 });
        Composite.add(engine.world, fish);
        return fish;
    };

    // [수정 2] 급식 밥 투하 (글씨 적힌 동그라미 밥 형태)
    const dropFood = (menuText) => {
        // 글씨 길이에 비례해 동그라미 반지름(radius) 결정 (물고기보다 작게)
        const radius = Math.min(Math.max(menuText.length * 5 + 12, 18), 45);
        const spawnX = Math.random() * (canvas.width - 300) + 150;

        const food = Bodies.circle(spawnX, -30, radius, {
            frictionAir: 0.05,
            restitution: 0.4,
            label: 'food'
        });

        food.word = menuText;
        food.size = radius;
        food.color = '#fffb00'; // 급식 밥 네온 노란색

        Composite.add(engine.world, food);
    };

    // [수정 1] 물고기 생성 속도 낮춤 (기존 1500ms -> 5000ms 로 변경)
    const checkPopulation = () => {
        const bodies = Composite.allBodies(engine.world).filter(b => b.label === 'fish');
        if (bodies.length < 7) {
            const randomWord = autoWords[Math.floor(Math.random() * autoWords.length)];
            spawn(randomWord);
        }
    };
    setInterval(checkPopulation, 5000); 

    Events.on(engine, 'afterUpdate', () => {
        const bodies = Composite.allBodies(engine.world);
        const fishes = bodies.filter(b => b.label === 'fish');
        const foods = bodies.filter(b => b.label === 'food');
        const center = { x: canvas.width / 2, y: canvas.height / 2 };

        // 바닥 나간 밥 제거
        foods.forEach(food => {
            if (food.position.y > canvas.height + 50) {
                Composite.remove(engine.world, food);
            }
        });

        fishes.forEach(me => {
            // 물고기는 중력에 영향받지 않도록 보정
            Body.setVelocity(me, { x: me.velocity.x, y: me.velocity.y - engine.world.gravity.y * 0.9 });

            if (me.size > 350) {
                for (let i = 0; i < 3; i++) {
                    spawn(me.word, me.position.x, me.position.y, 30);
                }
                Composite.remove(engine.world, me);
                return;
            }

            const dx = me.position.x - center.x;
            const dy = me.position.y - center.y;
            if (Math.abs(dx) > boundarySize.width / 2 || Math.abs(dy) > boundarySize.height / 2) {
                const toCenter = Vector.normalise(Vector.sub(center, me.position));
                Body.applyForce(me, me.position, Vector.mult(toCenter, 0.007 * me.size));
            }

            // 1. 급식 밥 동그라미를 우선 탐색 및 추적
            let nearestFood = null;
            let minFoodDist = Infinity;
            foods.forEach(food => {
                const dist = Vector.magnitude(Vector.sub(me.position, food.position));
                if (dist < 400 && dist < minFoodDist) {
                    minFoodDist = dist;
                    nearestFood = food;
                }
            });

            if (nearestFood) {
                const diff = Vector.sub(nearestFood.position, me.position);
                const norm = Vector.normalise(diff);
                Body.applyForce(me, me.position, Vector.mult(norm, 0.0007 * me.size));

                // 밥 섭취
                if (minFoodDist < me.size + nearestFood.size) {
                    const oldSize = me.size;
                    me.size += nearestFood.size * 0.4;
                    Body.scale(me, me.size / oldSize, me.size / oldSize);
                    me.eatEffect = 20;
                    Composite.remove(engine.world, nearestFood);
                }
            } else {
                // 2. 밥이 없으면 다른 물고기 유영
                let nearest = null;
                let minDist = Infinity;
                const visionRange = me.size * 12; 

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
                    const forceMag = me.size > nearest.size ? 0.0006 : -0.0004;
                    Body.applyForce(me, me.position, Vector.mult(norm, forceMag * me.size));
                }
            }

            // 3. 기존 물고기 상호 포식 유지
            fishes.forEach(other => {
                if (me === other || other.removed) return;
                const dist = Vector.magnitude(Vector.sub(me.position, other.position));
                if (dist < me.size * 0.9 && me.size > other.size * 1.05) {
                    const oldSize = me.size;
                    me.size += other.size * 0.25;
                    Body.scale(me, me.size / oldSize, me.size / oldSize);
                    me.eatEffect = 20;
                    other.removed = true;
                    Composite.remove(engine.world, other);
                }
            });

            const speed = Vector.magnitude(me.velocity);
            const limit = me.size > 150 ? 2.0 : 3.5; 
            if (speed > limit) {
                Body.setVelocity(me, Vector.mult(Vector.normalise(me.velocity), limit));
            }
        });
    });

    // 물고기 그리기 (기존 코드 100% 동일)
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
            ctx.shadowBlur = (f.isJackpot ? 60 : 30);
            ctx.shadowColor = f.color;
        }

        ctx.fillStyle = f.color.replace('hsl', 'hsla').replace(')', `, ${f.eatEffect > 0 ? 0.8 : 0.2})`); 
        ctx.beginPath();
        ctx.ellipse(0, 0, f.size, f.size/2.2, 0, 0, Math.PI * 2);
        ctx.fill();
        
        ctx.strokeStyle = f.color;
        ctx.lineWidth = f.isJackpot ? 3 : 1.5;
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
        ctx.font = `bold ${f.size * 0.4}px Arial`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(f.word, 0, 0);
        ctx.restore();
    }

    // 동그라미 밥 그리기
    function drawFood(ctx, food) {
        const { x, y } = food.position;
        ctx.save();
        ctx.translate(x, y);

        ctx.shadowBlur = 15;
        ctx.shadowColor = food.color;

        ctx.fillStyle = "rgba(255, 251, 0, 0.25)";
        ctx.strokeStyle = food.color;
        ctx.lineWidth = 1.5;

        ctx.beginPath();
        ctx.arc(0, 0, food.size, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        ctx.shadowBlur = 0;
        ctx.fillStyle = "white";
        ctx.font = `bold ${Math.max(food.size * 0.45, 10)}px Arial`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(food.word, 0, 0);

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
    // 나이스(NEIS) 급식 API 연동
    // ==========================================
    const API_KEY = "7fa09fa2743e4c78844dfc42faef8a41";
    const OFFICE_CODE = "B10";   // 서울특별시교육청 (필요시 변경)
    const SCHOOL_CODE = "7010537"; // 강동고등학교 예시 (필요시 변경)

    async function loadNeisMeal() {
        const today = new Date().toISOString().slice(0, 10).replace(/-/g, '');
        const url = `https://open.neis.go.kr/hub/mealServiceDietInfo?Type=json&KEY=${API_KEY}&ATPT_OFCDC_SC_CODE=${OFFICE_CODE}&SD_SCHUL_CODE=${SCHOOL_CODE}&MLSV_YMD=${today}`;
        const mealListEl = document.getElementById('meal-list');

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

                // 오른쪽 급식표 UI 업데이트
                mealListEl.innerHTML = menuItems.map(item => `<li>• ${item}</li>`).join('');

                // 밥 투하 (0.5초 간격)
                menuItems.forEach((item, idx) => {
                    setTimeout(() => dropFood(item.trim()), idx * 500);
                });
            } else {
                // 주말/방학 등 급식 없을 때 샘플 급식으로 자동 전환
                mealListEl.innerHTML = `<li style="color:#ffb199;">(오늘 급식 없음 - 샘플)</li>`;
                dropSampleMeal();
            }
        } catch (e) {
            mealListEl.innerHTML = `<li style="color:#ffb199;">(샘플 급식 투하 중)</li>`;
            dropSampleMeal();
        }
    }

    function dropSampleMeal() {
        const samples = [ " 예시 "];
        const mealListEl = document.getElementById('meal-list');
        mealListEl.innerHTML += samples.map(item => `<li>• ${item}</li>`).join('');
        samples.forEach((item, idx) => {
            setTimeout(() => dropFood(item), idx * 500);
        });
    }

    document.getElementById('retry-btn').onclick = loadNeisMeal;

    document.getElementById('spawn-form').onsubmit = (e) => {
        e.preventDefault();
        spawn(document.getElementById('target-text').value);
        document.getElementById('target-text').value = '';
    };

    // 초기 실행
    loadNeisMeal();

    Runner.run(Runner.create(), engine);
    window.onresize = () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; };
</script>
</body>
</html>
