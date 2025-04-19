<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>地震情報アプリ</title>
    <link rel="icon" type="image/x-icon" href="/favicon.ico">
  <link rel="apple-touch-icon" sizes="180x180" href="/ewc.png">
  <link rel="icon" type="image/png" sizes="192x192" href="/ewc.png">
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            background-color: #f9f9f9;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
        }
        header {
            width: 100%;
            background-color: #ffffff;
            color: #333333;
            padding: 15px 0;
            text-align: center;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        main {
            width: 100%;
            max-width: 800px;
            margin: 20px;
            padding: 20px;
            background-color: #ffffff;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            border-radius: 12px;
        }
        #earthquakeInfo, #eewInfo {
            margin: 20px 0;
            padding: 15px;
            border: 1px solid #e0e0e0;
            border-radius: 12px;
            background-color: #fafafa;
        }
        textarea {
            width: 100%;
            height: 100px;
            margin-top: 10px;
            padding: 10px;
            border: 1px solid #e0e0e0;
            border-radius: 8px;
            box-sizing: border-box;
            font-family: inherit;
            font-size: 16px;
        }
        button {
            display: block;
            padding: 8px 16px;
            margin-top: 10px;
            background-color: #007aff;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
        }
        button:hover {
            background-color: #005bb5;
        }
    </style>
</head>
<body>
    <header>
        <h1>地震情報アプリ</h1>
        <p>通知関連のプログラムがおそらく治りました。（4/7時点）ぜひ下のボタンから通知を許可してみてください！</p>
        <button id="subscribeButton">通知を許可</button>
        <button id="testNotificationButton">テスト通知</button>
        <button class="scroll-button" onclick="scrollToEewSection()">緊急地震速報へ</button>
    </header>
    <main>
        <div id="earthquakeInfo">地震情報を取得中...</div>
        <div id="eewInfo">緊急地震速報を取得中...</div>
    </main>

    <script>
        const publicVapidKey = 'BC1T1-o4f9vLJ11ngQXOZTdKY8xd38vUyWeWyPosJ7JDJxnCPrAGtJZE_CUW4dqdh60eEUf5G-qzWjaojsSMer0';

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

        async function sendTestNotification() {
            if ('serviceWorker' in navigator) {
                const registration = await navigator.serviceWorker.getRegistration();
                if (registration) {
                    const options = {
                        body: 'これはテスト通知です。',
                        icon: '/icon.png'
                    };
                    registration.showNotification('テスト通知', options);
                } else {
                    alert('サービスワーカーが登録されていません。');
                }
            } else {
                alert('このブラウザはプッシュ通知をサポートしていません。');
            }
        }

        document.getElementById('subscribeButton').addEventListener('click', () => {
            subscribe().catch(err => console.error(err));
        });

        document.getElementById('testNotificationButton').addEventListener('click', () => {
            sendTestNotification().catch(err => console.error(err));
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

        async function fetchEarthquakeInfo() {
            try {
                const response = await fetch('https://api.p2pquake.net/v2/history?codes=551&limit=5');
                const data = await response.json();
                displayEarthquakeInfo(data);
            } catch (error) {
                console.error('Error fetching earthquake information:', error);
            }
        }

        async function fetchEewInfo() {
            try {
                const response = await fetch('https://api.wolfx.jp/jma_eew.json');
                const data = await response.json();
                displayEewInfo(data);
            } catch (error) {
                console.error('Error fetching EEW information:', error);
            }
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

                let formattedMessage;

                if (earthquake.issue && earthquake.issue.type === 'ScalePrompt') {
                    formattedMessage = `[震度速報]\n${date} ${timeStr}ごろ、地震による強い揺れを感じました。震度３以上が観測された地域をお知らせします。\n`;

                    Object.keys(pointsByScale).sort((a, b) => b - a).forEach(scale => {
                        formattedMessage += `\n《震度${scale}》`;
                        Object.keys(pointsByScale[scale]).forEach(pref => {
                            formattedMessage += `\n【${pref}】${pointsByScale[scale][pref].join('  ')}`;
                        });
                    });
                } else {
                    formattedMessage = `[地震情報]\n${date} ${timeStr}頃\n震源地：${hypocenter}\n最大震度：${maxScale}\n深さ：${depth}\n規模：M${magnitude}\n${tsunamiInfo}\n\n［各地の震度］`;

                    const scaleOrder = ['7', '6強', '6弱', '5強', '5弱','5弱以上と推定されるが震度情報を入手していない', '4', '3', '2', '1'];
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
                }

                const earthquakeInfo = document.createElement('div');
                earthquakeInfo.innerHTML = `
                    <p>${formattedMessage.replace(/\n/g, '<br>')}</p>
                    
                    <textarea id="editedMessage${index}" placeholder="通知メッセージを編集" style="width: 100%; height: 150px;">${formattedMessage}</textarea>
                    <button onclick="shareEditedMessage(${index})">編集したメッセージを共有</button>
                `;
                container.appendChild(earthquakeInfo);
            });
        }

        function displayEewInfo(data) {
            const container = document.getElementById('eewInfo');
            container.innerHTML = '';
            if (data.Title && data.CodeType) {
                let formattedMessage;
                if (data.isCancel) {
                    formattedMessage = "先程の緊急地震速報は取り消されました。";
                } else {
                    formattedMessage = formatEewMessage(data);
                    if (data.isAssumption) {
                        formattedMessage += "\n※この緊急地震速報は精度が低い可能性があります※";
                    }
                }

                const eewInfo = document.createElement('div');
                eewInfo.innerHTML = `
                    <p>${formattedMessage.replace(/\n/g, '<br>')}</p>
                    //<textarea id="eewComment" placeholder="自作緊急地震速報文"></textarea>
                    //<button onclick="shareEewComment()">自作情報文を共有</button>
                    <textarea id="editedEewMessage" placeholder="通知メッセージを編集" style="width: 100%; height: 150px;">${formattedMessage}</textarea>
                    <button onclick="shareEditedEewMessage()">編集したメッセージを共有</button>
                `;
                container.appendChild(eewInfo);
            }
        }

        function formatEewMessage(data) {
            return `【${data.Title} 推定最大震度${data.MaxIntensity}】\n(第${data.Serial}報)\n${data.OriginTime.split(' ')[1]}頃、${data.Hypocenter}を震源とする地震がありました。地震の規模はM${data.Magunitude}程度、震源の深さは約${data.Depth}km、最大震度${data.MaxIntensity}程度と推定されています。`;
        }

        function getScaleDescription(scale) {
            const scaleDescriptions = {
                10: '1',
                20: '2',
                30: '3',
                40: '4',
                45: '5弱',
                46: '5弱以上と推定されるが震度情報を入手していない',
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
                    title: '自作地震情報文',
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

        function shareEewComment() {
            const comment = document.getElementById('eewComment').value;
            if (navigator.share) {
                navigator.share({
                    title: '自作緊急地震速報文',
                    text: comment
                }).catch(console.error);
            } else {
                alert('このブラウザは共有機能をサポートしていません。');
            }
        }

        function shareEditedEewMessage() {
            const editedMessage = document.getElementById('editedEewMessage').value;
            if (navigator.share) {
                navigator.share({
                    title: '緊急地震速報',
                    text: editedMessage
                }).catch(console.error);
            } else {
                alert('このブラウザは共有機能をサポートしていません。');
            }
        }

        function scrollToEewSection() {
            const eewSection = document.getElementById('eewInfo');
            eewSection.scrollIntoView({ behavior: 'smooth' });
        }

        document.addEventListener('DOMContentLoaded', () => {
            fetchEarthquakeInfo();
            fetchEewInfo();
        });
    </script>
</body>
</html>
