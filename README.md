<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<!-- 适配移动端，禁止缩放以获得类似原生App的体验 -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>午餐大转盘</title>
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
  
  /* 添加区域样式 */
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
    box-shadow: inset 0 1px 3px rgba(0,0,0,0.05);
  }
  .add-btn { 
    padding: 12px 20px; 
    border: none; 
    background-color: #07c160; /* 微信绿 */
    color: white; 
    border-radius: 8px; 
    font-size: 16px; 
    cursor: pointer; 
    font-weight: bold;
  }
  .add-btn:active { background-color: #06ad56; }

  /* 选项列表区域样式 */
  #options-list {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 10px;
    margin-bottom: 30px;
    padding: 0 10px;
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
    box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  }
  .delete-btn {
    background: none;
    border: none;
    color: #ff4d4f;
    cursor: pointer;
    font-weight: bold;
    padding: 0 2px;
    font-size: 16px;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .delete-btn:hover { color: #cf1322; }

  /* 转盘区域样式 */
  #wheel-wrapper { 
    position: relative; 
    width: 300px; 
    height: 300px; 
    margin: 0 auto; 
  }
  /* 顶部指针 */
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
    filter: drop-shadow(0 2px 2px rgba(0,0,0,0.2));
  }
  #wheel {
    width: 100%;
    height: 100%;
    border-radius: 50%;
    box-shadow: 0 6px 15px rgba(0,0,0,0.15);
    /* 核心动画：控制旋转速度和贝塞尔曲线实现阻尼减速效果 */
    transition: transform 4s cubic-bezier(0.25, 0.1, 0.15, 1);
  }

  /* 底部按钮与结果展示 */
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
    box-shadow: 0 4px 10px rgba(24,144,255,0.3);
  }
  .spin-btn:active { transform: translateY(2px); box-shadow: 0 2px 5px rgba(24,144,255,0.3); }
  .spin-btn:disabled { background-color: #ccc; box-shadow: none; cursor: not-allowed; }
  #result { 
    margin-top: 25px; 
    font-size: 22px; 
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

  <!-- 新增：显示所有选项及删除按钮 -->
  <div id="options-list"></div>

  <div id="wheel-wrapper">
    <div id="pointer"></div>
    <!-- 物理尺寸为600x600，CSS尺寸为300x300，以解决移动端Canvas文字模糊问题 -->
    <canvas id="wheel" width="600" height="600"></canvas>
  </div>

  <div class="controls">
    <button class="spin-btn" id="spinBtn" onclick="spin()">开始随机</button>
  </div>

  <div id="result"></div>

  <script>
    const canvas = document.getElementById("wheel");
    const ctx = canvas.getContext("2d");
    const spinBtn = document.getElementById("spinBtn");
    
    // 默认午餐选项
    let options = ["小炒先生", "香菇面", "青海牛肉面", "面相爷", "烩面"];
    let currentRotation = 0;
    let isSpinning = false;

    // 预设的高级配色表
    const colors = ["#ff9e9d", "#ffb56b", "#f9e076", "#c3e27d", "#89d3e8", "#c8a2c8", "#ffc0cb", "#ffd700", "#98fb98", "#afeeee"];

    // 渲染选项列表（新增功能）
    function renderOptionsList() {
      const listContainer = document.getElementById("options-list");
      listContainer.innerHTML = "";
      options.forEach((opt, index) => {
        const tag = document.createElement("div");
        tag.className = "option-tag";
        tag.innerHTML = `
          <span>${opt}</span>
          <button class="delete-btn" onclick="deleteItem(${index})" title="删除">×</button>
        `;
        listContainer.appendChild(tag);
      });
    }

    // 删除选项（新增功能）
    function deleteItem(index) {
      if (isSpinning) return;
      if (options.length <= 2) {
        alert("转盘至少需要保留 2 个选项哦！");
        return;
      }
      const removed = options.splice(index, 1);
      renderOptionsList();
      drawWheel();
      document.getElementById("result").innerText = "已删除：" + removed[0];
    }

    // 绘制转盘
    function drawWheel() {
      const numOptions = options.length;
      const arcSize = (2 * Math.PI) / numOptions;

      ctx.clearRect(0, 0, 600, 600);
      ctx.translate(300, 300);

      for(let i = 0; i < numOptions; i++) {
        // 计算起始角度，偏移 -90度(置顶) 并减去半个扇形角度，使得第一个选项完美居中于正上方
        const angle = i * arcSize - Math.PI / 2 - arcSize / 2; 
        
        // 绘制扇形
        ctx.beginPath();
        ctx.fillStyle = colors[i % colors.length];
        ctx.moveTo(0, 0);
        ctx.arc(0, 0, 300, angle, angle + arcSize, false);
        ctx.fill();
        
        // 绘制扇形分隔线
        ctx.strokeStyle = "#ffffff";
        ctx.lineWidth = 4;
        ctx.stroke();

        // 绘制文字
        ctx.save();
        ctx.fillStyle = "#333333";
        ctx.font = "bold 28px sans-serif";
        // 将画布原点移动到扇形中间偏外围的位置
        ctx.translate(Math.cos(angle + arcSize / 2) * 190, Math.sin(angle + arcSize / 2) * 190);
        // 旋转画布使文字顺着半径方向
        ctx.rotate(angle + arcSize / 2 + Math.PI / 2);
        ctx.fillText(options[i], -ctx.measureText(options[i]).width / 2, 10);
        ctx.restore();
      }
      ctx.translate(-300, -300);
    }

    // 添加选项
    function addItem() {
      if (isSpinning) return;
      const input = document.getElementById("newItem");
      const val = input.value.trim();
      if(val !== "") {
        options.push(val);
        input.value = "";
        renderOptionsList();
        drawWheel();
        document.getElementById("result").innerText = "已成功添加：" + val;
      }
    }

    // 监听回车键添加
    document.getElementById("newItem").addEventListener("keypress", function(event) {
      if (event.key === "Enter") {
        event.preventDefault();
        addItem();
      }
    });

    // 旋转逻辑
    function spin() {
      if (isSpinning || options.length === 0) return;
      isSpinning = true;
      spinBtn.disabled = true;
      document.getElementById("result").innerText = "命运的齿轮开始转动...";

      // 核心公平性保障：Math.random() 生成完全均匀的 [0, 1) 随机数
      const randomDegree = Math.random() * 360;
      // 基础旋转圈数（至少转5圈增加视觉效果）
      const baseSpins = 5 * 360; 
      
      currentRotation += baseSpins + randomDegree;
      canvas.style.transform = `rotate(${currentRotation}deg)`;

      // 等待CSS动画结束（4秒）后计算结果
      setTimeout(() => {
        const actualRotation = currentRotation % 360;
        // 抵消顺时针旋转，计算此时指向正上方(0度)的是哪一个区块的索引
        const pointingAngle = (360 - actualRotation) % 360;
        const sliceAngle = 360 / options.length;
        const winnerIndex = Math.floor(pointingAngle / sliceAngle);

        document.getElementById("result").innerText = "🎉 午餐决定是：" + options[winnerIndex] + "！";
        isSpinning = false;
        spinBtn.disabled = false;
      }, 4000);
    }

    // 页面加载完毕后初始化转盘和列表
    window.onload = function() {
      renderOptionsList();
      drawWheel();
    };
  </script>
</body>
</html>
