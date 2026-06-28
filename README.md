<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
    <meta http-equiv="Pragma" content="no-cache">
    <meta http-equiv="Expires" content="0">
    
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob: https://cdn-icons-png.flaticon.com; media-src 'self' blob:; connect-src 'self' https://cdn.jsdelivr.net;">
    
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="theme-color" content="#f0f0f0">
    <link rel="apple-touch-icon" href="https://cdn-icons-png.flaticon.com/512/3069/3069186.png">

    <title>射箭動作線性觀察與控制系統</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose/pose.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    
    <style>
        html, body { height: 100%; margin: 0; padding: 0; }
        body { font-family: Arial, sans-serif; text-align: center; background-color: #f0f0f0; padding-top: env(safe-area-inset-top); display: flex; flex-direction: column; -webkit-tap-highlight-color: transparent; }
        
        .menu-overlay { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.6); z-index: 9998; backdrop-filter: blur(2px); }
        .side-menu { position: fixed; top: 0; left: -280px; width: 260px; height: 100%; background: white; z-index: 9999; box-shadow: 2px 0 15px rgba(0,0,0,0.4); transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); display: flex; flex-direction: column; padding-top: calc(60px + env(safe-area-inset-top)); }
        .side-menu.open { left: 0; }
        .close-menu-btn { position: absolute; top: calc(15px + env(safe-area-inset-top)); right: 20px; font-size: 32px; cursor: pointer; color: #333; font-weight: bold; line-height: 1; }
        .side-menu button { background: none; border: none; padding: 18px 20px; text-align: left; font-size: 18px; font-weight: bold; color: #333; cursor: pointer; border-bottom: 1px solid #eee; transition: background 0.2s; }
        .side-menu button:active { background: #e0e0e0; }
        .menu-title { padding: 0 20px 10px 20px; font-size: 14px; color: #888; border-bottom: 2px solid #3498DB; margin-bottom: 10px; text-align: left; font-weight: bold; }

        .toast-msg { display: none; position: fixed; top: calc(20px + env(safe-area-inset-top)); left: 50%; transform: translateX(-50%); background: rgba(46, 204, 113, 0.95); color: white; padding: 10px 20px; border-radius: 20px; font-weight: bold; z-index: 10000; box-shadow: 0 4px 6px rgba(0,0,0,0.2); font-size: 14px; }

        .controls { background: white; padding: 10px; border-radius: 8px; margin: 10px auto; box-shadow: 0 2px 5px rgba(0,0,0,0.2); width: 95%; max-width: 450px; flex-shrink: 0; display: flex; flex-direction: column; gap: 8px; box-sizing: border-box; }
        .controls-row { display: flex; align-items: stretch; justify-content: space-between; width: 100%; gap: 8px; }

        .hamburger-btn { background: #f9f9f9; padding: 8px 10px; border-radius: 8px; cursor: pointer; display: flex; flex-direction: column; justify-content: space-around; width: 45px; box-sizing: border-box; border: 1px solid #ccc; flex-shrink: 0; margin: 0; }
        .hamburger-btn div { width: 100%; height: 3px; background-color: #333; border-radius: 2px; }
        
        .controls button { padding: 8px 5px; margin: 0; cursor: pointer; font-size: 14px; border-radius: 5px; border: 1px solid #ccc; font-weight: bold; flex-grow: 1; }
        #recordBtn { background-color: #2ECC71; color: white; border: none; }
        #recordBtn.recording { background-color: #E74C3C; animation: pulse 1.5s infinite; }
        
        @keyframes pulse { 0% { transform: scale(1); } 50% { transform: scale(1.05); } 100% { transform: scale(1); } }

        .mode-container { display: flex; align-items: center; justify-content: center; gap: 5px; background: #f9f9f9; border: 1px solid #ccc; border-radius: 8px; padding: 0 8px; flex-grow: 2; }
        .switch { position: relative; display: inline-block; width: 42px; height: 24px; flex-shrink: 0; }
        .switch input { opacity: 0; width: 0; height: 0; }
        .slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #ccc; transition: .4s; border-radius: 34px; }
        .slider:before { position: absolute; content: ""; height: 18px; width: 18px; left: 3px; bottom: 3px; background-color: white; transition: .4s; border-radius: 50%; box-shadow: 0 2px 4px rgba(0,0,0,0.2); }
        input:checked + .slider { background-color: #2ECC71; }
        input:checked + .slider:before { transform: translateX(18px); }
        #timerInput { width: 36px; text-align: center; font-size: 15px; padding: 4px; border-radius: 5px; border: 1px solid #ccc; }

        .canvas-container { position: relative; flex-grow: 1; width: 100vw; background-color: #000; display: flex; justify-content: center; align-items: center; overflow: hidden; }
        video { display: none; } 
        /* 確保即時相機畫面不會被放大變形 */
        canvas { width: 100%; height: 100%; object-fit: contain; display: block; }
        
        /* Modal 通用樣式 */
        .modal { display: none; position: fixed; z-index: 10001; left: 0; top: 0; width: 100%; height: 100%; background-color: rgba(0,0,0,0.95); flex-direction: column; justify-content: flex-start; align-items: center; overflow-y: auto; padding-top: calc(20px + env(safe-area-inset-top)); padding-bottom: 20px;}
        .modal-content-wrapper { display: flex; justify-content: center; align-items: center; width: 100%; height: 100%; padding: 10px; box-sizing: border-box; }
        .close-modal { position: absolute; top: calc(20px + env(safe-area-inset-top)); right: 30px; color: white; font-size: 40px; font-weight: bold; cursor: pointer; z-index: 10006; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
        
        /* 修改：讓彈出視窗中的相片與影片自動根據手機縮放，不放大不變形 */
        .modal img, .modal video { max-width: 100%; max-height: 80vh; object-fit: contain; border-radius: 8px; background: transparent; }

        /* 面板與列表 */
        .score-container, .gallery-container, .sight-container, .match-container { background: white; width: 95%; max-width: 500px; border-radius: 8px; margin: 0 auto; padding: 15px; box-sizing: border-box; text-align: center; }
        .score-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px; }
        .score-nav { display: flex; justify-content: center; flex-wrap: wrap; gap: 5px; }
        .score-nav button { padding: 8px 12px; cursor: pointer; background: #eee; border: 1px solid #ccc; border-radius: 5px; font-weight: bold; flex-grow: 1;}
        .score-nav button.active { background: #3498DB; color: white; border-color: #2980B9; }
        
        .setup-group { margin-bottom: 15px; text-align: left; font-size: 16px; display: flex; align-items: center; }
        .setup-group label { min-width: 130px; font-weight: bold; color: #333; }
        .setup-group input, .setup-group select { font-size: 16px; text-align: center; padding: 5px; border-radius: 5px; border: 1px solid #ccc; flex-grow: 1; }
        
        .start-score-btn { background: #2ECC71; color: white; width: 100%; padding: 12px; border: none; border-radius: 8px; font-size: 18px; font-weight: bold; cursor: pointer; margin-top: 10px;}
        
        .score-grid { overflow-x: auto; margin-bottom: 15px; }
        .score-table { width: 100%; border-collapse: collapse; text-align: center; font-size: 16px; }
        .score-table th, .score-table td { border: 1px solid #ddd; padding: 6px 2px; min-width: 25px; }
        .score-table th { background: #f9f9f9; }
        .score-cell { cursor: pointer; transition: background 0.2s; }
        .score-cell:active { background-color: #e0e0e0; }
        .active-cell { background-color: #FFF3CD !important; border: 2px solid #E67E22 !important; box-shadow: inset 0 0 5px rgba(230,126,34,0.5); font-weight: bold; }
        .total-text { font-weight: bold; color: #E74C3C; font-size: 18px; margin-bottom: 10px; }
        
        .keypad { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; margin-bottom: 15px; }
        .keypad button { padding: 15px; font-size: 20px; font-weight: bold; border-radius: 8px; border: none; cursor: pointer; color: white; text-shadow: 1px 1px 2px rgba(0,0,0,0.5); }
        .keypad button:active { opacity: 0.8; transform: scale(0.95); }
        .btn-m { background-color: #95A5A6; } .btn-1, .btn-2 { background-color: #FFF; color: #333; text-shadow: none; border: 1px solid #ccc; }
        .btn-3, .btn-4 { background-color: #2C3E50; } .btn-5, .btn-6 { background-color: #3498DB; }
        .btn-7, .btn-8 { background-color: #E74C3C; } .btn-9, .btn-10, .btn-x { background-color: #F1C40F; color: #333; text-shadow: none; border: 1px solid #D4AC0D; }
        .btn-undo { background-color: #7F8C8D; grid-column: span 3; color: white; text-shadow: none; }
        
        .action-btns { display: flex; gap: 10px; justify-content: center; }
        .save-score-btn { background: #F39C12; color: white; padding: 10px; border: none; border-radius: 5px; font-weight: bold; cursor: pointer; flex-grow: 1; }
        .reset-score-btn { background: #E74C3C; color: white; padding: 10px; border: none; border-radius: 5px; font-weight: bold; cursor: pointer; flex-grow: 1; }
        
        /* 歷史紀錄與瞄準器列表 */
        .history-list, .sight-list { text-align: left; max-height: 50vh; overflow-y: auto; padding-right: 5px;}
        .history-item, .sight-item { border: 1px solid #eee; padding: 10px; margin-bottom: 10px; border-radius: 5px; background: #fafafa; }
        .history-item input { width: 100%; border: none; font-weight: bold; font-size: 16px; border-bottom: 1px dashed #ccc; padding: 2px 0; margin-bottom: 5px; background: transparent; color: #2C3E50; box-sizing: border-box; }
        .history-summary { font-size: 14px; color: #666; display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; flex-wrap: wrap; gap: 5px; }
        .history-actions { display: flex; gap: 5px; justify-content: flex-end; width: 100%; }
        
        .del-history-btn, .export-history-btn, .stats-history-btn { color: white; border: none; border-radius: 3px; padding: 6px 10px; cursor: pointer; font-size: 12px; font-weight: bold; }
        .del-history-btn { background: #E74C3C; } .export-history-btn { background: #3498DB; } .stats-history-btn { background: #9B59B6; }
        .no-data { color: #999; margin-top: 20px; text-align: center; }
        
        .stats-details-badges { display: flex; flex-wrap: wrap; gap: 5px; justify-content: center; margin-bottom: 15px; }
        .stats-badge { padding: 4px 8px; border-radius: 4px; font-size: 13px; font-weight: bold; background: #eee; color: #333; border: 1px solid #ddd; }
        
        /* 骨架開關設定專用 */
        .pose-toggles { display: flex; flex-direction: column; gap: 15px; margin-top: 15px; text-align: left; }
        .pose-toggle-item { display: flex; justify-content: space-between; align-items: center; font-size: 16px; font-weight: bold; padding: 12px; background: #f9f9f9; border-radius: 8px; border: 1px solid #eee; }
        
        /* 媒體庫 */
        #gallery { display: grid; grid-template-columns: repeat(auto-fill, minmax(110px, 1fr)); gap: 10px; justify-items: center;}
        .gallery-item { text-align: center; width: 100%; position: relative; overflow: hidden; border-radius: 5px; border: 1px solid #ddd; background: black; }
        /* 縮圖維持填滿，讓版面整齊 */
        .gallery-item img, .gallery-item video { width: 100%; height: 110px; display: block; cursor: pointer; object-fit: cover; }
        .select-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.3); display: none; pointer-events: none; border: 3px solid transparent; box-sizing: border-box; border-radius: 5px; }
        .select-overlay::after { content: ''; position: absolute; top: 5px; right: 5px; width: 22px; height: 22px; border-radius: 50%; border: 2px solid #fff; background: rgba(0,0,0,0.5); box-sizing: border-box; }
        .gallery-item.selected .select-overlay { display: block; border-color: #2ECC71; background: rgba(46,204,113,0.2); }
        .gallery-item.selected .select-overlay::after { background: #2ECC71; content: '✔'; color: white; line-height: 20px; font-size: 14px; text-align: center; border-color: #2ECC71; }
        .select-mode-on .gallery-item { cursor: pointer; }
        .select-mode-on .gallery-item:not(.selected) .select-overlay { display: block; }
        .indiv-save-btn { width: 100%; padding: 8px 0; font-size: 13px; cursor: pointer; background-color: #3498DB; color: white; border: none; font-weight: bold; display: block; }
        .select-mode-on .indiv-save-btn { display: none; }
        #bulkActions { display: none; justify-content: space-between; margin-bottom: 15px; background: #f1f2f6; padding: 10px; border-radius: 8px; width: 100%; box-sizing: border-box; align-items: center; flex-wrap: wrap; gap: 8px; }
        #bulkActions button { padding: 8px 12px; border-radius: 5px; font-weight: bold; cursor: pointer; border: none; font-size: 14px; }
    </style>
</head>
<body>

    <div id="toastMsg" class="toast-msg">已存入紀錄庫</div>

    <div class="menu-overlay" id="menuOverlay"></div>
    <div class="side-menu" id="sideMenu">
        <span class="close-menu-btn" id="closeMenuBtn">&times;</span>
        <div class="menu-title">系統功能選單</div>
        <button id="menuScoreBtn">🎯 計分系統</button>
        <button id="menuMatchBtn">⚔️ 雙人對抗賽</button>
        <button id="menuSightBtn">📏 瞄準器紀錄</button>
        <button id="menuPoseBtn">🧍 骨架顯示設定</button>
        <button id="menuGalleryBtn">📸 媒體紀錄庫</button>
    </div>

    <div class="controls">
        <div class="controls-row">
            <div class="hamburger-btn" id="openMenuBtn"><div></div><div></div><div></div></div>
            <button id="actionShootBtn" style="background-color: #3498DB; color: white; border: none;">📸 拍照</button>
            <div class="mode-container">
                <label class="switch"><input type="checkbox" id="photoModeToggle"><span class="slider round"></span></label>
                <span id="modeLabel" style="font-size: 14px; font-weight: bold; color: #333;">立即</span>
                <input type="number" id="timerInput" value="5" style="display: none;">
                <span id="secLabel" style="display: none; font-size: 14px; font-weight: bold; color: #333;">秒</span>
            </div>
        </div>
        <div class="controls-row">
            <button id="recordBtn">🎥 開始錄影</button>
            <button id="switchCamBtn">🔄 翻轉鏡頭</button>
        </div>
    </div>
    
    <div class="canvas-container">
        <video id="videoElement" autoplay playsinline></video>
        <canvas id="outputCanvas"></canvas>
    </div>

    <div id="poseModalUI" class="modal">
        <div class="score-container">
            <div class="score-header">
                <h3 style="margin:0; color: #2C3E50;">🧍 骨架顯示設定</h3>
                <span id="closePoseModal" style="font-size: 28px; cursor: pointer; font-weight: bold;">&times;</span>
            </div>
            <div class="pose-toggles">
                <div class="pose-toggle-item">
                    <span>重心 (中心線)</span>
                    <label class="switch"><input type="checkbox" id="toggleCG" checked><span class="slider round"></span></label>
                </div>
                <div class="pose-toggle-item">
                    <span>鼻尖 (頭部對齊)</span>
                    <label class="switch"><input type="checkbox" id="toggleNose" checked><span class="slider round"></span></label>
                </div>
                <div class="pose-toggle-item">
                    <span>手部 (手臂連線)</span>
                    <label class="switch"><input type="checkbox" id="toggleArms" checked><span class="slider round"></span></label>
                </div>
                <div class="pose-toggle-item">
                    <span>身體 (軀幹連線)</span>
                    <label class="switch"><input type="checkbox" id="toggleBody" checked><span class="slider round"></span></label>
                </div>
            </div>
        </div>
    </div>

    <div id="galleryModalUI" class="modal">
        <div class="gallery-container" id="galleryContainer">
            <div class="gallery-header">
                <p class="gallery-title">📸 媒體紀錄庫<br><span>(永久儲存直到手動刪除)</span></p>
                <div style="display: flex; align-items: center; gap: 10px;">
                    <button id="toggleSelectModeBtn" style="background: #34495E; color: white; border: none; padding: 6px 12px; border-radius: 5px; font-weight: bold; cursor: pointer;">☑️ 多選模式</button>
                    <span id="closeGalleryModal" style="font-size: 32px; cursor: pointer; font-weight: bold; line-height: 1;">&times;</span>
                </div>
            </div>
            <div id="bulkActions">
                <div style="display: flex; gap: 8px;">
                    <button id="saveSelectedBtn" style="background:#2ECC71; color:white;">💾 儲存</button>
                    <button id="deleteSelectedBtn" style="background:#E74C3C; color:white;">🗑️ 刪除</button>
                </div>
                <button id="deleteAllBtn" style="background:#c0392b; color:white;">❌ 一鍵全刪</button>
            </div>
            <div id="gallery"></div>
        </div>
    </div>

    <div id="mediaModal" class="modal" style="z-index: 10005;">
        <span class="close-modal" id="closeModal">&times;</span>
        <div class="modal-content-wrapper" id="modalContent"></div>
    </div>

    <div id="sightModalUI" class="modal">
        <div class="sight-container">
            <div class="score-header">
                <h3 style="margin:0; color: #2C3E50;">📏 瞄準器紀錄</h3>
                <span id="closeSightModal" style="font-size: 28px; cursor: pointer; font-weight: bold;">&times;</span>
            </div>
            <div class="setup-group">
                <label style="min-width: 60px;">距離:</label>
                <input type="number" id="sightDist" placeholder="m" style="width:60px; flex-grow:0;">
                <label style="min-width: 45px; margin-left:10px;">刻度:</label>
                <input type="text" id="sightMark" placeholder="例如: 7.5" style="width:75px; flex-grow:0;">
            </div>
            <div class="setup-group">
                <label style="min-width: 60px;">備註:</label>
                <input type="text" id="sightNote" placeholder="例如: 左風、下雨">
            </div>
            <button class="start-score-btn" onclick="saveSightMark()" style="margin-bottom:15px;">➕ 新增紀錄</button>
            <div class="sight-list" id="sightList"></div>
        </div>
    </div>

    <div id="matchModalUI" class="modal">
        <div class="match-container">
            <div class="score-header">
                <h3 style="margin:0; color: #2C3E50;">⚔️ 雙人對抗賽</h3>
                <span id="closeMatchModal" style="font-size: 28px; cursor: pointer; font-weight: bold;">&times;</span>
            </div>
            
            <div id="matchSetupSection">
                <div class="setup-group">
                    <label>賽制規則：</label>
                    <select id="matchType" style="flex-grow: 1;">
                        <option value="recurve">反曲弓 (局分制 先得6點勝)</option>
                        <option value="compound">複合弓 (總分制 滿分150分)</option>
                    </select>
                </div>
                <div class="setup-group">
                    <label>選手 A：</label>
                    <input type="text" id="matchP1Name" value="選手 A">
                </div>
                <div class="setup-group">
                    <label>選手 B：</label>
                    <input type="text" id="matchP2Name" value="選手 B">
                </div>
                <p style="font-size:12px; color:#888;">規則：依序進行5趟，每趟每人各3支箭。</p>
                <button class="start-score-btn" onclick="startMatch()">開始對抗</button>
            </div>

            <div id="matchScoringSection" style="display: none;">
                <div class="total-text" style="font-size: 16px;">
                    <span id="p1DisplayScore" style="color:#2980B9;">A: 0</span> 
                    <span style="color:#333; margin:0 10px;">VS</span> 
                    <span id="p2DisplayScore" style="color:#E74C3C;">B: 0</span>
                </div>
                <p id="matchStatusText" style="font-size: 14px; font-weight:bold; color: #2ECC71; margin-top:-5px;"></p>
                
                <div class="score-grid">
                    <table class="score-table" id="matchTable"></table>
                </div>
                
                <div class="keypad">
                    <button class="btn-x" onclick="inputMatchScore('X')">X</button>
                    <button class="btn-10" onclick="inputMatchScore('10')">10</button>
                    <button class="btn-9" onclick="inputMatchScore('9')">9</button>
                    <button class="btn-8" onclick="inputMatchScore('8')">8</button>
                    <button class="btn-7" onclick="inputMatchScore('7')">7</button>
                    <button class="btn-6" onclick="inputMatchScore('6')">6</button>
                    <button class="btn-5" onclick="inputMatchScore('5')">5</button>
                    <button class="btn-4" onclick="inputMatchScore('4')">4</button>
                    <button class="btn-3" onclick="inputMatchScore('3')">3</button>
                    <button class="btn-2" onclick="inputMatchScore('2')">2</button>
                    <button class="btn-1" onclick="inputMatchScore('1')">1</button>
                    <button class="btn-m" onclick="inputMatchScore('M')">M</button>
                    <button class="btn-undo" onclick="undoMatchScore()">⬅️ 退回上一格</button>
                </div>
                
                <div class="action-btns">
                    <button class="reset-score-btn" onclick="if(confirm('確定要放棄這場比賽重新設定嗎？')){document.getElementById('matchScoringSection').style.display='none'; document.getElementById('matchSetupSection').style.display='block';}">重新設定</button>
                    <button class="save-score-btn" onclick="saveMatchRecord()">結算並存檔</button>
                </div>
            </div>
        </div>
    </div>

    <div id="scoreModalUI" class="modal">
        <div class="score-container">
            <div class="score-header">
                <h3 style="margin:0;">🎯 射箭計分系統</h3>
                <span id="closeScoreModal" style="font-size: 28px; cursor: pointer; font-weight: bold;">&times;</span>
            </div>
            <div class="score-nav">
                <button id="tabSetup" class="active">設定</button>
                <button id="tabScoring">計分板</button>
                <button id="tabHistory">歷史紀錄</button>
            </div>
            <hr style="border: 0; border-top: 1px solid #eee; margin: 15px 0;">

            <div id="sectionSetup">
                <div class="setup-group">
                    <label>預計趟數 (Ends)：</label>
                    <input type="number" id="inputEnds" value="6" min="1" style="flex-grow:0; width:80px;">
                </div>
                <div class="setup-group">
                    <label>單趟箭數 (Arrows)：</label>
                    <input type="number" id="inputArrows" value="6" min="1" style="flex-grow:0; width:80px;">
                </div>
                <div class="setup-group">
                    <label>射箭距離 (Dist)：</label>
                    <div style="display: flex; align-items: center; flex-grow: 1;">
                        <select id="inputDistance" style="font-size: 16px; padding: 5px; border-radius: 5px; border: 1px solid #ccc; flex-grow: 1;">
                            <option value="70">70 公尺</option>
                            <option value="50">50 公尺</option>
                            <option value="30">30 公尺</option>
                            <option value="15">15 公尺</option>
                            <option value="10">10 公尺</option>
                            <option value="custom">自訂...</option>
                        </select>
                        <input type="number" id="inputCustomDistance" placeholder="m" style="display: none; width: 60px; margin-left: 5px;">
                    </div>
                </div>
                <button class="start-score-btn" id="startScoreBtn">開始計分</button>
            </div>

            <div id="sectionScoring" style="display: none;">
                <div class="total-text" style="line-height: 1.5;">總分: <span id="grandTotal">0</span> <br><span style="font-size: 14px; color: #555; font-weight: normal;">(距離: <span id="displayScoreDist">70</span>m / 平均: <span id="displayScoreAvg">0.00</span>)</span></div>
                <p style="font-size: 12px; color: #888; margin-top: 0px; margin-bottom: 10px;">💡 提示：點擊下方表格的格子可以直接跳過去修改成績。</p>
                <div class="score-grid">
                    <table class="score-table" id="scoreTable"></table>
                </div>
                <div class="keypad">
                    <button class="btn-x" onclick="inputScore('X')">X</button>
                    <button class="btn-10" onclick="inputScore('10')">10</button>
                    <button class="btn-9" onclick="inputScore('9')">9</button>
                    <button class="btn-8" onclick="inputScore('8')">8</button>
                    <button class="btn-7" onclick="inputScore('7')">7</button>
                    <button class="btn-6" onclick="inputScore('6')">6</button>
                    <button class="btn-5" onclick="inputScore('5')">5</button>
                    <button class="btn-4" onclick="inputScore('4')">4</button>
                    <button class="btn-3" onclick="inputScore('3')">3</button>
                    <button class="btn-2" onclick="inputScore('2')">2</button>
                    <button class="btn-1" onclick="inputScore('1')">1</button>
                    <button class="btn-m" onclick="inputScore('M')">M</button>
                    <button class="btn-undo" onclick="undoScore()">⬅️ 退回</button>
                </div>
                <div class="action-btns">
                    <button class="reset-score-btn" id="resetScoreBtn">重新開始</button>
                    <button class="save-score-btn" id="saveScoreRecBtn">儲存成績</button>
                </div>
            </div>

            <div id="sectionHistory" style="display: none;">
                <div class="history-list" id="historyList"></div>
            </div>
        </div>
    </div>

    <div id="statsModalUI" class="modal" style="z-index: 10005;">
        <div class="score-container" style="max-height: 90vh; overflow-y: auto; width: 90%; max-width: 450px;">
            <div class="score-header" style="border-bottom: 2px solid #3498DB;">
                <h3 id="statsTitle" style="margin:0; color: #2C3E50;">🎯 數據統計</h3>
                <span id="closeStatsModal" style="font-size: 28px; cursor: pointer; font-weight: bold;">&times;</span>
            </div>
            <h4 id="statsAvg" style="color:#E74C3C; margin: 10px 0;"></h4>
            <div id="statsDetails" class="stats-details-badges"></div>
            <div style="position: relative; width: 100%; height: 250px;">
                <canvas id="statsChart"></canvas>
            </div>
        </div>
    </div>

    <div id="exportCardWrapper" style="position: absolute; top: -9999px; left: -9999px; z-index: -1;">
        <div id="exportCard" style="background: white; padding: 20px; border-radius: 10px; min-width: 400px; color: black; font-family: Arial; text-align: center;">
            <h2 id="exName" style="margin-top:0; border-bottom: 2px solid #ccc; padding-bottom: 10px;"></h2>
            <h3 id="exTotal" style="color:#E74C3C; font-size: 24px; margin-bottom: 5px;"></h3>
            <p id="exAvg" style="font-size: 16px; color: #555; margin-top: 0; font-weight: bold;"></p>
            <table class="score-table" id="exTable" style="width:100%; text-align:center; border-collapse:collapse; margin-top: 15px;"></table>
            <p style="margin-top: 15px; font-size: 12px; color: #888;">射箭動作線性觀察與控制系統 - 自動生成</p>
        </div>
    </div>

    <script>
        // 初始化且只宣告一次圖表實例
        let statsChartInstance = null;

        // ========== 通用與選單邏輯 ==========
        const openMenuBtn = document.getElementById('openMenuBtn');
        const sideMenu = document.getElementById('sideMenu');
        const menuOverlay = document.getElementById('menuOverlay');

        function toggleMenu(show) {
            if (show) { sideMenu.classList.add('open'); menuOverlay.style.display = 'block'; } 
            else { sideMenu.classList.remove('open'); menuOverlay.style.display = 'none'; }
        }

        openMenuBtn.addEventListener('click', () => toggleMenu(true));
        document.getElementById('closeMenuBtn').addEventListener('click', () => toggleMenu(false));
        menuOverlay.addEventListener('click', () => toggleMenu(false));

        document.getElementById('menuScoreBtn').addEventListener('click', () => { toggleMenu(false); document.getElementById('scoreModalUI').style.display = 'flex'; });
        document.getElementById('menuGalleryBtn').addEventListener('click', () => { toggleMenu(false); document.getElementById('galleryModalUI').style.display = 'flex'; });
        document.getElementById('menuSightBtn').addEventListener('click', () => { toggleMenu(false); document.getElementById('sightModalUI').style.display = 'flex'; renderSightMarks(); });
        document.getElementById('menuMatchBtn').addEventListener('click', () => { toggleMenu(false); document.getElementById('matchModalUI').style.display = 'flex'; });
        document.getElementById('menuPoseBtn').addEventListener('click', () => { toggleMenu(false); document.getElementById('poseModalUI').style.display = 'flex'; });

        document.getElementById('closeGalleryModal').addEventListener('click', () => { document.getElementById('galleryModalUI').style.display = 'none'; });
        document.getElementById('closeScoreModal').addEventListener('click', () => { document.getElementById('scoreModalUI').style.display = 'none'; });
        document.getElementById('closeSightModal').addEventListener('click', () => { document.getElementById('sightModalUI').style.display = 'none'; });
        document.getElementById('closeMatchModal').addEventListener('click', () => { document.getElementById('matchModalUI').style.display = 'none'; });
        document.getElementById('closeStatsModal').addEventListener('click', () => { document.getElementById('statsModalUI').style.display = 'none'; });
        document.getElementById('closePoseModal').addEventListener('click', () => { document.getElementById('poseModalUI').style.display = 'none'; });

        function showToast(msg) {
            const toast = document.getElementById('toastMsg');
            toast.innerText = msg; toast.style.display = 'block';
            setTimeout(() => { toast.style.display = 'none'; }, 2500);
        }

        // ========== 骨架繪製控制 (開關設定) ==========
        let drawOptions = { cg: true, nose: true, arms: true, body: true };
        document.getElementById('toggleCG').addEventListener('change', e => drawOptions.cg = e.target.checked);
        document.getElementById('toggleNose').addEventListener('change', e => drawOptions.nose = e.target.checked);
        document.getElementById('toggleArms').addEventListener('change', e => drawOptions.arms = e.target.checked);
        document.getElementById('toggleBody').addEventListener('change', e => drawOptions.body = e.target.checked);

        // ========== 相機與錄影邏輯 ==========
        const videoElement = document.getElementById('videoElement');
        const canvasElement = document.getElementById('outputCanvas');
        const canvasCtx = canvasElement.getContext('2d');
        let cameraMode = 'user'; let isCounting = false; let countdownVal = 0; let countdownStartTime = 0;
        let mediaRecorder; let recordedChunks = []; let isRecording = false; let cameraStream = null; let isProcessingAI = false;

        function isStraight(p1, p2, p3) {
            const v1 = { x: p2.x - p1.x, y: p2.y - p1.y }; const v2 = { x: p3.x - p2.x, y: p3.y - p2.y };
            const norm1 = Math.sqrt(v1.x * v1.x + v1.y * v1.y); const norm2 = Math.sqrt(v2.x * v2.x + v2.y * v2.y);
            if (norm1 === 0 || norm2 === 0) return false;
            let cosAngle = (v1.x * v2.x + v1.y * v2.y) / (norm1 * norm2);
            return Math.max(-1.0, Math.min(1.0, cosAngle)) < -0.96;
        }

        const pose = new Pose({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`});
        pose.setOptions({ modelComplexity: 1, smoothLandmarks: true, minDetectionConfidence: 0.7, minTrackingConfidence: 0.7 });

        pose.onResults((results) => {
            canvasElement.width = results.image.width; canvasElement.height = results.image.height;
            const w = canvasElement.width; const h = canvasElement.height;
            canvasCtx.save(); canvasCtx.clearRect(0, 0, w, h); canvasCtx.drawImage(results.image, 0, 0, w, h);

            if (results.poseLandmarks) {
                const lm = results.poseLandmarks;
                const drawLine = (pt1, pt2, color, thickness) => { canvasCtx.beginPath(); canvasCtx.moveTo(pt1.x * w, pt1.y * h); canvasCtx.lineTo(pt2.x * w, pt2.y * h); canvasCtx.strokeStyle = color; canvasCtx.lineWidth = thickness; canvasCtx.stroke(); };
                const drawCircle = (pt, radius, color) => { canvasCtx.beginPath(); canvasCtx.arc(pt.x * w, pt.y * h, radius, 0, 2 * Math.PI); canvasCtx.fillStyle = color; canvasCtx.fill(); };

                if (drawOptions.nose) { drawLine({x: lm[0].x, y: 0}, {x: lm[0].x, y: 1}, 'rgb(135, 206, 235)', 2); drawCircle(lm[0], 8, 'rgb(135, 206, 235)'); }
                if (drawOptions.cg) { const upperMidX = (lm[11].x + lm[12].x) / 2; const lowerMidX = (lm[23].x + lm[24].x) / 2; drawLine({x: upperMidX, y: 0}, {x: lowerMidX, y: 1}, 'rgb(0, 100, 255)', 3); }
                if (drawOptions.body) { [[11, 23], [12, 24], [23, 24], [11, 12]].forEach(([p1, p2]) => { drawLine(lm[p1], lm[p2], 'rgb(100, 100, 100)', 2); drawCircle(lm[p1], 6, 'rgb(150, 150, 150)'); drawCircle(lm[p2], 6, 'rgb(150, 150, 150)'); }); }
                if (drawOptions.arms) {
                    const checkAndDrawArm = (pts) => {
                        const straight = isStraight(lm[pts[0]], lm[pts[1]], lm[pts[2]]) && isStraight(lm[pts[1]], lm[pts[2]], lm[pts[3]]);
                        const color = straight ? 'rgb(0, 255, 0)' : 'rgb(255, 0, 0)';
                        for (let i = 0; i < pts.length - 1; i++) { drawLine(lm[pts[i]], lm[pts[i+1]], color, 6); drawCircle(lm[pts[i]], 8, color); drawCircle(lm[pts[i+1]], 8, color); }
                    };
                    checkAndDrawArm([11, 13, 15, 17]); checkAndDrawArm([12, 14, 16, 18]);
                }
            }
            if (isCounting) {
                const elapsed = (Date.now() - countdownStartTime) / 1000; const remaining = Math.ceil(countdownVal - elapsed);
                if (remaining > 0) {
                    canvasCtx.font = "bold 15vw Arial"; canvasCtx.fillStyle = "red"; canvasCtx.strokeStyle = "white"; canvasCtx.lineWidth = 3;
                    canvasCtx.fillText(remaining, w * 0.05, h * 0.15); canvasCtx.strokeText(remaining, w * 0.05, h * 0.15);
                } else { isCounting = false; capturePhoto(); }
            }
            if (isRecording) { canvasCtx.beginPath(); canvasCtx.arc(w - (w * 0.05), h * 0.05, 20, 0, 2 * Math.PI); canvasCtx.fillStyle = "red"; canvasCtx.fill(); }
            canvasCtx.restore();
        });

        async function startCamera() {
            if (cameraStream) cameraStream.getTracks().forEach(track => track.stop());
            const constraints = { video: { facingMode: cameraMode, width: { ideal: 1920 }, height: { ideal: 1080 }, frameRate: { ideal: 120, min: 60 } } };
            try {
                cameraStream = await navigator.mediaDevices.getUserMedia(constraints);
                videoElement.srcObject = cameraStream; await videoElement.play();
                pose.setOptions({ selfieMode: cameraMode === 'user' });
                if (!isProcessingAI) { isProcessingAI = true; processVideoFrame(); }
            } catch (error) { alert("無法存取相機。"); }
        }
        async function processVideoFrame() {
            if (!isProcessingAI) return;
            if (videoElement.readyState >= 2 && !videoElement.paused) await pose.send({image: videoElement});
            requestAnimationFrame(processVideoFrame);
        }
        startCamera(); 

        document.getElementById('switchCamBtn').addEventListener('click', () => { cameraMode = (cameraMode === 'user') ? 'environment' : 'user'; startCamera(); });
        
        const photoModeToggle = document.getElementById('photoModeToggle');
        const timerInput = document.getElementById('timerInput');
        const modeLabel = document.getElementById('modeLabel');
        const secLabel = document.getElementById('secLabel');
        
        photoModeToggle.addEventListener('change', (e) => {
            if (e.target.checked) { modeLabel.innerText = '倒數'; timerInput.style.display = 'inline-block'; secLabel.style.display = 'inline-block'; } 
            else { modeLabel.innerText = '立即'; timerInput.style.display = 'none'; secLabel.style.display = 'none'; }
        });
        
        document.getElementById('actionShootBtn').addEventListener('click', () => {
            if (photoModeToggle.checked) {
                const val = parseInt(timerInput.value);
                if (isNaN(val) || val <= 0) { alert("請輸入有效的秒數"); return; }
                countdownVal = val; isCounting = true; countdownStartTime = Date.now();
            } else { capturePhoto(); }
        });
        
        document.getElementById('recordBtn').addEventListener('click', () => {
            const recBtn = document.getElementById('recordBtn');
            if (!isRecording) {
                recordedChunks = []; const stream = canvasElement.captureStream(120); let options = { videoBitsPerSecond: 12000000 }; 
                if (MediaRecorder.isTypeSupported('video/mp4;codecs=avc1')) options.mimeType = 'video/mp4;codecs=avc1';
                else if (MediaRecorder.isTypeSupported('video/webm;codecs=vp9')) options.mimeType = 'video/webm;codecs=vp9';
                else if (MediaRecorder.isTypeSupported('video/webm')) options.mimeType = 'video/webm';
                try { mediaRecorder = new MediaRecorder(stream, options); } catch (e) { mediaRecorder = new MediaRecorder(stream); }
                mediaRecorder.ondataavailable = (e) => { if (e.data.size > 0) recordedChunks.push(e.data); };
                mediaRecorder.onstop = saveVideoToGallery; mediaRecorder.start();
                isRecording = true; recBtn.innerText = '⏹️ 停止錄影'; recBtn.classList.add('recording');
            } else {
                mediaRecorder.stop(); isRecording = false; recBtn.innerText = '🎥 開始錄影'; recBtn.classList.remove('recording');
            }
        });

        function dataURLtoFile(dataurl, filename) {
            let arr = dataurl.split(','), mime = arr[0].match(/:(.*?);/)[1], bstr = atob(arr[1]), n = bstr.length, u8arr = new Uint8Array(n);
            while(n--) { u8arr[n] = bstr.charCodeAt(n); } return new File([u8arr], filename, {type:mime});
        }
        function capturePhoto() {
            const dataURL = canvasElement.toDataURL('image/jpeg', 1.0); const now = new Date();
            const filename = `archery_photo_${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}.jpg`;
            addMediaItem(dataURLtoFile(dataURL, filename), 'image');
        }
        function saveVideoToGallery() {
            const blob = new Blob(recordedChunks, { type: mediaRecorder.mimeType || 'video/mp4' }); const now = new Date();
            const ext = blob.type.includes('mp4') ? 'mp4' : 'webm';
            const filename = `archery_video_${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}.${ext}`;
            addMediaItem(new File([blob], filename, { type: blob.type }), 'video');
        }

        document.getElementById('closeModal').addEventListener('click', () => { document.getElementById('mediaModal').style.display = 'none'; document.getElementById('modalContent').innerHTML = ''; });
        
        function openModal(src, type) {
            const modal = document.getElementById('mediaModal'); const content = document.getElementById('modalContent'); content.innerHTML = ''; 
            if (type === 'image') { const img = document.createElement('img'); img.src = src; content.appendChild(img); } 
            else { const vid = document.createElement('video'); vid.src = src; vid.controls = true; vid.autoplay = true; content.appendChild(vid); }
            modal.style.display = 'flex';
        }

        // ========== IndexedDB 媒體庫管理 ==========
        const DB_NAME = 'ArcheryMediaDB'; let db; let mediaItems = []; let isSelectMode = false;
        function initDB() {
            return new Promise((resolve, reject) => {
                const request = indexedDB.open(DB_NAME, 1); request.onerror = e => reject(e); request.onsuccess = e => { db = e.target.result; resolve(db); };
                request.onupgradeneeded = e => { const tempDb = e.target.result; if (!tempDb.objectStoreNames.contains('media')) { tempDb.createObjectStore('media', { keyPath: 'id' }); } };
            });
        }
        async function loadMediaFromDB() {
            try { await initDB(); const tx = db.transaction('media', 'readonly'); const store = tx.objectStore('media'); const request = store.getAll();
                request.onsuccess = () => { mediaItems = request.result; mediaItems.sort((a, b) => b.timestamp - a.timestamp); renderGallery(); };
            } catch (e) { console.error(e); }
        }
        function saveToDB(item) { if (!db) return; const tx = db.transaction('media', 'readwrite'); tx.objectStore('media').put(item); }
        function deleteFromDB(id) { if (!db) return; const tx = db.transaction('media', 'readwrite'); tx.objectStore('media').delete(id); }
        function clearDB() { if (!db) return; const tx = db.transaction('media', 'readwrite'); tx.objectStore('media').clear(); }
        function addMediaItem(file, type) {
            const item = { id: Date.now().toString() + Math.random().toString(36).substr(2, 5), file: file, type: type, timestamp: Date.now(), filename: file.name };
            mediaItems.unshift(item); saveToDB(item); renderGallery(); showToast(`已存入紀錄庫`);
        }
        document.addEventListener('DOMContentLoaded', loadMediaFromDB);

        const galleryContainer = document.getElementById('gallery');
        const toggleSelectModeBtn = document.getElementById('toggleSelectModeBtn');
        function renderGallery() {
            galleryContainer.innerHTML = '';
            if (mediaItems.length === 0) { galleryContainer.innerHTML = '<p style="color:#999; grid-column: 1/-1; text-align:center;">目前沒有暫存紀錄</p>'; return; }
            if (isSelectMode) galleryContainer.parentElement.classList.add('select-mode-on'); else galleryContainer.parentElement.classList.remove('select-mode-on');
            mediaItems.forEach(item => {
                const url = URL.createObjectURL(item.file); const div = document.createElement('div'); div.className = 'gallery-item'; div.id = `media-${item.id}`;
                div.onclick = () => { if (isSelectMode) { div.classList.toggle('selected'); } else { openModal(url, item.type); } };
                const mediaTag = item.type === 'image' ? `<img src="${url}">` : `<video src="${url}" preload="metadata"></video>`;
                div.innerHTML = `${mediaTag}<div class="select-overlay"></div><button class="indiv-save-btn" onclick="event.stopPropagation(); saveIndividual('${item.id}')">單獨存檔</button>`;
                galleryContainer.appendChild(div);
            });
        }
        window.saveIndividual = async function(id) {
            const item = mediaItems.find(i => i.id === id); if (!item) return;
            if (navigator.canShare && navigator.canShare({ files: [item.file] })) { try { await navigator.share({ files: [item.file] }); } catch (e) {} } 
            else { const url = URL.createObjectURL(item.file); const link = document.createElement('a'); link.href = url; link.download = item.filename; document.body.appendChild(link); link.click(); document.body.removeChild(link); }
        };
        toggleSelectModeBtn.addEventListener('click', () => {
            isSelectMode = !isSelectMode;
            if (isSelectMode) { toggleSelectModeBtn.innerText = '完成管理'; toggleSelectModeBtn.style.background = '#E67E22'; document.getElementById('bulkActions').style.display = 'flex'; } 
            else { toggleSelectModeBtn.innerText = '☑️ 多選模式'; toggleSelectModeBtn.style.background = '#34495E'; document.getElementById('bulkActions').style.display = 'none'; document.querySelectorAll('.gallery-item.selected').forEach(el => el.classList.remove('selected')); }
            renderGallery();
        });
        document.getElementById('deleteSelectedBtn').addEventListener('click', () => {
            const selectedEls = document.querySelectorAll('.gallery-item.selected'); if (selectedEls.length === 0) { alert('請先點選要刪除的檔案'); return; }
            if (confirm(`確定要刪除這 ${selectedEls.length} 個檔案嗎？`)) { selectedEls.forEach(el => { const id = el.id.replace('media-', ''); mediaItems = mediaItems.filter(i => i.id !== id); deleteFromDB(id); }); renderGallery(); }
        });
        document.getElementById('saveSelectedBtn').addEventListener('click', async () => {
            const selectedEls = document.querySelectorAll('.gallery-item.selected'); if (selectedEls.length === 0) { alert('請先點選要儲存的檔案'); return; }
            const filesToSave = []; selectedEls.forEach(el => { const id = el.id.replace('media-', ''); const item = mediaItems.find(i => i.id === id); if (item) filesToSave.push(item.file); });
            if (navigator.canShare && navigator.canShare({ files: filesToSave })) { try { await navigator.share({ files: filesToSave, title: '儲存選取的紀錄' }); } catch (e) {} } 
            else { filesToSave.forEach(file => { const url = URL.createObjectURL(file); const link = document.createElement('a'); link.href = url; link.download = file.name; document.body.appendChild(link); link.click(); document.body.removeChild(link); }); }
            toggleSelectModeBtn.click();
        });
        document.getElementById('deleteAllBtn').addEventListener('click', () => {
            if (mediaItems.length === 0) return;
            if (confirm('⚠️ 確定要清空紀錄庫裡的所有照片與影片嗎？\n此動作無法復原！')) { mediaItems = []; clearDB(); renderGallery(); showToast('紀錄庫已清空'); }
        });

        // ========== 瞄準器紀錄系統 ==========
        function saveSightMark() {
            const dist = document.getElementById('sightDist').value;
            const mark = document.getElementById('sightMark').value;
            const note = document.getElementById('sightNote').value;
            
            if (!dist || !mark) { alert("請輸入距離與刻度！"); return; }
            
            const record = { id: Date.now(), dist: dist, mark: mark, note: note, date: new Date().toLocaleDateString() };
            let marks = JSON.parse(localStorage.getItem('archerySightMarks')) || [];
            marks.unshift(record);
            localStorage.setItem('archerySightMarks', JSON.stringify(marks));
            
            document.getElementById('sightDist').value = '';
            document.getElementById('sightMark').value = '';
            document.getElementById('sightNote').value = '';
            showToast('瞄準器紀錄已儲存！');
            renderSightMarks();
        }

        function renderSightMarks() {
            const list = document.getElementById('sightList');
            let marks = JSON.parse(localStorage.getItem('archerySightMarks')) || [];
            if (marks.length === 0) { list.innerHTML = '<p class="no-data">目前無瞄準器紀錄</p>'; return; }
            
            list.innerHTML = '';
            marks.forEach(m => {
                const item = document.createElement('div'); item.className = 'history-item';
                let html = `<div style="display:flex; justify-content:space-between; align-items:center;">
                    <div><b style="color:#E74C3C; font-size:18px;">${m.dist}m</b> ➔ <b style="color:#3498DB; font-size:18px;">刻度: ${m.mark}</b>`;
                if(m.note) html += `<br><span style="font-size:13px; color:#555;">備註: ${m.note}</span>`;
                html += `<br><span style="font-size:11px; color:#aaa;">${m.date}</span></div>
                    <button class="del-history-btn" onclick="deleteSightMark(${m.id})">刪除</button></div>`;
                item.innerHTML = html;
                list.appendChild(item);
            });
        }

        window.deleteSightMark = function(id) {
            if(confirm("確定刪除此瞄準器紀錄嗎？")) {
                let marks = JSON.parse(localStorage.getItem('archerySightMarks')) || [];
                marks = marks.filter(m => m.id !== id);
                localStorage.setItem('archerySightMarks', JSON.stringify(marks));
                renderSightMarks();
            }
        };

        // ========== 雙人對抗賽邏輯 ==========
        let matchData = []; 
        let mEnd = 0, mPlayer = 0, mArrow = 0;
        let matchType = 'recurve';
        let p1Name = "A", p2Name = "B";
        
        let isShootOff = false;
        let shootOffData = [null, null];
        let shootOffWinner = null; 

        function startMatch() {
            matchType = document.getElementById('matchType').value;
            p1Name = document.getElementById('matchP1Name').value || "選手 A";
            p2Name = document.getElementById('matchP2Name').value || "選手 B";
            
            matchData = Array.from({length: 5}, () => [Array(3).fill(null), Array(3).fill(null)]);
            mEnd = 0; mPlayer = 0; mArrow = 0;
            isShootOff = false; shootOffData = [null, null]; shootOffWinner = null;
            
            document.getElementById('matchSetupSection').style.display = 'none';
            document.getElementById('matchScoringSection').style.display = 'block';
            renderMatchTable();
        }

        function parseScore(val) {
            if (val === 'X') return 10; if (val === 'M' || val === null) return 0;
            return parseInt(val) || 0;
        }

        // 加射排名判定：X(12) > 10(11) > 9(10) > ... > M(1)
        function getShootOffRank(val) {
            const ranks = {'X':12, '10':11, '9':10, '8':9, '7':8, '6':7, '5':6, '4':5, '3':4, '2':3, '1':2, 'M':1};
            return ranks[val] || 0;
        }

        window.setMatchActiveCell = function(endIdx, playerIdx, arrowIdx) {
            if (endIdx === 'so') { mPlayer = playerIdx; } 
            else { mEnd = endIdx; mPlayer = playerIdx; mArrow = arrowIdx; }
            renderMatchTable();
        };

        window.resolveShootOff = function(winnerIdx) {
            shootOffWinner = winnerIdx;
            renderMatchTable();
        }
        window.resetShootOff = function() {
            shootOffData = [null, null]; shootOffWinner = null; mPlayer = 0;
            renderMatchTable();
        }

        function renderMatchTable() {
            const table = document.getElementById('matchTable'); table.innerHTML = '';
            
            let p1TotalScore = 0, p2TotalScore = 0;
            let p1SetPoints = 0, p2SetPoints = 0;
            let statusText = "";

            for (let e = 0; e < 5; e++) {
                let p1EndSum = 0, p2EndSum = 0;
                let p1EndFull = true, p2EndFull = true;

                for (let a = 0; a < 3; a++) {
                    const v1 = matchData[e][0][a]; p1EndSum += parseScore(v1); if (v1 === null) p1EndFull = false;
                    const v2 = matchData[e][1][a]; p2EndSum += parseScore(v2); if (v2 === null) p2EndFull = false;
                }
                
                p1TotalScore += p1EndSum; p2TotalScore += p2EndSum;

                if (p1EndFull && p2EndFull) {
                    if (matchType === 'recurve') {
                        if (p1EndSum > p2EndSum) p1SetPoints += 2;
                        else if (p2EndSum > p1EndSum) p2SetPoints += 2;
                        else { p1SetPoints += 1; p2SetPoints += 1; }
                    }
                }
            }

            // 判斷是否結束與是否需要加射
            if (!isShootOff) {
                let matchOverNormal = false; let tied = false;
                if (matchType === 'recurve') {
                    if (p1SetPoints >= 6 || p2SetPoints >= 6) matchOverNormal = true;
                    else if (matchData[4][1][2] !== null) { 
                        if (p1SetPoints === p2SetPoints) tied = true;
                        else matchOverNormal = true;
                    }
                } else {
                    if (matchData[4][1][2] !== null) {
                        if (p1TotalScore === p2TotalScore) tied = true;
                        else matchOverNormal = true;
                    }
                }

                if (tied) {
                    isShootOff = true; mPlayer = 0; shootOffData = [null, null]; shootOffWinner = null;
                }
            }

            // 顯示加射專屬畫面
            if (isShootOff) {
                let s1Disp = matchType === 'recurve' ? `${p1SetPoints} 點` : `${p1TotalScore} 分`;
                let s2Disp = matchType === 'recurve' ? `${p2SetPoints} 點` : `${p2TotalScore} 分`;
                
                let html = `
                    <tr><th colspan="2" style="background:#f1c40f; color:#fff; font-size:18px;">🔥 加射 (Shoot-Off) 🔥</th></tr>
                    <tr>
                        <td style="font-size:18px; width:50%; padding:10px;">${p1Name}<br><b style="color:#2980B9">${s1Disp}</b></td>
                        <td style="font-size:18px; width:50%; padding:10px;">${p2Name}<br><b style="color:#E74C3C">${s2Disp}</b></td>
                    </tr>
                    <tr>
                        <td class="score-cell ${mPlayer===0?'active-cell':''}" style="font-size:24px; padding:15px; font-weight:bold;" onclick="setMatchActiveCell('so', 0, 0)">${shootOffData[0] !== null ? shootOffData[0] : '第一箭'}</td>
                        <td class="score-cell ${mPlayer===1?'active-cell':''}" style="font-size:24px; padding:15px; font-weight:bold;" onclick="setMatchActiveCell('so', 1, 0)">${shootOffData[1] !== null ? shootOffData[1] : '第二箭'}</td>
                    </tr>
                `;
                
                if (shootOffWinner === 'tie') {
                    html += `<tr><td colspan="2" style="padding:15px; background:#FDEDEC;">
                        <p style="color:#E74C3C; font-weight:bold; margin-top:0;">加射分數完全相同！<br>請以靶心距離判定最終勝負：</p>
                        <button onclick="resolveShootOff(0)" style="background:#2980B9; color:white; padding:8px 15px; border-radius:5px; border:none; margin:5px; font-weight:bold;">${p1Name} 較近 (勝)</button>
                        <button onclick="resolveShootOff(1)" style="background:#E74C3C; color:white; padding:8px 15px; border-radius:5px; border:none; margin:5px; font-weight:bold;">${p2Name} 較近 (勝)</button>
                        <br><button onclick="resetShootOff()" style="margin-top:10px; padding:5px 10px; border-radius:5px; border:1px solid #ccc; font-weight:bold;">雙方重新加射</button>
                    </td></tr>`;
                }

                table.innerHTML = html;

                if (shootOffWinner === 0) statusText = `🎉 加射結束，${p1Name} 獲得最終勝利！`;
                else if (shootOffWinner === 1) statusText = `🎉 加射結束，${p2Name} 獲得最終勝利！`;
                else statusText = "⚔️ 加射進行中，一箭定勝負...";
                
            } else {
                // 正常顯示5趟畫面
                let thead = `<tr><th>趟</th><th style="min-width:50px;">選手</th><th>1</th><th>2</th><th>3</th><th>小計</th><th>${matchType==='recurve'?'積點':'總計'}</th></tr>`;
                table.innerHTML = thead;
                
                for (let e = 0; e < 5; e++) {
                    let p1ArrowsStr = "", p2ArrowsStr = "";
                    let p1Sum = 0, p2Sum = 0;
                    
                    for (let a = 0; a < 3; a++) {
                        const val = matchData[e][0][a]; p1Sum += parseScore(val);
                        let aClass = (e === mEnd && mPlayer === 0 && a === mArrow) ? 'active-cell' : '';
                        p1ArrowsStr += `<td class="score-cell ${aClass}" onclick="setMatchActiveCell(${e}, 0, ${a})">${val !== null ? val : ''}</td>`;
                    }
                    for (let a = 0; a < 3; a++) {
                        const val = matchData[e][1][a]; p2Sum += parseScore(val);
                        let aClass = (e === mEnd && mPlayer === 1 && a === mArrow) ? 'active-cell' : '';
                        p2ArrowsStr += `<td class="score-cell ${aClass}" onclick="setMatchActiveCell(${e}, 1, ${a})">${val !== null ? val : ''}</td>`;
                    }
                    
                    let p1Disp = "-", p2Disp = "-";
                    let e_p1Pts=0, e_p2Pts=0;
                    for (let t=0; t<=e; t++) {
                        let t_p1S=0, t_p2S=0; let full = true;
                        for(let a=0;a<3;a++){ if(matchData[t][0][a]===null||matchData[t][1][a]===null) full=false; t_p1S+=parseScore(matchData[t][0][a]); t_p2S+=parseScore(matchData[t][1][a]); }
                        if(full) { if(t_p1S>t_p2S) e_p1Pts+=2; else if (t_p2S>t_p1S) e_p2Pts+=2; else { e_p1Pts+=1; e_p2Pts+=1; } }
                    }
                    if (matchType === 'recurve') { p1Disp = e_p1Pts; p2Disp = e_p2Pts; }
                    else {
                        let t_p1T=0, t_p2T=0;
                        for (let t=0; t<=e; t++) { for(let a=0;a<3;a++){ t_p1T+=parseScore(matchData[t][0][a]); t_p2T+=parseScore(matchData[t][1][a]); } }
                        p1Disp = t_p1T; p2Disp = t_p2T;
                    }

                    let row1 = `<tr><td rowspan="2" style="background:#fff; border-bottom:2px solid #333;"><b>${e+1}</b></td>
                                <td style="background:#EBF5FB;">${p1Name.substring(0,3)}</td>${p1ArrowsStr}
                                <td style="background:#EBF5FB;"><b>${p1Sum}</b></td><td style="background:#D6EAF8; font-weight:bold;">${p1Disp}</td></tr>`;
                    let row2 = `<tr><td style="background:#FDEDEC; border-bottom:2px solid #333;">${p2Name.substring(0,3)}</td>${p2ArrowsStr}
                                <td style="background:#FDEDEC; border-bottom:2px solid #333;"><b>${p2Sum}</b></td><td style="background:#FADBD8; border-bottom:2px solid #333; font-weight:bold;">${p2Disp}</td></tr>`;
                    table.innerHTML += row1 + row2;
                }
                
                if (matchType === 'recurve' && (p1SetPoints >= 6 || p2SetPoints >= 6)) {
                    if (p1SetPoints >= 6) statusText = `🎉 ${p1Name} 獲得勝利！`; else statusText = `🎉 ${p2Name} 獲得勝利！`;
                } else if (matchData[4][1][2] !== null) {
                    if (matchType === 'recurve') {
                        if (p1SetPoints > p2SetPoints) statusText = `🎉 ${p1Name} 獲得勝利！`;
                        else if (p2SetPoints > p1SetPoints) statusText = `🎉 ${p2Name} 獲得勝利！`;
                    } else {
                        if (p1TotalScore > p2TotalScore) statusText = `🎉 ${p1Name} 獲得勝利！`;
                        else if (p2TotalScore > p1TotalScore) statusText = `🎉 ${p2Name} 獲得勝利！`;
                    }
                }
            }

            document.getElementById('matchStatusText').innerText = statusText;
            if (matchType === 'recurve') {
                document.getElementById('p1DisplayScore').innerText = `${p1Name}: ${p1SetPoints}點`; document.getElementById('p2DisplayScore').innerText = `${p2Name}: ${p2SetPoints}點`;
            } else {
                document.getElementById('p1DisplayScore').innerText = `${p1Name}: ${p1TotalScore}分`; document.getElementById('p2DisplayScore').innerText = `${p2Name}: ${p2TotalScore}分`;
            }
        }

        window.inputMatchScore = function(val) {
            if (isShootOff) {
                if (shootOffWinner !== null) return;
                shootOffData[mPlayer] = val;
                if (mPlayer === 0) mPlayer = 1;
                else {
                    let r1 = getShootOffRank(shootOffData[0]); let r2 = getShootOffRank(shootOffData[1]);
                    // 自動判定 X > 10
                    if (r1 > r2) shootOffWinner = 0; else if (r2 > r1) shootOffWinner = 1; else shootOffWinner = 'tie';
                }
                renderMatchTable(); return;
            }

            if (mEnd >= 5) return;
            matchData[mEnd][mPlayer][mArrow] = val;
            mArrow++;
            if (mArrow >= 3) { mArrow = 0; mPlayer++; if (mPlayer >= 2) { mPlayer = 0; mEnd++; } }
            renderMatchTable();
        };

        window.undoMatchScore = function() {
            if (isShootOff) {
                if (shootOffWinner !== null) { shootOffWinner = null; if (shootOffData[1] !== null) mPlayer = 1; }
                else if (shootOffData[1] !== null) { shootOffData[1] = null; mPlayer = 1; }
                else if (shootOffData[0] !== null) { shootOffData[0] = null; mPlayer = 0; }
                else {
                    isShootOff = false; mEnd = 4; mPlayer = 1; mArrow = 2; matchData[4][1][2] = null;
                }
                renderMatchTable(); return;
            }

            if (mEnd === 0 && mPlayer === 0 && mArrow === 0) return; 
            if (mArrow === 0) { 
                if (mPlayer === 1) { mPlayer = 0; mArrow = 2; }
                else { mEnd--; mPlayer = 1; mArrow = 2; }
            } else { mArrow--; }
            matchData[mEnd][mPlayer][mArrow] = null;
            renderMatchTable();
        };

        window.saveMatchRecord = function() {
            const now = new Date();
            const dateStr = `${now.getFullYear()}/${now.getMonth()+1}/${now.getDate()} ${String(now.getHours()).padStart(2,'0')}:${String(now.getMinutes()).padStart(2,'0')}`;
            
            let p1TotalScore = 0, p2TotalScore = 0, p1SetPoints = 0, p2SetPoints = 0;
            for (let e = 0; e < 5; e++) {
                let p1Sum = 0, p2Sum = 0;
                for (let a = 0; a < 3; a++) { p1Sum += parseScore(matchData[e][0][a]); p2Sum += parseScore(matchData[e][1][a]); }
                p1TotalScore += p1Sum; p2TotalScore += p2Sum;
                if (p1Sum > p2Sum) p1SetPoints += 2; else if (p2Sum > p1Sum) p2SetPoints += 2; else if (p1Sum > 0 || p2Sum > 0) { p1SetPoints += 1; p2SetPoints += 1; }
            }

            const record = { 
                id: Date.now(), isMatch: true, matchType: matchType, name: `${dateStr} 對抗賽`, 
                p1Name: p1Name, p2Name: p2Name, p1Total: p1TotalScore, p2Total: p2TotalScore, p1Set: p1SetPoints, p2Set: p2SetPoints,
                isShootOff: isShootOff, shootOffData: isShootOff ? shootOffData : null, shootOffWinner: shootOffWinner,
                data: JSON.parse(JSON.stringify(matchData)) 
            };
            
            let history = JSON.parse(localStorage.getItem('archeryHistory')) || [];
            history.unshift(record); localStorage.setItem('archeryHistory', JSON.stringify(history));
            
            showToast('✅ 對抗賽已存檔！');
            document.getElementById('matchModalUI').style.display = 'none';
            document.getElementById('scoreModalUI').style.display = 'flex';
            switchTab('History');
        };

        // ========== 個人計分系統邏輯 ==========
        let scoreData = []; let totalEnds = 6; let arrowsPerEnd = 6; let currentDistance = '70'; let currentEndIndex = 0; let currentArrowIndex = 0; let currentGrandTotal = 0; let currentAvg = "0.00";

        const tabs = ['Setup', 'Scoring', 'History'];
        tabs.forEach(tab => { document.getElementById(`tab${tab}`).addEventListener('click', () => switchTab(tab)); });

        function switchTab(activeTab) {
            tabs.forEach(tab => { document.getElementById(`tab${tab}`).classList.remove('active'); document.getElementById(`section${tab}`).style.display = 'none'; });
            document.getElementById(`tab${activeTab}`).classList.add('active'); document.getElementById(`section${activeTab}`).style.display = 'block';
            if (activeTab === 'History') renderHistory();
        }

        document.getElementById('inputDistance').addEventListener('change', function(e) {
            const customInput = document.getElementById('inputCustomDistance');
            if (e.target.value === 'custom') { customInput.style.display = 'inline-block'; customInput.focus(); } 
            else { customInput.style.display = 'none'; }
        });

        document.getElementById('startScoreBtn').addEventListener('click', () => {
            totalEnds = parseInt(document.getElementById('inputEnds').value) || 6;
            arrowsPerEnd = parseInt(document.getElementById('inputArrows').value) || 6;
            const distSelect = document.getElementById('inputDistance').value;
            currentDistance = distSelect === 'custom' ? (document.getElementById('inputCustomDistance').value || '0') : distSelect;
            document.getElementById('displayScoreDist').innerText = currentDistance;

            scoreData = Array.from({ length: totalEnds }, () => Array(arrowsPerEnd).fill(null));
            currentEndIndex = 0; currentArrowIndex = 0;
            renderScoreTable(); 
            switchTab('Scoring'); 
        });

        document.getElementById('resetScoreBtn').addEventListener('click', () => { if(confirm('確定要重新開始嗎？')) document.getElementById('startScoreBtn').click(); });

        window.setActiveCell = function(endIdx, arrowIdx) { currentEndIndex = endIdx; currentArrowIndex = arrowIdx; renderScoreTable(); };

        function renderScoreTable() {
            const table = document.getElementById('scoreTable'); table.innerHTML = '';
            let thead = '<tr><th>趟</th>';
            for (let i = 1; i <= arrowsPerEnd; i++) thead += `<th>${i}</th>`;
            thead += '<th>小計</th></tr>'; table.innerHTML += thead;

            currentGrandTotal = 0; let actualArrows = 0; 
            for (let e = 0; e < totalEnds; e++) {
                let rowHtml = `<tr><td><b>${e + 1}</b></td>`; let endTotal = 0;
                for (let a = 0; a < arrowsPerEnd; a++) {
                    const val = scoreData[e][a]; endTotal += parseScore(val);
                    if (val !== null) actualArrows++;
                    let activeClass = (e === currentEndIndex && a === currentArrowIndex) ? 'active-cell' : '';
                    rowHtml += `<td class="score-cell ${activeClass}" onclick="setActiveCell(${e}, ${a})">${val !== null ? val : ''}</td>`;
                }
                currentGrandTotal += endTotal; rowHtml += `<td><b>${endTotal}</b></td></tr>`; table.innerHTML += rowHtml;
            }
            
            currentAvg = actualArrows > 0 ? (currentGrandTotal / actualArrows).toFixed(2) : "0.00";
            document.getElementById('grandTotal').innerText = currentGrandTotal; 
            document.getElementById('displayScoreAvg').innerText = currentAvg;
        }

        window.inputScore = function(val) {
            if (currentEndIndex >= totalEnds) { alert('已經射完了，可以儲存成績了。'); return; }
            scoreData[currentEndIndex][currentArrowIndex] = val;
            currentArrowIndex++;
            if (currentArrowIndex >= arrowsPerEnd) { currentArrowIndex = 0; currentEndIndex++; }
            renderScoreTable();
        };

        window.undoScore = function() {
            if (currentEndIndex === 0 && currentArrowIndex === 0) return; 
            if (currentArrowIndex === 0) { currentEndIndex--; currentArrowIndex = arrowsPerEnd - 1; } 
            else { currentArrowIndex--; }
            scoreData[currentEndIndex][currentArrowIndex] = null; renderScoreTable();
        };

        document.getElementById('saveScoreRecBtn').addEventListener('click', () => {
            const now = new Date();
            const defaultName = `${now.getFullYear()}/${now.getMonth()+1}/${now.getDate()} ${String(now.getHours()).padStart(2,'0')}:${String(now.getMinutes()).padStart(2,'0')} 紀錄`;
            const record = { 
                id: Date.now(), isMatch: false, name: defaultName, totalScore: currentGrandTotal, avgScore: currentAvg, 
                ends: totalEnds, arrows: arrowsPerEnd, distance: currentDistance, data: JSON.parse(JSON.stringify(scoreData)) 
            };
            let history = JSON.parse(localStorage.getItem('archeryHistory')) || [];
            history.unshift(record); localStorage.setItem('archeryHistory', JSON.stringify(history));
            showToast('✅ 成績已儲存！'); switchTab('History');
        });

        function renderHistory() {
            const list = document.getElementById('historyList');
            let history = JSON.parse(localStorage.getItem('archeryHistory')) || [];
            if (history.length === 0) { list.innerHTML = '<div class="no-data">目前沒有歷史紀錄。</div>'; return; }

            list.innerHTML = '';
            history.forEach(rec => {
                const item = document.createElement('div'); item.className = 'history-item';
                const nameInput = document.createElement('input'); nameInput.value = rec.name;
                nameInput.addEventListener('change', (e) => {
                    let h = JSON.parse(localStorage.getItem('archeryHistory')) || [];
                    const idx = h.findIndex(x => x.id === rec.id);
                    if (idx !== -1) { h[idx].name = e.target.value; localStorage.setItem('archeryHistory', JSON.stringify(h)); }
                });
                
                const summaryDiv = document.createElement('div'); summaryDiv.className = 'history-summary';
                
                if (rec.isMatch) {
                    const matchTypeStr = rec.matchType === 'recurve' ? '反曲對抗 (局分制)' : '複合對抗 (總分制)';
                    const p1ScoreDisp = rec.matchType === 'recurve' ? `${rec.p1Set}點` : `${rec.p1Total}分`;
                    const p2ScoreDisp = rec.matchType === 'recurve' ? `${rec.p2Set}點` : `${rec.p2Total}分`;
                    
                    let winnerText = "";
                    if (rec.isShootOff) {
                        let p1SO = rec.shootOffData[0] || '0'; let p2SO = rec.shootOffData[1] || '0';
                        let winnerName = rec.shootOffWinner === 0 ? rec.p1Name : (rec.shootOffWinner === 1 ? rec.p2Name : '未定');
                        winnerText = `<br><span style="color:#F39C12; font-size:12px;">加射: ${p1SO} - ${p2SO} (${winnerName} 勝)</span>`;
                    } else {
                        let winnerName = "";
                        if (rec.matchType === 'recurve') { if(rec.p1Set > rec.p2Set) winnerName = rec.p1Name; else if (rec.p2Set > rec.p1Set) winnerName = rec.p2Name; } 
                        else { if(rec.p1Total > rec.p2Total) winnerName = rec.p1Name; else if (rec.p2Total > rec.p1Total) winnerName = rec.p2Name; }
                        if (winnerName) winnerText = `<br><span style="color:#2ECC71; font-size:12px;">🏆 ${winnerName} 獲勝</span>`;
                    }
                    summaryDiv.innerHTML = `<span style="flex-grow:1;"><b>⚔️ ${matchTypeStr}</b><br><span style="color:#2980B9;">${rec.p1Name} (${p1ScoreDisp})</span> vs <span style="color:#E74C3C;">${rec.p2Name} (${p2ScoreDisp})</span>${winnerText}</span>`;
                } else {
                    const distStr = rec.distance ? rec.distance : '-';
                    const avgStr = rec.avgScore ? rec.avgScore : '-';
                    summaryDiv.innerHTML = `<span style="flex-grow:1;"><b>總分: ${rec.totalScore}</b> <span style="font-size:12px; color:#E74C3C; font-weight:bold;">(均: ${avgStr})</span><br><span style="font-size:12px;">(${rec.ends}趟x${rec.arrows}箭 / ${distStr}m)</span></span>`;
                }
                
                const actionsDiv = document.createElement('div'); actionsDiv.className = 'history-actions';

                const delBtn = document.createElement('button'); delBtn.className = 'del-history-btn'; delBtn.innerText = '刪除';
                delBtn.addEventListener('click', () => {
                    if(confirm('確定要刪除這筆紀錄嗎？')) {
                        history = history.filter(h => h.id !== rec.id);
                        localStorage.setItem('archeryHistory', JSON.stringify(history)); renderHistory();
                    }
                });

                if (rec.isMatch) {
                    const exportBtn = document.createElement('button'); exportBtn.className = 'export-history-btn'; exportBtn.innerText = '🖼️ 轉成相片';
                    exportBtn.addEventListener('click', () => exportMatchScoreToImage(rec.id));
                    actionsDiv.appendChild(exportBtn);
                } else {
                    const exportBtn = document.createElement('button'); exportBtn.className = 'export-history-btn'; exportBtn.innerText = '🖼️ 轉成相片';
                    exportBtn.addEventListener('click', () => exportScoreToImage(rec.id));
                    const statsBtn = document.createElement('button'); statsBtn.className = 'stats-history-btn'; statsBtn.innerText = '📊 統計';
                    statsBtn.addEventListener('click', () => showStats(rec.id));
                    actionsDiv.appendChild(statsBtn); actionsDiv.appendChild(exportBtn);
                }

                actionsDiv.appendChild(delBtn);
                item.appendChild(nameInput); item.appendChild(summaryDiv); item.appendChild(actionsDiv); list.appendChild(item);
            });
        }

        // 統計圓餅圖
        function getScoreColor(val) {
            const colors = { 'X':'#FFD700', '10':'#F1C40F', '9':'#F39C12', '8':'#E74C3C', '7':'#C0392B', '6':'#3498DB', '5':'#2980B9', '4':'#34495E', '3':'#2C3E50', '2':'#ECF0F1', '1':'#BDC3C7', 'M':'#95A5A6' };
            return colors[val] || '#ccc';
        }

        function showStats(id) {
            const history = JSON.parse(localStorage.getItem('archeryHistory')) || []; const rec = history.find(h => h.id === id); if (!rec) return;

            const counts = { 'X':0, '10':0, '9':0, '8':0, '7':0, '6':0, '5':0, '4':0, '3':0, '2':0, '1':0, 'M':0 };
            let actualArrows = 0;
            rec.data.forEach(end => { end.forEach(arr => { if (arr !== null && counts[arr] !== undefined) { counts[arr]++; actualArrows++; } }); });
            
            let avg = actualArrows > 0 ? (rec.totalScore / actualArrows).toFixed(2) : "0.00";
            document.getElementById('statsTitle').innerText = rec.name;
            const distStr = rec.distance ? rec.distance : '-';
            document.getElementById('statsAvg').innerText = `總分: ${rec.totalScore} / 平均: ${avg}分 \n(距離: ${distStr}m / 共 ${actualArrows} 箭)`;
            
            const scoreOrder = ['X', '10', '9', '8', '7', '6', '5', '4', '3', '2', '1', 'M'];
            let textStats = '';
            scoreOrder.forEach(k => {
                if(counts[k] > 0) {
                    let bgColor = (k === 'X' || k === '10' || k === '9') ? '#FEF9E7' : '#f9f9f9';
                    let bColor = getScoreColor(k);
                    textStats += `<span class="stats-badge" style="background:${bgColor}; border-color:${bColor};">${k}: ${counts[k]}</span>`;
                }
            });
            document.getElementById('statsDetails').innerHTML = textStats;

            const labels = scoreOrder.filter(k => counts[k] > 0);
            const data = labels.map(k => counts[k]);
            const bgColors = labels.map(k => getScoreColor(k));

            const ctx = document.getElementById('statsChart').getContext('2d');
            if (statsChartInstance) statsChartInstance.destroy(); 
            statsChartInstance = new Chart(ctx, { type: 'pie', data: { labels: labels, datasets: [{ data: data, backgroundColor: bgColors, borderWidth: 1, borderColor: '#fff' }] }, options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { position: 'right' } } } });
            document.getElementById('statsModalUI').style.display = 'flex';
        }

        function exportScoreToImage(id) {
            const history = JSON.parse(localStorage.getItem('archeryHistory')) || []; const record = history.find(h => h.id === id); if (!record) return;
            const card = document.getElementById('exportCard'); document.getElementById('exName').innerText = record.name;
            const distStr = record.distance ? record.distance : '-'; document.getElementById('exTotal').innerText = `總分: ${record.totalScore} (${distStr}m)`;
            const avgStr = record.avgScore ? record.avgScore : '-'; document.getElementById('exAvg').innerText = `平均: ${avgStr} 分`;
            let thead = '<tr><th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">趟</th>';
            for (let i = 1; i <= record.arrows; i++) thead += `<th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">${i}</th>`;
            thead += '<th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">小計</th></tr>';
            
            let tbody = '';
            for (let e = 0; e < record.ends; e++) {
                tbody += `<tr><td style="border:1px solid #ccc; padding:8px; font-weight:bold;">${e + 1}</td>`; let endTotal = 0;
                for (let a = 0; a < record.arrows; a++) {
                    const val = record.data[e][a]; endTotal += parseScore(val);
                    tbody += `<td style="border:1px solid #ccc; padding:8px;">${val !== null ? val : ''}</td>`;
                }
                tbody += `<td style="border:1px solid #ccc; padding:8px; font-weight:bold;">${endTotal}</td></tr>`;
            }
            document.getElementById('exTable').innerHTML = thead + tbody;
            html2canvas(card, { scale: 2 }).then(canvas => {
                const dataURL = canvas.toDataURL('image/png'); const now = new Date();
                const filename = `score_card_${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}.png`;
                addMediaItem(dataURLtoFile(dataURL, filename), 'image');
            });
        }

        // 對抗賽匯出相片
        function exportMatchScoreToImage(id) {
            const history = JSON.parse(localStorage.getItem('archeryHistory')) || [];
            const record = history.find(h => h.id === id); if (!record) return;

            const card = document.getElementById('exportCard');
            document.getElementById('exName').innerText = record.name;
            document.getElementById('exTotal').innerText = `⚔️ ${record.matchType === 'recurve' ? '反曲弓對抗' : '複合弓對抗'}`;

            let winStr = "";
            if (record.isShootOff) {
                let winnerName = record.shootOffWinner === 0 ? record.p1Name : (record.shootOffWinner === 1 ? record.p2Name : '平手');
                winStr = `加射: ${record.shootOffData[0] || 'M'} vs ${record.shootOffData[1] || 'M'} (${winnerName} 勝)`;
            } else {
                let w = "";
                if (record.matchType === 'recurve') {
                    if (record.p1Set > record.p2Set) w = record.p1Name; else if (record.p2Set > record.p1Set) w = record.p2Name;
                } else {
                    if (record.p1Total > record.p2Total) w = record.p1Name; else if (record.p2Total > record.p1Total) w = record.p2Name;
                }
                if(w) winStr = `🏆 ${w} 獲勝`; else winStr = "平手";
            }
            
            let p1S = record.matchType === 'recurve' ? `${record.p1Set} 點` : `${record.p1Total} 分`;
            let p2S = record.matchType === 'recurve' ? `${record.p2Set} 點` : `${record.p2Total} 分`;

            document.getElementById('exAvg').innerHTML = `<span style="color:#2980B9">${record.p1Name}: ${p1S}</span> vs <span style="color:#E74C3C">${record.p2Name}: ${p2S}</span><br><span style="color:#27AE60">${winStr}</span>`;

            let thead = `<tr><th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">趟</th>
                        <th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">選手</th>
                        <th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">1</th>
                        <th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">2</th>
                        <th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">3</th>
                        <th style="border:1px solid #ccc; padding:8px; background:#f9f9f9;">小計</th></tr>`;
            
            let tbody = '';
            for (let e = 0; e < 5; e++) {
                let p1Sum = 0, p2Sum = 0; let p1A = "", p2A = "";
                for(let a=0; a<3; a++){ 
                    let v1 = record.data[e][0][a]; p1Sum += parseScore(v1); p1A += `<td style="border:1px solid #ccc; padding:5px;">${v1||''}</td>`; 
                    let v2 = record.data[e][1][a]; p2Sum += parseScore(v2); p2A += `<td style="border:1px solid #ccc; padding:5px;">${v2||''}</td>`; 
                }
                tbody += `<tr><td rowspan="2" style="border:1px solid #ccc; padding:8px; font-weight:bold; background:#fff;">${e + 1}</td>
                              <td style="border:1px solid #ccc; padding:5px; background:#EBF5FB;">${record.p1Name.substring(0,3)}</td>${p1A}
                              <td style="border:1px solid #ccc; padding:5px; font-weight:bold; background:#EBF5FB;">${p1Sum}</td></tr>`;
                tbody += `<tr><td style="border:1px solid #ccc; padding:5px; background:#FDEDEC;">${record.p2Name.substring(0,3)}</td>${p2A}
                              <td style="border:1px solid #ccc; padding:5px; font-weight:bold; background:#FDEDEC;">${p2Sum}</td></tr>`;
            }
            document.getElementById('exTable').innerHTML = thead + tbody;

            html2canvas(document.getElementById('exportCard'), { scale: 2 }).then(canvas => {
                const dataURL = canvas.toDataURL('image/png'); const now = new Date();
                const filename = `match_card_${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}.png`;
                addMediaItem(dataURLtoFile(dataURL, filename), 'image');
            });
        }
    </script>
</body>
</html>
