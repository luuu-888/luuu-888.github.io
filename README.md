<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>午餐大转盘-Supabase云同步版</title>
<!-- 引入 Supabase SDK -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<style>
  /* 保持之前的样式不变... */
  body { font-family: -apple-system, sans-serif; text-align: center; background: #f7f7f7; padding: 20px; }
  .input-group { display: flex; justify-content: center; gap: 10px; margin-bottom: 15px; }
  input { padding: 12px; border: 1px solid #ddd; border-radius: 8px; width: 60%; font-size: 16px; }
  .add-btn { padding: 12px 20px; border: none; background: #07c160; color: white; border-radius: 8px; font-weight: bold; cursor: pointer; }
  #options-list { display: flex; flex-wrap: wrap; justify-content: center; gap: 10px; margin-bottom: 30px; }
  .option-tag { background: #e6f7ff; border: 1px solid #91d5ff; color: #1890ff; padding: 6px 12px; border-radius: 20px; font-size: 14px; }
  .delete-btn { background: none; border: none; color: #ff4d4f; font-weight: bold; cursor: pointer; }
  #wheel-wrapper { position: relative; width: 300px; height: 300px; margin: 0 auto; }
  #pointer { position: absolute; top: -15px; left: 50%; transform: translateX(-50%); border-left: 15px solid transparent; border-right: 15px solid transparent; border-top: 35px solid #ff4d4f; z-index: 10; }
  #wheel { width: 100%; height: 100%; border-radius: 50%; box-shadow: 0 6px 15px rgba(0,0,0,0.15); transition: transform 4s cubic-bezier(0.25, 0.1, 0.15, 1); }
  .controls { margin-top: 40px; }
  .spin-btn { background: #1890ff; color: white; padding: 15px 50px; font-size: 20px; border-radius: 30px; border: none; font-weight: bold; }
  #result { margin-top: 25px; font-size: 18px; font-weight: bold; color: #ff4d4f; min-height: 33px; }
</style>
</head>
<body>

  <h2>今天午餐吃什么？</h2>

  <div class="input-group">
    <input type="text" id="newItem" placeholder="输入新的午餐选项">
    <button class="add-btn" onclick="addItem()">添加</button>
  </div>
  <div id="options-list"></div>
  <div id="wheel-wrapper"><div id="pointer"></div><canvas id="wheel" width="600" height="600"></canvas></div>
  <div class="controls"><button class="spin-btn" id="spinBtn" onclick="spin()">开始随机</button></div>
  <div id="result">正在连接云端...</div>

  <script>
    // === 配置区 ===
    const SUPABASE_URL = 'https://wfkgbfubwlxecymjtvvm.supabase.co';
    const SUPABASE_KEY = 'sb_publishable_TYlYIcT9SvkZdj-T1sW3KA_UTugHksC';
    // =============

    const _supabase = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
    const resultEl = document.getElementById("result");
    const canvas = document.getElementById("wheel");
    const ctx = canvas.getContext("2d");
    
    let options = [];
    let currentRotation = 0;
    let isSpinning = false;
    const colors = ["#ff9e9d", "#ffb56b", "#f9e076", "#c3e27d", "#89d3e8", "#c8a2c8", "#ffc0cb", "#ffd700"];

    // 从云端获取数据
    async function loadData() {
      const { data, error } = await _supabase.from('lunch_data').select('list').eq('id', 1).single();
      if (data) {
        options = data.list;
        resultEl.innerText = "云端同步就绪";
        renderOptionsList();
        drawWheel();
      } else {
        resultEl.innerText = "数据初始化失败，请检查配置";
      }
    }

    // 更新云端数据
    async function updateCloud() {
      resultEl.innerText = "⏳ 正在同步...";
      const { error } = await _supabase.from('lunch_data').update({ list: options }).eq('id', 1);
      if (error) resultEl.innerText = "❌ 同步失败";
      else resultEl.innerText = "✅ 已同步";
    }

    function renderOptionsList() {
      const listContainer = document.getElementById("options-list");
      listContainer.innerHTML = "";
      options.forEach((opt, index) => {
        const tag = document.createElement("div");
        tag.className = "option-tag";
        tag.innerHTML = `<span>${opt}</span><button class="delete-btn" onclick="deleteItem(${index})">×</button>`;
        listContainer.appendChild(tag);
      });
    }

    function drawWheel() {
      const numOptions = options.length;
      if (numOptions === 0) return;
      const arcSize = (2 * Math.PI) / numOptions;
      ctx.clearRect(0, 0, 600, 600);
      ctx.save(); ctx.translate(300, 300);
      for(let i = 0; i < numOptions; i++) {
        const angle = i * arcSize - Math.PI / 2 - arcSize / 2; 
        ctx.beginPath(); ctx.fillStyle = colors[i % colors.length];
        ctx.moveTo(0, 0); ctx.arc(0, 0, 300, angle, angle + arcSize, false); ctx.fill();
        ctx.strokeStyle = "#fff"; ctx.lineWidth = 4; ctx.stroke();
        ctx.save(); ctx.fillStyle = "#333"; ctx.font = "bold 28px sans-serif";
        ctx.translate(Math.cos(angle + arcSize / 2) * 190, Math.sin(angle + arcSize / 2) * 190);
        ctx.rotate(angle + arcSize / 2 + Math.PI / 2);
        ctx.fillText(options[i], -ctx.measureText(options[i]).width / 2, 10); ctx.restore();
      }
      ctx.restore();
    }

    async function addItem() {
      const val = document.getElementById("newItem").value.trim();
      if(val && !isSpinning) {
        options.push(val);
        document.getElementById("newItem").value = "";
        renderOptionsList(); drawWheel();
        await updateCloud();
      }
    }

    async function deleteItem(index) {
      if (!isSpinning && options.length > 2) {
        options.splice(index, 1);
        renderOptionsList(); drawWheel();
        await updateCloud();
      }
    }

    function spin() {
      if (isSpinning || options.length === 0) return;
      isSpinning = true;
      const randomDegree = Math.random() * 360;
      currentRotation += 1800 + randomDegree;
      canvas.style.transform = `rotate(${currentRotation}deg)`;
      setTimeout(() => {
        const actualRotation = currentRotation % 360;
        const sliceAngle = 360 / options.length;
        const winnerIndex = Math.floor(((360 - actualRotation + sliceAngle / 2) % 360) / sliceAngle);
        resultEl.innerText = "🎉 决定是：" + options[winnerIndex];
        isSpinning = false;
      }, 4000);
    }

    window.onload = loadData;
  </script>
</body>
</html>
