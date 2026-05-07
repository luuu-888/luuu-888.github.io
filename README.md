<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>午餐大转盘-GitHub同步版</title>
<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    text-align: center;
    background-color: #f7f7f7;
    margin: 0;
    padding: 20px;
    touch-action: manipulation;
  }
  h2 { color: #333; margin-bottom: 20px; }
  .input-group { 
    display: flex; 
    justify-content: center; 
    gap: 10px; 
    margin-bottom: 15px; 
  }
  input { 
    padding: 12px; 
    border: 1px solid #ddd; 
    border-radius: 8px; 
    outline: none; 
    width: 60%; 
    font-size: 16px; 
  }
  .add-btn { 
    padding: 12px 20px; 
    border: none; 
    background-color: #07c160; 
    color: white; 
    border-radius: 8px; 
    font-size: 16px; 
    cursor: pointer; 
    font-weight: bold;
  }
  #options-list {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 10px;
    margin-bottom: 30px;
  }
  .option-tag {
    background-color: #e6f7ff;
    border: 1px solid #91d5ff;
    color: #1890ff;
    padding: 6px 12px;
    border-radius: 20px;
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 14px;
  }
  .delete-btn {
    background: none;
    border: none;
    color: #ff4d4f;
    cursor: pointer;
    font-weight: bold;
    font-size: 16px;
  }
  #wheel-wrapper { 
    position: relative; 
    width: 300px; 
    height: 300px; 
    margin: 0 auto; 
  }
  #pointer {
    position: absolute;
    top: -15px;
    left: 50%;
    transform: translateX(-50%);
    width: 0;
    height: 0;
    border-left: 15px solid transparent;
    border-right: 15px solid transparent;
    border-top: 35px solid #ff4d4f;
    z-index: 10;
  }
  #wheel {
    width: 100%;
    height: 100%;
    border-radius: 50%;
    box-shadow: 0 6px 15px rgba(0,0,0,0.15);
    transition: transform 4s cubic-bezier(0.25, 0.1, 0.15, 1);
  }
  .controls { margin-top: 40px; }
  .spin-btn { 
    background-color: #1890ff; 
    color: white;
    padding: 15px 50px; 
    font-size: 20px; 
    border-radius: 30px;
    border: none;
    font-weight: bold;
    cursor: pointer;
  }
  .spin-btn:disabled { background-color: #ccc; }
  #result { 
    margin-top: 25px; 
    font-size: 18px; 
    font-weight: bold; 
    color: #ff4d4f; 
    min-height: 33px; 
  }
</style>
</head>
<body>

  <h2>今天午餐吃什么？</h2>

  <div class="input-group">
    <input type="text" id="newItem" placeholder="输入新的午餐选项">
    <button class="add-btn" onclick="addItem()">添加</button>
  </div>

  <div id="options-list"></div>

  <div id="wheel-wrapper">
    <div id="pointer"></div>
    <canvas id="wheel" width="600" height="600"></canvas>
  </div>

  <div class="controls">
    <button class="spin-btn" id="spinBtn" onclick="spin()">开始随机</button>
  </div>

  <div id="result">正在加载数据...</div>

  <script>
    // ================= 配置区（请修改这里） =================
    const OWNER = "luuu-888"; 
    const REPO = "luuu-888.github.io";
    const GITHUB_TOKEN = "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"; 
    // ======================================================

    const canvas = document.getElementById("wheel");
    const ctx = canvas.getContext("2d");
    const spinBtn = document.getElementById("spinBtn");
    const resultEl = document.getElementById("result");
    
    let options = [];
    const colors = ["#ff9e9d", "#ffb56b", "#f9e076", "#c3e27d", "#89d3e8", "#c8a2c8", "#ffc0cb", "#ffd700", "#98fb98", "#afeeee"];
    let currentRotation = 0;
    let isSpinning = false;

    // 1. 从 GitHub 读取数据
    async function loadOptions() {
      try {
        const response = await fetch(`https://raw.githubusercontent.com/${OWNER}/${REPO}/main/options.json?v=${new Date().getTime()}`);
        if (response.ok) {
          options = await response.json();
          resultEl.innerText = "数据加载成功";
        } else {
          options = ["小炒先生", "香菇面", "牛肉面"]; // 失败备选
          resultEl.innerText = "读取失败，使用默认值";
        }
      } catch (e) {
        options = ["小炒先生", "香菇面", "牛肉面"];
        resultEl.innerText = "网络错误";
      }
      renderOptionsList();
      drawWheel();
    }

    // 2. 同步到 GitHub (触发 Action)
    async function syncToGitHub() {
      resultEl.innerText = "⏳ 正在同步到云端...";
      try {
        const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/dispatches`, {
          method: 'POST',
          headers: {
            'Authorization': `token ${GITHUB_TOKEN}`,
            'Accept': 'application/vnd.github.v3+json',
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            event_type: 'update-options',
            client_payload: { options: JSON.stringify(options) }
          })
        });

        if (res.ok) {
          resultEl.innerText = "✅ 同步中！GitHub 正在保存，预计 1 分钟后生效";
        } else {
          const err = await res.json();
          resultEl.innerText = "❌ 同步失败: " + (err.message || "未知错误");
        }
      } catch (e) {
        resultEl.innerText = "❌ 网络请求失败";
      }
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
      ctx.save();
      ctx.translate(300, 300);

      for(let i = 0; i < numOptions; i++) {
        const angle = i * arcSize - Math.PI / 2 - arcSize / 2; 
        ctx.beginPath();
        ctx.fillStyle = colors[i % colors.length];
        ctx.moveTo(0, 0);
        ctx.arc(0, 0, 300, angle, angle + arcSize, false);
        ctx.fill();
        ctx.strokeStyle = "#ffffff";
        ctx.lineWidth = 4;
        ctx.stroke();

        ctx.save();
        ctx.fillStyle = "#333333";
        ctx.font = "bold 28px sans-serif";
        ctx.translate(Math.cos(angle + arcSize / 2) * 190, Math.sin(angle + arcSize / 2) * 190);
        ctx.rotate(angle + arcSize / 2 + Math.PI / 2);
        ctx.fillText(options[i], -ctx.measureText(options[i]).width / 2, 10);
        ctx.restore();
      }
      ctx.restore();
    }

    function addItem() {
      if (isSpinning) return;
      const input = document.getElementById("newItem");
      const val = input.value.trim();
      if(val !== "") {
        options.push(val);
        input.value = "";
        renderOptionsList();
        drawWheel();
        syncToGitHub();
      }
    }

    function deleteItem(index) {
      if (isSpinning) return;
      if (options.length <= 2) {
        alert("至少保留 2 个选项！");
        return;
      }
      options.splice(index, 1);
      renderOptionsList();
      drawWheel();
      syncToGitHub();
    }

    function spin() {
      if (isSpinning || options.length === 0) return;
      isSpinning = true;
      spinBtn.disabled = true;
      resultEl.innerText = "命运的齿轮开始转动...";

      const randomDegree = Math.random() * 360;
      const baseSpins = 5 * 360; 
      currentRotation += baseSpins + randomDegree;
      canvas.style.transform = `rotate(${currentRotation}deg)`;

      setTimeout(() => {
        const actualRotation = currentRotation % 360;
        const sliceAngle = 360 / options.length;
        const pointingAngle = (360 - actualRotation) % 360;
        const adjustedAngle = (pointingAngle + sliceAngle / 2) % 360;
        const winnerIndex = Math.floor(adjustedAngle / sliceAngle);

        resultEl.innerText = "🎉 午餐决定是：" + options[winnerIndex] + "！";
        isSpinning = false;
        spinBtn.disabled = false;
      }, 4000);
    }

    document.getElementById("newItem").addEventListener("keypress", (e) => {
      if (e.key === "Enter") addItem();
    });

    window.onload = loadOptions;
  </script>
</body>
</html>
