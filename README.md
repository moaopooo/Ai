<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8">
  <title>百家樂預測系統</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    button { margin: 5px; padding: 5px 10px; }
    table { margin-top: 10px; width: 100%; border-collapse: collapse; }
    th, td { text-align: center; padding: 6px; border: 1px solid #ccc; }
    .predict-area { margin-top: 15px; font-size: 18px; }
  </style>
</head>
<body>

<h2>🎰 百家樂預測系統</h2>

<div>
  <button onclick="addResult('B')">莊</button>
  <button onclick="addResult('P')">閒</button>
  <button onclick="addResult('T')">和</button>
  <button onclick="undo()">復原</button>
</div>

<div style="margin-top: 10px;">
  <label for="strategy">預測策略：</label>
  <select id="strategy" onchange="manualChangeStrategy()">
    <option value="default">預設策略（前5局 + 大路）</option>
    <option value="reverse">反向預測策略</option>
    <option value="small-road">小路參考策略</option>
  </select>
  <button onclick="clearHistory()">清除預測紀錄</button>
</div>

<div class="predict-area" id="analysis">預測分析結果會出現在這裡</div>

<h3>📋 預測歷史記錄</h3>
<table>
  <thead>
    <tr>
      <th>局數</th>
      <th>預測</th>
      <th>實際</th>
      <th>命中</th>
      <th>策略</th>
    </tr>
  </thead>
  <tbody id="historyBody"></tbody>
</table>

<script>
let results = [];
let predictionHistory = JSON.parse(localStorage.getItem("predictionHistory")) || [];

let predictionLog = {
  lastPrediction: null,
  correctCount: 0,
  wrongCount: 0,
  currentStrategy: "default"
};

function addResult(result) {
  results.push(result);

  if (predictionLog.lastPrediction && (result === "B" || result === "P")) {
    if (result === predictionLog.lastPrediction) {
      predictionLog.correctCount++;
      predictionLog.wrongCount = 0;
    } else {
      predictionLog.wrongCount++;
    }

    predictionHistory.push({
      round: predictionHistory.length + 1,
      predict: predictionLog.lastPrediction,
      actual: result,
      correct: result === predictionLog.lastPrediction,
      strategy: predictionLog.currentStrategy
    });

    localStorage.setItem("predictionHistory", JSON.stringify(predictionHistory));
    updatePredictionHistoryTable();

    if (predictionLog.wrongCount >= 2) {
      if (predictionLog.currentStrategy === "default") {
        predictionLog.currentStrategy = "reverse";
      } else if (predictionLog.currentStrategy === "reverse") {
        predictionLog.currentStrategy = "small-road";
      } else {
        predictionLog.currentStrategy = "default";
      }
      predictionLog.wrongCount = 0;
    }
  }

  renderAll();
}

function undo() {
  if (results.length > 0) results.pop();
  renderAll();
}

function manualChangeStrategy() {
  const selected = document.getElementById("strategy").value;
  predictionLog.currentStrategy = selected;
  predictionLog.wrongCount = 0;
  predictNext();
}

function renderAll() {
  document.getElementById("strategy").value = predictionLog.currentStrategy;
  predictNext();
  updatePredictionHistoryTable();
}

function predictNext() {
  if (results.length < 5) {
    document.getElementById("analysis").textContent = "資料不足，請至少輸入 5 局以上資料。";
    predictionLog.lastPrediction = null;
    return;
  }

  const last5 = results.slice(-5).filter(r => r === "B" || r === "P");
  let prediction = "";

  switch (predictionLog.currentStrategy) {
    case "default":
      const allSame = last5.every(r => r === last5[0]);
      if (allSame && last5.length >= 3) {
        prediction = last5[0];
      } else if (last5.length === 5 && last5.every((v, i, arr) => i === 0 || v !== arr[i - 1])) {
        prediction = last5[4] === "B" ? "P" : "B";
      } else {
        const trend = getLastBigRoadColumn();
        const reds = trend.filter(x => x === "B").length;
        const blues = trend.filter(x => x === "P").length;
        prediction = reds > blues ? "B" : blues > reds ? "P" : last5[last5.length - 1];
      }
      break;
    case "reverse":
      prediction = predictionLog.lastPrediction === "B" ? "P" : "B";
      break;
    case "small-road":
      const small = getSmallRoadTrend();
      prediction = small === "R" ? "B" : "P";
      break;
  }

  predictionLog.lastPrediction = prediction;
  document.getElementById("analysis").textContent =
    `預測下一局：${prediction === "B" ? "莊" : "閒"}（策略：${predictionLog.currentStrategy}） ｜ 錯誤次數：${predictionLog.wrongCount}`;
}

function getLastBigRoadColumn() {
  let grid = Array.from({ length: 6 }, () => []);
  let col = 0, row = 0, last = null;

  for (let i = 0; i < results.length; i++) {
    const curr = results[i];
    if (curr !== "B" && curr !== "P") continue;

    if (curr === last) {
      if (row < 5 && !grid[row + 1][col]) row++;
      else col++;
    } else {
      col++;
      row = 0;
    }
    grid[row][col] = curr;
    last = curr;
  }

  const column = [];
  for (let y = 0; y < 6; y++) {
    if (grid[y][col]) column.push(grid[y][col]);
  }
  return column;
}

function getSmallRoadTrend() {
  let baseGrid = Array.from({ length: 6 }, () => []);
  let col = 0, row = 0, last = null;

  for (let i = 0; i < results.length; i++) {
    const curr = results[i];
    if (curr !== "B" && curr !== "P") continue;

    if (curr === last) {
      if (row < 5 && !baseGrid[row + 1][col]) row++;
      else col++;
    } else {
      col++;
      row = 0;
    }
    baseGrid[row][col] = curr;
    last = curr;
  }

  const small = generateDerivedRoad(baseGrid, 2);
  if (small.length < 3) return "R";
  const reds = small.slice(-3).filter(x => x === "R").length;
  const blues = small.slice(-3).filter(x => x === "B").length;
  return reds >= blues ? "R" : "B";
}

function generateDerivedRoad(grid, offset) {
  const result = [];
  for (let col = offset; col < grid[0].length; col++) {
    const a = getColumnHeight(grid, col - offset);
    const b = getColumnHeight(grid, col - offset + 1);
    result.push(a === b ? "R" : "B");
  }
  return result;
}

function getColumnHeight(grid, col) {
  let height = 0;
  for (let i = 0; i < 6; i++) {
    if (grid[i][col]) height++;
  }
  return height;
}

function updatePredictionHistoryTable() {
  const tbody = document.getElementById("historyBody");
  tbody.innerHTML = "";

  predictionHistory.forEach(entry => {
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${entry.round}</td>
      <td>${entry.predict === "B" ? "莊" : "閒"}</td>
      <td>${entry.actual === "B" ? "莊" : "閒"}</td>
      <td style="color:${entry.correct ? 'green' : 'red'};">
        ${entry.correct ? "✔" : "✘"}
      </td>
      <td>${entry.strategy}</td>
    `;
    tbody.appendChild(tr);
  });
}

function clearHistory() {
  if (confirm("確定要清除預測歷史記錄嗎？")) {
    predictionHistory = [];
    localStorage.removeItem("predictionHistory");
    updatePredictionHistoryTable();
    alert("預測記錄已清除");
  }
}

// 初始化
renderAll();
</script>
</body>
</html>
