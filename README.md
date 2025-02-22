# earthquake-alert
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>全球地震警报系统</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        /* 样式部分保持不变 */
        body { margin: 0; padding: 0; font-family: -apple-system, sans-serif; background: #f5f5f5; overflow: hidden; }
        #map { height: 100vh; width: 100vw; }
        .alert-bar { position: fixed; bottom: 0; left: 0; right: 0; background: rgba(255, 59, 48, 0.95); color: white; padding: 12px; text-align: center; font-size: 16px; transform: translateY(100%); transition: transform 0.3s ease; z-index: 1000; }
        .alert-bar.visible { transform: translateY(0); }
        .quake-info { position: fixed; top: 10px; left: 10px; right: 10px; background: rgba(255, 255, 255, 0.95); padding: 15px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1); z-index: 1000; display: none; }
        .quake-info.visible { display: block; }
        .control-panel { position: fixed; bottom: 20px; right: 20px; z-index: 1000; }
        .control-panel button { background: white; border: none; border-radius: 50%; width: 50px; height: 50px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1); margin: 5px; font-size: 24px; display: flex; align-items: center; justify-content: center; cursor: pointer; }
    </style>
</head>
<body>
    <div id="map"></div>
    <div class="alert-bar" id="alertBar"><span id="alertText"></span></div>
    <div class="quake-info" id="quakeInfo">
        <h3 id="quakeTitle"></h3>
        <p>震级: <span class="magnitude" id="quakeMag"></span></p>
        <p id="quakeLocation"></p>
        <p id="quakeTime"></p>
        <div class="tsunami-warning" id="tsunamiWarning">海啸预警</div>
    </div>
    <div class="control-panel">
        <button onclick="toggleVoice()">🎤</button>
        <button onclick="refreshData()">🔄</button>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // 初始化地图
        const map = L.map('map', { zoomControl: false, tap: false }).setView([35, 105], 4);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '© OpenStreetMap' }).addTo(map);

        // 语言配置
        const languageSettings = {
            enabled: true,
            defaultLang: 'zh-CN',
            voices: {
                'ja-JP': { name: 'Kyoko', gender: 'female' }, // 日语女声
                'en-US': { name: 'Alex', gender: 'male' },   // 英语男声
                'ru-RU': { name: 'Milena', gender: 'female' } // 俄语女声
            }
        };

        // 语音合成器
        let speechSynth;
        let voices = [];

        // 初始化语音合成
        function initSpeech() {
            if ('speechSynthesis' in window) {
                speechSynth = window.speechSynthesis;
                speechSynth.onvoiceschanged = loadVoices;
            } else {
                console.warn('您的浏览器不支持语音合成');
            }
        }

        // 加载可用语音
        function loadVoices() {
            voices = speechSynth.getVoices();
            console.log('可用语音列表:', voices);
        }

        // 获取指定语言的语音
        function getVoice(lang) {
            const voiceConfig = languageSettings.voices[lang];
            if (voiceConfig) {
                return voices.find(voice => 
                    voice.lang === lang && 
                    voice.name.includes(voiceConfig.name)
                );
            }
            return voices.find(voice => voice.lang === lang);
        }

        // 多语言播报
        function speak(text, lang = languageSettings.defaultLang) {
            if (!languageSettings.enabled || !speechSynth) return;

            const utterance = new SpeechSynthesisUtterance(text);
            utterance.lang = lang;

            // 设置语音
            const voice = getVoice(lang);
            if (voice) {
                utterance.voice = voice;
                console.log(`使用语音: ${voice.name} (${voice.lang})`);
            }

            // 播放语音
            speechSynth.speak(utterance);
        }

        // 多语言警报生成
        function createAlertText(event, lang) {
            const { type, place, mag, time, areas } = event;
            const date = new Date(time).toLocaleString(lang);

            switch (type) {
                case 'earthquake':
                    return {
                        'zh-CN': `地震警报！${date}，在${place}发生规模${mag}级地震`,
                        'en-US': `Earthquake alert! Magnitude ${mag} earthquake occurred at ${place} on ${date}`,
                        'ja-JP': `地震警報！${date}、${place}でマグニチュード${mag}の地震が発生しました`,
                        'ru-RU': `Предупреждение о землетрясении! Землетрясение магнитудой ${mag} произошло в ${place} в ${date}`
                    }[lang];

                case 'tsunami':
                    return {
                        'zh-CN': `海啸警报！以下地区请注意：${areas.join('、')}`,
                        'en-US': `Tsunami warning! Please note the following areas: ${areas.join(', ')}`,
                        'ja-JP': `津波警報！以下の地域に注意してください：${areas.join('、')}`,
                        'ru-RU': `Предупреждение о цунами! Обратите внимание на следующие регионы: ${areas.join(', ')}`
                    }[lang];

                default:
                    return '';
            }
        }

        // 显示警报
        function showAlert(event) {
            const { type, place, mag, time, areas } = event;

            // 显示警报面板
            const alertText = createAlertText(event, languageSettings.defaultLang);
            document.getElementById('alertText').textContent = alertText;
            document.getElementById('alertBar').classList.add('visible');

            // 多语言播报
            if (languageSettings.enabled) {
                const languages = ['zh-CN', 'en-US', 'ja-JP', 'ru-RU'];
                languages.forEach(lang => {
                    const text = createAlertText(event, lang);
                    speak(text, lang);
                });
            }

            // 自动隐藏
            setTimeout(() => {
                document.getElementById('alertBar').classList.remove('visible');
            }, 10000);
        }

        // 获取地震数据
        async function fetchEarthquakes() {
            try {
                const response = await fetch('https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_day.geojson');
                const data = await response.json();
                return data.features;
            } catch (error) {
                console.error('数据获取失败:', error);
                return [];
            }
        }

        // 更新地图标记
        function updateMap(earthquakes) {
            earthquakes.forEach(quake => {
                const [lng, lat] = quake.geometry.coordinates;
                const mag = quake.properties.mag;
                
                const marker = L.circleMarker([lat, lng], {
                    radius: mag * 2,
                    color: getColor(mag),
                    fillOpacity: 0.7
                }).addTo(map);

                marker.on('click', () => {
                    showQuakeInfo(quake);
                });

                if (mag >= 6.0) {
                    showAlert({
                        type: 'earthquake',
                        place: quake.properties.place,
                        mag: quake.properties.mag,
                        time: quake.properties.time,
                        areas: []
                    });
                }
            });
        }

        // 显示地震详情
        function showQuakeInfo(quake) {
            const infoPanel = document.getElementById('quakeInfo');
            const tsunamiWarning = document.getElementById('tsunamiWarning');
            
            document.getElementById('quakeTitle').textContent = quake.properties.place;
            document.getElementById('quakeMag').textContent = quake.properties.mag;
            document.getElementById('quakeLocation').textContent = `位置: ${lat},${lng}`;
            document.getElementById('quakeTime').textContent = `时间: ${new Date(quake.properties.time).toLocaleString()}`;
            
            tsunamiWarning.style.display = quake.properties.tsunami ? 'block' : 'none';
            infoPanel.classList.add('visible');
            
            setTimeout(() => {
                infoPanel.classList.remove('visible');
            }, 5000);
        }

        // 获取标记颜色
        function getColor(mag) {
            return mag > 7 ? '#d62728' :
                   mag > 5 ? '#ff7f0e' :
                   mag > 4 ? '#2ca02c' : '#1f77b4';
        }

        // 刷新数据
        async function refreshData() {
            const quakes = await fetchEarthquakes();
            updateMap(quakes);
        }

        // 初始化
        window.onload = function() {
            initSpeech();
            refreshData();
            setInterval(refreshData, 300000); // 每5分钟更新
        };
    </script>
</body>
</html>
