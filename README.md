# quakeapps

<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>EWCアプリ</title>
  <link rel="icon" type="image/x-icon" href="/favicon.ico">
  <link rel="apple-touch-icon" sizes="180x180" href="/ewc.png">
<link rel="icon" type="image/png" sizes="192x192" href="/ewc.png">
</head>
<body>
  <h1>地震情報通知アプリ</h1>
  <button id="subscribeButton">通知を許可</button>
  <div id="earthquakeInfo"></div>
  <script>
    const publicVapidKey = 'BC1T1-o4f9vLJ11ngQXOZTdKY8xd38vUyWeWyPosJ7JDJxnCPrAGtJZE_CUW4dqdh60eEUf5G-qzWjaojsSMer0';

    // Register service worker and subscribe to push notifications
    async function subscribe() {
      if ('serviceWorker' in navigator) {
        const registration = await navigator.serviceWorker.register('/service-worker.js');
        const subscription = await registration.pushManager.subscribe({
          userVisibleOnly: true,
          applicationServerKey: urlBase64ToUint8Array(publicVapidKey)
        });

        await fetch('/subscribe', {
          method: 'POST',
          body: JSON.stringify(subscription),
          headers: {
            'Content-Type': 'application/json'
          }
        });

        alert('通知が許可されました！');
      } else {
        alert('このブラウザはプッシュ通知をサポートしていません。');
      }
    }

    document.getElementById('subscribeButton').addEventListener('click', () => {
      subscribe().catch(err => console.error(err));
    });

    function urlBase64ToUint8Array(base64String) {
      const padding = '='.repeat((4 - base64String.length % 4) % 4);
      const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
      const rawData = window.atob(base64);
      const outputArray = new Uint8Array(rawData.length);

      for (let i = 0; i < rawData.length; ++i) {
        outputArray[i] = rawData.charCodeAt(i);
      }
      return outputArray;
    }

    // Fetch earthquake information and display it
    async function fetchEarthquakeInfo() {
      const response = await fetch('https://api.p2pquake.net/v2/history?codes=551&limit=5');
      const data = await response.json();
      displayEarthquakeInfo(data);
    }

    function displayEarthquakeInfo(earthquakes) {
      const container = document.getElementById('earthquakeInfo');
      container.innerHTML = '';
      earthquakes.forEach((earthquake, index) => {
        const time = new Date(earthquake.earthquake.time);
        const date = time.toLocaleDateString('ja-JP', { month: '2-digit', day: '2-digit' });
        const timeStr = time.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' });

        const hypocenter = earthquake.earthquake.hypocenter.name;
        const maxScale = getScaleDescription(earthquake.earthquake.maxScale);
        let magnitude = earthquake.earthquake.hypocenter.magnitude;
        let depth = earthquake.earthquake.hypocenter.depth;

        depth = depth === -1 ? '不明' : depth === 0 ? 'ごく浅い' : `約${depth}km`;
        magnitude = magnitude === -1 ? '不明' : magnitude.toFixed(1);

        const pointsByScale = groupPointsByScale(earthquake.points);
        const tsunamiInfo = getTsunamiInfo(earthquake.earthquake.domesticTsunami);
        const freeFormComment = earthquake.comments?.freeFormComment || '';

        let formattedMessage = `[地震情報]\n${date} ${timeStr}頃\n震源地：${hypocenter}\n最大震度：${maxScale}\n深さ：${depth}\n規模：M${magnitude}\n${tsunamiInfo}\n\n［各地の震度］`;

        const scaleOrder = ['7', '6強', '6弱', '5強', '5弱', '4', '3', '2', '1'];
        const sortedScales = Object.keys(pointsByScale).sort((a, b) => {
          return scaleOrder.indexOf(a) - scaleOrder.indexOf(b);
        });

        sortedScales.forEach(scale => {
          formattedMessage += `\n《震度${scale}》`;
          Object.keys(pointsByScale[scale]).forEach(pref => {
            const uniqueCities = new Set();
            pointsByScale[scale][pref].forEach(addr => {
              const cityMatch = addr.match(/([^市区町村]+[市区町村])/);
              if (cityMatch) {
                uniqueCities.add(cityMatch[1]);
              }
            });
            formattedMessage += `\n【${pref}】${Array.from(uniqueCities).join('  ')}`;
          });
        });

        if (freeFormComment) {
          formattedMessage += `\n\n【情報】\n${freeFormComment}`;
        }

        const earthquakeInfo = document.createElement('div');
        earthquakeInfo.innerHTML = `
          <p>${formattedMessage.replace(/\n/g, '<br>')}</p>
          <textarea id="comment${index}" placeholder="自作地震情報文"></textarea>
          <button onclick="shareComment(${index})">自作情報分を共有</button>
          <textarea id="editedMessage${index}" placeholder="通知メッセージを編集" style="width: 100%; height: 150px;">${formattedMessage}</textarea>
          <button onclick="shareEditedMessage(${index})">編集したメッセージを共有</button>
        `;
        container.appendChild(earthquakeInfo);
      });
    }

    function getScaleDescription(scale) {
      const scaleDescriptions = {
        10: '1',
        20: '2',
        30: '3',
        40: '4',
        45: '5弱',
        50: '5強',
        55: '6弱',
        60: '6強',
        70: '7'
      };
      return scaleDescriptions[scale] || '不明';
    }

    function groupPointsByScale(points) {
      const pointsByScale = {};
      points.forEach(point => {
        const scale = getScaleDescription(point.scale);
        const addr = point.addr;
        const prefecture = point.pref;

        if (!scale) return;

        pointsByScale[scale] = pointsByScale[scale] || {};
        pointsByScale[scale][prefecture] = pointsByScale[scale][prefecture] || [];
        pointsByScale[scale][prefecture].push(addr);
      });
      return pointsByScale;
    }

    function getTsunamiInfo(domesticTsunami) {
      const tsunamiMessages = {
        "None": "この地震による津波の心配はありません。",
        "Unknown": "不明",
        "Checking": "津波の有無を調査中です。今後の情報に注意してください。",
        "NonEffective": "若干の海面変動があるかもしれませんが、被害の心配はありません。",
        "Watch": "現在、津波注意報を発表中です。",
        "Warning": "津波警報等（大津波警報・津波警報あるいは津波注意報）を発表中です。"
      };
      return tsunamiMessages[domesticTsunami] || "（津波情報なし）";
    }

    function shareComment(index) {
      const comment = document.getElementById(`comment${index}`).value;
      if (navigator.share) {
        navigator.share({
          title: 'コメント',
          text: comment
        }).catch(console.error);
      } else {
        alert('このブラウザは共有機能をサポートしていません。');
      }
    }

    function shareEditedMessage(index) {
      const editedMessage = document.getElementById(`editedMessage${index}`).value;
      if (navigator.share) {
        navigator.share({
          title: '地震情報',
          text: editedMessage
        }).catch(console.error);
      } else {
        alert('このブラウザは共有機能をサポートしていません。');
      }
    }

    // Fetch earthquake info on page load
    fetchEarthquakeInfo();
  </script>
</body>
</html>

# server

const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const bodyParser = require('body-parser');
const webPush = require('web-push');

// WebSocket サーバーのエンドポイント
const wsUrl = 'wss://api.p2pquake.net/v2/ws';

// WebSocket 再接続の間隔（ミリ秒）
const reconnectInterval = 5000;

// 環境変数 PORT からポートを取得（Render の要件）
const PORT = process.env.PORT || 3000;

// VAPID keys for web push notifications
const vapidKeys = {
  publicKey: 'BC1T1-o4f9vLJ11ngQXOZTdKY8xd38vUyWeWyPosJ7JDJxnCPrAGtJZE_CUW4dqdh60eEUf5G-qzWjaojsSMer0',
  privateKey: 'ya3LjwPTzOVTnLaz5S5qrtPne7I_C_tAEr60jzEQZAY'
};

webPush.setVapidDetails(
  'mailto:example@yourdomain.org',
  vapidKeys.publicKey,
  vapidKeys.privateKey
);

const app = express();
app.use(bodyParser.json());

let subscriptions = [];

// HTTP サーバーを作成して Render が要求するポートをリッスン
const server = http.createServer(app);

server.listen(PORT, () => {
  console.log(`HTTP server is running on port ${PORT}`);
});

let ws;
function connectWebSocket() {
  console.log('Connecting to WebSocket server...');
  ws = new WebSocket(wsUrl);

  ws.on('open', () => {
    console.log('WebSocket connection established');
  });

  ws.on('message', async (data) => {
    try {
      const message = JSON.parse(data);
      console.log("Received Data:", message);

      if (message.code === 551) {
        console.log('Processing earthquake data with code 551.');
        if (!message.earthquake) {
          console.error("Invalid earthquake data received.");
          return;
        }

        const earthquakeInfo = formatEarthquakeInfo(message.earthquake, message);
        sendWebPushNotification(earthquakeInfo);
      } else if (message.code === 552) {
        console.log('Processing tsunami warning data with code 552.');
        const tsunamiInfo = formatTsunamiWarningInfo(message);
        sendWebPushNotification(tsunamiInfo);
      } else {
        console.log(`Ignored message with code: ${message.code}`);
      }
    } catch (error) {
      console.error("Error processing message data:", error);
    }
  });

  ws.on('error', (error) => {
    console.error('WebSocket error:', error);
  });

  ws.on('close', () => {
    console.log('WebSocket connection closed. Reconnecting in 5 seconds...');
    setTimeout(connectWebSocket, reconnectInterval);
  });
}

function formatEarthquakeInfo(earthquake, message) {
  const time = new Date(earthquake.time);
  const date = time.toLocaleDateString('ja-JP', { month: '2-digit', day: '2-digit' });
  const timeStr = time.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' });

  const hypocenter = earthquake.hypocenter.name;
  const maxScale = getScaleDescription(earthquake.maxScale);
  let magnitude = earthquake.hypocenter.magnitude;
  let depth = earthquake.hypocenter.depth;

  depth = depth === -1 ? '不明' : depth === 0 ? 'ごく浅い' : `約${depth}km`;
  magnitude = magnitude === -1 ? '不明' : magnitude.toFixed(1);

  const pointsByScale = groupPointsByScale(message.points);
  const tsunamiInfo = getTsunamiInfo(earthquake.domesticTsunami);
  const freeFormComment = message.comments?.freeFormComment || '';

  // 震度速報のフォーマット
  if (message.issue && message.issue.type === 'ScalePrompt') {
    let formattedMessage = `[震度速報]\n${date} ${timeStr}ころ、地震による強い揺れを感じました。震度３以上が観測された地域をお知らせします。\n`;

    Object.keys(pointsByScale).sort((a, b) => b - a).forEach(scale => {
      formattedMessage += `\n《震度${scale}》`;
      Object.keys(pointsByScale[scale]).forEach(pref => {
        formattedMessage += `\n【${pref}】${pointsByScale[scale][pref].join('  ')}`;
      });
    });

    return formattedMessage;
  }

  // 通常の地震情報のフォーマット
  let formattedMessage = `[地震情報]\n${date} ${timeStr}頃\n震源地：${hypocenter}\n最大震度：${maxScale}\n深さ：${depth}\n規模：M${magnitude}\n${tsunamiInfo}\n\n［各地の震度］`;

  // 震度順序を定義
  const scaleOrder = ['7', '6強', '6弱', '5強', '5弱', '4', '3', '2', '1'];

  // 震度順にソート
  const sortedScales = Object.keys(pointsByScale).sort((a, b) => {
    return scaleOrder.indexOf(a) - scaleOrder.indexOf(b);
  });

  sortedScales.forEach(scale => {
    formattedMessage += `\n《震度${scale}》`;
    Object.keys(pointsByScale[scale]).forEach(pref => {
      const uniqueCities = new Set();
      pointsByScale[scale][pref].forEach(addr => {
        // 市区町村名を抽出してセットに追加
        const cityMatch = addr.match(/([^市区町村]+[市区町村])/);
        if (cityMatch) {
          uniqueCities.add(cityMatch[1]);
        }
      });
      // 市区町村のみを表示
      formattedMessage += `\n【${pref}】${Array.from(uniqueCities).join('  ')}`;
    });
  });

  if (freeFormComment) {
    formattedMessage += `\n\n【情報】\n${freeFormComment}`;
  }

  return formattedMessage;
}

function formatTsunamiWarningInfo(message) {
  if (message.cancelled) {
    return "津波警報等（大津波警報・津波警報あるいは津波注意報）は解除されました。";
  }

  // 津波警報の種類に応じてメッセージを作成
  const warnings = {
    MajorWarning: '【大津波警報🟪】\n大津波警報を発表しました！\n今すぐ高台やビルに避難！！\n【対象地域】',
    Warning: '【津波警報🟥】\n津波警報を発表しています！\n高台や近くのビルへ避難！\n【対象地域】',
    Watch: '【津波注意報🟨】\n津波注意報を発表しています。\n海や川から離れて下さい！\n【対象地域】',
    Unknown: '【津波情報❓️】\n津波の状況は不明です。\n今後の情報に注意してください。\n※プログラムエラーの可能性大。開発者をメンションしてください。\n【対象地域】'
  };

  let formattedMessage = warnings[message.areas[0].grade] || '【津波情報】\n津波の状況が不明です。\n【対象地域】';

  const areas = message.areas.map(area => {
    const name = area.name;
    const maxHeight = area.maxHeight.description;

    return `［${name}］${maxHeight}`;
  }).join('\n');

  formattedMessage += `\n${areas}\n津波は１mでも人や物を押し倒します！`;

  if (message.areas[0].grade === 'MajorWarning') {
    formattedMessage += `\n⚠️絶対に避難⚠️`;
  }

  return formattedMessage;
}

function groupPointsByScale(points) {
  const pointsByScale = {};
  points.forEach(point => {
    const scale = getScaleDescription(point.scale);
    const addr = point.addr;
    const prefecture = point.pref;

    if (!scale) return;

    pointsByScale[scale] = pointsByScale[scale] || {};
    pointsByScale[scale][prefecture] = pointsByScale[scale][prefecture] || [];
    pointsByScale[scale][prefecture].push(addr);
  });
  return pointsByScale;
}

function getScaleDescription(scale) {
  const scaleDescriptions = {
    10: '1',
    20: '2',
    30: '3',
    40: '4',
    45: '5弱',
    50: '5強',
    55: '6弱',
    60: '6強',
    70: '7'
  };
  return scaleDescriptions[scale] || '不明';
}

function getTsunamiInfo(domesticTsunami) {
  const tsunamiMessages = {
    "None": "この地震による津波の心配はありません。",
    "Unknown": "不明",
    "Checking": "津波の有無を調査中です。今後の情報に注意してください。",
    "NonEffective": "若干の海面変動があるかもしれませんが、被害の心配はありません。",
    "Watch": "現在、津波注意報を発表中です。",
    "Warning": "津波警報等（大津波警報・津波警報あるいは津波注意報）を発表中です。"
  };
  return tsunamiMessages[domesticTsunami] || "（津波情報なし）";
}

function sendWebPushNotification(message) {
  const payload = JSON.stringify({ title: '地震情報', body: message });

  subscriptions.forEach(subscription => {
    webPush.sendNotification(subscription, payload).catch(error => {
      console.error('Error sending notification, reason: ', error);
    });
  });
}

connectWebSocket();

// Web Push notification subscription endpoint
app.post('/subscribe', (req, res) => {
  const subscription = req.body;
  subscriptions.push(subscription);
  res.status(201).json({});
});

// Serve static files (for PWA)
app.use(express.static('public'));
