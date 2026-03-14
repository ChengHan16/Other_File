```
<!DOCTYPE html>

<html lang="zh-TW">

<head>

    <meta charset="UTF-8">

    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">    

    <title>法院通訊 - 訊息中心</title>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">

    <link href="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.5.13/cropper.min.css" rel="stylesheet">

    <script src="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.5.13/cropper.min.js"></script>

    <link rel="icon" href="data:,">

    <style>

        /* --- 全域基礎與阻擋選取 --- */

        * { -webkit-touch-callout: none !important; -webkit-user-select: none !important; user-select: none !important; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }

        input, textarea { -webkit-user-select: auto !important; user-select: auto !important; }



        :root { 

            --ly-blue: #003153; --ly-blue-trans: rgba(0, 49, 83, 0.85); 

            --ly-gold: #cfa900; --bg-gray: #f2f4f7; --white: #ffffff; 

            --border-color: #dee2e6; --container-width: 790px; 

        }



        html, body {

            height: 100dvh !important; width: 100vw !important; overflow: hidden;

            margin: 0; padding: 0; font-family: "Noto Sans TC", sans-serif;

            background: var(--bg-gray); color: #333;

        }



        /* --- 雙視圖切換架構 (SPA 核心) --- */

        .app-view {

            position: fixed; top: 0; left: 0;

            width: 100vw; height: 100dvh;

            display: flex; flex-direction: column;

            background: var(--bg-gray);

            transition: transform 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);

            will-change: transform;

        }



        #messengerView { z-index: 10; transform: translateX(0%); }

        #chatView { 

            z-index: 20; 

            transform: translateX(100%); 

            box-shadow: -5px 0 25px rgba(0,0,0,0.15);

        }



        /* --- 頂部導覽列 --- */

        .messenger-header {

            background: var(--ly-blue-trans); backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px);

            position: sticky; top: 0; z-index: 100; box-shadow: 0 2px 10px rgba(0,0,0,0.2);

            border-bottom: 1px solid rgba(255,255,255,0.1); padding-top: env(safe-area-inset-top);

        }

        .header-content {

            max-width: var(--container-width); margin: 0 auto; 

            padding: 0 41px; /* 【關鍵對齊】：精準對應下方列表的偏移量 41px */

            height: 80px;

            display: flex; align-items: center; justify-content: space-between; box-sizing: border-box;

        }

        @media (max-width: 750px) { .header-content { padding: 0 41px; } }

        /* 確保左側個人資料區塊具有彈性，不會硬擠右邊的按鈕 */

        .user-profile { display: flex; align-items: center; flex: 1; min-width: 0; }

        .user-avatar { width: 55px; height: 55px; border-radius: 50%; object-fit: cover; margin-right: 15px; flex-shrink: 0; }



        /* 加入防長檔名換行設計，太長會自動變 ... */

/* 加入防長檔名換行設計，稍微縮小字體並強制加上右側安全間距 */

        .user-name { 

            font-weight: 800; 

            font-size: 1.2rem; /* 【修改】：從 1.35rem 縮小到 1.2rem，看起來更精緻 */

            color: white; 

            letter-spacing: 1px; 

            white-space: nowrap; 

            overflow: hidden; 

            text-overflow: ellipsis; 

            margin-right: 15px; /* 【關鍵修改】：強制文字右邊一定要留 15px 的空白，絕對不會撞到按鈕 */

        }

        /* 【關鍵修復】：強制右側容器水平並排，且絕對不壓縮變形 */

        .header-actions { display: flex; flex-direction: row; align-items: center; gap: 12px; flex-shrink: 0; }



        /* 移除原有的 margin-left 改用 flex gap，並加上 flex-shrink: 0 防止按鈕變形 */

        .header-actions button {

            background: rgba(255, 255, 255, 0.15); border: 1px solid rgba(255, 255, 255, 0.2); 

            color: white; width: 42px; height: 42px; border-radius: 50%; cursor: pointer; margin-left: 0;

            font-size: 1.1rem; transition: 0.3s; display: flex; align-items: center; justify-content: center; flex-shrink: 0;

        }

        

        /* --- Messenger 列表樣式 --- */

        .chat-list { 

            flex: 1; overflow-y: auto; -webkit-overflow-scrolling: touch; width: 100%;

            max-width: var(--container-width); margin: 0 auto; padding: 15px 15px 100px 15px; box-sizing: border-box;

        }

        .chat-item {

            background: white; display: flex; align-items: center; padding: 18px 20px; 

            margin-bottom: 0px; border-radius: 8px; cursor: pointer; transition: all 0.3s ease;

            border: 1px solid var(--border-color); border-left: 6px solid transparent;

            position: relative; /* 【關鍵新增】 */

        }

        .chat-item:hover { transform: translateY(-3px); box-shadow: 0 8px 20px rgba(0,0,0,0.08); border-color: #ffffff; }

        .chat-info { 

            flex-grow: 1; padding-left: 5px; 

            min-width: 0; /* 【關鍵新增】 */

        }

        .chat-info h4 { 

            margin: 0 0 6px 0; font-size: 1.25rem; color: var(--ly-blue); 

            white-space: nowrap; overflow: hidden; text-overflow: ellipsis; /* 【關鍵新增】不換行處理 */

            flex: 1; min-width: 0; /* 配合右側時間排版 */

        }

        .chat-info p { margin: 0; color: #666; font-size: 1.05rem; display: -webkit-box; line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; word-break: break-all; }

        .chat-time { 

            font-size: 0.85rem; color: #999; font-weight: 500; 

            flex-shrink: 0; margin-left: 10px; /* 【關鍵新增】 */

        }

        .chat-item.unread { background-color: #fdfcf5; border-left: 6px solid var(--ly-gold) !important; }

        .chat-item.unread h4 { font-weight: 900 !important; color: var(--ly-blue); }

        .chat-item.unread p { color: #000 !important; font-weight: 800 !important; background: transparent !important; }

        .fab {

            position: fixed; bottom: 40px; right: 40px; background: var(--ly-gold); color: white;

            width: 65px; height: 65px; border-radius: 50%; display: flex; align-items: center; justify-content: center;

            font-size: 28px; box-shadow: 0 6px 20px rgba(207, 169, 0, 0.4); cursor: pointer; border: none; z-index: 15;

        }



        /* --- Chat 內部樣式 --- */

        .back-btn { background: none; border: none; margin-right: 15px; cursor: pointer; color: var(--ly-gold); font-size: 1.4rem; transition: 0.3s; display: flex; align-items: center; }

        .messages {

            flex: 1; overflow-y: auto; padding: 20px 15px; display: flex; flex-direction: column; gap: 4px;

            background: var(--bg-gray); -webkit-overflow-scrolling: touch; max-width: var(--container-width); margin: 0 auto; width: 100%;

        }

        .msg-wrapper { display: flex; flex-direction: column; width: 100%; margin-bottom: 4px; }

        .msg-wrapper.mine { align-items: flex-end; }

        .msg-wrapper.theirs { align-items: flex-start; }

        .msg-row { display: flex; align-items: flex-end; gap: 5px; width: 100%; }

        .msg-row.mine { justify-content: flex-end; }

        .msg-row.theirs { justify-content: flex-start; }

        .msg-avatar { width: 35px; height: 35px; border-radius: 50%; flex-shrink: 0; border: 1px solid #ddd; object-fit: cover; background: #fff; margin:0;}

        .msg-avatar-container { display: flex; flex-direction: column; align-items: center; justify-content: flex-end; width: 42px; flex-shrink: 0; gap: 4px; }

        .msg-avatar-name { font-size: 10px; color: #888; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 45px; text-align: center; line-height: 1; }

        .msg { max-width: 75%; padding: 12px 16px; font-size: 15px; line-height: 1.5; word-wrap: break-word; box-shadow: 0 2px 5px rgba(0,0,0,0.05); border-radius: 18px; -webkit-user-select: none !important; user-select: none !important; cursor: default; }

        .msg.media-only { background: transparent !important; border: none !important; box-shadow: none !important; padding: 0 !important; -webkit-user-select: none !important; user-select: none !important; }

        .msg.media-only .msg-media { border: 1px solid rgba(0,0,0,0.05); }

        .msg.mine { background: var(--ly-blue); color: var(--white); border-radius: 18px 18px 4px 18px; border-right: 3px solid var(--ly-gold); }

        .msg.theirs { background: var(--white); color: #333; border-radius: 18px 18px 18px 4px; border: 1px solid #dee2e6; }

        .time-divider { align-self: center; font-size: 0.75rem; color: #888; margin: 20px 0 10px 0; background: rgba(0,0,0,0.05); padding: 4px 14px; border-radius: 20px; font-weight: 500; }

        

        /* --- 訊息時間與已讀狀態 --- */

        .msg-meta { display: flex; flex-direction: column; justify-content: flex-end; margin: 0 6px; min-width: 40px; padding-bottom: 2px;}

        .msg-meta.mine { align-items: flex-end; }

        .msg-meta.theirs { align-items: flex-start; }

        .msg-time { font-size: 11px; color: #999; line-height: 1; }

        .read-status { font-size: 11px; color: #999; line-height: 1; margin-bottom: 4px; padding: 0; white-space: nowrap; }

        .read-status.group-read { cursor: pointer; color: var(--ly-blue); font-weight: 600; transition: opacity 0.2s; }

        .read-status.group-read:hover { opacity: 0.7; text-decoration: underline; }

        

        .input-area { flex-shrink: 0; background: var(--white); padding: 0; display: flex; flex-direction: column; align-items: center; border-top: 1px solid #e9ecef; box-shadow: 0 -2px 10px rgba(0,0,0,0.03); width: 100%; position: relative;}

        .input-container { max-width: var(--container-width); width: 100%; display: flex; align-items: center; gap: 12px; padding: 12px 15px; padding-bottom: calc(12px + env(safe-area-inset-bottom)); box-sizing: border-box; }

        #msgInput { flex: 1; height: 44px; padding: 0 18px; border: 1.5px solid #eee; border-radius: 25px; outline: none; font-size: 16px; background: #f8f9fa; transition: border-color 0.3s; }

        #msgInput:focus { border-color: var(--ly-gold); background: #fff; }

        .send-btn { background: var(--ly-blue); color: var(--white); border: none; width: 44px; height: 44px; border-radius: 50%; display: flex; align-items: center; justify-content: center; cursor: pointer; transition: 0.3s; box-shadow: 0 4px 10px rgba(0, 49, 83, 0.2); flex-shrink: 0; }

        .attach-btn, .expand-menu-btn { background: none; border: none; color: #888; font-size: 1.4rem; cursor: pointer; padding: 0 5px; display: flex; align-items: center; justify-content: center; transition: 0.3s;}

        .attach-btn:hover, .expand-menu-btn:hover { color: var(--ly-blue); }

        .msg-media { max-width: 100%; max-height: 250px; border-radius: 12px; margin-top: 5px; display: block; cursor: pointer; }

        

        .msg.mine .mention-highlight { color: var(--ly-gold); font-weight: 800; background: rgba(255, 255, 255, 0.2); padding: 1px 6px; border-radius: 10px; }

        .msg.theirs .mention-highlight { color: var(--ly-blue); font-weight: 800; background: rgba(0, 49, 83, 0.1); padding: 1px 6px; border-radius: 10px; }

        

        .pinned-bar { display: none; align-items: center; background: rgba(255, 255, 255, 0.75); backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px); padding: 10px 20px; box-shadow: 0 4px 10px rgba(0,0,0,0.04); border-bottom: 1px solid rgba(0,0,0,0.05); position: relative; z-index: 90; max-width: var(--container-width); margin: 0 auto; width: 100%; flex-shrink: 0; }

        .pinned-icon { color: var(--ly-gold); font-size: 1.1rem; margin-right: 12px; transform: rotate(-45deg); }

        .pinned-content { flex: 1; font-size: 0.95rem; color: #333; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; font-weight: 600; cursor: pointer; }

        .unpin-btn { background: none; border: none; color: #999; cursor: pointer; font-size: 1rem; padding: 5px; transition: 0.3s; }

        .unpin-btn:hover:not(:disabled) { color: #ff4d4d; }



        .reply-preview-bar { display: none; background: #f2f4f7; padding: 8px 15px; font-size: 0.85rem; color: #555; border-left: 4px solid var(--ly-blue); align-items: center; justify-content: space-between; width: 100%; border-top: 1px solid #e9ecef; }

        .reply-preview-bar .close-reply { cursor: pointer; color: #999; font-size: 1.1rem; padding: 5px; }

        .reply-snippet { font-size: 0.75rem; color: #666; background: rgba(0, 0, 0, 0.05); padding: 4px 8px; border-radius: 4px; border-left: 3px solid var(--ly-blue); margin-bottom: 5px; cursor: pointer; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%; display: block; }

        .msg.mine .reply-snippet { background: rgba(255, 255, 255, 0.2); color: #f1f1f1; border-left: 3px solid var(--ly-gold); }



        .expand-menu-panel { display: none; position: absolute; bottom: 100%; left: 0; width: 100%; background: var(--bg-gray); border-top: 1px solid #dee2e6; box-shadow: 0 -4px 15px rgba(0,0,0,0.05); padding: 20px; z-index: 10; gap: 25px; flex-direction: row; justify-content: flex-start; animation: slideUp 0.3s ease-out; box-sizing: border-box; }

        @keyframes slideUp { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

        .expand-menu-item { display: flex; flex-direction: column; align-items: center; gap: 8px; cursor: pointer; }

        .expand-menu-item .icon-circle { width: 55px; height: 55px; background: white; border-radius: 50%; display: flex; justify-content: center; align-items: center; font-size: 1.4rem; color: var(--ly-blue); box-shadow: 0 2px 8px rgba(0,0,0,0.05); transition: 0.2s; }

        .expand-menu-item:hover .icon-circle { background: var(--ly-gold); color: white; transform: scale(1.05); }

        .expand-menu-item span { font-size: 0.85rem; color: #555; font-weight: 700; }



        .mention-list { display: none; position: absolute; bottom: 100%; left: 0; width: 100%; max-height: 200px; overflow-y: auto; background: var(--white); border-top: 1px solid var(--border-color); box-shadow: 0 -10px 20px rgba(0,0,0,0.15); z-index: 10000; }

        .mention-item { display: flex; align-items: center; padding: 12px 20px; cursor: pointer; border-bottom: 1px solid #f2f4f7; transition: background 0.2s; }

        .mention-item:hover { background: #f8f9fa; }

        .mention-item img { width: 30px; height: 30px; border-radius: 50%; margin-right: 12px; object-fit: cover; border: 1px solid #ddd; }

        .mention-item span { font-size: 14px; font-weight: 700; color: var(--ly-blue); }



        /* 【修改】：將 width 改為 min-width，讓它能自然貼齊對話氣泡 */

        .msg-event-card { 

            background: #fffdf5; 

            border: 1px solid var(--ly-gold); 

            border-radius: 8px; 

            padding: 12px 15px; 

            margin-top: 5px; 

            box-shadow: 0 2px 8px rgba(207, 169, 0, 0.15); 

            display: flex; 

            flex-direction: column; 

            gap: 6px; 

            cursor: pointer; 

            text-decoration: none; 

            color: inherit; 

            min-width: 230px; /* 保留最小寬度，避免文字太短時卡片變形 */

            max-width: 100%; /* 避免超出手機螢幕 */

        }

        /* 活動卡片頂部標題列：改為左右並排 */

        .msg-event-header { 

            font-size: 0.85rem; 

            color: #888; 

            border-bottom: 1px solid #eee; 

            padding-bottom: 4px; 

            display: flex; 

            justify-content: space-between; 

            align-items: center; 

        }

        /* 倒數時間的警告樣式 */

        .msg-event-countdown {

            color: #d93025; /* 紅色警示 */

            font-weight: 900;

            font-size: 0.8rem;

            background: #fff5f5;

            padding: 2px 6px;

            border-radius: 4px;

        }

        /* 已截止的灰色樣式 */

        .msg-event-countdown.expired {

            color: #999;

            background: #f5f5f5;

        }

        .msg-event-title { font-size: 1.05rem; font-weight: 800; color: var(--ly-blue); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

        .msg-event-time { font-size: 0.85rem; color: #555; display: flex; align-items: center; gap: 5px; }



        /* --- 其他 Modal 與組件 (完美適配 iOS dVH) --- */

        .modal-overlay { position: fixed; inset: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.6); backdrop-filter: blur(4px); -webkit-backdrop-filter: blur(4px); display: none; justify-content: center; align-items: center; z-index: 3000; padding: env(safe-area-inset-top, 20px) 20px 20px 20px; box-sizing: border-box; overflow: hidden;}        .modal-card { background: white; width: 100%; max-width: 450px; max-height: 85dvh; display: flex; flex-direction: column; border-radius: 16px; overflow: hidden; box-shadow: 0 20px 40px rgba(0,0,0,0.3); border: 1px solid var(--border-color); animation: modalFadeIn 0.3s cubic-bezier(0.25, 0.8, 0.25, 1); margin: 0;}

        @keyframes modalFadeIn { from { opacity: 0; transform: translateY(-20px); } to { opacity: 1; transform: translateY(0); } }

        .modal-header { flex-shrink: 0; background: var(--ly-blue-trans); padding: 15px 20px; color: white; display: flex; justify-content: space-between; align-items: center; border-bottom: 3px solid var(--ly-gold); }

        .modal-header h3 { margin: 0; font-size: 1.1rem; }

        .modal-body { flex: 1; padding: 20px; overflow-x: hidden; overflow-y: auto; -webkit-overflow-scrolling: touch; overscroll-behavior: none; width: 100%; box-sizing: border-box; }



        /* --- 新增活動 (Event) 專用樣式 --- */

        .field-adder-bar { display: flex; flex-wrap: wrap; gap: 8px; margin: 15px 0; padding: 10px; background: #f8f9fa; border-radius: 8px; border: 1px solid #eee; width: 100%; box-sizing: border-box; }

        .tag-btn { background: white; border: 1.5px solid var(--ly-blue); color: var(--ly-blue); padding: 5px 12px; border-radius: 20px; cursor: pointer; font-size: 0.85rem; transition: all 0.2s; font-weight: bold; }

        .tag-btn:hover { background: var(--ly-blue); color: white; }

        .tag-btn-img { background: #fdf7e3; border-color: var(--ly-gold); color: #856404; }

        .tag-btn-img:hover { background: var(--ly-gold); color: white; }

        .draggable-item { display: flex; align-items: flex-end; justify-content: space-between; gap: 10px; padding: 12px; margin-bottom: 10px; background: #fff; border-radius: 8px; border: 1px solid #ddd; box-sizing: border-box; width: 100%;}

        .drag-handle { color: #ccc; padding: 10px; cursor: grab; }

        .btn-remove-field { width: 35px; height: 35px; border-radius: 8px; background: #fff0f0; color: #d93025; border: 1px solid #f8d7da; display: flex; align-items: center; justify-content: center; cursor: pointer; flex-shrink: 0;}

        .image-field-preview { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 10px; width: 100%; box-sizing: border-box; }

        .preview-card { position: relative; width: 80px; height: 80px; border-radius: 8px; overflow: hidden; border: 1px solid #eee; cursor: grab; }

        .preview-card img { width: 100%; height: 100%; object-fit: cover; }

        .preview-remove-btn { position: absolute; bottom: 2px; right: 2px; background: rgba(255,255,255,0.9); color: #d93025; border-radius: 4px; width: 20px; height: 20px; display: flex; align-items: center; justify-content: center; cursor: pointer; font-size: 0.7rem; border: 1px solid #d93025; }

        .member-preview-list { display: flex; flex-wrap: wrap; gap: 8px; padding: 12px; background: #fafafa; border: 1.5px dashed #d0d0d0; border-radius: 8px; min-height: 40px; align-items: center; width: 100%; box-sizing: border-box; }

        .ev-dynamic-input { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 16px; appearance: none; box-sizing: border-box; }

        .ev-dynamic-input:focus { border-color: var(--ly-gold); outline: none; }

        .picker-search-wrapper { position: relative; flex: 1; min-width: 150px; }

        .picker-search-wrapper i { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); color: #aaa; }

        .picker-search-wrapper input { width: 100%; padding: 8px 10px 8px 30px; border-radius: 20px; border: 1px solid #ddd; font-size: 16px; appearance: none; box-sizing: border-box; }

        .modal-input-group { margin-bottom: 20px; }

        .modal-input-group label { display: block; font-size: 13px; color: var(--ly-blue); font-weight: 700; margin-bottom: 8px; text-align: left;}

        .modal-input-group input { width: 100%; padding: 12px 15px; border: 1px solid var(--border-color); border-radius: 8px; font-size: 16px; outline: none; box-sizing: border-box; appearance: none;}        .modal-input-group input:focus { border-color: var(--ly-gold); }

        .modal-footer { flex-shrink: 0; padding: 15px 20px; background: #f8f9fa; display: flex; justify-content: flex-end; gap: 12px; }

        .btn-modal { padding: 10px 20px; border-radius: 8px; font-size: 14px; font-weight: 700; cursor: pointer; border: none; transition: 0.3s;}

        .btn-confirm { background: var(--ly-blue); color: white; }

        .btn-cancel { background: white; color: #666; border: 1px solid #ddd; }

        .btn-danger { background: #ff4d4d; color: white; }



        /* 多選模式專用樣式 */

        .msg-checkbox { display: none; font-size: 1.4rem; color: #ccc; margin-right: 10px; transition: 0.2s; align-self: center; flex-shrink: 0; }

        .select-mode-active .msg-checkbox { display: flex; }

        .select-mode-active .msg-row.mine { cursor: pointer; }

        .select-mode-active .msg-wrapper.mine .msg { pointer-events: none; opacity: 0.85; transition: 0.2s; }

        .msg-checkbox.checked { color: var(--ly-gold); }

        .msg-checkbox.checked + .msg { opacity: 1 !important; box-shadow: 0 0 0 3px var(--ly-gold); }



        /* 卡片式選擇 */

        .user-select-list { display: grid; grid-template-columns: repeat(auto-fill, minmax(85px, 1fr)); gap: 15px; padding: 10px; background: transparent; border: none; }

        .user-select-item { display: block; margin: 0; padding: 0; cursor: pointer; }

        .user-select-item input { display: none; }

        .user-select-card { display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 10px 5px; background: white; border: 1.5px solid #eee; border-radius: 12px; transition: all 0.2s ease; box-shadow: 0 2px 6px rgba(0,0,0,0.03); }

        .user-select-card:hover { border-color: #ddd; box-shadow: 0 4px 10px rgba(0,0,0,0.08); transform: translateY(-2px); }

        .user-select-card img { width: 55px; height: 55px; border-radius: 50%; object-fit: cover; margin-bottom: 8px; border:none; flex-shrink:0;}

        .user-select-card span { font-size: 13px; font-weight: 600; color: #333; text-align: center; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; width: 100%; padding: 0 4px; }

        .user-select-item input:checked + .user-select-card { border-color: var(--ly-gold); background: #fffdf5; box-shadow: 0 4px 12px rgba(207, 169, 0, 0.15); }

        .user-select-item input:checked + .user-select-card img { border: 2px solid var(--ly-gold); }

        .user-select-item input:checked + .user-select-card span { color: var(--ly-gold); }



        /* 群組設定/裁切/成員 */

        .group-avatar-preview { width: 90px; height: 90px; border-radius: 50%; object-fit: cover; margin-bottom: 5px; border: 3px solid var(--ly-gold); cursor: pointer; transition:0.2s;}

        .group-avatar-preview:hover { opacity:0.8;}

        .avatar-edit-wrapper { position: relative; display: inline-block; cursor: pointer; }

        .avatar-edit-icon { position: absolute; bottom: 5px; right: 0; background: var(--ly-blue); color: white; border-radius: 50%; width: 28px; height: 28px; display: flex; align-items: center; justify-content: center; border: 2px solid white; font-size: 12px; transition:0.2s;}

        .avatar-edit-wrapper:hover .avatar-edit-icon{ background: var(--ly-gold); transform: scale(1.1);}

        

        .zoom-slider-container { display: flex; flex-direction: column; align-items: center; justify-content: center; height: 250px; width: 40px; background: rgba(255, 255, 255, 0.1); border-radius: 20px; padding: 15px 0; }

        #cropperZoomSlider { 

            appearance: auto; 

            writing-mode: vertical-lr; /* 現代標準：垂直排列 */

            direction: rtl; /* 讓數值方向由下往上增加 */

            height: 100%; 

            width: 8px; /* 稍微加寬一點點，讓手機更好滑動 */

            cursor: pointer; 

            accent-color: var(--ly-gold); 

            outline: none; 

        }

        .cropper-view-box, .cropper-face { border-radius: 50%; }



        .group-members-list { display: flex; flex-direction: column; gap: 10px; margin-top: 5px; background: #f8f9fa; padding: 12px; border-radius: 8px; border: 1px solid var(--border-color); max-height: 200px; overflow-y: auto; }

        .member-item { display: flex; align-items: center; gap: 12px; }

        .member-item img { width: 32px; height: 32px; border-radius: 50%; object-fit: cover; border: 1px solid #ddd; }

        .member-item span { font-size: 14px; color: #333; font-weight: 600; }

        .more-members-btn { background: none; border: none; color: var(--ly-blue); font-size: 13px; font-weight: bold; cursor: pointer; padding: 8px 0 4px 0; text-align: center; width: 100%; transition: color 0.2s; }



        @keyframes pulse-save { 0% { box-shadow: 0 0 0 0 rgba(207, 169, 0, 0.7); background-color: var(--ly-gold); color: #fff; } 70% { box-shadow: 0 0 0 10px rgba(207, 169, 0, 0); background-color: #e6bc00; color: #fff; } 100% { box-shadow: 0 0 0 0 rgba(207, 169, 0, 0); background-color: var(--ly-gold); color: #fff; } }

        .btn-highlight { animation: pulse-save 1.5s infinite !important; border: none !important; }



        /* UI 元件: Scroll/Menu/Preview */

        .scroll-bottom-btn { 

            display: none; 

            position: fixed; 

            bottom: 110px; /* 【修改】：從 90px 提高到 110px，避開輸入框 */

            left: 50%; 

            transform: translateX(-50%); 

            background: var(--white); 

            color: var(--ly-blue); 

            border: 1px solid #ddd; 

            border-radius: 20px; 

            padding: 8px 15px; 

            font-size: 0.9rem; 

            box-shadow: 0 4px 10px rgba(0,0,0,0.15); 

            cursor: pointer; 

            z-index: 100; /* 【修改】：提高層級確保在訊息上方 */

            align-items: center; 

            gap: 8px; 

            font-weight: bold; 

        }        

        @keyframes highlightMsg { 0% { box-shadow: 0 0 0 3px var(--ly-gold); background-color: #fff8dc; } 100% { box-shadow: 0 0 0 0 transparent; background-color: inherit; } }

        .highlight-animation { animation: highlightMsg 2s ease-out; }



        .context-menu { display: none; position: fixed; background: var(--white); border-radius: 12px; box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15); padding: 6px; z-index: 10000; border: 1px solid rgba(0,0,0,0.05); }

        .context-menu button { background: none; border: none; color: #444; font-size: 14px; padding: 10px 16px; cursor: pointer; display: flex; align-items: center; gap: 8px; border-radius: 8px; font-weight: bold; white-space: nowrap; }

        .context-menu button:hover { background: #f2f4f7; }

        #revokeMenuBtn { color: #ff4d4d; }



        /* 圖片/文件預覽 */

        .image-modal { display: none; position: fixed; z-index: 100000; left: 0; top: 0; width: 100vw; height: 100dvh; background-color: rgba(0, 0, 0, 0.85); backdrop-filter: blur(5px); justify-content: center; align-items: center; flex-direction: column; }

        .image-modal-close { position: absolute; top: 20px; right: 30px; color: #f1f1f1; font-size: 40px; font-weight: bold; cursor: pointer; z-index: 100001; }

        .image-modal-content { max-width: 95%; max-height: 90%; object-fit: contain; border-radius: 8px; animation: zoomIn 0.3s ease-out; }

        @keyframes zoomIn { from { transform: scale(0.8); opacity: 0; } to { transform: scale(1); opacity: 1; } }



        /* 活動行程總覽 */

        .event-mini-list { flex: 1; overflow-y: auto; display: flex; flex-direction: column; gap: 15px; padding-right:5px;}

        .event-mini-item { background: white; border-left: 5px solid var(--ly-blue); border-radius: 8px; padding: 15px 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); transition: all 0.2s ease; cursor: pointer; display: block; color: inherit; }

        .event-mini-item:hover { transform: translateX(4px); border-color: var(--ly-gold); box-shadow: 0 4px 12px rgba(0,0,0,0.1); }

        .event-mini-title { font-size: 1.1rem; color: var(--ly-blue); font-weight: 800; margin: 0 0 8px 0; line-height: 1.4; }

        .event-mini-meta { font-size: 0.85rem; color: #666; display: flex; align-items: center; gap: 8px; }

        .event-mini-meta i { color: var(--ly-gold); }

        .event-empty-state { text-align: center; color: #999; padding: 40px 20px; font-size: 0.95rem; }

        .event-year-title { font-size: 1.1rem; color: var(--ly-blue); font-weight: 800; margin: 20px 0 5px 0; padding-left: 10px; border-left: 4px solid var(--ly-gold); background: linear-gradient(90deg, rgba(0,49,83,0.05) 0%, transparent 100%); height: 30px; display: flex; align-items: center; }



        /* Doc Preview & Slider (精簡版) */

        .doc-modal { display: none; position: fixed; z-index: 9999; left: 0; top: 0; width: 100vw; height: 100dvh; background: rgba(0, 0, 0, 0.7); backdrop-filter: blur(8px); justify-content: center; align-items: flex-start; padding: env(safe-area-inset-top, 20px) 0 20px 0; overflow-y: auto; overflow-x: hidden; }

        .doc-paper { background: #ffffff; width: 92%; max-width: 600px; margin: 20px auto 50px; padding: 40px 20px; position: relative; border: 1px solid #dcdcdc; box-shadow: 0 0 40px rgba(0,0,0,0.15); border-radius: 8px; background-image: linear-gradient(90deg, var(--ly-blue) 0px, var(--ly-blue) 4px, transparent 4px, transparent 7px, var(--ly-gold) 7px, var(--ly-gold) 9px, transparent 9px), linear-gradient(#fdfdfd 1px, transparent 1px); background-size: 100% 100%, 100% 2.5rem; }

        .doc-close-btn { position: absolute; right: 15px; top: 15px; font-size: 2rem; cursor: pointer; color: #999; background: none; border: none; z-index: 10; }

        .doc-sub-title { color: var(--ly-blue); border-left: 5px solid var(--ly-gold); padding-left: 15px; margin: 30px 0 20px; font-size: 1.2rem; font-weight: bold; }

        .vote-panel { background: #ffffff; padding: 0; border-radius: 8px; border: 1px solid #e0e0e0; border-top: 5px solid var(--ly-blue); text-align: center; margin-top: 40px; overflow: hidden; box-shadow: 0 10px 30px rgba(0, 49, 83, 0.08); }

        .countdown-header { background: linear-gradient(135deg, #003153 0%, #004c82 100%); color: #ffcc00; padding: 12px; font-weight: 800; font-size: 1.05rem; }

        .btn-vote { padding: 12px 25px; margin: 10px; border-radius: 50px; border: none; font-weight: bold; cursor: pointer; transition: 0.3s; font-size: 1rem; color: white; display: inline-flex; align-items: center; gap: 8px; }

        .btn-join { background: var(--ly-blue); } .btn-next { background: #777; }

        .btn-vote:disabled { background-color: #e0e0e0 !important; color: #a0a0a0 !important; }

        .btn-joined-locked { background: linear-gradient(135deg, #d4af37, #f9e27d, #b8860b); color: #003153 !important; }

        .photo-stack-container { position: relative; width: 220px; height: 220px; margin: 20px auto; cursor: pointer; }

        .photo-stack-item { position: absolute; top: 0; left: 0; width: 100%; height: 100%; box-shadow: 0 8px 20px rgba(0,0,0,0.35); border-radius: 8px; overflow: hidden; }

        .photo-stack-item img { width: 100%; height: 100%; object-fit: cover; }

        .photo-stack-item:nth-child(1) { z-index: 5; transform: rotate(-3deg); } .photo-stack-item:nth-child(2) { z-index: 4; transform: rotate(4deg) translate(4px, 4px); } .photo-stack-item:nth-child(3) { z-index: 3; transform: rotate(-5deg) translate(-4px, 6px); }

        .voter-grid { display: flex; flex-wrap: wrap; gap: 15px; margin-top: 15px; justify-content: center; }

        .voter-box { text-align: center; width: 60px; }

        .voter-box img { width: 50px; height: 50px; border-radius: 50%; border: 2px solid var(--ly-gold); object-fit: cover; }

        .voter-box span { display: block; font-size: 0.75rem; margin-top: 5px; font-weight: bold; color: #444; }



        .slider-container { flex-grow: 1; width: 100%; overflow: hidden; display: flex; align-items: center; position: relative; cursor: grab; }

        .slider-container:active { cursor: grabbing; }

        .slider-track { display: flex; height: 80vh; transition: transform 0.3s ease-out; will-change: transform; }

        .slider-slide { min-width: 100vw; height: 100%; display: flex; justify-content: center; align-items: center; padding: 0 10px; box-sizing: border-box; }

        .slider-slide img { max-width: 100%; max-height: 100%; object-fit: contain; pointer-events: auto; user-select: none; -webkit-user-drag: none; box-shadow: 0 0 20px rgba(0,0,0,0.5); }

        .viewer-header { position: absolute; top: 0; left: 0; width: 100%; height: 80px; display: flex; justify-content: space-between; align-items: flex-start; padding: env(safe-area-inset-top, 20px) 20px 20px; z-index: 20010; box-sizing: border-box; pointer-events: none; transition: opacity 1.5s ease-in-out; opacity: 1; }

        .viewer-controls-group { display: flex; align-items: center; gap: 12px; pointer-events: auto; }

        .viewer-btn-text { background: rgba(255, 255, 255, 0.15); border: 1px solid rgba(255, 255, 255, 0.3); color: white; padding: 6px 12px; border-radius: 6px; cursor: pointer; font-size: 0.85rem; transition: all 0.2s; white-space: nowrap; }

        .viewer-counter { color: rgba(255, 255, 255, 0.8); font-size: 1rem; font-family: sans-serif; background: rgba(0,0,0,0.3); padding: 5px 12px; border-radius: 15px; pointer-events: auto; margin-top: 5px; }

        .viewer-close { background: rgba(255,255,255,0.15); border: 1px solid rgba(255,255,255,0.2); color: white; width: 44px; height: 44px; border-radius: 50%; font-size: 1.8rem; cursor: pointer; display: flex; align-items: center; justify-content: center; pointer-events: auto; }

        .viewer-hint { position: absolute; bottom: 30px; width: 100%; text-align: center; color: rgba(255,255,255,0.5); font-size: 0.9rem; pointer-events: none; transition: opacity 1.5s ease-in-out; opacity: 1; z-index: 20010; }

        .viewer-thumbnails { position: absolute; bottom: 70px; left: 0; width: 100%; height: 60px; display: flex; justify-content: center; align-items: center; gap: 12px; padding: 0 20px; box-sizing: border-box; overflow-x: auto; overflow-y: hidden; z-index: 20010; transition: opacity 1.5s ease-in-out; opacity: 1; pointer-events: auto; -webkit-overflow-scrolling: touch; scrollbar-width: none; }

        .viewer-thumbnails::-webkit-scrollbar { display: none; }

        .thumbnail-item { width: 40px; height: 40px; flex-shrink: 0; border-radius: 6px; overflow: hidden; opacity: 0.4; border: 2px solid transparent; cursor: pointer; transition: all 0.2s ease; transform: scale(0.9); }

        .thumbnail-item img { width: 100%; height: 100%; object-fit: cover; }

        .thumbnail-item.active { opacity: 1; border-color: var(--ly-gold); transform: scale(1.15); box-shadow: 0 4px 10px rgba(0,0,0,0.5); }

        .user-idle .viewer-header, .user-idle .viewer-hint, .user-idle .viewer-thumbnails { opacity: 0; pointer-events: none; }

        

        #settingsModal .modal-card { max-width: 100% !important; width: 100% !important; height: 100% !important; margin: 0 !important; border-radius: 0 !important; display: flex; flex-direction: column; }

        #settingsModal .modal-header { padding-top: env(safe-area-inset-top, 20px); height: calc(60px + env(safe-area-inset-top, 20px)); display: flex; align-items: center; justify-content: space-between; background: var(--ly-blue); color: white; }

    

        /* --- 載入中「立」字旋轉動畫 --- */

        @keyframes spin-char {

            0% { transform: rotate(0deg); }

            100% { transform: rotate(360deg); }

        }

        .spin-logo-char {

            font-size: 45px;

            color: var(--ly-blue);

            font-weight: 900;

            margin-bottom: 20px;

            animation: spin-char 1.5s linear infinite;

            display: inline-block; /* 必須加上這個，transform 旋轉才會生效 */

        }



        /* --- 團購系統專用樣式 --- */

        .gb-item-row { display: flex; gap: 8px; margin-bottom: 12px; align-items: center; background: #f9f9f9; padding: 10px; border-radius: 8px; border: 1px solid #eee; flex-wrap: wrap; position: relative;}

.gb-item-row input { flex: 1; min-width: 70px; padding: 10px 8px; border: 1px solid #ddd; border-radius: 6px; font-size: 16px; appearance: none;}        .gb-item-row input:focus { border-color: var(--ly-gold); outline: none; }

        .img-upload-btn { background: var(--ly-blue); color: white; padding: 10px; border-radius: 6px; cursor: pointer; white-space: nowrap; font-size: 13px; font-weight: bold;}

        .gb-item-row img { width: 40px; height: 40px; object-fit: cover; border-radius: 6px; display: none; border: 1px solid #ccc;}

        .btn-add-item { background: rgba(0, 49, 83, 0.1); color: var(--ly-blue); border: none; padding: 10px 20px; border-radius: 20px; cursor: pointer; font-size: 14px; font-weight: bold; margin-bottom: 15px; display: block; width: 100%; transition: 0.2s;}

        .btn-add-item:hover { background: rgba(0, 49, 83, 0.2); }

        .btn-remove-item { position: absolute; right: -5px; top: -5px; background: #ff4d4d; color: white; border-radius: 50%; width: 22px; height: 22px; display: flex; align-items: center; justify-content: center; font-size: 10px; cursor: pointer; z-index: 5;}

        

        .gb-order-item { display: flex; justify-content: space-between; align-items: center; padding: 15px 10px; border-bottom: 1px solid #eee; }

        .gb-order-item img { width: 55px; height: 55px; border-radius: 8px; object-fit: cover; cursor: pointer; border: 1px solid #ddd;}

        .gb-order-info { flex: 1; padding: 0 15px; }

        .gb-order-info h4 { margin: 0 0 5px 0; font-size: 1rem; color: var(--ly-blue); font-weight: 800;}

        .gb-order-info p { margin: 0; font-size: 0.9rem; color: #555; font-weight: 600;}

        .gb-qty-controls { display: flex; align-items: center; gap: 12px; }

        .gb-qty-btn { width: 32px; height: 32px; border-radius: 50%; border: 1.5px solid var(--ly-blue); background: white; color: var(--ly-blue); font-weight: bold; cursor: pointer; display: flex; justify-content: center; align-items: center; font-size: 18px; transition: 0.2s;}

        .gb-qty-btn:active { background: var(--ly-blue); color: white; }

        .gb-qty-btn:disabled { border-color: #ccc; color: #ccc; cursor: not-allowed; background: #f9f9f9;}



        /* --- 團購訂單分支明細與備註樣式 --- */

        .gb-buyers-wrapper { background: #f8f9fa; border-radius: 8px; margin-top: 12px; border-left: 3px solid var(--ly-gold); overflow: hidden; }

        .gb-buyer-row { display: flex; justify-content: space-between; align-items: center; padding: 6px 12px; border-bottom: 1px dashed #eee; font-size: 0.85rem; color: #444; }

        .gb-buyer-row:last-child { border-bottom: none; }

        .gb-buyer-name { font-weight: 800; color: var(--ly-blue); width: 60px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;}

        .gb-buyer-note { flex: 1; color: #888; margin: 0 10px; text-align: left; font-size: 0.8rem; }

        .gb-buyer-qty { font-weight: 900; color: #d93025; white-space: nowrap; }

        

.gb-my-note-input { width: 100%; padding: 10px; margin-top: 12px; border: 1px solid #ddd; border-radius: 6px; font-size: 16px; background: #fff; box-sizing: border-box; display: none; appearance: none;}        .gb-my-note-input:focus { border-color: var(--ly-gold); outline: none; }



        /* --- 團購清單左滑刪除 --- */

        .swipe-container { position: relative; overflow: hidden; border-radius: 8px; width: 100%; }

        .swipe-content { position: relative; z-index: 2; transition: transform 0.3s ease; width: 100%; box-sizing: border-box; }

        .swipe-actions { position: absolute; top: 0; right: 0; height: 100%; width: 80px; display: flex; justify-content: flex-end; align-items: center; z-index: 1; background: #ff4d4d; border-radius: 8px; }

        .swipe-delete-btn { color: white; background: none; border: none; font-size: 1.4rem; width: 100%; height: 100%; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: background 0.2s;}

        .swipe-delete-btn:active { background: #cc0000; }

        

        /* --- 活動清單左滑雙按鈕 (編輯/刪除) --- */

        .swipe-actions-dual { position: absolute; top: 0; right: 0; height: 100%; width: 150px; display: flex; z-index: 1; border-radius: 8px; overflow: hidden; }

        .swipe-edit-btn { background: var(--ly-blue); color: white; border: none; font-size: 1.3rem; width: 50%; height: 100%; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: 0.2s; }

        .swipe-delete-btn-half { background: #ff4d4d; color: white; border: none; font-size: 1.3rem; width: 50%; height: 100%; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: 0.2s; }

    </style>

    </style>

</head>

<body>

    <div id="appInitLoader" style="position: fixed; top: 0; left: 0; width: 100vw; height: 100dvh; background: var(--bg-gray); z-index: 99999; display: flex; flex-direction: column; justify-content: center; align-items: center; transition: opacity 0.3s ease;">

        <div class="spin-logo-char">立</div>

        <div style="color: var(--ly-blue); font-weight: 800; font-size: 1.1rem; letter-spacing: 1px;">資料同步中...</div>

    </div>



    <div id="messengerView" class="app-view">

        <header class="messenger-header">

            <div class="header-content">

                <div class="user-profile">

                    <img id="myAvatar" class="user-avatar" src="https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true" alt="Profile" onclick="openSettings()" style="cursor: pointer; width: 60px; height: 60px;">

                    <span id="myName" class="user-name">載入中...</span>

                </div>

                <div class="header-actions">

                    <button title="新增對話" onclick="startNewChat()"><i class="fas fa-plus"></i></button>

                    <button title="建立群組" onclick="createGroup()"><i class="fas fa-users"></i></button>

                </div>

            </div>

        </header>



        <main class="chat-list" id="chatList"></main>



        <button class="fab" onclick="startNewChat()" title="快速發起聊天">

            <i class="fas fa-comment-dots"></i>

        </button>

    </div>



    <div id="chatView" class="app-view">

        <header class="messenger-header">

            <div class="header-content" style="padding: 15px 20px;">

                <div class="user-profile">

                    <button class="back-btn" onclick="closeChatRoom()">

                        <i class="fas fa-chevron-left"></i>

                    </button>

                    <img id="targetAvatar" class="user-avatar" src="https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true">

                    <span id="chatTitle" class="user-name">載入中...</span>

                </div>

                <div class="header-actions" id="groupSettingsContainer" style="display: none;">

                    <button onclick="openGroupSettings()" title="群組設定">

                        <i class="fas fa-cog"></i>

                    </button>

                </div>

            </div>

        </header>



        <div id="pinnedMessageBar" class="pinned-bar" style="display: none;">

            <i class="fas fa-thumbtack pinned-icon"></i>

            <div id="pinnedText" class="pinned-content"></div>

            <button id="unpinBtn" class="unpin-btn" onclick="unpinMessage()" title="移除釘選"><i class="fas fa-times"></i></button>

        </div>



        <div class="messages" id="messageList"></div>

        

        <button class="scroll-bottom-btn" id="scrollBottomBtn" onclick="scrollToBottom(true)">

            <i class="fas fa-arrow-down"></i> 最新訊息

        </button>



        <div class="input-area">

            <div id="expandMenuPanel" class="expand-menu-panel">

                <div class="expand-menu-item" onclick="handleActivitySchedule()">

                    <div class="icon-circle"><i class="fas fa-calendar-alt"></i></div>

                    <span>活動行程</span>

                </div>

                <div class="expand-menu-item" onclick="openGroupBuyListModal()">

                    <div class="icon-circle"><i class="fas fa-shopping-basket"></i></div>

                    <span>團購功能</span>

                </div>

            </div>

            

            <div id="mentionList" class="mention-list"></div>



            <div id="replyPreview" class="reply-preview-bar" style="max-width: var(--container-width); width: 100%; box-sizing: border-box;">

                <span>回覆：<span id="replyPreviewText" style="font-weight: bold;"></span></span>

                <i class="fas fa-times close-reply" onclick="cancelReply()"></i>

            </div>

            

            <div class="input-container">

                <button class="expand-menu-btn" onclick="toggleExpandMenu(event)"><i class="fas fa-plus"></i></button>

                <input type="file" id="fileInput" accept="image/*,video/*" style="display: none;">

                <button class="attach-btn" onclick="document.getElementById('fileInput').click()"><i class="fas fa-paperclip"></i></button>

                <input type="text" id="msgInput" placeholder="請輸入訊息..." autocomplete="off">

                <button class="send-btn" onclick="sendMessage()"><i class="fas fa-paper-plane"></i></button>

            </div>

        </div>



        <div id="selectActionBar" style="display: none; flex-shrink: 0; width: 100%; height: auto; min-height: 80px; background: var(--bg-gray); box-shadow: 0 -5px 15px rgba(0,0,0,0.1); z-index: 150; align-items: center; justify-content: space-between; padding: 15px 20px; padding-bottom: calc(15px + env(safe-area-inset-bottom)); border-top: 1px solid #dee2e6; box-sizing: border-box;">

            <button onclick="cancelSelectMode()" style="background: none; border: none; font-size: 1rem; color: #666; font-weight: bold; cursor: pointer; padding: 10px;">取消</button>

            <span id="selectCountText" style="font-weight: 800; color: var(--ly-blue); font-size: 1.1rem;">已選擇 0 項</span>

            <button onclick="confirmMultiRevoke()" style="background: #ff4d4d; border: none; color: white; padding: 10px 20px; border-radius: 25px; font-weight: bold; cursor: pointer; font-size: 1rem; box-shadow: 0 4px 10px rgba(255, 77, 77, 0.3);"><i class="fas fa-trash-alt"></i> 收回</button>

        </div>



    </div>



    <div id="newChatModal" class="modal-overlay">

        <div class="modal-card">

            <div class="modal-header">

                <h3><i class="fas fa-comment-medical"></i> 發起新對話</h3>

                <span style="cursor:pointer;" onclick="closeModal('newChatModal')">&times;</span>

            </div>

            <div class="modal-body">

                <div class="modal-input-group">

                    <label>選擇對象</label>

                    <div id="singleUserList" class="user-select-list"></div>

                </div>

                <div class="modal-input-group">

                    <input type="text" id="targetUserName" placeholder="手動輸入或點擊上方選擇...">

                </div>

            </div>

            <div class="modal-footer">

                <button class="btn-modal btn-cancel" onclick="closeModal('newChatModal')">取消</button>

                <button class="btn-modal btn-confirm" onclick="handleStartChat()">確認開啟</button>

            </div>

        </div>

    </div>



    <div id="createGroupModal" class="modal-overlay">

        <div class="modal-card">

            <div class="modal-header">

                <h3><i class="fas fa-users"></i> 建立群組聊天</h3>

                <span style="cursor:pointer;" onclick="closeModal('createGroupModal')">&times;</span>

            </div>

            <div class="modal-body">

                <div class="modal-input-group" style="text-align: center;">

                    <input type="file" id="createGroupAvatarInput" accept="image/*" style="display: none;" onchange="openCropper(event, 'createGroupAvatarPreview')">

                    <div class="avatar-edit-wrapper" onclick="document.getElementById('createGroupAvatarInput').click()">

                        <img id="createGroupAvatarPreview" class="group-avatar-preview" src="https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true">

                        <div class="avatar-edit-icon"><i class="fas fa-pencil-alt"></i></div>

                    </div>

                </div>

                <div class="modal-input-group">

                    <input type="text" id="groupNameInput" placeholder="請輸入群組名稱...">

                </div>

                <div class="modal-input-group">

                    <div id="groupUserList" class="user-select-list"></div>

                </div>

            </div>

            <div class="modal-footer">

                <button class="btn-modal btn-cancel" onclick="closeModal('createGroupModal')">取消</button>

                <button class="btn-modal btn-confirm" onclick="handleCreateGroup()">建立群組</button>

            </div>

        </div>

    </div>



    <div id="groupSettingsModal" class="modal-overlay" onclick="if(event.target === this && window.innerWidth <= 750) closeModal('groupSettingsModal')">

        <div class="modal-card">

            <div class="modal-header">

                <h3><i class="fas fa-cog"></i> 群組設定</h3>

                <span style="cursor:pointer;" onclick="closeModal('groupSettingsModal')">&times;</span>

            </div>

            <div class="modal-body" style="text-align: center;">

                <div class="modal-input-group" style="text-align: center;">

                    <input type="file" id="editGroupAvatarInput" accept="image/*" style="display: none;" onchange="openCropper(event, 'editGroupAvatarPreview')">

                    <div class="avatar-edit-wrapper" onclick="document.getElementById('editGroupAvatarInput').click()">

                        <img id="editGroupAvatarPreview" class="group-avatar-preview" src="https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true">

                        <div class="avatar-edit-icon"><i class="fas fa-pencil-alt"></i></div>

                    </div>

                </div>

                <div class="modal-input-group" style="text-align: left;">

                    <label>群組名稱</label>

                    <input type="text" id="groupNameEdit" oninput="markAsUnsaved()">

                </div>

                <div class="modal-input-group" style="text-align: left; margin-bottom: 0;">

                    <label>群組成員</label>

                    <div id="groupMembersList" class="group-members-list"></div>

                    <button id="editMembersBtn" class="btn-modal btn-cancel" style="width: 100%; margin-top: 10px; display: none;" onclick="openEditMembersModal()">

                        <i class="fas fa-user-edit"></i> 編輯群組成員

                    </button>

                </div>

            </div>

            <div class="modal-footer" style="flex-direction: column; align-items: center; gap: 15px; padding-top: 20px;">

                <button id="saveGroupBtn" class="btn-modal btn-confirm" style="display: none; width: 100%;" onclick="saveGroupSettings()">儲存</button>

                <div id="moreOptionsToggle" style="color: #999; font-size: 12px; cursor: pointer; padding: 5px;" onclick="toggleGroupOptions()">

                    <i class="fas fa-chevron-down"></i> 更多選項

                </div>

                <div id="moreOptionsContainer" style="display: none; width: 100%; flex-direction: column; gap: 10px;">

                    <button class="btn-modal btn-cancel" style="width: 100%;" onclick="closeModal('groupSettingsModal')">取消</button>

                    <button id="deleteGroupBtn" class="btn-modal btn-danger" style="width: 100%;" onclick="deleteGroup()">刪除群組</button>

                </div>

            </div>

        </div>

    </div>



    <div id="editMembersModal" class="modal-overlay" style="z-index: 4000;">

        <div class="modal-card" style="max-width: 450px;">

            <div class="modal-header">

                <h3><i class="fas fa-users-cog"></i> 編輯群組成員</h3>

                <span style="cursor:pointer;" onclick="closeModal('editMembersModal')">&times;</span>

            </div>

            <div class="modal-body" style="padding: 15px;">

                <div id="editMembersList" class="user-select-list"></div>

            </div>

            <div class="modal-footer">

                <button class="btn-modal btn-cancel" onclick="closeModal('editMembersModal')">取消</button>

                <button class="btn-modal btn-confirm" onclick="saveGroupMembers()">儲存成員</button>

            </div>

        </div>

    </div>



    <div id="cropperModal" class="modal-overlay" style="z-index: 5000;">

        <div class="modal-card" style="max-width: 450px;">

            <div class="modal-header">

                <h3><i class="fas fa-crop-alt"></i> 調整圖片</h3>

                <span style="cursor:pointer;" onclick="closeCropper()">&times;</span>

            </div>

            <div class="modal-body" style="padding: 15px; background: #111; display: flex; align-items: center; gap: 15px;">

                <div style="height: 300px; flex: 1; overflow: hidden; background: #000; border-radius: 8px;">

                    <img id="cropperImage" src="" style="max-width: 100%; display: block;">

                </div>

                <div class="zoom-slider-container">

                    <i class="fas fa-search-plus" style="color:white; margin-bottom:5px;"></i>

                    <input type="range" id="cropperZoomSlider" min="0.1" max="3" step="0.01" value="1" orient="vertical" oninput="changeCropperZoom(this.value)">

                    <i class="fas fa-search-minus" style="color:white; margin-top:5px;"></i>

                </div>

            </div>

            <div class="modal-footer">

                <button class="btn-modal btn-cancel" onclick="closeCropper()">取消</button>

                <button class="btn-modal btn-confirm" onclick="confirmCrop()">確認</button>

            </div>

        </div>

    </div>



    <div id="activityScheduleModal" class="modal-overlay" style="z-index: 4000;">

        <div class="modal-card" style="max-width: 450px !important; height: 80vh; max-height: 600px; display: flex; flex-direction: column; background: #f8f9fa;">

            <div class="modal-header">

                <h3><i class="fas fa-calendar-check"></i> 活動行程總覽</h3>

                <span style="cursor:pointer;" onclick="closeModal('activityScheduleModal')">&times;</span>

            </div>

            <div id="activityScheduleList" class="event-mini-list" style="padding: 15px;"></div>

            <div class="modal-footer" style="justify-content: space-between; background: white; border-top: 1px solid #eee;">

                <button class="btn-modal btn-cancel" onclick="closeModal('activityScheduleModal')">關閉</button>

                <button class="btn-modal btn-confirm" onclick="openCreateEventModal()"><i class="fas fa-plus"></i> 新增活動</button>

            </div>

        </div>

    </div> 



    <div id="createEventModal" class="modal-overlay" style="z-index: 5000;">

        <div class="modal-card" style="max-width: 500px; max-height: 85dvh; display: flex; flex-direction: column;">

            <div class="modal-header">

                <h3><i class="fas fa-calendar-plus"></i> 發起新活動</h3>

                <span style="cursor:pointer;" onclick="closeModal('createEventModal')">&times;</span>

            </div>

            <div class="modal-body" style="background: white;">

                

                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; padding: 10px; background: #fff5f5; border-radius: 8px; border: 1px dashed #d93025;">

                    <label style="font-weight: bold; color: #d93025; margin: 0;"><i class="fas fa-user-secret"></i> 設為私人活動 (僅限授權可見)</label>

                    <input type="checkbox" id="ev-is-private" onchange="toggleEventPrivateMode(this.checked)" style="width: 20px; height: 20px; accent-color: #d93025;">

                </div>



                <div class="modal-input-group">

                    <label>① 活動標題 <small style="color:red;">(必填)</small></label>

                    <input type="text" id="ev-title" class="ev-dynamic-input" placeholder="請輸入標題">

                </div>



                <div id="dynamic-fields-container" style="width: 100%;"></div>



                <div class="field-adder-bar">

                    <button class="tag-btn" onclick="addEventField('headline', '標題')">+ 標題</button>

                    <button class="tag-btn" onclick="addEventField('location', '地點')">+ 地點</button>

                    <button class="tag-btn" onclick="addEventField('meetup', '集合時間')">+ 集合時間</button>

                    <button class="tag-btn" onclick="addEventField('time', '活動時間')">+ 活動時間</button>

                    <button class="tag-btn" onclick="addEventField('content_text', '活動內容')">+ 內容</button>

                    <button class="tag-btn tag-btn-img" onclick="addEventField('image_upload', '活動照片')">

                        <i class="fas fa-image"></i> + 照片

                    </button>

                </div>



                <div id="public-participants-section" style="margin-top: 20px;">

                    <label style="font-weight: bold; color: var(--ly-blue); display: block; margin-bottom: 8px;">參加成員名單</label>

                    <div id="selected-members-preview" class="member-preview-list">

                        <p style="color: #999; font-size: 0.9em; margin:0;">尚未選擇成員</p>

                    </div>

                    <div style="text-align: center; margin-top: 10px;">

                        <button class="btn-modal btn-cancel" onclick="openEventMemberPicker('participants')" style="width: 100%;">

                            <i class="fas fa-users-cog"></i> 編輯參加名單

                        </button>

                    </div>

                </div>



                <div id="private-auth-section" style="display: none; background: #fff5f5; padding: 15px; border-radius: 8px; border: 1px dashed #d93025; margin-top: 20px;">

                    <label style="font-weight: bold; color: #d93025; display: block; margin-bottom: 8px;">授權檢視成員 (私人模式)</label>

                    <div id="authorized-members-preview" class="member-preview-list" style="background: white; border-color: #ffcccb;">

                        <p style="color: #999; font-size: 0.9em; margin:0;">尚未授權任何成員</p>

                    </div>

                    <div style="text-align: center; margin-top: 10px;">

                        <button class="btn-modal btn-cancel" onclick="openEventMemberPicker('authorized')" style="width: 100%; color: #d93025; border-color: #d93025;">

                            <i class="fas fa-user-plus"></i> 編輯授權名單

                        </button>

                    </div>

                </div> 



                <div class="modal-input-group" style="margin-top: 20px;">

                    <label>投票/報名 截止時間</label>

                    <input type="datetime-local" id="ev-deadline" class="ev-dynamic-input">

                </div>

            </div>

            <div class="modal-footer" style="justify-content: space-between;">

                <button class="btn-modal btn-cancel" onclick="closeModal('createEventModal')">取消</button>

                <button class="btn-modal btn-confirm" id="btnSubmitEvent" onclick="submitNewEvent()">確認發布</button>

            </div>

        </div>

    </div>



    <div id="eventMemberPickerModal" class="modal-overlay" style="z-index: 6000;">

        <div class="modal-card" style="max-width: 450px; max-height: 80dvh; display: flex; flex-direction: column;">

            <div class="modal-header">

                <h3 id="evPickerTitle"><i class="fas fa-users"></i> 選擇成員</h3>

                <span style="cursor:pointer;" onclick="closeModal('eventMemberPickerModal')">&times;</span>

            </div>

            <div style="padding: 15px; background: white; border-bottom: 1px solid #eee; flex-shrink: 0;">

                <div style="display:flex; flex-wrap: wrap; gap: 8px; align-items: center;">

                    <button class="tag-btn" onclick="toggleAllEventMembers(true)" style="border-color: #28a745; color: #28a745;">全選</button>

                    <button class="tag-btn" onclick="toggleAllEventMembers(false)" style="border-color: #d93025; color: #d93025;">移除</button>

                    <div class="picker-search-wrapper">

                        <i class="fas fa-search"></i>

                        <input type="text" id="ev-member-search" placeholder="搜尋..." oninput="searchEventMember()">

                    </div>

                </div>

            </div>

            <div class="modal-body" style="background: #f8f9fa;">

                <div id="ev-picker-grid" class="user-select-list"></div>

            </div>

            <div class="modal-footer" style="justify-content: center;">

                <button class="btn-modal btn-confirm" style="width: 100%;" onclick="saveEventSelectedMembers()">確認勾選</button>

            </div>

        </div> 

    </div>

    

    <div id="groupBuyListModal" class="modal-overlay" style="z-index: 4000;">

        <div class="modal-card" style="max-width: 450px !important; height: 80vh; max-height: 600px; display: flex; flex-direction: column; background: #f8f9fa;">

            <div class="modal-header">

                <h3><i class="fas fa-shopping-basket"></i> 團購清單</h3>

                <span style="cursor:pointer;" onclick="closeModal('groupBuyListModal')">&times;</span>

            </div>

            <div id="groupBuyListContainer" class="event-mini-list" style="padding: 15px;"></div>

            

            <div class="modal-footer" style="justify-content: space-between; background: white; border-top: 1px solid #eee;">

                <button class="btn-modal btn-cancel" onclick="closeModal('groupBuyListModal')">關閉</button>

                <button class="btn-modal btn-confirm" onclick="openCreateGroupBuyModal()"><i class="fas fa-plus"></i> 新增團購</button>

            </div>

        </div>

    </div>



    <div id="createGroupBuyModal" class="modal-overlay" style="z-index: 4000;">

        <div class="modal-card" style="max-width: 500px; max-height: 85vh; display: flex; flex-direction: column;">

            <div class="modal-header">

                <h3><i class="fas fa-shopping-cart"></i> 發起團購訂單</h3>

                <span style="cursor:pointer;" onclick="closeModal('createGroupBuyModal')">&times;</span>

            </div>

            <div class="modal-body" style="flex: 1; overflow-y: auto; padding: 20px;">

                <div class="modal-input-group">

                    <label>代購成員 (發起人)</label>

                    <input type="text" id="gbInitiator" disabled style="background:#f8f9fa; font-weight:bold; color:var(--ly-blue); border:none;">

                </div>

                <div class="modal-input-group">

                    <label>團購標題</label>

                    <input type="text" id="gbTitle" placeholder="例如：LOCH 卡包...">

                </div>

                <div class="modal-input-group">

                    <label>截止時間</label>

                    <input type="datetime-local" id="gbDeadline">

                </div>

                

                <div style="height: 1px; background: #eee; margin: 20px 0;"></div>

                <label style="display:block; font-size: 14px; color: var(--ly-blue); font-weight: 800; margin-bottom: 12px;"><i class="fas fa-box-open"></i> 商品列表</label>

                

                <div id="gbItemsContainer"></div>

                <button class="btn-add-item" onclick="addGroupBuyItemRow()"><i class="fas fa-plus"></i> 增加一項商品</button>

            </div>

            <div class="modal-footer">

                <button class="btn-modal btn-cancel" onclick="closeModal('createGroupBuyModal')">取消</button>

                <button class="btn-modal btn-confirm" id="gbSubmitBtn" onclick="submitGroupBuy()">發布到聊天室</button>

            </div>

        </div>

    </div>



    <div id="viewGroupBuyModal" class="modal-overlay" style="z-index: 4000;">

        <div class="modal-card" style="max-width: 500px; max-height: 85vh; display: flex; flex-direction: column;">

            <div class="modal-header">

                <h3 id="vgbTitle" style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 80%;">團購詳情</h3>

                <span style="cursor:pointer;" onclick="closeModal('viewGroupBuyModal')">&times;</span>

            </div>

            <div style="padding: 15px 20px; background: #fffdf5; border-bottom: 2px solid #eee; font-size: 0.95rem; color: #555;">

                <div style="margin-bottom: 5px;"><strong><i class="fas fa-user-tag" style="color:var(--ly-gold);"></i> 代購成員：</strong> <span id="vgbInitiator" style="font-weight:bold; color:var(--ly-blue);"></span></div>

                <div><strong><i class="far fa-clock" style="color:var(--ly-gold);"></i> 截止時間：</strong> <span id="vgbDeadline" style="font-weight:bold; color:#d93025;"></span></div>

            </div>

            <div class="modal-body" id="vgbItemsList" style="flex: 1; overflow-y: auto; padding: 10px 20px;">

                </div>

            <div class="modal-footer" style="background: white; border-top: 1px solid #eee;">

                <button class="btn-modal btn-cancel" onclick="closeModal('viewGroupBuyModal')">關閉</button>

                <button class="btn-modal btn-confirm" id="vgbSaveBtn" onclick="saveGroupBuyOrder()">儲存我的訂單</button>

            </div>

        </div>

    </div>



    <div id="readReceiptModal" class="modal-overlay" style="z-index: 6000;">

        <div class="modal-card" style="max-width: 350px;">

            <div class="modal-header">

                <h3><i class="fas fa-eye"></i> 已讀成員</h3>

                <span style="cursor:pointer;" onclick="closeModal('readReceiptModal')">&times;</span>

            </div>

            <div class="modal-body" style="padding: 15px;">

                <div id="readReceiptList" class="group-members-list" style="max-height: 300px;"></div>

            </div>

            <div class="modal-footer" style="justify-content: center; background: white;">

                <button class="btn-modal btn-cancel" style="width: 100%;" onclick="closeModal('readReceiptModal')">關閉</button>

            </div>

        </div>

    </div>



    <div id="pinnedMessageModal" class="modal-overlay" style="z-index: 6000;">

        <div class="modal-card" style="max-width: 350px;">

            <div class="modal-header">

                <h3><i class="fas fa-thumbtack" style="transform: rotate(-45deg);"></i> 釘選訊息內容</h3>

                <span style="cursor:pointer;" onclick="closeModal('pinnedMessageModal')">&times;</span>

            </div>

            <div class="modal-body" style="padding: 20px; background: #f8f9fa;">

                <div style="font-size: 0.85rem; color: #666; margin-bottom: 10px; display: flex; justify-content: space-between; border-bottom: 1px dashed #ddd; padding-bottom: 8px;" id="pinnedMsgSenderTime">

                    </div>

                <div style="font-size: 1.05rem; color: #333; line-height: 1.6; background: white; padding: 15px; border-radius: 12px; border: 1px solid #dee2e6; word-break: break-word; box-shadow: 0 2px 8px rgba(0,0,0,0.03);" id="pinnedMsgContent">

                    </div>

            </div>

            <div class="modal-footer" style="justify-content: center; background: white;">

                <button class="btn-modal btn-cancel" style="width: 100%;" onclick="closeModal('pinnedMessageModal')">關閉視窗</button>

            </div>

        </div>

    </div>



    <div id="contextMenu" class="context-menu">

        <button id="pinMenuBtn" onclick="pinMessage()"><i class="fas fa-thumbtack"></i> 釘選</button>

        <button id="replyMenuBtn" onclick="triggerMenuReply()"><i class="fas fa-reply"></i> 回覆</button>

        <button id="copyMenuBtn" onclick="copyMessage()"><i class="fas fa-copy"></i> 複製</button>

        <button id="multiSelectMenuBtn" onclick="enterSelectMode()"><i class="fas fa-trash-alt"></i> 收回</button>

    </div>



    <div id="settingsView" class="app-view" style="z-index: 30; transform: translateX(100%);">

        <header class="messenger-header">

            <div class="header-content" style="padding: 15px 20px;">

                <div class="user-profile">

                    <button class="back-btn" onclick="closeSettings()"><i class="fas fa-chevron-left"></i></button>

                    <span class="user-name">個人設定</span>

                </div>

            </div>

        </header>

        <div style="flex: 1; display: flex; flex-direction: column; align-items: center; padding: 40px 20px; background: white; overflow-y: auto;">

            <input type="file" id="settingsAvatarInput" accept="image/*" style="display: none;" onchange="handleSettingsAvatarChange(event)">

            

            <div class="avatar-edit-wrapper" onclick="document.getElementById('settingsAvatarInput').click()" style="margin-bottom: 20px;">

                <img id="settingsAvatar" src="" style="width: 120px; height: 120px; border-radius: 50%; object-fit: cover; border: 3px solid var(--ly-gold); transition: 0.2s;">

                <div class="avatar-edit-icon" style="width: 32px; height: 32px; font-size: 14px;"><i class="fas fa-camera"></i></div>

            </div>



            <h2 id="settingsName" style="color: var(--ly-blue); margin-bottom: 5px;">載入中...</h2>

            <p id="settingsEmail" style="color: #666; font-size: 1rem; margin-bottom: 5px;">載入中...</p>

            <p id="settingsUid" style="color: #aaa; font-size: 0.8rem; margin-bottom: 20px; font-family: monospace;">UID: ...</p>

            

            <div style="width: 100%; max-width: 300px; display: flex; gap: 10px; margin-bottom: 30px;">

                <button class="btn-modal btn-cancel" style="flex: 1; padding: 10px; border-radius: 8px; font-size: 0.9rem;" onclick="handleChangeEmail()"><i class="fas fa-envelope"></i> 修改信箱</button>

                <button class="btn-modal btn-cancel" style="flex: 1; padding: 10px; border-radius: 8px; font-size: 0.9rem;" onclick="handleChangePassword()"><i class="fas fa-key"></i> 修改密碼</button>

            </div>



            <button class="btn-modal btn-confirm" style="width: 100%; max-width: 300px; background: #ff4d4d; font-size: 1.1rem; padding: 15px; border-radius: 30px; box-shadow: 0 4px 15px rgba(255,77,77,0.3);" onclick="handleLogout()">

                <i class="fas fa-sign-out-alt"></i> 登出帳號

            </button>

        </div>

    </div>



    <div id="doc-preview-modal" class="doc-modal" onclick="if(event.target === this) closeModal('doc-preview-modal')">

        <article class="doc-paper">

            <button class="doc-close-btn" onclick="closeModal('doc-preview-modal')">&times;</button>

            <header class="doc-header"></header>

            <section id="doc-content" style="line-height: 1.8; color: #333;"></section>

            <section id="vote-area" class="vote-panel" style="display: none;"></section>

            <h3 class="doc-sub-title" style="font-size: 1rem; margin-top: 30px;">參與成員 <span id="participant-count-tag" style="font-size: 0.85rem; color: #666; font-weight: normal;"></span></h3>

            <div id="voter-list" class="voter-grid"></div>

            <div id="doc-creation-info" style="color: #aaa; text-align: center; font-size: 0.8rem; margin-top: 30px;"></div>

        </article>

    </div>



    <div id="imageModal" class="image-modal" onclick="if(event.target === this) closeModal('imageModal')">

        <span class="image-modal-close" onclick="closeModal('imageModal')">&times;</span>

        <img class="image-modal-content" id="expandedImg">

    </div>



    <div id="image-viewer-modal" onclick="handleViewerClick(event)" style="display: none; position: fixed; z-index: 20000; left: 0; top: 0; width: 100vw; height: 100dvh; background-color: rgba(0, 0, 0, 0.95); flex-direction: column; overflow: hidden;">

        <div class="viewer-header">

            <div class="viewer-counter" id="viewer-counter">1 / 1</div>

            <div class="viewer-controls-group">

                <button class="viewer-btn-text" onclick="event.stopPropagation(); downloadCurrentImage()"><i class="fas fa-download"></i> 下載此張</button>

                <button class="viewer-btn-text" onclick="event.stopPropagation(); downloadAllImages()"><i class="fas fa-images"></i> 下載全部</button>

                <button class="viewer-close" onclick="event.stopPropagation(); closeModal('image-viewer-modal'); detachSliderEvents();">&times;</button>

            </div>

        </div>

        <div class="slider-container" id="slider-container"><div class="slider-track" id="slider-track"></div></div>

        <div class="viewer-thumbnails" id="viewer-thumbnails" onclick="event.stopPropagation()"></div>

        <div class="viewer-hint"><i class="fas fa-hand-pointer"></i> 左右拖曳切換</div>

    </div>



    <script src="https://www.gstatic.com/firebasejs/10.11.0/firebase-app-compat.js"></script>

    <script src="https://www.gstatic.com/firebasejs/10.11.0/firebase-auth-compat.js"></script>

    <script src="https://www.gstatic.com/firebasejs/10.11.0/firebase-firestore-compat.js"></script>

    <script src="https://www.gstatic.com/firebasejs/10.11.0/firebase-storage-compat.js"></script>

    <script src="https://unpkg.com/@ffmpeg/ffmpeg@0.11.6/dist/ffmpeg.min.js"></script>

    <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js"></script>



    <script>

    // ==========================================

    // 1. Firebase 與全域變數初始化

    // ==========================================

    const firebaseConfig = {

        apiKey: "AIzaSyACnoimIASfb1rb59SbgLDkUmyYR6ODbUU",

        authDomain: "llwb-ed686.firebaseapp.com",

        projectId: "llwb-ed686",

        storageBucket: "llwb-ed686.firebasestorage.app",

        messagingSenderId: "940345852074",

        appId: "1:940345852074:web:7a30cca5a6d997a92350d3"

    };

    firebase.initializeApp(firebaseConfig);

    const auth = firebase.auth();

    const db = firebase.firestore();

    const storage = firebase.storage();

    const MAX_FILE_SIZE = 10 * 1024 * 1024;



    db.settings({ experimentalForceLongPolling: true, useFetchStreams: false });



    // 全域快取與狀態

    const userCache = {};

    let currentChatId = null;

    let chatSnapshotUnsubscribe = null;

    let messagesSnapshotUnsubscribe = null;

    let opponentInChat = false;

    let opponentReadTime = new Date(0);

    window.isCurrentChatGroup = false;

    window.currentGroupMembers = [];

    window.groupReadTimes = {};

    window.chatMembersCache = {};

    let messagesDocsData = []; // 存儲當前聊天訊息

    let currentMessageLimit = 5; // 【新增】：初始只載入最新 5 則訊息

    let isFetchingOlder = false; // 【新增】：標記是否正在向上載入舊訊息

    let currentSubscribeMessages = null; // 【新增】：儲存重新監聽的函式



    // 工具函式：字串中段省略 (Middle Ellipsis)

    function formatMiddleEllipsis(str, maxLength = 14) {

        if (!str || str.length <= maxLength) return str;

        // 以 "立法院 Legislature" (15字) 為例：

        // 前面取 8 個字 ("立法院 Legi")，後面取 4 個字 ("ture")，中間加上 "…"

        const front = str.substring(0, 8);

        const back = str.substring(str.length - 4);

        return front + '…' + back;

    }



    // 工具函式

    function escapeHtml(str) { return str ? str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "&#039;") : ""; }

    function linkify(text) { return text ? text.replace(/(\b(https?|ftp|file):\/\/[-A-Z0-9+&@#\/%?=~_|!:,.;]*[-A-Z0-9+&@#\/%=~_|])/ig, '<a href="$1" target="_blank" style="color: inherit; text-decoration: underline; word-break: break-all;">$1</a>') : ""; }

    function highlightMentions(text) { return text ? text.replace(/@([^\s]+)/g, '<span class="mention-highlight">@$1</span>') : ""; }

    function formatChatDate(timestamp) {

        if (!timestamp) return "";

        const date = timestamp.toDate(), now = new Date();

        return date.toDateString() === now.toDateString() ? date.toLocaleTimeString('zh-TW', { hour: '2-digit', minute: '2-digit', hour12: true }) : `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`;

    }



    // 計算卡片上的精簡版倒數時間

    function getCardCountdownText(deadlineStr) {

        if (!deadlineStr) return "";

        const now = new Date().getTime();

        const deadline = new Date(deadlineStr).getTime();

        const diff = deadline - now;

        if (diff <= 0) return "已截止";

        

        const days = Math.floor(diff / (1000 * 60 * 60 * 24));

        const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));

        const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));

        

        if (days >= 1) return `倒數 ${days} 天`;

        if (hours >= 1) return `倒數 ${hours} 小時`;

        return `倒數 ${minutes} 分鐘`;

    }



    // ==========================================

    // 活動卡片即時倒數系統

    // ==========================================

    // ==========================================

    // 活動卡片即時倒數系統 (修復初次載入延遲)

    // ==========================================

    // ==========================================

    // 活動卡片即時倒數系統 (終極修復：解決 Firebase 雙次渲染覆蓋問題)

    // ==========================================

    window.eventDeadlineCache = window.eventDeadlineCache || {}; 



    function updateDynamicCountdowns() {

        const countdownEls = document.querySelectorAll('.dynamic-countdown');

        if (countdownEls.length === 0) return;



        countdownEls.forEach((el) => {

            const eventId = el.getAttribute('data-event-id');

            // 【新增】：讀取所屬集合名稱 (預設為 events，若為團購則是 group-orders)

            const collectionName = el.getAttribute('data-collection') || 'events';

            if (!eventId) return;



            const cacheKey = collectionName + '_' + eventId;



            const applyCountdownUI = (deadlineStr) => {

                if (deadlineStr) {

                    const cdText = getCardCountdownText(deadlineStr);

                    el.innerText = cdText;

                    el.style.display = 'inline-block'; 

                    if (cdText === "已截止") el.classList.add('expired');

                    else el.classList.remove('expired');

                } else {

                    el.style.display = 'none';

                }

            };



            if (typeof window.eventDeadlineCache[cacheKey] === 'string') {

                applyCountdownUI(window.eventDeadlineCache[cacheKey]);

                return;

            }



            if (window.eventDeadlineCache[cacheKey] instanceof Promise) {

                window.eventDeadlineCache[cacheKey].then(applyCountdownUI);

                return;

            }



            window.eventDeadlineCache[cacheKey] = db.collection(collectionName).doc(eventId).get().then(evDoc => {

                let deadlineStr = (evDoc.exists && evDoc.data().deadline) ? evDoc.data().deadline : "";

                window.eventDeadlineCache[cacheKey] = deadlineStr;

                return deadlineStr;

            }).catch(e => {

                window.eventDeadlineCache[cacheKey] = "";

                return "";

            });



            window.eventDeadlineCache[cacheKey].then(applyCountdownUI);

        });

    }



    // 設定每 60 秒自動刷新一次畫面上的倒數時間

    if (!window.countdownTimerStarted) {

        window.countdownTimerStarted = true;

        setInterval(updateDynamicCountdowns, 60000); 

    }



    // Modal 控制

    function openModal(id) { document.getElementById(id).style.display = 'flex'; }

    function closeModal(id) { document.getElementById(id).style.display = 'none'; }

    function removeInitLoader() { const loader = document.getElementById('appInitLoader'); if(loader) { loader.style.opacity='0'; setTimeout(()=>loader.style.display='none', 300); } }



    // ==========================================

    // 2. SPA 切換與手勢控制

    // ==========================================

    const chatViewDOM = document.getElementById('chatView');

    // ==========================================

    // 2. SPA 切換與手勢控制 (包含速度判定)

    // ==========================================

    let isChatSwiping = false, chatPageStartX = 0, chatPageCurrentX = 0, chatPageStartTime = 0;



    function openChatRoom(chatId) {

        currentChatId = chatId;

        

        // 【新增】：在動畫開始前，立刻清空舊的變數與畫面，避免殘留

        messagesDocsData = []; 

        document.getElementById('messageList').innerHTML = "";

        

        chatViewDOM.style.transform = 'translateX(0%)';

        initChatData(chatId);

    }



    function closeChatRoom() {

        chatViewDOM.style.transform = 'translateX(100%)';

        

        // 取消 Firebase 監聽器

        if (chatSnapshotUnsubscribe) { chatSnapshotUnsubscribe(); chatSnapshotUnsubscribe = null; }

        if (messagesSnapshotUnsubscribe) { messagesSnapshotUnsubscribe(); messagesSnapshotUnsubscribe = null; }

        

        // 【新增】：徹底清除狀態與資料，確保下次進來一定是全新的狀態

        currentChatId = null;

        currentSubscribeMessages = null;

        messagesDocsData = []; 

        

        updateMyStatus(false);

    }



    chatViewDOM.addEventListener('touchstart', (e) => {

        // 觸發區域限制在螢幕最左側 40px 內

        if (e.touches[0].clientX < 40) { 

            isChatSwiping = true; 

            chatPageStartX = e.touches[0].clientX; 

            chatPageStartTime = Date.now(); // 紀錄開始時間

            chatViewDOM.style.transition = 'none'; 

        }

    });

    

    chatViewDOM.addEventListener('touchmove', (e) => {

        if (!isChatSwiping) return;

        chatPageCurrentX = e.touches[0].clientX - chatPageStartX;

        if (chatPageCurrentX > 0) { e.preventDefault(); chatViewDOM.style.transform = `translateX(${chatPageCurrentX}px)`; }

    }, { passive: false });

    

    chatViewDOM.addEventListener('touchend', (e) => {

        if (!isChatSwiping) return;

        isChatSwiping = false;

        chatViewDOM.style.transition = 'transform 0.3s cubic-bezier(0.25, 0.8, 0.25, 1)';

        

        const swipeTime = Date.now() - chatPageStartTime;

        const swipeSpeed = chatPageCurrentX / swipeTime; // 計算滑動速度 (px/ms)



        // 【修改】：放寬滑動速度門檻至 0.3 (輕輕一撇即觸發)，並增加「至少滑動 30px」防呆以免誤觸

        if (chatPageCurrentX > window.innerWidth / 3 || (swipeSpeed > 0.3 && chatPageCurrentX > 30)) {

            closeChatRoom();

        } else {

            chatViewDOM.style.transform = 'translateX(0%)';

        }

        chatPageCurrentX = 0;

    });



    // ==========================================

    // 3. Auth, 推播與 Messenger 列表載入

    // ==========================================

    async function initPushNotifications(userId) {

        if (typeof Capacitor === 'undefined' || !Capacitor.isNativePlatform()) return;



        try {

            // 【修改 1】：同時引入 PushNotifications 和 FCM

            const { PushNotifications, FCM } = Capacitor.Plugins;



            let permStatus = await PushNotifications.checkPermissions();

            if (permStatus.receive === 'prompt') {

                permStatus = await PushNotifications.requestPermissions();

            }

            if (permStatus.receive !== 'granted') return;



            await PushNotifications.removeAllListeners();



            // 【修改 2】：轉換 Token 並存入資料庫

            PushNotifications.addListener('registration', async (token) => {

                console.log("拿到 Apple 原始 Token:", token.value);

                

                try {

                    // 透過 FCM 套件，將 Apple Token 轉換為 Firebase 專用 Token

                    const fcmResult = await FCM.getToken();

                    console.log("成功轉換為 FCM Token:", fcmResult.token);

                    

                    // 確保你存入的是 fcmResult.token，而不是 token.value

                    await db.collection("act").doc(userId).set({ 

                        pushToken: fcmResult.token, 

                        platform: 'ios',

                        env: 'production',

                        lastUpdated: firebase.firestore.FieldValue.serverTimestamp()

                    }, { merge: true });

                    

                } catch (err) {

                    console.error("FCM Token 轉換失敗:", err);

                }

            });



            // --- 找到這段並替換 ---

            PushNotifications.addListener('pushNotificationReceived', (notification) => {

                

                // 【新增】：盤查邏輯。如果這則推播的 chatId，等於我目前正在看的 currentChatId，就直接略過！

                if (notification.data && notification.data.chatId === currentChatId) {

                    console.log("📍 已在當前聊天室，攔截並隱藏通知橫幅");

                    return; 

                }



                // 以下維持原樣：呼叫我們自訂的漂亮橫幅，並自動在 4 秒後收合

                const notifEl = document.getElementById('inAppNotification');

                document.getElementById('inAppNotifTitle').innerText = notification.title || '新訊息';

                document.getElementById('inAppNotifBody').innerText = notification.body || '';



                notifEl.style.display = 'flex';

                void notifEl.offsetWidth; // 觸發重繪

                notifEl.style.top = 'env(safe-area-inset-top, 20px)'; // 避開瀏海

                notifEl.style.opacity = '1';



                setTimeout(() => {

                    notifEl.style.top = '-100px';

                    notifEl.style.opacity = '0';

                    setTimeout(() => { notifEl.style.display = 'none'; }, 400);

                }, 4000);

            });

            PushNotifications.addListener('pushNotificationActionPerformed', (notification) => {

                console.log("使用者點開了通知:", notification);

            });



            await PushNotifications.register();



        } catch(e) { 

            console.error("推播初始化錯誤:", e); 

        }

    }



    auth.onAuthStateChanged(user => {

        if (user) {

            // --- 1. 抓取身分資料 (加上防呆檢查) ---

            db.collection("act").doc(user.uid).get().then(doc => {

                const nameEl = document.getElementById('myName');

                const avatarEl = document.getElementById('myAvatar');



                if (doc.exists) {

                    const data = doc.data();

                    // 只有真的有名字時才更新，否則顯示提示

                    if (nameEl) nameEl.innerText = data.displayName || "身分未設定";

                    if (avatarEl) avatarEl.src = data.photoURL || 'https://via.placeholder.com/45';

                    

                    // --- 2. 確定有身分資料後，才啟動推播註冊 ---

                    if (data.displayName) {

                        initPushNotifications(user.uid);

                    } else {

                        console.warn("偵測到帳號無姓名資料，推播註冊暫緩");

                    }

                } else {

                    if (nameEl) nameEl.innerText = "找不到資料";

                    // 如果連文件都不存在，建議引導去補填資料

                    console.error("Firestore 'act' 集合中找不到此 UID 的文件");

                }

            }).catch(err => {

                console.error("讀取身分失敗 (權限錯誤):", err);

                // 如果報 Permission Error，請檢查 Firebase Rules

            });



            // --- 3. 聊天室列表監聽 ---

            db.collection("chats")

            .where("members", "array-contains", user.uid)

            .orderBy("updatedAt", "desc")

            .onSnapshot(renderChatList);

            

            // --- 4. 監聽上下線 ---

            window.addEventListener('focus', () => updateMyStatus(true));

            window.addEventListener('blur', () => updateMyStatus(false));

            

        } else {

            // 使用 replace 避免按返回鍵又回到這一頁

            window.location.replace("index.html");

        }

    });



    async function renderChatList(snapshot) {

        const listElement = document.getElementById('chatList');

        if (snapshot.empty) { listElement.innerHTML = '<p style="text-align:center; color:#999; margin-top:20px;">暫無對話</p>'; removeInitLoader(); return; }



        const currentUser = auth.currentUser;

        let docs = snapshot.docs;

        

        const pinnedChats = [], normalChats = [];

        docs.forEach(doc => {

            const data = doc.data();

            if (data.isGroup && data.groupName === "立法院 Legislature") pinnedChats.push(doc);

            else normalChats.push(doc);

        });

        const sortedDocs = [...pinnedChats, ...normalChats];



        const htmlPromises = sortedDocs.map(async (doc) => {

            const chat = doc.data(), chatId = doc.id;

            let title = "", avatar = "", avatarStyle = "";



            if (chat.isGroup) {

                title = chat.groupName || "未命名群組";

                avatar = chat.groupAvatar || `https://ui-avatars.com/api/?name=${encodeURIComponent(title)}&background=003153&color=fff`;

            } else {

                const targetUid = chat.members.find(uid => uid !== currentUser.uid);

                if (targetUid && !userCache[targetUid]) {

                    const userDoc = await db.collection("act").doc(targetUid).get();

                    if (userDoc.exists) userCache[targetUid] = userDoc.data();

                }

                const targetData = userCache[targetUid] || { displayName: "未知成員", photoURL: "https://via.placeholder.com/45" };

                title = targetData.displayName; avatar = targetData.photoURL;

                avatarStyle = "width: 60px; height: 60px; margin-right: 10px;"; 

            }



            const updatedAt = chat.updatedAt ? chat.updatedAt.toDate() : new Date();

            const myLastRead = chat.lastReadAt?.[currentUser.uid] ? chat.lastReadAt[currentUser.uid].toDate() : new Date(0);

            const isUnread = chat.lastSenderId !== currentUser.uid && updatedAt > myLastRead;

            const unreadClass = isUnread ? "unread" : "";

            // 1. 修改 pinIcon：加入絕對定位 (position: absolute)，釘在卡片左上角

            const pinIcon = (chat.isGroup && chat.groupName === "立法院 Legislature") ? '<i class="fas fa-thumbtack" style="color:var(--ly-gold); position:absolute; top:10px; left:12px; transform:rotate(-45deg); font-size:14px;"></i>' : '';



            let timeStr = "";

            if (chat.updatedAt) {

                const now = new Date();

                if (updatedAt.toDateString() === now.toDateString()) timeStr = updatedAt.toLocaleTimeString('zh-TW', {hour: '2-digit', minute:'2-digit', hour12: false});

                else timeStr = `${updatedAt.getFullYear()}/${(updatedAt.getMonth() + 1).toString().padStart(2,'0')}/${updatedAt.getDate().toString().padStart(2,'0')}`;

            }



            let displayLastMsg = chat.lastMessage || '尚無訊息';

            if (displayLastMsg.includes('"type":"event_share"')) {

                try { const parsedMsg = JSON.parse(displayLastMsg); displayLastMsg = `分享了活動：${parsedMsg.title}`; } catch(e) {}

            } 

            else if (displayLastMsg.includes('"type":"group_buy"')) {

                try { const parsedMsg = JSON.parse(displayLastMsg); displayLastMsg = `發起了團購：${parsedMsg.title}`; } catch(e) {}

            }



            // 2. 重構 HTML：將 pinIcon 獨立拉出來，並確保 h4 標題乾淨

            return `

                <div class="chat-item ${unreadClass}" onclick="openChatRoom('${chatId}')">

                    ${pinIcon} <img class="user-avatar" src="${avatar}" style="${avatarStyle}">

                    <div class="chat-info">

                        <div style="display:flex; justify-content:space-between; align-items:center;">

                            <h4>${title}</h4> <span class="chat-time">${timeStr}</span>

                        </div>

                        <p>${displayLastMsg}</p>

                    </div>

                </div>`;

        });



        const html = await Promise.all(htmlPromises);

        listElement.innerHTML = html.join("");

        removeInitLoader();

    }



    // ==========================================

    // 4. Chat 視窗資料載入與 UI 渲染

    // ==========================================

    function updateMyStatus(isInChat) {

        if (!currentChatId || !auth.currentUser) return;

        const updateData = { [`presence.${auth.currentUser.uid}`]: isInChat };

        if (isInChat) updateData[`lastReadAt.${auth.currentUser.uid}`] = firebase.firestore.FieldValue.serverTimestamp();

        db.collection("chats").doc(currentChatId).update(updateData).catch(()=>{});

    }



    function initChatData(chatId) {

        const user = auth.currentUser;

        if (!user) return;

        

        window.isFirstChatLoad = true; 

        

        // 【修改】：雙重防呆，確保剛點進來時畫面一定是乾淨的

        messagesDocsData = [];

        document.getElementById('messageList').innerHTML = "<div style='text-align:center; padding:20px; color:#999; font-size:14px;'><i class='fas fa-spinner fa-spin'></i> 載入對話中...</div>";

        document.getElementById('chatTitle').innerText = "載入中...";

        window.chatMembersCache = {};

        

        const chatRef = db.collection("chats").doc(chatId);

        updateMyStatus(true);



        chatSnapshotUnsubscribe = chatRef.onSnapshot(doc => {

            if (!doc.exists) return closeChatRoom();

            const data = doc.data();

            

            // 釘選列處理

            // 釘選列處理

            const pinnedBar = document.getElementById('pinnedMessageBar'), pinnedText = document.getElementById('pinnedText'), unpinBtn = document.getElementById('unpinBtn');

            if (data.pinnedMessage?.text) {

                // 【新增】：把釘選的完整資料存到全域變數，供彈窗讀取

                window.currentPinnedMessage = data.pinnedMessage;

                pinnedBar.style.display = 'flex'; 

                pinnedText.innerText = data.pinnedMessage.text;

                

                // 【修改】：點擊時不再往上滑，改為開啟彈窗

                pinnedText.onclick = openPinnedModal; 

                unpinBtn.disabled = false;

            } else { 

                window.currentPinnedMessage = null;

                pinnedBar.style.display = 'none'; 

            }

            

            window.isCurrentChatGroup = !!data.isGroup;

            window.currentGroupMembers = data.members || [];

            window.groupReadTimes = data.lastReadAt || {};



            // 快取成員資料

            window.currentGroupMembers.forEach(uid => {

                if (!window.chatMembersCache[uid]) {

                    window.chatMembersCache[uid] = { displayName: "載入中", photoURL: "https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true" };

                    db.collection("act").doc(uid).get().then(uDoc => {

                        if (uDoc.exists) { window.chatMembersCache[uid] = uDoc.data(); renderChatUI(); }

                    });

                }

            });



            if (data.isGroup) {

                const rawGroupName = data.groupName || "未命名群組";

                window.currentRawGroupName = rawGroupName; // 【新增】：儲存未縮減的原始群組名稱

                // 【修改】：套用中段省略函式

                document.getElementById('chatTitle').innerText = formatMiddleEllipsis(rawGroupName);

                document.getElementById('targetAvatar').src = data.groupAvatar || `https://ui-avatars.com/api/?name=${encodeURIComponent(data.groupName)}&background=003153&color=fff`;

                document.getElementById('groupSettingsContainer').style.display = 'block';

                updateReadStatusUI();

            } else {

                document.getElementById('groupSettingsContainer').style.display = 'none';

                const targetUid = data.members.find(uid => uid !== user.uid);

                opponentInChat = data.presence ? data.presence[targetUid] : false;

                if (data.lastReadAt && data.lastReadAt[targetUid]) opponentReadTime = data.lastReadAt[targetUid].toDate();

                db.collection("act").doc(targetUid).get().then(uDoc => {

                    if (uDoc.exists) {

                        const rawUserName = uDoc.data().displayName || "未知成員";

                        // 【修改】：套用中段省略函式

                        document.getElementById('chatTitle').innerText = formatMiddleEllipsis(rawUserName);

                        document.getElementById('targetAvatar').src = uDoc.data().photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true';

                    }

                    updateReadStatusUI();

                });

            }

        });



        // 【新增】：每次進入新對話時，重置為 5 則

        // 【修改】：將 5 改為 20，確保訊息數量足夠撐出捲動條

        currentMessageLimit = 10;

        isFetchingOlder = false;



        // 將監聽器包裝成函式，以便往上滑時可以更新 Limit 數量

        currentSubscribeMessages = () => {

            if (messagesSnapshotUnsubscribe) messagesSnapshotUnsubscribe();

            

            // 改用降序(desc)並加上 limit 來只抓取最新 N 筆

            messagesSnapshotUnsubscribe = chatRef.collection("messages")

                .orderBy("timestamp", "desc")

                .limit(currentMessageLimit)

                .onSnapshot(snapshot => {

                    if (user && document.hasFocus() && currentChatId === chatId) {

                        chatRef.update({ [`lastReadAt.${user.uid}`]: firebase.firestore.FieldValue.serverTimestamp() }).catch(()=>{});

                    }

                    

                    // 因為降序抓下來最新訊息在最上面，我們需要將陣列反轉，讓舊的在上、新的在下

                    messagesDocsData = snapshot.docs.slice().reverse();

                    renderChatUI();

                });

        };

        

        currentSubscribeMessages(); // 初次觸發監聽

    } // <-- 這是 initChatData 函式的結尾右括號



    function renderChatUI() {

        if (!messagesDocsData || !auth.currentUser) return;

        const currentUid = auth.currentUser.uid;

        const list = document.getElementById('messageList');

        

        // 【新增】：在記錄高度前，先把頂部的載入動畫移除，這樣高度計算才會精準

        const loader = document.getElementById('historyLoader');

        if (loader) loader.remove();



        // 【新增這行】：記錄渲染前的高度，用來把畫面推回原本的相對位置

        const oldScrollHeight = list.scrollHeight;

        

        // 放寬置底判斷區間

        const isAtBottom = list.scrollHeight - list.scrollTop - list.clientHeight < 150;

        const isLastMessageMine = messagesDocsData.length > 0 && messagesDocsData[messagesDocsData.length - 1].data().senderId === currentUid;



        list.innerHTML = "";

        let lastTime = null;



        messagesDocsData.forEach((msgDoc, i) => {

            const msg = msgDoc.data(), msgId = msgDoc.id;

            const isMine = msg.senderId === currentUid;

            const msgTime = msg.timestamp ? msg.timestamp.toDate() : new Date();



            if (!lastTime || (msgTime - lastTime > 30 * 60 * 1000)) {

                const timeDiv = document.createElement('div');

                timeDiv.className = 'time-divider'; timeDiv.innerText = formatChatDate(msg.timestamp);

                list.appendChild(timeDiv);

            }

            lastTime = msgTime;



            const nextMsg = messagesDocsData[i + 1] ? messagesDocsData[i + 1].data() : null;

            const isLastInSequence = !nextMsg || nextMsg.senderId !== msg.senderId;



            const wrapper = document.createElement('div');

            wrapper.className = `msg-wrapper ${isMine ? 'mine' : 'theirs'} ${isLastInSequence ? 'last-in-group' : ''}`;



            // 【修改】：加入動態 style。如果是在群組且是別人的訊息，就切換為頂部對齊 (flex-start)

            let rowStyle = (!isMine && window.isCurrentChatGroup) ? 'align-items: flex-start;' : '';

            let html = `<div class="msg-row ${isMine ? 'mine' : 'theirs'}" onclick="toggleMsgSelect('${msgId}', ${isMine})" style="${rowStyle}">`;

            

            if (isMine) {

                const isChecked = window.selectedMsgIds && window.selectedMsgIds.has(msgId);

                const icon = isChecked ? '<i class="fas fa-check-circle"></i>' : '<i class="far fa-circle"></i>';

                const checkClass = isChecked ? 'checked' : '';

                html += `<div class="msg-checkbox ${checkClass}" id="cb-${msgId}">${icon}</div>`;

            }

            

            if (!isMine) {

                const avatarVisibility = isLastInSequence ? 'visible' : 'hidden';

                const senderCache = window.chatMembersCache[msg.senderId] || {};

                const senderAvatar = senderCache.photoURL || "https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true";

                const senderName = senderCache.displayName || "成員";



                if (window.isCurrentChatGroup) {

                    html += `<div class="msg-avatar-container" style="visibility: ${avatarVisibility}"><img src="${senderAvatar}" class="msg-avatar"><span class="msg-avatar-name">${senderName}</span></div>`;

                } else {

                    html += `<img src="${senderAvatar}" class="msg-avatar" style="visibility: ${avatarVisibility}">`;

                }

            }



            let contentHtml = "", isMediaOnly = false, safeText = '[圖片/影片]', isEventCard = false;



            if (msg.text) {

                try {

                    const parsed = JSON.parse(msg.text);

                    if (parsed.type === 'event_share') {

                        isEventCard = true; safeText = `分享了活動：${parsed.title}`;

                        let countdownHtml = `<span class="msg-event-countdown dynamic-countdown" data-event-id="${parsed.eventId}" data-collection="events" style="display:none;"></span>`;

                        contentHtml += `

                            <div onclick="openDocPreview('${parsed.eventId}')" class="msg-event-card">

                                <div class="msg-event-header">

                                    <span><i class="fas fa-bullhorn"></i> 活動分享</span>

                                    ${countdownHtml}

                                </div>

                                <div class="msg-event-title">${parsed.title}</div>

                                <div class="msg-event-time"><i class="far fa-clock"></i> ${parsed.timeStr}</div>

                            </div>`;

                    } 

                    // 【新增】：團購卡片渲染邏輯

                    else if (parsed.type === 'group_buy') {

                        isEventCard = true; safeText = `發起了團購：${parsed.title}`;

                        let countdownHtml = `<span class="msg-event-countdown dynamic-countdown" data-event-id="${parsed.orderId}" data-collection="group-orders" style="display:none;"></span>`;

                        contentHtml += `

                            <div onclick="openGroupBuyDetail('${parsed.orderId}')" class="msg-event-card" style="border-color: #28a745;">

                                <div class="msg-event-header">

                                    <span style="color:#28a745;"><i class="fas fa-shopping-cart"></i> 團購訂單</span>

                                    ${countdownHtml}

                                </div>

                                <div class="msg-event-title">${parsed.title}</div>

                                <div class="msg-event-time" style="font-weight:bold; color:#555; margin-top:4px;">

                                    [ 代購: ${parsed.initiator} ]

                                </div>

                            </div>`;

                    }

                    else safeText = msg.text.replace(/"/g, '&quot;').replace(/'/g, "\\'");

                } catch (e) { safeText = msg.text.replace(/"/g, '&quot;').replace(/'/g, "\\'"); }

            }



            if (msg.replyToId) contentHtml += `<div class="reply-snippet" onclick="scrollToOriginalMsg('${msg.replyToId}')"><i class="fas fa-reply"></i> ${msg.replyToText}</div>`;



            if (msg.fileUrl) {

                if (msg.fileType === 'video') {

                    // 【修改】：加上 onloadeddata="scrollToBottom()"

                    contentHtml += `<video src="${msg.fileUrl}" controls class="msg-media" onloadeddata="scrollToBottom()"></video>`;

                } else {

                    // 【修改】：加上 onload="scrollToBottom()"

                    contentHtml += `<img src="${msg.fileUrl}" class="msg-media" onload="scrollToBottom()" onclick="if(!skipNextClick) openImageModal(this.src)">`;

                }

            }

            

            if (msg.text && !isEventCard) contentHtml += (msg.fileUrl ? '<br>' : '') + highlightMentions(linkify(escapeHtml(msg.text)));

            else if (msg.fileUrl && !msg.replyToId && !isEventCard) isMediaOnly = true;



            const mediaClass = isMediaOnly ? 'media-only' : '';

            

            // 【移除左右滑動回覆，僅保留長按呼叫選單功能】

            // 【移除左右滑動回覆，僅保留長按呼叫選單功能】

            let interactEvents = isMine ? 

                `ontouchstart="handlePressStart(event, '${msgId}', 'true', '${safeText}')" ontouchmove="handlePressEnd()" ontouchend="handlePressEnd()" onmousedown="handlePressStart(event, '${msgId}', 'true', '${safeText}')" onmouseup="handlePressEnd()" onmouseleave="handlePressEnd()" oncontextmenu="event.preventDefault(); activeMsgId='${msgId}'; activeMsgTextForMenu='${safeText}'; showContextMenu(event, true);"` : 

                `ontouchstart="handlePressStart(event, '${msgId}', 'false', '${safeText}')" ontouchmove="handlePressEnd()" ontouchend="handlePressEnd()" onmousedown="handlePressStart(event, '${msgId}', 'false', '${safeText}')" onmouseup="handlePressEnd()" onmouseleave="handlePressEnd()" oncontextmenu="event.preventDefault(); activeMsgId='${msgId}'; activeMsgTextForMenu='${safeText}'; showContextMenu(event, false);"`;



            // 產生時間與已讀的排版 HTML

            // 產生時間的字串

            const timeStr = msgTime.toLocaleTimeString('zh-TW', {hour: '2-digit', minute:'2-digit', hour12: false});

            

            // 【修改】：判斷是否為群組對話，給予不同的排版結構

            // 【修改】：完美還原群組對話「上下堆疊」的排版

            if (window.isCurrentChatGroup) {

                const alignStyle = isMine ? 'align-items: flex-end;' : 'align-items: flex-start;';

                html += `<div style="display: flex; flex-direction: column; max-width: 75%; ${alignStyle}">`;

                // 訊息氣泡本體：將 margin-bottom 加大到 8px 增加一點點間隔

                html += `<div id="msg-${msgId}" class="msg ${isMine ? 'mine' : 'theirs'} ${mediaClass}" style="max-width: 100%; margin-bottom: 8px;" ${interactEvents}>${contentHtml}</div>`;

                

                // 下方區塊存放時間與已讀

                html += `<div class="msg-meta ${isMine ? 'mine' : 'theirs'}" id="meta-${msgId}" style="margin: 0 4px; display: flex; flex-direction: column; ${alignStyle}">`;

                html += `<div class="msg-time">${timeStr}</div>`;

                html += `</div></div>`;

            } else {

                // 單人對話：維持原本時間在泡泡兩側的排版

                const metaHtml = `<div class="msg-meta ${isMine ? 'mine' : 'theirs'}" id="meta-${msgId}"><div class="msg-time">${timeStr}</div></div>`;

                if (isMine) html += metaHtml;

                html += `<div id="msg-${msgId}" class="msg ${isMine ? 'mine' : 'theirs'} ${mediaClass}" ${interactEvents}>${contentHtml}</div>`;

                if (!isMine) html += metaHtml;

            }



            html += `</div>`; // 關閉 .msg-row

            wrapper.innerHTML = html;

            list.appendChild(wrapper);

        });



        // 在最底下的這邊加入這行

        updateDynamicCountdowns(); // 觸發即時倒數更新

        updateReadStatusUI();

        

        // 【新增】：如果是往上拉載入舊訊息，把卷軸推回原本的相對位置，不要置底

        if (isFetchingOlder) {

            setTimeout(() => {

                list.scrollTop = list.scrollHeight - oldScrollHeight;

                isFetchingOlder = false; // 恢復狀態

            }, 0);

        } 

        // 否則如果是初次載入、滑動在最底部、或是最新訊息是自己發的，就強制置底

        // 【修改】：如果是初次載入、滑動在最底部、或是最新訊息是自己發的，就強制置底

        else if (window.isFirstChatLoad || isAtBottom || isLastMessageMine) {

            const shouldSmooth = !window.isFirstChatLoad; 

            

            setTimeout(() => {

                scrollToBottom(shouldSmooth); 

                // 【關鍵修復】：等待延遲捲動確實執行完畢後，才解除「初次載入」的狀態

                // 這能完美包容 Firebase 在零點幾秒內發生的連續雙次渲染

                window.isFirstChatLoad = false; 

            }, 150);

        }

    }



    // 已讀邏輯

    // 已讀邏輯：計算誰讀了這則訊息 (排除發送者本人)

    function getReaders(msgTime, senderId) {

        if (!window.isCurrentChatGroup || !window.groupReadTimes || !msgTime) return [];

        const readers = [];

        window.currentGroupMembers.forEach(uid => {

            // 如果該成員不是發送者，且已讀時間大於訊息發送時間，就列入已讀名單

            if (uid !== senderId && window.groupReadTimes[uid] && window.groupReadTimes[uid].toDate() >= msgTime) {

                readers.push(uid);

            }

        });

        return readers;

    }



    function openReadModal(uidsStr) {

        if (!uidsStr) return;

        const uids = uidsStr.split(','), listEl = document.getElementById('readReceiptList');

        listEl.innerHTML = '';

        uids.forEach(uid => {

            const member = window.chatMembersCache[uid] || {}, name = member.displayName || "成員", photo = member.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true';

            listEl.innerHTML += `<div class="member-item"><img src="${photo}" alt="avatar"><span>${name}</span></div>`;

        });

        openModal('readReceiptModal');

    }



    function updateReadStatusUI() {

        if (!messagesDocsData || !auth.currentUser) return;

        document.querySelectorAll('.read-status').forEach(el => el.remove());

        const currentUid = auth.currentUser.uid;



        if (!window.isCurrentChatGroup) {

            let lastReadMsgId = null;

            for (let i = messagesDocsData.length - 1; i >= 0; i--) {

                const msg = messagesDocsData[i].data(), msgTime = msg.timestamp ? msg.timestamp.toDate() : new Date();

                if (msg.senderId === currentUid && msgTime <= opponentReadTime) { lastReadMsgId = messagesDocsData[i].id; break; }

            }

            if (lastReadMsgId) {

                const metaEl = document.getElementById('meta-' + lastReadMsgId);

                if (metaEl) metaEl.insertAdjacentHTML('afterbegin', `<div class="read-status">已讀</div>`);

            }

        } else {

            messagesDocsData.forEach(doc => {

                const msg = doc.data(), msgTime = msg.timestamp ? msg.timestamp.toDate() : new Date();

                let readers = getReaders(msgTime, msg.senderId);

                

                // 1. 過濾掉自己，絕對不會在已讀名單看到「你」

                readers = readers.filter(uid => uid !== currentUid);

                

                if (readers.length > 0) {

                    let names = readers.map(uid => window.chatMembersCache[uid]?.displayName || "成員");

                    let displayText = names.length <= 2 ? names.join("、") : names.slice(0, 2).join("、") + `...等 ${names.length} 人`;

                    const readHtml = `<div class="read-status group-read" onclick="openReadModal('${readers.join(',')}')">${displayText}</div>`;

                    

                    // 2. 將已讀狀態插入到時間容器 (.msg-meta) 內，達成圖片中的堆疊效果

                    const metaEl = document.getElementById('meta-' + doc.id);

                    if (metaEl) metaEl.insertAdjacentHTML('afterbegin', readHtml);

                }

            });

        }

    }



    // ==========================================

    // 5. Chat 互動功能 (發送、傳檔、選單、收回)

    // ==========================================

    function scrollToBottom(smooth = false) {

        const list = document.getElementById('messageList');

        if (list) list.scrollTo({ top: list.scrollHeight, behavior: smooth ? 'smooth' : 'auto' });

    }

    // --- 監聽訊息清單捲動，控制按鈕與載入歷史紀錄 ---

    // --- 監聽訊息清單捲動，控制按鈕與載入歷史紀錄 ---

    document.getElementById('messageList').addEventListener('scroll', function() {

        const btn = document.getElementById('scrollBottomBtn');

        btn.style.display = (this.scrollHeight - this.scrollTop - this.clientHeight > 150) ? 'flex' : 'none';



        // 【新增】：當捲動到最頂部 (scrollTop === 0) 且可能還有舊訊息時，觸發載入

        if (this.scrollTop < 10 && !isFetchingOlder && messagesDocsData.length >= currentMessageLimit) {

            isFetchingOlder = true;

            currentMessageLimit += 15; // 往上滑一次，多載入 15 則舊訊息

            

            // 先清掉可能殘留的舊動畫

            const oldLoader = document.getElementById('historyLoader');

            if (oldLoader) oldLoader.remove();



            // 在頂部插入一個臨時的載入中動畫

            const loaderHtml = '<div id="historyLoader" style="text-align:center; padding:15px; color:#999; font-size:0.85rem;"><i class="fas fa-spinner fa-spin"></i> 載入過往紀錄...</div>';

            this.insertAdjacentHTML('afterbegin', loaderHtml);

            

            // 呼叫重新監聽 (抓取更多資料)

            if (currentSubscribeMessages) currentSubscribeMessages();

        }

    });



    async function sendMessage() {

        const input = document.getElementById('msgInput'), text = input.value.trim();

        if (!text || !currentChatId) return;

        const user = auth.currentUser;

        input.value = ""; scrollToBottom(true); 



        try {

            const batch = db.batch(), chatRef = db.collection("chats").doc(currentChatId), msgRef = chatRef.collection("messages").doc();

            batch.set(msgRef, { text: text, senderId: user.uid, timestamp: firebase.firestore.FieldValue.serverTimestamp(), replyToId: activeReplyId || null, replyToText: activeReplyText || null });

            batch.update(chatRef, { lastMessage: text, lastSenderId: user.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp(), [`lastReadAt.${user.uid}`]: firebase.firestore.FieldValue.serverTimestamp() });

            await batch.commit();

            cancelReply();

        } catch (error) { console.error("發送失敗:", error); }

    }

    document.getElementById('msgInput').addEventListener('keypress', (e) => { if (e.key === 'Enter') sendMessage(); });

    document.getElementById('messageList').addEventListener('click', () => { document.getElementById('msgInput').blur(); });



    // ==========================================

    // 前端影片壓縮引擎 (FFmpeg.wasm)

    // ==========================================

    const { createFFmpeg, fetchFile } = FFmpeg;

    // 初始化 FFmpeg，設定 log: false 避免終端機洗版

    const ffmpeg = createFFmpeg({ log: false }); 



    async function compressVideoClientSide(file, inputField) {

        return new Promise(async (resolve, reject) => {

            try {

                // 1. 載入 FFmpeg 核心 (只需載入一次)

                if (!ffmpeg.isLoaded()) {

                    inputField.placeholder = "初次啟動影片壓縮引擎中...";

                    await ffmpeg.load();

                }



                inputField.placeholder = "影片壓縮中，請勿關閉視窗...";

                

                // 2. 將使用者的影片寫入 FFmpeg 的虛擬記憶體中

                const inputName = 'input' + file.name.substring(file.name.lastIndexOf('.'));

                const outputName = 'output.mp4';

                ffmpeg.FS('writeFile', inputName, await fetchFile(file));



                // 3. 執行壓縮指令

                // 指令說明：

                // -vf scale=-2:720 (最高畫質限制為 720p，維持比例)

                // -vcodec libx264 (使用高壓縮比的 h264 編碼)

                // -crf 28 (畫質與檔案大小的平衡點，數字越大檔案越小，28 是不錯的手機觀看畫質)

                // -preset veryfast (加快壓縮速度)

                await ffmpeg.run('-i', inputName, '-vf', 'scale=-2:720', '-vcodec', 'libx264', '-crf', '28', '-preset', 'veryfast', outputName);



                // 4. 從虛擬記憶體讀取壓縮後的檔案

                const data = ffmpeg.FS('readFile', outputName);

                

                // 5. 轉換為 Blob 與 File 物件回傳

                const blob = new Blob([data.buffer], { type: 'video/mp4' });

                const compressedFile = new File([blob], file.name.replace(/\.[^/.]+$/, "") + "_compressed.mp4", { type: 'video/mp4' });

                

                // 清理虛擬記憶體

                ffmpeg.FS('unlink', inputName);

                ffmpeg.FS('unlink', outputName);

                

                resolve(compressedFile);

            } catch (error) {

                console.error("影片壓縮失敗:", error);

                reject(error);

            }

        });

    }



    // 檔案上傳與壓縮

    function compressImageClientSide(file, maxSizeMB) {

        return new Promise((resolve, reject) => {

            const maxSize = maxSizeMB * 1024 * 1024, reader = new FileReader();

            reader.readAsDataURL(file);

            reader.onload = event => {

                const img = new Image(); img.src = event.target.result;

                img.onload = () => {

                    const canvas = document.createElement('canvas');

                    let width = img.width, height = img.height, maxDim = 4000;

                    if (width > maxDim || height > maxDim) {

                        if (width > height) { height = Math.round(height * (maxDim / width)); width = maxDim; } 

                        else { width = Math.round(width * (maxDim / height)); height = maxDim; }

                    }

                    canvas.width = width; canvas.height = height;

                    const ctx = canvas.getContext('2d'); ctx.drawImage(img, 0, 0, width, height);

                    let quality = 0.9;

                    const compress = () => {

                        canvas.toBlob(blob => {

                            if (!blob) return reject(new Error("處理失敗"));

                            if (blob.size <= maxSize || quality <= 0.1) resolve(new File([blob], file.name.replace(/\.[^/.]+$/, "") + "_compressed.jpg", { type: 'image/jpeg' }));

                            else { quality -= 0.1; compress(); }

                        }, 'image/jpeg', quality);

                    };

                    compress();

                };

            };

        });

    }



    // ==========================================

    // 團購專用：圖片極致縮小壓縮 (限制 800px，檔案極小化)

    // ==========================================

    function compressGroupBuyImage(file) {

        return new Promise((resolve, reject) => {

            const reader = new FileReader();

            reader.readAsDataURL(file);

            reader.onload = event => {

                const img = new Image(); 

                img.src = event.target.result;

                img.onload = () => {

                    const canvas = document.createElement('canvas');

                    // 團購商品圖不需要太大，限制在 800px 以內即可大幅縮小體積

                    let width = img.width, height = img.height, maxDim = 800;

                    if (width > maxDim || height > maxDim) {

                        if (width > height) { height = Math.round(height * (maxDim / width)); width = maxDim; } 

                        else { width = Math.round(width * (maxDim / height)); height = maxDim; }

                    }

                    canvas.width = width; canvas.height = height;

                    const ctx = canvas.getContext('2d'); 

                    ctx.drawImage(img, 0, 0, width, height);

                    

                    // 直接輸出 0.7 畫質的 JPEG，通常會在 50KB ~ 150KB 左右

                    canvas.toBlob(blob => {

                        if (!blob) return reject(new Error("處理失敗"));

                        resolve(new File([blob], file.name.replace(/\.[^/.]+$/, "") + "_gb_min.jpg", { type: 'image/jpeg' }));

                    }, 'image/jpeg', 0.7);

                };

            };

        });

    }



    // ==========================================

    // 檔案上傳與前端極致壓縮邏輯

    // ==========================================

    document.getElementById('fileInput').addEventListener('change', async (e) => {

        let file = e.target.files[0]; 

        if (!file || !currentChatId) return;



        const fileType = file.type.startsWith('video/') ? 'video' : 'image';

        const user = auth.currentUser;

        const inputField = document.getElementById('msgInput');

        const originalPlaceholder = inputField.placeholder;

        const sendBtn = document.querySelector('.send-btn');



        // 鎖定輸入框，防止壓縮期間使用者亂點

        inputField.disabled = true;

        sendBtn.disabled = true;



        try {

            // --- 處理大圖片壓縮 ---

            if (fileType === 'image') {

                if (file.size > MAX_FILE_SIZE || file.size > 2 * 1024 * 1024) { // 超過 2MB 就強制壓縮

                    inputField.placeholder = "圖片智慧壓縮中...";

                    file = await compressImageClientSide(file, 9.5); // 目標壓到 10MB 以下

                }

            } 

            

            // --- 處理大影片壓縮 (FFmpeg.wasm) ---

            else if (fileType === 'video') {

                if (file.size > 15 * 1024 * 1024) { 

                    // 如果影片大於 15MB，提示使用者這需要時間

                    if (!confirm("此影片較大，前端壓縮可能需要 10~30 秒，確定要處理嗎？")) {

                        throw new Error("USER_CANCEL");

                    }

                }

                

                // 只要超過 5MB 就進行前端影片壓縮

                if (file.size > 5 * 1024 * 1024) {

                    file = await compressVideoClientSide(file, inputField);

                    

                    // 壓縮完如果還是大於 10MB (非常罕見)，做最後防線攔截

                    if (file.size > MAX_FILE_SIZE) {

                        alert("影片壓縮後仍大於 10MB，請上傳較短的影片。");

                        throw new Error("FILE_TOO_LARGE");

                    }

                }

            }



            // --- 正式上傳至 Firebase Storage ---

            inputField.placeholder = "加密上傳中，請稍候...";

            

            const snapshot = await storage.ref(`chat/${currentChatId}/${Date.now()}_${file.name}`).put(file);

            const downloadUrl = await snapshot.ref.getDownloadURL();

            

            // 寫入 Firestore 訊息資料庫

            const batch = db.batch();

            const chatRef = db.collection("chats").doc(currentChatId);

            const msgRef = chatRef.collection("messages").doc();

            

            batch.set(msgRef, { 

                text: "", 

                fileUrl: downloadUrl, 

                fileType: fileType, 

                senderId: user.uid, 

                timestamp: firebase.firestore.FieldValue.serverTimestamp() 

            });

            

            batch.update(chatRef, { 

                lastMessage: fileType === 'image' ? '[圖片]' : '[影片]', 

                lastSenderId: user.uid, 

                updatedAt: firebase.firestore.FieldValue.serverTimestamp(), 

                [`lastReadAt.${user.uid}`]: firebase.firestore.FieldValue.serverTimestamp() 

            });

            

            await batch.commit();



        } catch (error) {

            if (error.message !== "USER_CANCEL" && error.message !== "FILE_TOO_LARGE") {

                console.error("處理或上傳失敗:", error);

                alert("檔案處理或上傳失敗，請更換檔案重試。");

            }

        } finally {

            // 無論成功或失敗，恢復 UI 狀態

            inputField.placeholder = originalPlaceholder;

            inputField.disabled = false;

            sendBtn.disabled = false;

            e.target.value = '';

            scrollToBottom(true);

        }

    });



    // 長按選單邏輯

    let pressTimer, activeMsgId = null, activeMsgTextForMenu = "", longPressActive = false, skipNextClick = false;

    function handlePressStart(e, msgId, isMineStr, text) {

        if (e.button === 2) return;

        activeMsgId = msgId; activeMsgTextForMenu = text; longPressActive = false;

        pressTimer = setTimeout(() => { longPressActive = true; showContextMenu(e, isMineStr === 'true'); }, 300);

    }

    function handlePressEnd() {

        clearTimeout(pressTimer);

        if (longPressActive) { skipNextClick = true; longPressActive = false; setTimeout(() => skipNextClick = false, 100); }

    }

    function showContextMenu(e, isMine) {

        if(e && e.cancelable) e.preventDefault(); 

        const menu = document.getElementById('contextMenu');

        

        // 判斷是否為純文字訊息，如果是才顯示「複製」

        const isText = activeMsgTextForMenu && !activeMsgTextForMenu.includes('[圖片/影片]') && !activeMsgTextForMenu.includes('分享了活動') && !activeMsgTextForMenu.includes('發起了團購');

        document.getElementById('copyMenuBtn').style.display = isText ? 'flex' : 'none';

        

        document.getElementById('multiSelectMenuBtn').style.display = isMine ? 'flex' : 'none';

        menu.style.display = 'flex'; menu.style.flexDirection = 'row'; menu.style.gap = '2px';

        

        const msgEl = document.getElementById('msg-' + activeMsgId); if (!msgEl) return;

        const rect = msgEl.getBoundingClientRect();

        let topPos = rect.top - menu.offsetHeight - 8; if (topPos < 90) topPos = rect.bottom + 8;

        let leftPos = rect.left + (rect.width / 2) - (menu.offsetWidth / 2);

        if (leftPos < 10) leftPos = 10; if (leftPos + menu.offsetWidth > window.innerWidth - 10) leftPos = window.innerWidth - menu.offsetWidth - 10;

        menu.style.left = `${leftPos}px`; menu.style.top = `${topPos}px`;

    }

    function closeContextMenu() { document.getElementById('contextMenu').style.display = 'none'; activeMsgId = null; }

    document.addEventListener('click', (e) => { if (skipNextClick) { e.preventDefault(); e.stopPropagation(); return; } if (!e.target.closest('.context-menu')) closeContextMenu(); }, true); 

    document.addEventListener('scroll', closeContextMenu, true);



    async function confirmRevoke() {

        if (!activeMsgId || !confirm("確定收回？")) return;

        try {

            const chatRef = db.collection("chats").doc(currentChatId), msgRef = chatRef.collection("messages").doc(activeMsgId), msgDoc = await msgRef.get();

            if (msgDoc.exists) {

                if (msgDoc.data().fileUrl) await storage.refFromURL(msgDoc.data().fileUrl).delete().catch(()=>{});

                await msgRef.delete();

                const lastMsgSnap = await chatRef.collection("messages").orderBy("timestamp", "desc").limit(1).get();

                if (!lastMsgSnap.empty) {

                    const lastData = lastMsgSnap.docs[0].data();

                    let newLastMsg = lastData.text || (lastData.fileType === 'video' ? '[影片]' : '[圖片]');

                    

                    // 【新增】：如果退回的上一則訊息是活動卡片，正確處理預覽文字

                    if (newLastMsg.includes('"type":"event_share"')) {

                        try {

                            const parsedMsg = JSON.parse(newLastMsg);

                            newLastMsg = `分享了活動：${parsedMsg.title}`;

                        } catch(e) {}

                    }

                    

                    await chatRef.update({ lastMessage: newLastMsg, lastSenderId: lastData.senderId, updatedAt: lastData.timestamp });

                } else {

                    await chatRef.update({ lastMessage: "尚無訊息", lastSenderId: "" });

                }

            }

        } catch (e) { alert("收回失敗"); } finally { closeContextMenu(); }

    }



    async function pinMessage() {

        if (!activeMsgId) return;

        

        // 1. 從目前畫面快取中找出這則訊息，撈出時間與發送者ID

        const msgDoc = messagesDocsData.find(doc => doc.id === activeMsgId);

        let senderName = "成員";

        let timeStr = "未知時間";



        if (msgDoc) {

            const msgData = msgDoc.data();

            const isMine = msgData.senderId === auth.currentUser.uid;

            

            // 判斷發送者名字

            if (isMine) {

                senderName = document.getElementById('myName').innerText;

            } else {

                const senderCache = window.chatMembersCache[msgData.senderId] || {};

                senderName = senderCache.displayName || "成員";

            }

            

            // 格式化時間

            if (msgData.timestamp) {

                const d = msgData.timestamp.toDate();

                timeStr = `${d.getFullYear()}/${(d.getMonth()+1).toString().padStart(2,'0')}/${d.getDate().toString().padStart(2,'0')} ${d.toLocaleTimeString('zh-TW', {hour: '2-digit', minute:'2-digit', hour12: false})}`;

            }

        }



        // 2. 打包成完整的釘選物件

        const pinnedData = { 

            id: activeMsgId, 

            text: activeMsgTextForMenu,

            senderName: senderName,

            timeStr: timeStr

        };



        // 3. 上傳更新 (Firebase 會自動把這包新資料覆蓋掉舊的，達成「自動刪除舊釘選」的效果)

        await db.collection("chats").doc(currentChatId).update({ pinnedMessage: pinnedData });

        closeContextMenu();

    }



    async function unpinMessage() { 

        if (confirm("移除釘選？")) await db.collection("chats").doc(currentChatId).update({ pinnedMessage: firebase.firestore.FieldValue.delete() }); 

    }



    // 【新增】：點擊上方橫幅時，跳出詳細視窗的函式

    function openPinnedModal() {

        if (!window.currentPinnedMessage) return;

        const pm = window.currentPinnedMessage;

        

        // 相容舊資料，若舊釘選沒有這些欄位就給預設值

        const sender = pm.senderName || "群組成員";

        const time = pm.timeStr || "";

        const text = pm.text || "";

        

        document.getElementById('pinnedMsgSenderTime').innerHTML = `<span><i class="fas fa-user" style="color:var(--ly-gold);"></i> ${sender}</span> <span><i class="far fa-clock"></i> ${time}</span>`;

        document.getElementById('pinnedMsgContent').innerHTML = escapeHtml(text).replace(/\n/g, '<br>');

        

        openModal('pinnedMessageModal');

    }

    // 滑動與回覆

    /*let touchStartX = 0, swipeMsgId = null, swipeMsgText = "", isSwipeMine = false;

    function handleSwipeStart(e, id, text, isMineStr) { touchStartX = e.touches[0].clientX; swipeMsgId = id; swipeMsgText = text; isSwipeMine = isMineStr === 'true'; }

    function handleSwipeMove(e, el) {

        if (!touchStartX) return; let diffX = e.touches[0].clientX - touchStartX;

        if (isSwipeMine && diffX < 0 && diffX > -80) el.style.transform = `translateX(${diffX}px)`;

        else if (!isSwipeMine && diffX > 0 && diffX < 80) el.style.transform = `translateX(${diffX}px)`;

    }

    function handleSwipeEnd(e, el) {

        if (!touchStartX) return; let diffX = e.changedTouches[0].clientX - touchStartX;

        el.style.transition = 'transform 0.3s ease'; el.style.transform = 'translateX(0px)'; setTimeout(() => { el.style.transition = ''; }, 300);

        if ((isSwipeMine && diffX < -45) || (!isSwipeMine && diffX > 45)) setReplyMode(swipeMsgId, swipeMsgText);

        touchStartX = 0;

    }*/



    let activeReplyId = null, activeReplyText = null;

    function setReplyMode(id, text) { activeReplyId = id; activeReplyText = text; document.getElementById('replyPreview').style.display = 'flex'; document.getElementById('replyPreviewText').innerText = text.length > 15 ? text.substring(0, 15) + '...' : text; document.getElementById('msgInput').focus(); }

    function cancelReply() { activeReplyId = null; activeReplyText = null; document.getElementById('replyPreview').style.display = 'none'; }

    function triggerMenuReply() { if (activeMsgId) { setReplyMode(activeMsgId, activeMsgTextForMenu); closeContextMenu(); } }

    function scrollToOriginalMsg(id) { const el = document.getElementById('msg-' + id); if (el) { el.scrollIntoView({ behavior: 'smooth', block: 'center' }); el.classList.remove('highlight-animation'); void el.offsetWidth; el.classList.add('highlight-animation'); } }



    // @標記功能

    let mentionStartIndex = -1;

    function showMentionList(searchStr) {

        const listEl = document.getElementById('mentionList');

        if (!window.isCurrentChatGroup) { listEl.style.display = 'none'; return; }

        listEl.innerHTML = ''; let hasMatches = false;

        

        if ("all (全員)".includes(searchStr.toLowerCase())) {

            hasMatches = true; const item = document.createElement('div'); item.className = 'mention-item';

            item.innerHTML = `<div style="width: 30px; height: 30px; border-radius: 50%; background: var(--ly-gold); color: white; display: flex; align-items: center; justify-content: center; margin-right: 12px; font-size: 14px; flex-shrink: 0;"><i class="fas fa-users"></i></div><span>All (全員)</span>`;

            

            // 【修改】：移除 ontouchstart，改用 onclick 支援滑動，並保留 mousedown 防止輸入框失去焦點

            item.onmousedown = (e) => { e.preventDefault(); }; 

            item.onclick = (e) => { e.preventDefault(); insertMention("All"); }; 

            listEl.appendChild(item);

        }



        window.currentGroupMembers.forEach(uid => {

            if (uid === auth.currentUser?.uid) return;

            const member = window.chatMembersCache[uid] || {}, name = member.displayName || "未知";

            if (name.toLowerCase().includes(searchStr.toLowerCase())) {

                hasMatches = true; const item = document.createElement('div'); item.className = 'mention-item';

                item.innerHTML = `<img src="${member.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'}"><span>${name}</span>`;

                

                // 【修改】：移除 ontouchstart，改用 onclick 支援滑動，並保留 mousedown 防止輸入框失去焦點

                item.onmousedown = (e) => { e.preventDefault(); }; 

                item.onclick = (e) => { e.preventDefault(); insertMention(name); }; 

                listEl.appendChild(item);

            }

        });

        listEl.style.display = hasMatches ? 'block' : 'none';

    }

    function insertMention(name) {

        const input = document.getElementById('msgInput'), val = input.value, pos = input.selectionStart;

        if (mentionStartIndex !== -1) {

            input.value = val.substring(0, mentionStartIndex) + '@' + name + ' ' + val.substring(pos);

            const newPos = mentionStartIndex + name.length + 2; input.focus(); input.setSelectionRange(newPos, newPos);

        }

        document.getElementById('mentionList').style.display = 'none'; mentionStartIndex = -1;

    }

    const msgInputEl = document.getElementById('msgInput');

    if(msgInputEl){

        const checkMention = () => {

            const val = msgInputEl.value, pos = msgInputEl.selectionStart; if (pos === null) return document.getElementById('mentionList').style.display = 'none';

            const match = val.substring(0, pos).match(/(?:^|\s)@([^@\s]*)$/);

            if (match && window.isCurrentChatGroup) { mentionStartIndex = val.substring(0, pos).lastIndexOf('@'); showMentionList(match[1]); }

            else { document.getElementById('mentionList').style.display = 'none'; mentionStartIndex = -1; }

        };

        msgInputEl.addEventListener('input', checkMention); msgInputEl.addEventListener('keyup', checkMention); msgInputEl.addEventListener('click', checkMention);

    }

    document.addEventListener('click', (e) => { if (!e.target.closest('.mention-list') && e.target.id !== 'msgInput') document.getElementById('mentionList').style.display = 'none'; });



    // ==========================================

    // 6. 對話建立與群組管理

    // ==========================================

    async function startNewChat() {

        openModal('newChatModal'); document.getElementById('targetUserName').value = '';

        const listEl = document.getElementById('singleUserList'); listEl.innerHTML = '<div style="text-align:center;">載入中...</div>';

        try {

            const snap = await db.collection("act").get(); let html = '';

            snap.forEach(doc => {

                if (doc.id !== auth.currentUser.uid) {

                    const data = doc.data();

                    html += `<label class="user-select-item"><input type="radio" name="singleChatUser" value="${data.displayName}" onclick="document.getElementById('targetUserName').value = this.value"><div class="user-select-card"><img src="${data.photoURL||'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'}"><span>${data.displayName}</span></div></label>`;

                }

            });

            listEl.innerHTML = html || '<div style="text-align:center;">無資料</div>';

        } catch(e){ listEl.innerHTML="載入失敗"; }

    }



    async function handleStartChat() {

        const targetName = document.getElementById('targetUserName').value.trim(); if (!targetName) return;

        try {

            const userQuery = await db.collection("act").where("displayName", "==", targetName).limit(1).get();

            if (userQuery.empty) return alert("找不到該人員");

            const targetUid = userQuery.docs[0].id; if (targetUid === auth.currentUser.uid) return alert("不支援與自己對話");



            const chatQuery = await db.collection("chats").where("members", "array-contains", auth.currentUser.uid).get();

            let existId = null; chatQuery.forEach(doc => { if (doc.data().members?.length === 2 && doc.data().members.includes(targetUid)) existId = doc.id; });



            closeModal('newChatModal');

            if (existId) openChatRoom(existId);

            else {

                const newRef = await db.collection("chats").add({ members: [auth.currentUser.uid, targetUid], lastMessage: "剛開啟了對話", lastSenderId: auth.currentUser.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp() });

                openChatRoom(newRef.id);

            }

        } catch(e) { alert("開啟錯誤"); }

    }



    // 裁切器共用邏輯

    let cropperInstance = null, targetPreviewId = null; window.croppedAvatarBlob = null;

    function openCropper(event, previewId) {

        const file = event.target.files[0]; targetPreviewId = previewId;

        if (file) {

            const reader = new FileReader(); reader.onload = e => {

                document.getElementById('cropperImage').src = e.target.result; openModal('cropperModal');

                if (cropperInstance) cropperInstance.destroy();

                cropperInstance = new Cropper(document.getElementById('cropperImage'), { aspectRatio: 1, viewMode: 1, dragMode: 'move', autoCropArea: 1, cropBoxMovable: false, cropBoxResizable: false, ready: () => { document.getElementById('cropperZoomSlider').value = cropperInstance.getImageData().ratio; }, zoom: e => document.getElementById('cropperZoomSlider').value = e.detail.ratio });

            }; reader.readAsDataURL(file);

        } event.target.value = '';

    }

    function changeCropperZoom(val) { if(cropperInstance) cropperInstance.zoomTo(parseFloat(val)); }

    function closeCropper() { closeModal('cropperModal'); if(cropperInstance) cropperInstance.destroy(); cropperInstance=null; }

    function confirmCrop() {

        if(!cropperInstance || !targetPreviewId) return;

        const cvs = cropperInstance.getCroppedCanvas({ width:300, height:300, fillColor:'#fff' });

        document.getElementById(targetPreviewId).src = cvs.toDataURL('image/jpeg');

        cvs.toBlob(b => { window.croppedAvatarBlob = b; if(targetPreviewId==='editGroupAvatarPreview') markAsUnsaved(); closeCropper(); }, 'image/jpeg', 0.85);

    }



    async function createGroup() {

        openModal('createGroupModal'); document.getElementById('createGroupAvatarPreview').src='https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'; window.croppedAvatarBlob = null;

        const listEl = document.getElementById('groupUserList'); listEl.innerHTML = '<div style="text-align:center;">載入中...</div>';

        try {

            const snap = await db.collection("act").get(); let html = '';

            snap.forEach(doc => {

                if (doc.id !== auth.currentUser.uid) html += `<label class="user-select-item"><input type="checkbox" value="${doc.id}" class="group-member-checkbox"><div class="user-select-card"><img src="${doc.data().photoURL||'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'}"><span>${doc.data().displayName}</span></div></label>`;

            }); listEl.innerHTML = html || '無成員';

        } catch(e){ listEl.innerHTML="載入失敗"; }

    }



    async function handleCreateGroup() {

        const groupName = document.getElementById('groupNameInput').value.trim(), checkboxes = document.querySelectorAll('.group-member-checkbox:checked');

        if (!groupName) return alert('請輸入名稱！'); if (checkboxes.length === 0) return alert('請選成員！');

        

        const members = [auth.currentUser.uid]; checkboxes.forEach(cb => members.push(cb.value));

        document.querySelector('#createGroupModal .btn-confirm').disabled = true;



        try {

            let groupAvatarUrl = "";

            if (window.croppedAvatarBlob) {

                const snap = await storage.ref(`groupAvatars/${db.collection("chats").doc().id}/${Date.now()}_avatar.jpg`).put(window.croppedAvatarBlob);

                groupAvatarUrl = await snap.ref.getDownloadURL();

            }

            const newRef = await db.collection("chats").add({ isGroup: true, groupName: groupName, members: members, groupAvatar: groupAvatarUrl || null, lastMessage: "群組已建立", lastSenderId: auth.currentUser.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp(), [`lastReadAt.${auth.currentUser.uid}`]: firebase.firestore.FieldValue.serverTimestamp() });

            closeModal('createGroupModal'); document.getElementById('groupNameInput').value = ''; openChatRoom(newRef.id);

        } catch(e) { alert('失敗'); } finally { document.querySelector('#createGroupModal .btn-confirm').disabled = false; }

    }



    // 群組設定 (Edit)

    function markAsUnsaved() { const b = document.getElementById('saveGroupBtn'); if(b) { b.style.display='block'; b.classList.add('btn-highlight'); b.innerText="記得儲存！"; } }

    function resetSaveButtonState() { const b = document.getElementById('saveGroupBtn'); if(b) { b.style.display='none'; b.classList.remove('btn-highlight'); b.innerText="儲存"; } }

    function toggleGroupOptions() {

        const c = document.getElementById('moreOptionsContainer'), t = document.getElementById('moreOptionsToggle');

        if(c.style.display==='none'){ c.style.display='flex'; t.innerHTML='<i class="fas fa-chevron-up"></i> 收起'; }

        else { c.style.display='none'; t.innerHTML='<i class="fas fa-chevron-down"></i> 更多選項'; }

    }



    // ==========================================

    // 群組設定與成員展開邏輯

    // ==========================================

    async function openGroupSettings() {

        openModal('groupSettingsModal'); 

        document.getElementById('moreOptionsContainer').style.display='none'; 

        document.getElementById('moreOptionsToggle').innerHTML='<i class="fas fa-chevron-down"></i> 更多選項';

        

        const nameInput = document.getElementById('groupNameEdit');

        const editMembersBtn = document.getElementById('editMembersBtn');

        

        // 【修改】：優先使用儲存的原始群組名稱，不使用被縮減的文字

        nameInput.value = window.currentRawGroupName || document.getElementById('chatTitle').innerText;

        document.getElementById('editGroupAvatarPreview').src = document.getElementById('targetAvatar').src;

        window.croppedAvatarBlob = null; 

        resetSaveButtonState();



        if (nameInput.value === "立法院 Legislature" && document.getElementById('myName').innerText !== "班長") {

            nameInput.disabled = true; 

            document.querySelector('.avatar-edit-wrapper').style.pointerEvents='none';

            document.querySelector('.avatar-edit-icon').style.display='none'; 

            document.getElementById('deleteGroupBtn').style.display='none'; 

            editMembersBtn.style.display='none';

        } else {

            nameInput.disabled = false; 

            document.querySelector('.avatar-edit-wrapper').style.pointerEvents='auto';

            document.querySelector('.avatar-edit-icon').style.display='flex'; 

            document.getElementById('deleteGroupBtn').style.display='block'; 

            editMembersBtn.style.display='block';

        }



        const listEl = document.getElementById('groupMembersList'); 

        listEl.innerHTML = '<div style="font-size: 13px; color: #999; text-align: center;">載入成員中... <i class="fas fa-spinner fa-spin"></i></div>';

        

        try {

            // 【修改】：先將抓到的成員資料存進全域快取陣列中

            const memberPromises = (window.currentGroupMembers || []).map(async uid => {

                const doc = await db.collection("act").doc(uid).get();

                if (doc.exists) return { uid, ...doc.data() };

                return { uid, displayName: "未知成員", photoURL: "https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true" };

            });



            window.groupMembersDataCache = await Promise.all(memberPromises);

            

            // 預設為收合狀態 (false)，最多顯示 3 人

            renderGroupMembers(false);

            

        } catch(e) { 

            listEl.innerHTML = '<div style="font-size: 13px; color: red; text-align: center;">載入失敗</div>'; 

        }

    }



    // 【新增】：獨立的成員列表渲染與展開/收合邏輯

    function renderGroupMembers(isExpanded) {

        const listEl = document.getElementById('groupMembersList');

        const membersData = window.groupMembersDataCache || [];

        

        let html = '';

        // 決定要顯示幾個人 (若為 true 顯示全部，false 最多顯示 3 個)

        const displayCount = isExpanded ? membersData.length : Math.min(3, membersData.length);



        for (let i = 0; i < displayCount; i++) {

            const user = membersData[i];

            const avatar = user.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true';

            const name = user.displayName || '未命名成員';

            html += `

                <div class="member-item">

                    <img src="${avatar}" alt="avatar">

                    <span>${name}</span>

                </div>

            `;

        }



        // 判斷是否需要顯示「展開 / 收起」按鈕

        if (!isExpanded && membersData.length > 3) {

            const hiddenCount = membersData.length - 3;

            html += `<button class="more-members-btn" onclick="renderGroupMembers(true)">

                        <i class="fas fa-ellipsis-h"></i> 展開其餘 ${hiddenCount} 位成員

                     </button>`;

        } else if (isExpanded && membersData.length > 3) {

            html += `<button class="more-members-btn" onclick="renderGroupMembers(false)">

                        <i class="fas fa-chevron-up"></i> 收起成員列表

                     </button>`;

        }



        listEl.innerHTML = html;

    }



    async function saveGroupSettings() {

        const newName = document.getElementById('groupNameEdit').value.trim(); if (!newName) return alert('不能為空');

        if (window.croppedAvatarBlob && !confirm("確認更改圖片？")) { window.croppedAvatarBlob=null; document.getElementById('editGroupAvatarPreview').src = document.getElementById('targetAvatar').src; return; }

        

        const b = document.getElementById('saveGroupBtn'); b.innerText="儲存中..."; b.disabled=true;

        try {

            const updateData = { groupName: newName };

            if (window.croppedAvatarBlob) {

                const snap = await storage.ref(`groupAvatars/${currentChatId}/${Date.now()}_avatar.jpg`).put(window.croppedAvatarBlob);

                updateData.groupAvatar = await snap.ref.getDownloadURL();

            }

            await db.collection("chats").doc(currentChatId).update(updateData); closeModal('groupSettingsModal'); window.croppedAvatarBlob=null;

        } catch(e){ alert("失敗"); } finally { b.innerText="儲存"; b.disabled=false; }

    }

    

    async function deleteGroup() {

        if (!confirm("確定刪除此群組？無法還原！")) return;

        try { await db.collection("chats").doc(currentChatId).delete(); closeModal('groupSettingsModal'); closeChatRoom(); } catch(e){ alert("失敗"); }

    }



    async function openEditMembersModal() {

        openModal('editMembersModal'); const listEl = document.getElementById('editMembersList'); listEl.innerHTML='載入中...';

        try {

            const snap = await db.collection("act").get(); let html = '';

            snap.forEach(doc => {

                const isChecked = window.currentGroupMembers.includes(doc.id) ? 'checked' : '';

                html += `<label class="user-select-item"><input type="checkbox" value="${doc.id}" class="edit-group-member-checkbox" ${isChecked}><div class="user-select-card"><img src="${doc.data().photoURL}"><span>${doc.data().displayName}</span></div></label>`;

            }); listEl.innerHTML = html;

        } catch(e){ listEl.innerHTML="失敗"; }

    }

    

    async function saveGroupMembers() {

        const checkboxes = document.querySelectorAll('.edit-group-member-checkbox:checked'); if(checkboxes.length===0) return alert('需保留一人');

        const newMembers = []; checkboxes.forEach(cb => newMembers.push(cb.value));

        if(!newMembers.includes(auth.currentUser.uid)) newMembers.push(auth.currentUser.uid);

        try { await db.collection("chats").doc(currentChatId).update({ members: newMembers }); window.currentGroupMembers = newMembers; closeModal('editMembersModal'); openGroupSettings(); } catch(e){ alert("失敗"); }

    }



    // ==========================================

    // 7. 活動與圖片預覽系統

    // ==========================================

    function toggleExpandMenu(e) {

        const panel = document.getElementById('expandMenuPanel'), btn = e.currentTarget;

        if(panel.style.display==='flex') { panel.style.display='none'; btn.style.transform='rotate(0deg)'; }

        else { panel.style.display='flex'; btn.style.transform='rotate(45deg)'; }

    }

    document.addEventListener('click', (e) => {

        const p = document.getElementById('expandMenuPanel'), b = document.querySelector('.expand-menu-btn');

        if(p && p.style.display==='flex' && !p.contains(e.target) && !b.contains(e.target)) { p.style.display='none'; b.style.transform='rotate(0deg)'; }

    }, true);



    async function handleActivitySchedule() {

        document.getElementById('expandMenuPanel').style.display='none'; document.querySelector('.expand-menu-btn').style.transform='rotate(0deg)';

        openModal('activityScheduleModal'); const listEl = document.getElementById('activityScheduleList'); listEl.innerHTML = '<div style="text-align:center; padding:20px;"><i class="fas fa-spinner fa-spin"></i> 載入中...</div>';

        try {

            const currentUser = auth.currentUser;

            const currentUserEmail = currentUser ? currentUser.email : '';

            const currentUid = currentUser ? currentUser.uid : '';



            const snap = await db.collection("events").get(); 

            let events = [];

            snap.forEach(doc => {

                const ev = doc.data(); 

                

                // 【修復】：加入 creatorUid 的判斷，最精準辨識是不是你建立的

                const isCreator = (ev.creatorUid === currentUid) || (ev.creator === currentUserEmail) || (ev.creator === currentUid);

                

                if (ev.isPrivate) {

                    const authMembers = ev.authorizedMembers || [];

                    if (!isCreator && !authMembers.includes(currentUid)) {

                        return; // 沒有權限，隱藏此活動

                    }

                }



                // 【修復】：安全處理時間轉換，避免剛存入時 createdAt 為 null 導致報錯消失

                let sortDate = new Date();

                const timeField = ev.fields?.find(f => f.type === 'time');

                if (timeField && timeField.content) {

                    sortDate = new Date(timeField.content);

                } else if (ev.createdAt && typeof ev.createdAt.toDate === 'function') {

                    sortDate = ev.createdAt.toDate();

                }



                events.push({ id: doc.id, ...ev, sortDate, isCreator }); // 把 isCreator 一併存入陣列供後續判斷

            });

            

            events.sort((a,b)=>b.sortDate - a.sortDate); 

            if(events.length===0){ listEl.innerHTML="<div class='event-empty-state'>目前無活動行程</div>"; return; }

            

            let html = '', currentYear = null;

            events.forEach(ev => {

                const year = ev.sortDate.getFullYear(); 

                if(year!==currentYear){ currentYear=year; html+=`<div class="event-year-title">${year} 年度</div>`; }

                const timeStr = ev.sortDate.toLocaleString('zh-TW', {year:'numeric',month:'2-digit',day:'2-digit',hour:'2-digit',minute:'2-digit',hour12:false}).replace(/\//g,'-');

                const safeTitle = (ev.title||'').replace(/'/g, "\\'").replace(/"/g, '&quot;');

                const deadlineStr = ev.deadline || '';

                

                const lockIcon = ev.isPrivate ? '<i class="fas fa-lock" style="color:#d93025; margin-right:5px;" title="私人活動"></i>' : '';

                const innerContent = `<h4 class="event-mini-title">${lockIcon}${ev.title}</h4><div class="event-mini-meta"><i class="far fa-clock"></i> ${timeStr}</div>`;



                if (ev.isCreator) {

                    // 發起人：套用左滑編輯/刪除模組

                    html += `

                    <div class="swipe-container">

                        <div class="swipe-actions-dual">

                            <button class="swipe-edit-btn" onclick="editEvent('${ev.id}')"><i class="fas fa-edit"></i></button>

                            <button class="swipe-delete-btn-half" onclick="deleteEvent('${ev.id}')"><i class="fas fa-trash-alt"></i></button>

                        </div>

                        <div class="swipe-content event-mini-item" style="margin-bottom: 0;"

                             onmousedown="evSwipeStart(event, this)" 

                             onmousemove="evSwipeMove(event, this)" 

                             onmouseup="evSwipeEnd(event, this, '${ev.id}', '${safeTitle}', '${timeStr}', '${deadlineStr}')" 

                             onmouseleave="evSwipeEnd(event, this, '${ev.id}', '${safeTitle}', '${timeStr}', '${deadlineStr}')" 

                             ontouchstart="evSwipeStart(event, this)" 

                             ontouchmove="evSwipeMove(event, this)" 

                             ontouchend="evSwipeEnd(event, this, '${ev.id}', '${safeTitle}', '${timeStr}', '${deadlineStr}')">

                            ${innerContent}

                        </div>

                    </div>`;

                } else {

                    // 非發起人：純點擊發布卡片

                    html += `<div class="event-mini-item" onclick="sendEventCard('${ev.id}', '${safeTitle}', '${timeStr}', '${deadlineStr}')">${innerContent}</div>`;

                }

            }); 

            listEl.innerHTML = html;

        } catch(e){ listEl.innerHTML="<div style='text-align:center; color:red; padding:20px;'>載入失敗，請檢查網路</div>"; console.error(e); }

    }



    // ==========================================

    // 活動清單：左滑引擎與編輯/刪除邏輯

    // ==========================================

    let evSwipeStartX = 0, evSwipeCurrentX = 0, isEvItemSwiping = false, evSwipedItemOpen = null;



    function evSwipeStart(e, el) {

        if (evSwipedItemOpen && evSwipedItemOpen !== el) { evSwipedItemOpen.style.transform = 'translateX(0px)'; evSwipedItemOpen = null; }

        evSwipeStartX = e.type.includes('mouse') ? e.pageX : e.touches[0].clientX;

        evSwipeCurrentX = 0; isEvItemSwiping = true; el.style.transition = 'none';

    }

    function evSwipeMove(e, el) {

        if (!isEvItemSwiping) return;

        const x = e.type.includes('mouse') ? e.pageX : e.touches[0].clientX;

        evSwipeCurrentX = x - evSwipeStartX;

        if (evSwipeCurrentX < 0 && evSwipeCurrentX > -160) el.style.transform = `translateX(${evSwipeCurrentX}px)`;

    }

    function evSwipeEnd(e, el, eventId, title, timeStr, deadline) {

        if (!isEvItemSwiping) return;

        isEvItemSwiping = false; el.style.transition = 'transform 0.3s ease';

        

        if (evSwipeCurrentX < -75) {

            // 滑超過一半，展開兩個按鈕 (150px)

            el.style.transform = 'translateX(-150px)';

            evSwipedItemOpen = el;

        } else {

            el.style.transform = 'translateX(0px)';

            if (evSwipedItemOpen === el) evSwipedItemOpen = null;

            // 輕點直接發送

            if (Math.abs(evSwipeCurrentX) < 10 && eventId) sendEventCard(eventId, title, timeStr, deadline);

        }

        evSwipeCurrentX = 0;

    }



    async function deleteEvent(eventId) {

        if (!confirm("確定要刪除此活動嗎？\n(若有上傳照片也會一併從雲端清除)")) return;

        try {

            const doc = await db.collection("events").doc(eventId).get();

            if (doc.exists && doc.data().fields) {

                // 清理圖片 Storage

                doc.data().fields.forEach(f => {

                    if(f.type === 'image_upload' && f.content) {

                        try{ storage.refFromURL(f.content).delete(); } catch(e){}

                    }

                });

            }

            await db.collection("events").doc(eventId).delete();

            alert("活動已刪除！");

            handleActivitySchedule(); // 重新整理清單

        } catch(e) { alert("刪除失敗: " + e.message); }

    }



    let currentEditingEventId = null; // 全域變數紀錄編輯狀態

    async function editEvent(eventId) {

        currentEditingEventId = eventId;

        closeModal('activityScheduleModal');

        

        // 變更表單標題與按鈕

        document.querySelector('#createEventModal h3').innerHTML = '<i class="fas fa-edit"></i> 編輯活動';

        document.getElementById('btnSubmitEvent').innerHTML = '<i class="fas fa-save"></i> 儲存修改';

        

        const doc = await db.collection("events").doc(eventId).get();

        if (!doc.exists) return;

        const data = doc.data();



        document.getElementById('ev-title').value = data.title;

        document.getElementById('ev-deadline').value = data.deadline || "";

        

        document.getElementById('ev-is-private').checked = !!data.isPrivate;

        toggleEventPrivateMode(!!data.isPrivate);



        if (data.isPrivate) {

            evAuthorizedUIDs = data.authorizedMembers || [];

            renderEvMemberPreview(evAuthorizedUIDs, 'authorized');

            evSelectedUIDs = [];

        } else {

            evSelectedUIDs = data.participants || [];

            renderEvMemberPreview(evSelectedUIDs, 'participants');

            evAuthorizedUIDs = [];

        }



        const container = document.getElementById('dynamic-fields-container');

        container.innerHTML = ""; 

        if (data.fields) {

            let imageGroup = [];

            data.fields.forEach(f => {

                if(f.type === 'image_upload') imageGroup.push(f.content);

                else {

                    addEventField(f.type, f.label);

                    const last = container.lastElementChild;

                    const input = last.querySelector('.ev-dynamic-input');

                    if(input) input.value = f.content;

                }

            });

            if(imageGroup.length > 0) {

                addEventField('image_upload', '活動照片');

                const lastImgField = container.querySelector('.draggable-item[data-type="image_upload"]');

                if(lastImgField) {

                    const fid = lastImgField.id.replace('field-', '');

                    updateEvImageField(fid, imageGroup);

                }

            }

        }

        openModal('createEventModal');

    }



    // 【修改】：接收 deadlineStr 並存入 JSON 中

    async function sendEventCard(eventId, title, timeStr, deadlineStr) {

        if(!confirm(`發布「${title}」？`)) return; closeModal('activityScheduleModal');

        try {

            const batch = db.batch(), chatRef = db.collection("chats").doc(currentChatId), msgRef = chatRef.collection("messages").doc();

            // 將 deadline 一併包進 JSON 字串存入資料庫

            batch.set(msgRef, { text: JSON.stringify({ type:'event_share', eventId:eventId, title:title, timeStr:timeStr, deadline:deadlineStr }), senderId: auth.currentUser.uid, timestamp: firebase.firestore.FieldValue.serverTimestamp() });

            batch.update(chatRef, { lastMessage: `分享活動：${title}`, lastSenderId: auth.currentUser.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp(), [`lastReadAt.${auth.currentUser.uid}`]: firebase.firestore.FieldValue.serverTimestamp() });

            await batch.commit(); scrollToBottom(true);

        } catch(e){ alert("失敗"); }

    }



    // 投票系統

    // 投票系統

    let currentEventId = null, currentGalleryImages = [];

    let eventVotesUnsubscribe = null; // 【新增】：用來記錄並取消舊的監聽器

    window.eventProfileCache = window.eventProfileCache || {}; // 【新增】：全域成員資料快取

    async function openDocPreview(id) {

        currentEventId = id; currentGalleryImages = [];

        try {

            const docSnap = await db.collection("events").doc(id).get(); if (!docSnap.exists) return alert("找不到");

            const ev = docSnap.data(), contentArea = document.getElementById('doc-content');

            document.querySelector('.doc-header').innerHTML = `<h2 style="color:var(--ly-blue); text-align:center;">${ev.title}</h2><div style="text-align:center; font-size:0.85rem; color:#666;">存檔：${ev.formattedDate||'--'}</div>`;

            contentArea.innerHTML = "";

            ev.fields?.forEach(f => {

                if(f.content===ev.title) return;

                if(f.type==='image_upload' && f.content) currentGalleryImages.push(f.content);

                else contentArea.innerHTML += `<div style="margin-bottom:20px;"><strong style="display:block; color:var(--ly-blue);">[ ${f.label} ]</strong><div>${f.content.replace('T', ' ')}</div></div>`;

            });

            if (currentGalleryImages.length > 0) {

                let stack = `<div style="margin-bottom:20px;"><strong style="color:var(--ly-blue);">[ 活動剪影 ]</strong></div><div class="photo-stack-container" onclick="openImageViewer(0)">`;

                for(let i=0; i<Math.min(currentGalleryImages.length,3); i++) stack += `<div class="photo-stack-item"><img src="${currentGalleryImages[i]}"></div>`;

                contentArea.innerHTML += stack + `</div>`;

            }

            const voteArea = document.getElementById('vote-area'), isExp = ev.deadline && (new Date(ev.deadline).getTime() - new Date().getTime() <= 0);

            voteArea.style.display = 'block'; voteArea.innerHTML = `<div class="countdown-header">${isExp?'截止':'請確認行程'}</div><div style="padding:20px 10px;"><button class="btn-vote btn-join" onclick="handleVote('join')" ${isExp?'disabled':''}>參與</button><button class="btn-vote btn-next" onclick="handleVote('next')" ${isExp?'disabled':''}>下次</button></div>`;

            loadVotes(id); openModal('doc-preview-modal');

        } catch(err){ alert("失敗"); }

    }



    async function handleVote(choice) {

        if(!auth.currentUser || !currentEventId) return;

        const eventRef = db.collection("events").doc(currentEventId);

        try {

            let uName = document.getElementById('myName').innerText;

            const eDoc = await eventRef.get(); if(!eDoc.exists) return;

            let parts = eDoc.data().participants||[], names = eDoc.data().participantNames||[];

            if(choice==='join'){ if(!parts.includes(auth.currentUser.uid)) parts.push(auth.currentUser.uid); if(!names.includes(uName)) names.push(uName); }

            else { parts = parts.filter(i=>i!==auth.currentUser.uid); names = names.filter(n=>n!==uName); }

            await eventRef.update({ participants: parts, participantNames: names }); alert(choice==='join'?"已紀錄":"已移除");

        } catch(e){}

    }

    

    async function loadVotes(eventId) {

        const list = document.getElementById('voter-list');

        const countTag = document.getElementById('participant-count-tag');

        const user = auth.currentUser;



        // 【優化 1】：立即清空舊畫面，顯示載入中動畫，徹底解決「殘留上一次舊成員」的問題

        list.innerHTML = "<div style='width: 100%; text-align: center; color: #ccc; font-size: 0.9rem; padding: 10px 0;'><i class='fas fa-spinner fa-spin'></i> 載入成員資料中...</div>";

        if (countTag) countTag.innerText = "";



        // 【優化 2】：如果之前有綁定過監聽器，先把它取消掉，避免多個活動互相干擾

        if (eventVotesUnsubscribe) {

            eventVotesUnsubscribe();

            eventVotesUnsubscribe = null;

        }



        // 取得當前登入者的名稱 (用來比對 participantNames)

        let currentUserName = "";

        if (user) {

            try {

                if (window.eventProfileCache[user.uid]) {

                    currentUserName = window.eventProfileCache[user.uid].text || window.eventProfileCache[user.uid].displayName;

                } else {

                    let uDoc = await db.collection("content").doc(user.uid).get();

                    if (uDoc.exists) {

                        currentUserName = uDoc.data().text;

                        window.eventProfileCache[user.uid] = uDoc.data();

                    } else {

                        uDoc = await db.collection("act").doc(user.uid).get();

                        if (uDoc.exists) {

                            currentUserName = uDoc.data().displayName;

                            window.eventProfileCache[user.uid] = uDoc.data();

                        }

                    }

                }

            } catch (e) { console.error("無法取得名稱:", e); }

            if (!currentUserName) currentUserName = user.displayName || user.email.split('@')[0];

        }



        // 定義正確的排序名單

        const memberOrder = [

            { id: "001", name: "もも" }, { id: "002", name: "班長" }, { id: "003", name: "俊齊" },

            { id: "004", name: "博峻" }, { id: "005", name: "佳偉" }, { id: "006", name: "書銘" },

            { id: "007", name: "江旭" }, { id: "008", name: "尊玄" }, { id: "009", name: "俊諺" },

            { id: "010", name: "成一" }, { id: "011", name: "宗航" }, { id: "012", name: "崇憲" },

            { id: "013", name: "光庭" }, { id: "014", name: "威丞" }, { id: "015", name: "韋仁" },

            { id: "016", name: "祐瑜" }, { id: "017", name: "至軒" }, { id: "018", name: "奕賢" },

            { id: "019", name: "正德" }, { id: "020", name: "鈞儒" }, { id: "021", name: "凱瑞" },

            { id: "022", name: "承遠" }, { id: "023", name: "皓鈞" }, { id: "024", name: "鎮遠" },

            { id: "025", name: "智皓" }, { id: "026", name: "哲遠" }, { id: "027", name: "隆勳" },

            { id: "028", name: "柏鑫" }, { id: "029", name: "承佑" }, { id: "030", name: "凱駿" },

            { id: "031", name: "冠叡" }, { id: "032", name: "祐祥" }, { id: "033", name: "可愛柴" }

        ];



        // 將監聽器賦值給 eventVotesUnsubscribe 以便下次清除

        eventVotesUnsubscribe = db.collection("events").doc(eventId).onSnapshot(async (doc) => {

            if(!doc.exists) return; 

            const data = doc.data();

            const participants = data.participants || [];

            const participantNames = data.participantNames || [];



            if (countTag) countTag.innerText = `(共 ${participants.length} 人)`;



            const jBtn = document.querySelector('.btn-join'), nBtn = document.querySelector('.btn-next');

            if(jBtn && auth.currentUser){

                const isJ = participantNames.includes(currentUserName) || participants.includes(user.uid);

                if(data.deadline && new Date().getTime() > new Date(data.deadline).getTime()){ 

                    jBtn.disabled=nBtn.disabled=true; 

                } else if(isJ){ 

                    jBtn.className='btn-vote btn-join btn-joined-locked'; 

                    jBtn.innerHTML='<i class="fas fa-check"></i> 已參與'; 

                    jBtn.disabled=true; nBtn.disabled=false; 

                } else { 

                    jBtn.className='btn-vote btn-join'; 

                    jBtn.innerHTML='參與'; 

                    jBtn.disabled=false; nBtn.disabled=false; 

                }

            }



            if(!participants.length){ 

                list.innerHTML="<p style='color:#ccc; font-size:0.9rem; width: 100%; text-align: center;'>尚無成員參與...</p>"; 

                return; 

            }

            

            const memberMap = {};

            

            // 【優化 3】：加入全域快取 (Cache) 系統。

            // 抓過資料的成員，下次打開任何活動都會瞬間秒讀，不再浪費流量與等待時間。

            const memberPromises = participants.map(async (uid) => {

                if (window.eventProfileCache[uid]) {

                    memberMap[uid] = window.eventProfileCache[uid];

                    return;

                }

                

                // 如果快取沒有，才去資料庫抓

                let snap = await db.collection("content").doc(uid).get();

                if (snap.exists) {

                    window.eventProfileCache[uid] = snap.data();

                    memberMap[uid] = snap.data();

                } else {

                    snap = await db.collection("act").doc(uid).get();

                    if (snap.exists) {

                        window.eventProfileCache[uid] = snap.data();

                        memberMap[uid] = snap.data();

                    }

                }

            });



            await Promise.all(memberPromises); // 等待所有還沒快取的人抓完資料



            // 依照 memberOrder 進行配對與排序渲染

            let htmlContent = "";

            memberOrder.forEach(orderItem => {

                for (const uid in memberMap) {

                    const m = memberMap[uid];

                    const mName = m.text || m.displayName || "";

                    if (mName === orderItem.name) {

                        const avatarUrl = m.imageUrl || m.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true';

                        htmlContent += `

                            <div class="voter-box">

                                <img src="${avatarUrl}" onerror="this.src='https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'">

                                <span>${mName}</span>

                            </div>`;

                        break; 

                    }

                }

            });



            list.innerHTML = htmlContent; // 將排序好的 HTML 一次性塞入畫面

        });

    }



    // 單圖與幻燈片預覽

    function openImageModal(src) { document.getElementById('expandedImg').src = src; openModal('imageModal'); }

    

    let currentSlideIndex=0, isSliderDragging=false, sliderStartPos=0, currentTranslate=0, prevTranslate=0, animationID;

    function openImageViewer(startIndex) {

        if(!currentGalleryImages.length) return; currentSlideIndex = startIndex||0;

        const track = document.getElementById('slider-track'), thumbs = document.getElementById('viewer-thumbnails');

        track.innerHTML = ""; thumbs.innerHTML = "";

        currentGalleryImages.forEach((url, i) => {

            track.innerHTML += `<div class="slider-slide"><img src="${url}" draggable="false"></div>`;

            if(currentGalleryImages.length>1) {

                const thumb = document.createElement('div'); thumb.className = 'thumbnail-item'; thumb.innerHTML = `<img src="${url}">`;

                thumb.onclick = () => { currentSlideIndex = i; updateSliderPos(); }; thumbs.appendChild(thumb);

            }

        });

        document.getElementById('image-viewer-modal').style.display='flex'; updateSliderPos(true);

        attachSliderEvents();

    }

    

    function updateSliderPos(instant=false) {

        const target = -currentSlideIndex * window.innerWidth; prevTranslate = target; currentTranslate = target;

        const track = document.getElementById('slider-track');

        track.style.transition = instant ? 'none' : 'transform 0.3s ease-out'; track.style.transform = `translateX(${target}px)`;

        document.getElementById('viewer-counter').innerText = `${currentSlideIndex + 1} / ${currentGalleryImages.length}`;

        document.querySelectorAll('.thumbnail-item').forEach((t, i) => i===currentSlideIndex ? t.classList.add('active') : t.classList.remove('active'));

    }

    

    function touchSliderStart(e) { isSliderDragging=true; sliderStartPos = e.type.includes('mouse')?e.pageX:e.touches[0].clientX; prevTranslate = -currentSlideIndex*window.innerWidth; document.getElementById('slider-track').style.transition='none'; animationID=requestAnimationFrame(sliderAnim); }

    function touchSliderMove(e) { if(isSliderDragging){ const pos = e.type.includes('mouse')?e.pageX:e.touches[0].clientX; currentTranslate = prevTranslate + (pos - sliderStartPos); } }

    function touchSliderEnd() { isSliderDragging=false; cancelAnimationFrame(animationID); const moved = currentTranslate - prevTranslate; if(moved < -100 && currentSlideIndex < currentGalleryImages.length-1) currentSlideIndex++; else if(moved > 100 && currentSlideIndex > 0) currentSlideIndex--; updateSliderPos(); }

    function sliderAnim() { document.getElementById('slider-track').style.transform = `translateX(${currentTranslate}px)`; if(isSliderDragging) requestAnimationFrame(sliderAnim); }

    function attachSliderEvents() { const c = document.getElementById('slider-container'); c.addEventListener('touchstart', touchSliderStart); c.addEventListener('touchmove', touchSliderMove, {passive:false}); c.addEventListener('touchend', touchSliderEnd); c.addEventListener('mousedown', touchSliderStart); c.addEventListener('mousemove', touchSliderMove); c.addEventListener('mouseup', touchSliderEnd); c.addEventListener('mouseleave', touchSliderEnd); }

    function detachSliderEvents() { const c = document.getElementById('slider-container'); c.removeEventListener('touchstart', touchSliderStart); c.removeEventListener('touchmove', touchSliderMove); c.removeEventListener('touchend', touchSliderEnd); c.removeEventListener('mousedown', touchSliderStart); c.removeEventListener('mousemove', touchSliderMove); c.removeEventListener('mouseup', touchSliderEnd); }



    async function downloadCurrentImage() { forceDownload(currentGalleryImages[currentSlideIndex], `photo_${Date.now()}.jpg`); }

    async function downloadAllImages() { if(!confirm("確定下載全部？")) return; for(let i=0; i<currentGalleryImages.length; i++){ await forceDownload(currentGalleryImages[i], `batch_${i}.jpg`); await new Promise(r=>setTimeout(r,800)); } alert("已送出"); }

    async function forceDownload(url, filename) { try { const r = await fetch(url, {mode:'cors'}); const b = await r.blob(); const l = document.createElement('a'); l.href = URL.createObjectURL(b); l.download = filename; l.click(); URL.revokeObjectURL(l.href); } catch(e){ window.open(url, '_blank'); } }



    // 設定與登出

    // ==========================================

    // 設定頁面與滿版滑動邏輯

    // ==========================================

    const settingsViewDOM = document.getElementById('settingsView');

    let isSettingsSwiping = false, settingsPageStartX = 0, settingsPageCurrentX = 0;



    settingsViewDOM.addEventListener('touchstart', (e) => {

        if (e.touches[0].clientX < 40) { 

            isSettingsSwiping = true; 

            settingsPageStartX = e.touches[0].clientX; 

            settingsViewDOM.style.transition = 'none'; 

        }

    });

    settingsViewDOM.addEventListener('touchmove', (e) => {

        if (!isSettingsSwiping) return;

        settingsPageCurrentX = e.touches[0].clientX - settingsPageStartX;

        if (settingsPageCurrentX > 0) { e.preventDefault(); settingsViewDOM.style.transform = `translateX(${settingsPageCurrentX}px)`; }

    }, { passive: false });

    settingsViewDOM.addEventListener('touchend', (e) => {

        if (!isSettingsSwiping) return;

        isSettingsSwiping = false;

        settingsViewDOM.style.transition = 'transform 0.3s cubic-bezier(0.25, 0.8, 0.25, 1)';

        if (settingsPageCurrentX > window.innerWidth / 3 || settingsPageCurrentX > 30) {

            closeSettings();

        } else {

            settingsViewDOM.style.transform = 'translateX(0%)';

        }

        settingsPageCurrentX = 0;

    });



    async function openSettings() { 

        settingsViewDOM.style.transform = 'translateX(0%)'; 

        document.getElementById('settingsAvatar').src = document.getElementById('myAvatar').src;

        document.getElementById('settingsName').innerText = document.getElementById('myName').innerText;

        document.getElementById('settingsUid').innerText = `UID: ${auth.currentUser.uid}`;

        document.getElementById('settingsEmail').innerText = auth.currentUser.email || '讀取中...';

        

        try {

            const doc = await db.collection("act").doc(auth.currentUser.uid).get();

            if (doc.exists && doc.data().email) {

                document.getElementById('settingsEmail').innerText = doc.data().email;

            }

        } catch(e) { console.error('無法讀取 act 中的 email', e); }

    }

    

    function closeSettings() { settingsViewDOM.style.transform = 'translateX(100%)'; }

    function handleLogout() { if(confirm("確定登出？")) auth.signOut().then(()=>window.location.href="index.html"); }



    // 【新增】：修改信箱功能

    async function handleChangeEmail() {

        const user = auth.currentUser;

        if (!user) return;

        const newEmail = prompt("請輸入新的信箱地址：", user.email);

        if (!newEmail || newEmail === user.email) return;

        

        try {

            await user.updateEmail(newEmail);

            // 同步更新 act 集合裡的資料

            await db.collection("act").doc(user.uid).update({ email: newEmail });

            document.getElementById('settingsEmail').innerText = newEmail;

            alert("信箱修改成功！");

        } catch(e) {

            // 如果超過一段時間未登入，Firebase 會要求重新驗證

            if (e.code === 'auth/requires-recent-login') {

                alert("為了安全起見，修改信箱需要您重新登入。請登出後再試一次。");

            } else {

                alert("修改失敗：" + e.message);

            }

        }

    }



    // 【新增】：修改密碼功能

    async function handleChangePassword() {

        const user = auth.currentUser;

        if (!user) return;

        const newPassword = prompt("請輸入新的密碼 (至少 6 個字元)：");

        if (!newPassword) return;

        if (newPassword.length < 6) return alert("密碼長度至少需要 6 個字元！");

        

        try {

            await user.updatePassword(newPassword);

            alert("密碼修改成功！下次請使用新密碼登入。");

        } catch(e) {

            if (e.code === 'auth/requires-recent-login') {

                alert("為了安全起見，修改密碼需要您重新登入。請登出後再試一次。");

            } else {

                alert("修改失敗：" + e.message);

            }

        }

    }



    // --- 複製訊息功能 ---



    // --- 複製訊息功能 ---

    function copyMessage() {

        if (!activeMsgTextForMenu) return;

        // 移除可能殘留的 HTML 實體 (例如把 &quot; 轉回 ")

        const unescapedText = activeMsgTextForMenu.replace(/&quot;/g, '"').replace(/&#039;/g, "'").replace(/&lt;/g, '<').replace(/&gt;/g, '>');

        

        if (navigator.clipboard && window.isSecureContext) {

            navigator.clipboard.writeText(unescapedText).then(() => {

                closeContextMenu();

            });

        } else {

            // 舊版瀏覽器與 iOS webview 的備用複製方案

            const textArea = document.createElement("textarea");

            textArea.value = unescapedText;

            document.body.appendChild(textArea);

            textArea.select();

            try { document.execCommand('copy'); } catch (err) {}

            textArea.remove();

            closeContextMenu();

        }

    }



    // --- 設定頁面：上傳/更新個人大頭貼 ---

    async function handleSettingsAvatarChange(e) {

        const file = e.target.files[0];

        if (!file) return;

        const user = auth.currentUser;

        if (!user) return;



        try {

            // 1. UI 顯示載入中

            const avatarImg = document.getElementById('settingsAvatar');

            avatarImg.style.opacity = '0.5';



            // 2. 使用既有的前端函數壓縮圖片至 1MB 以下 (傳入 1 代表 1MB)

            const compressedFile = await compressImageClientSide(file, 1);



            // 3. 取得舊照片網址，準備稍後刪除

            const userDocRef = db.collection("act").doc(user.uid);

            const userDoc = await userDocRef.get();

            const oldPhotoUrl = userDoc.exists ? userDoc.data().photoURL : null;



            // 4. 上傳到 Storage 的 avatars 資料夾

            const snap = await storage.ref(`avatars/${user.uid}_${Date.now()}.jpg`).put(compressedFile);

            const newUrl = await snap.ref.getDownloadURL();



            // 5. 更新 Firestore act 集合

            await userDocRef.update({ photoURL: newUrl });



            // 6. 刪除舊照片 (加上防呆機制：只刪除 Storage 中 avatars 資料夾內的圖片)

            if (oldPhotoUrl && oldPhotoUrl.includes('firebasestorage') && oldPhotoUrl.includes('%2Favatars%2F')) {

                try { await storage.refFromURL(oldPhotoUrl).delete(); } catch(err) { console.warn('刪除舊頭像失敗', err); }

            }



            // 7. 更新畫面顯示

            avatarImg.src = newUrl;

            document.getElementById('myAvatar').src = newUrl;

            avatarImg.style.opacity = '1';

            

        } catch (error) {

            console.error("更新大頭貼失敗:", error);

            alert("圖片上傳失敗，請重試");

            document.getElementById('settingsAvatar').style.opacity = '1';

        }

        e.target.value = '';

    }





    // ==========================================

    // 移植：活動發起系統核心邏輯 (Event Creator)

    // ==========================================

    let evSelectedUIDs = [];

    let evAuthorizedUIDs = [];

    let evPickerMode = 'participants'; 

    let evTempSelectedUIDs = [];

    let isEvPrivate = false;



    function openCreateEventModal() {

        closeModal('activityScheduleModal'); // 關閉行程總覽

        currentEditingEventId = null; // 確保是新增模式

        document.querySelector('#createEventModal h3').innerHTML = '<i class="fas fa-calendar-plus"></i> 發起新活動';

        document.getElementById('btnSubmitEvent').innerHTML = '確認發布';



        // 重置表單

        document.getElementById('ev-title').value = '';

        document.getElementById('ev-deadline').value = '';

        document.getElementById('dynamic-fields-container').innerHTML = '';

        document.getElementById('ev-is-private').checked = false;

        toggleEventPrivateMode(false);

        evSelectedUIDs = [];

        evAuthorizedUIDs = [];

        renderEvMemberPreview(evSelectedUIDs, 'participants');

        renderEvMemberPreview(evAuthorizedUIDs, 'authorized');

        openModal('createEventModal');

    }



    function toggleEventPrivateMode(isPrivate) {

        isEvPrivate = isPrivate;

        const pubSec = document.getElementById('public-participants-section');

        const privSec = document.getElementById('private-auth-section');

        if (isPrivate) {

            pubSec.style.display = 'none';

            privSec.style.display = 'block';

            // 預設將自己加入授權名單

            if (auth.currentUser && !evAuthorizedUIDs.includes(auth.currentUser.uid)) {

                evAuthorizedUIDs.push(auth.currentUser.uid);

                renderEvMemberPreview(evAuthorizedUIDs, 'authorized');

            }

        } else {

            pubSec.style.display = 'block';

            privSec.style.display = 'none';

        }

    }



    function addEventField(type, label) {

        const container = document.getElementById('dynamic-fields-container');

        if (type === 'image_upload' && container.querySelector('.draggable-item[data-type="image_upload"]')) {

            return alert("「活動照片」欄位已存在，可在內部多選圖片。");

        }

        const fieldId = Date.now();

        let inputHtml = '';

        if (type === 'image_upload') {

            inputHtml = `

                <div style="flex-grow: 1; min-width: 0; display: flex; flex-direction: column;">

                    <label style="font-size:0.85em; color:var(--ly-blue); font-weight:bold; margin-bottom:5px;">${label} (可多選)</label>

                    <input type="file" class="ev-dynamic-file" accept="image/*" multiple onchange="handleEvDynamicImage(this, '${fieldId}')" style="width: 100%; margin: 0; font-size:16px;">

                    <div id="preview-${fieldId}" class="image-field-preview"></div>

                    <input type="hidden" class="ev-dynamic-input" id="hidden-${fieldId}" data-field-type="images_json">

                </div>`;

        } else if (type === 'time' || type === 'meetup') {

            inputHtml = `

                <div style="flex-grow: 1; min-width: 0; display: flex; flex-direction: column;">

                    <label style="font-size:0.85em; color:var(--ly-blue); font-weight:bold; margin-bottom:5px;">${label}</label>

                    <input type="datetime-local" class="ev-dynamic-input" style="width: 100%; margin: 0;">

                </div>`;

        } else {

            inputHtml = `

                <div style="flex-grow: 1; min-width: 0; display: flex; flex-direction: column;">

                    <label style="font-size:0.85em; color:var(--ly-blue); font-weight:bold; margin-bottom:5px;">${label}</label>

                    <textarea class="ev-dynamic-input" placeholder="請輸入${label}" style="width: 100%; margin: 0; min-height: 80px; resize: vertical; padding: 10px;"></textarea>

                </div>`;

        }

        const html = `

            <div class="draggable-item" data-type="${type}" id="field-${fieldId}">

                <i class="fas fa-grip-lines drag-handle"></i>

                ${inputHtml}

                <button class="btn-remove-field" onclick="this.closest('.draggable-item').remove()" title="刪除"><i class="fas fa-trash-alt"></i></button>

            </div>`;

        container.insertAdjacentHTML('beforeend', html);

        initEvSortable();

        if (type === 'image_upload') initEvImageSortable(fieldId);

    }



    function initEvSortable() {

        if (typeof Sortable !== 'undefined') {

            const el = document.getElementById('dynamic-fields-container');

            if (el && !el.sortableInst) el.sortableInst = Sortable.create(el, { handle: '.drag-handle', delay: 0 });

        }

    }

    function initEvImageSortable(fieldId) {

        if (typeof Sortable !== 'undefined') {

            const el = document.getElementById(`preview-${fieldId}`);

            const hidden = document.getElementById(`hidden-${fieldId}`);

            if (el) Sortable.create(el, {

                animation: 150, onEnd: function() {

                    const newOrder = [];

                    el.querySelectorAll('.preview-card img').forEach(img => newOrder.push(img.src));

                    hidden.value = JSON.stringify(newOrder);

                }

            });

        }

    }



    async function handleEvDynamicImage(input, fieldId) {

        const hiddenInput = document.getElementById(`hidden-${fieldId}`);

        let currentData = [];

        try { if(hiddenInput.value) currentData = JSON.parse(hiddenInput.value); } catch(e){}

        if (!Array.isArray(currentData)) currentData = currentData ? [currentData] : [];



        if (input.files && input.files.length > 0) {

            for (const file of Array.from(input.files)) {

                // 使用原有的前端壓縮函數 (壓至 2MB 以下)

                const compressed = await compressImageClientSide(file, 2); 

                const reader = new FileReader();

                const base64 = await new Promise(resolve => {

                    reader.onload = e => resolve(e.target.result);

                    reader.readAsDataURL(compressed);

                });

                currentData.push(base64);

            }

            updateEvImageField(fieldId, currentData);

        }

        input.value = '';

    }



    function updateEvImageField(fieldId, dataArray) {

        document.getElementById(`hidden-${fieldId}`).value = JSON.stringify(dataArray);

        const container = document.getElementById(`preview-${fieldId}`);

        container.innerHTML = "";

        dataArray.forEach((src, idx) => {

            const card = document.createElement('div');

            card.className = 'preview-card';

            card.innerHTML = `<img src="${src}"><div class="preview-remove-btn"><i class="fas fa-trash"></i></div>`;

            card.querySelector('.preview-remove-btn').onclick = (e) => {

                e.stopPropagation();

                if(confirm("移除此照片？")) {

                    dataArray.splice(idx, 1);

                    updateEvImageField(fieldId, dataArray);

                }

            };

            container.appendChild(card);

        });

        initEvImageSortable(fieldId);

    }



    async function openEventMemberPicker(mode) {

        evPickerMode = mode;

        const grid = document.getElementById('ev-picker-grid');

        grid.innerHTML = '<div style="text-align:center; padding:20px;"><i class="fas fa-spinner fa-spin"></i></div>';

        openModal('eventMemberPickerModal');

        

        document.getElementById('evPickerTitle').innerHTML = mode === 'authorized' ? '<i class="fas fa-user-lock"></i> 授權成員' : '<i class="fas fa-users"></i> 參加成員';

        evTempSelectedUIDs = mode === 'authorized' ? [...evAuthorizedUIDs] : [...evSelectedUIDs];



        try {

            const snap = await db.collection("act").get();

            let html = '';

            snap.forEach(doc => {

                const d = doc.data();

                if (mode === 'authorized' && doc.id === auth.currentUser.uid) {

                    if (!evTempSelectedUIDs.includes(doc.id)) evTempSelectedUIDs.push(doc.id);

                }

                const isChecked = evTempSelectedUIDs.includes(doc.id) ? 'checked' : '';

                html += `

                <label class="user-select-item">

                    <input type="checkbox" value="${doc.id}" class="ev-member-checkbox" ${isChecked} onchange="toggleEvTempMember(this)">

                    <div class="user-select-card">

                        <img src="${d.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'}">

                        <span class="ev-picker-name">${d.displayName || '未知'}</span>

                    </div>

                </label>`;

            });

            grid.innerHTML = html;

        } catch(e) { grid.innerHTML="讀取失敗"; }

    }



    function toggleEvTempMember(checkbox) {

        if (checkbox.checked) {

            if (!evTempSelectedUIDs.includes(checkbox.value)) evTempSelectedUIDs.push(checkbox.value);

        } else {

            evTempSelectedUIDs = evTempSelectedUIDs.filter(id => id !== checkbox.value);

        }

    }



    function toggleAllEventMembers(isSelect) {

        document.querySelectorAll('.ev-member-checkbox').forEach(cb => {

            cb.checked = isSelect;

            toggleEvTempMember(cb);

        });

    }



    function searchEventMember() {

        const kw = document.getElementById('ev-member-search').value.toLowerCase();

        document.querySelectorAll('#ev-picker-grid .user-select-item').forEach(item => {

            const name = item.querySelector('.ev-picker-name').innerText.toLowerCase();

            item.style.display = name.includes(kw) ? 'block' : 'none';

        });

    }



    async function saveEventSelectedMembers() {

        if (evPickerMode === 'authorized') {

            evAuthorizedUIDs = [...evTempSelectedUIDs];

            renderEvMemberPreview(evAuthorizedUIDs, 'authorized');

        } else {

            evSelectedUIDs = [...evTempSelectedUIDs];

            renderEvMemberPreview(evSelectedUIDs, 'participants');

        }

        closeModal('eventMemberPickerModal');

    }



    async function renderEvMemberPreview(uids, type) {

        const container = document.getElementById(type === 'authorized' ? 'authorized-members-preview' : 'selected-members-preview');

        if (uids.length === 0) return container.innerHTML = '<p style="color:#999; margin:0;">無</p>';

        container.innerHTML = '載入中...';

        try {

            const promises = uids.map(uid => db.collection("act").doc(uid).get());

            const docs = await Promise.all(promises);

            container.innerHTML = docs.map(d => {

                if(!d.exists) return '';

                const data = d.data();

                return `<div class="user-select-card" style="width: 70px; padding: 5px; background: white;">

                    <img src="${data.photoURL || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true'}" style="width:35px;height:35px;margin:0 0 5px 0;">

                    <span style="font-size:11px;">${data.displayName}</span>

                </div>`;

            }).join('');

        } catch(e) {}

    }



    async function submitNewEvent() {

        const title = document.getElementById('ev-title').value.trim();

        const deadline = document.getElementById('ev-deadline').value;

        if (!title) return alert("活動標題為必填項目");



        const btn = document.getElementById('btnSubmitEvent');

        btn.disabled = true; btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> 發布中...';



        try {

            const dynamicFields = [];

            const items = document.querySelectorAll('#dynamic-fields-container .draggable-item');

            for (const it of items) {

                const type = it.getAttribute('data-type');

                const label = it.querySelector('label').innerText.replace(/ \(.*\)/, '');

                let val = it.querySelector('.ev-dynamic-input').value;

                

                // 若為圖片，將 Base64 上傳至 Storage 換取 URL

                if (type === 'image_upload' && val) {

                    let images = JSON.parse(val);

                    for (const b64 of images) {

                        const name = `events/${Date.now()}_${Math.random().toString(36).substr(2, 9)}.jpg`;

                        const snap = await storage.ref(name).putString(b64, 'data_url');

                        const url = await snap.ref.getDownloadURL();

                        dynamicFields.push({ type, label, content: url });

                    }

                    continue;

                }

                dynamicFields.push({ type, label, content: val });

            }



            // 準備參與者/授權者名字 (從資料庫抓取以維持乾淨的 Array)

            const namesPromise = isEvPrivate ? evAuthorizedUIDs : evSelectedUIDs;

            const nameDocs = await Promise.all(namesPromise.map(id => db.collection("act").doc(id).get()));

            const names = nameDocs.map(d => d.exists ? d.data().displayName : '');



            const eventData = {

                title, deadline, fields: dynamicFields,

                participants: evSelectedUIDs, participantNames: isEvPrivate ? [] : names,

                isPrivate: isEvPrivate,

                authorizedMembers: isEvPrivate ? evAuthorizedUIDs : [],

                authorizedMemberNames: isEvPrivate ? names : [],

                creator: auth.currentUser.email || '',

                creatorUid: auth.currentUser.uid // 【關鍵修復】：加入絕對唯一且不會空的 UID 識別

            };



            if (currentEditingEventId) {

                // 【編輯模式】：更新資料，不發送新卡片

                eventData.updatedAt = firebase.firestore.FieldValue.serverTimestamp();

                await db.collection("events").doc(currentEditingEventId).update(eventData);

                closeModal('createEventModal');

                alert("活動修改成功！");

                handleActivitySchedule(); // 重新打開活動總覽刷新清單

            } else {

                // 【新增模式】：寫入集合並發送卡片

                eventData.createdAt = firebase.firestore.FieldValue.serverTimestamp();

                const newDoc = await db.collection("events").add(eventData);



                if (currentChatId) {

                    const batch = db.batch(), chatRef = db.collection("chats").doc(currentChatId), msgRef = chatRef.collection("messages").doc();

                    const timeStr = new Date().toLocaleTimeString('zh-TW', {hour: '2-digit', minute:'2-digit', hour12: false});

                    batch.set(msgRef, { text: JSON.stringify({ type:'event_share', eventId:newDoc.id, title:title, timeStr:timeStr, deadline:deadline }), senderId: auth.currentUser.uid, timestamp: firebase.firestore.FieldValue.serverTimestamp() });

                    batch.update(chatRef, { lastMessage: `分享活動：${title}`, lastSenderId: auth.currentUser.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp(), [`lastReadAt.${auth.currentUser.uid}`]: firebase.firestore.FieldValue.serverTimestamp() });

                    await batch.commit();

                    scrollToBottom(true);

                }

                closeModal('createEventModal');

                alert("活動建立成功並已發布至聊天室！");

                handleActivitySchedule(); // 【新增】：建立完成後，自動為你重新打開「活動行程總覽」

            }

        } catch(e) { alert("操作失敗: " + e.message); }

        finally {

            btn.disabled = false; btn.innerHTML = '確認發布';

        }

    }

    // ==========================================

    // 8. 團購系統核心邏輯

    // ==========================================

    let gbItemCount = 0;



    // 【新增】：打開團購清單視窗

    // 【修改】：打開團購清單視窗並支援左滑

    async function openGroupBuyListModal() {

        document.getElementById('expandMenuPanel').style.display = 'none';

        document.querySelector('.expand-menu-btn').style.transform = 'rotate(0deg)';

        

        openModal('groupBuyListModal');

        const listEl = document.getElementById('groupBuyListContainer');

        listEl.innerHTML = '<div style="text-align:center; padding:30px; color:#999;"><i class="fas fa-spinner fa-spin fa-2x"></i><br><br>載入團購資料中...</div>';

        

        try {

            const snap = await db.collection("group-orders").orderBy("createdAt", "desc").get();

            if (snap.empty) {

                listEl.innerHTML = '<div class="event-empty-state">尚無團購訂單，點擊下方新增。</div>';

                return;

            }

            

            let html = '';

            snap.forEach(doc => {

                const data = doc.data();

                const deadlineStr = data.deadline ? data.deadline.replace('T', ' ') : '無期限';

                const safeTitle = (data.title || '').replace(/'/g, "\\'").replace(/"/g, '&quot;');

                const safeInitiator = (data.initiatorName || '').replace(/'/g, "\\'").replace(/"/g, '&quot;');

                

                // 判斷當前使用者是否為該訂單的發起人

                const isInitiator = data.initiatorId === auth.currentUser.uid;

                

                const innerContent = `

                    <h4 class="event-mini-title">${data.title}</h4>

                    <div class="event-mini-meta">

                        <span style="color:var(--ly-blue); font-weight:bold;">[ 代購: ${data.initiatorName} ]</span>

                    </div>

                    <div class="event-mini-meta" style="margin-top: 4px; color: #d93025;">

                        <i class="far fa-clock" style="color: #d93025;"></i> 截止: ${deadlineStr}

                    </div>

                `;



                if (isInitiator) {

                    // 如果是發起人，加入左滑外掛結構

                    html += `

                    <div class="swipe-container">

                        <div class="swipe-actions">

                            <button class="swipe-delete-btn" onclick="deleteGroupBuy('${doc.id}')"><i class="fas fa-trash-alt"></i></button>

                        </div>

                        <div class="swipe-content event-mini-item" style="border-left-color: #28a745; margin-bottom: 0;"

                             onmousedown="gbSwipeStart(event, this)" 

                             onmousemove="gbSwipeMove(event, this)" 

                             onmouseup="gbSwipeEnd(event, this, '${doc.id}', '${safeTitle}', '${safeInitiator}', '${data.deadline}')" 

                             onmouseleave="gbSwipeEnd(event, this, '${doc.id}', '${safeTitle}', '${safeInitiator}', '${data.deadline}')" 

                             ontouchstart="gbSwipeStart(event, this)" 

                             ontouchmove="gbSwipeMove(event, this)" 

                             ontouchend="gbSwipeEnd(event, this, '${doc.id}', '${safeTitle}', '${safeInitiator}', '${data.deadline}')">

                            ${innerContent}

                        </div>

                    </div>`;

                } else {

                    // 不是發起人，保留一般的點擊卡片

                    html += `

                    <div class="event-mini-item" style="border-left-color: #28a745;" onclick="sendGroupBuyCard('${doc.id}', '${safeTitle}', '${safeInitiator}', '${data.deadline}')">

                        ${innerContent}

                    </div>`;

                }

            });

            listEl.innerHTML = html;

        } catch (e) {

            console.error(e);

            listEl.innerHTML = '<div style="text-align:center; padding:20px; color:red;">載入失敗，請檢查網路狀態。</div>';

        }

    }



    // 【新增】：點擊清單中的團購，發布到聊天室

    async function sendGroupBuyCard(orderId, title, initiator, deadline) {

        if(!confirm(`要將團購「${title}」發布到聊天室中嗎？`)) return; 

        closeModal('groupBuyListModal');

        try {

            const batch = db.batch();

            const chatRef = db.collection("chats").doc(currentChatId);

            const msgRef = chatRef.collection("messages").doc();

            

            const payload = {

                type: 'group_buy', orderId: orderId, title: title, initiator: initiator, deadline: deadline

            };



            batch.set(msgRef, { text: JSON.stringify(payload), senderId: auth.currentUser.uid, timestamp: firebase.firestore.FieldValue.serverTimestamp() });

            batch.update(chatRef, { lastMessage: `發起了團購：${title}`, lastSenderId: auth.currentUser.uid, updatedAt: firebase.firestore.FieldValue.serverTimestamp(), [`lastReadAt.${auth.currentUser.uid}`]: firebase.firestore.FieldValue.serverTimestamp() });

            

            await batch.commit(); 

            scrollToBottom(true);

        } catch(e){ alert("發布失敗"); }

    }



    // 【修改】：原本的新增團購函式，加入關閉清單視窗的動作

    function openCreateGroupBuyModal() {

        closeModal('groupBuyListModal'); // 如果是從清單打開的，先把清單關掉

        document.getElementById('expandMenuPanel').style.display = 'none';

        document.querySelector('.expand-menu-btn').style.transform = 'rotate(0deg)';

        

        // 自動帶入當前成員名稱

        document.getElementById('gbInitiator').value = document.getElementById('myName').innerText;

        document.getElementById('gbTitle').value = '';

        document.getElementById('gbDeadline').value = '';

        document.getElementById('gbItemsContainer').innerHTML = '';

        gbItemCount = 0;

        addGroupBuyItemRow(); // 預設先給一行

        openModal('createGroupBuyModal');

    }



    function addGroupBuyItemRow() {

        const container = document.getElementById('gbItemsContainer');

        const rowId = `gb-item-${gbItemCount++}`;

        const html = `

            <div class="gb-item-row" id="${rowId}">

                <div onclick="document.getElementById('${rowId}').remove()" class="btn-remove-item"><i class="fas fa-times"></i></div>

                <label class="img-upload-btn">

                    <i class="fas fa-camera"></i> 圖片

                    <input type="file" accept="image/*" style="display:none;" onchange="previewGBImage(this, '${rowId}')">

                </label>

                <img id="img-prev-${rowId}">

                <input type="text" class="gb-i-name" placeholder="商品名稱" style="flex: 2;">

                <input type="number" class="gb-i-price" placeholder="單價金額" style="flex: 1;">

                <input type="number" class="gb-i-qty" placeholder="數量限制(選填)" style="flex: 1;">

                <input type="text" class="gb-i-note" placeholder="備註說明(選填)" style="flex-basis: 100%; box-sizing: border-box;">

            </div>

        `;

        container.insertAdjacentHTML('beforeend', html);

    }



    function previewGBImage(input, rowId) {

        const file = input.files[0];

        if (file) {

            const reader = new FileReader();

            reader.onload = (e) => {

                const img = document.getElementById(`img-prev-${rowId}`);

                img.src = e.target.result;

                img.style.display = 'block';

                input.fileObj = file; 

            };

            reader.readAsDataURL(file);

        }

    }



    async function submitGroupBuy() {

        const title = document.getElementById('gbTitle').value.trim();

        const deadline = document.getElementById('gbDeadline').value;

        const initiatorName = document.getElementById('gbInitiator').value;

        

        if (!title || !deadline) return alert('請完整填寫「團購標題」與「截止時間」！');

        

        const rows = document.querySelectorAll('.gb-item-row');

        if (rows.length === 0) return alert('請至少新增一項商品！');



        const btn = document.getElementById('gbSubmitBtn');

        btn.disabled = true;

        btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> 建立中...';



        try {

            let items = [];

            for (let i = 0; i < rows.length; i++) {

                const row = rows[i];

                const name = row.querySelector('.gb-i-name').value.trim();

                const price = row.querySelector('.gb-i-price').value.trim();

                const qty = row.querySelector('.gb-i-qty').value.trim();

                const note = row.querySelector('.gb-i-note').value.trim();

                const fileInput = row.querySelector('input[type="file"]');

                

                if (!name || !price) throw new Error('每項商品的「名稱」與「金額」為必填項目。');



                let imgUrl = "";

                // 如果有上傳圖片，使用「團購專用極致壓縮」並上傳

                if (fileInput.fileObj) {

                    const compressedFile = await compressGroupBuyImage(fileInput.fileObj); // 壓縮到極小化 (約 50~150KB)

                    const snap = await storage.ref(`group-orders/${Date.now()}_${i}.jpg`).put(compressedFile);

                    imgUrl = await snap.ref.getDownloadURL();

                }



                items.push({

                    id: `item_${Date.now()}_${i}`,

                    name, 

                    price: Number(price), 

                    quantity: qty, 

                    note, 

                    imgUrl

                });

            }



            // 1. 寫入 group-orders 集合

            const newOrderRef = db.collection('group-orders').doc();

            await newOrderRef.set({

                title,

                initiatorId: auth.currentUser.uid,

                initiatorName,

                deadline,

                items,

                createdAt: firebase.firestore.FieldValue.serverTimestamp()

            });



            // 2. 傳送訊息卡片到當前聊天室

            const batch = db.batch();

            const chatRef = db.collection("chats").doc(currentChatId);

            const msgRef = chatRef.collection("messages").doc();

            

            const payload = {

                type: 'group_buy',

                orderId: newOrderRef.id,

                title: title,

                initiator: initiatorName,

                deadline: deadline

            };



            batch.set(msgRef, { 

                text: JSON.stringify(payload), 

                senderId: auth.currentUser.uid, 

                timestamp: firebase.firestore.FieldValue.serverTimestamp() 

            });

            batch.update(chatRef, { 

                lastMessage: `發起了團購：${title}`, 

                lastSenderId: auth.currentUser.uid, 

                updatedAt: firebase.firestore.FieldValue.serverTimestamp(), 

                [`lastReadAt.${auth.currentUser.uid}`]: firebase.firestore.FieldValue.serverTimestamp() 

            });

            

            await batch.commit();

            closeModal('createGroupBuyModal');

            scrollToBottom(true);



        } catch(e) {

            alert(e.message || "建立團購失敗，請稍後再試。");

        } finally {

            btn.disabled = false;

            btn.innerText = '發布到聊天室';

        }

    }



    // --- 查看與登記團購 ---

    // --- 查看與登記團購 ---

    let currentGroupBuyId = null;

    let currentGroupBuyItems = [];

    

    async function openGroupBuyDetail(orderId) {

        currentGroupBuyId = orderId;

        openModal('viewGroupBuyModal');

        document.getElementById('vgbItemsList').innerHTML = '<div style="text-align:center; padding:30px; color:#999;"><i class="fas fa-spinner fa-spin fa-2x"></i><br><br>載入表單中...</div>';

        

        try {

            // 1. 抓取團購主檔

            const doc = await db.collection('group-orders').doc(orderId).get();

            if (!doc.exists) {

                document.getElementById('vgbItemsList').innerHTML = '<div style="text-align:center; padding:20px; color:red;">此團購單已被刪除。</div>';

                return;

            }

            const data = doc.data();

            document.getElementById('vgbTitle').innerText = data.title;

            document.getElementById('vgbInitiator').innerText = data.initiatorName;

            document.getElementById('vgbDeadline').innerText = data.deadline.replace('T', ' ');

            currentGroupBuyItems = data.items || [];



            const isExpired = new Date(data.deadline).getTime() < Date.now();

            const saveBtn = document.getElementById('vgbSaveBtn');

            if (isExpired) {

                saveBtn.disabled = true; saveBtn.innerText = "登記已截止"; saveBtn.style.background = "#ccc";

            } else {

                saveBtn.disabled = false; saveBtn.innerText = "儲存我的訂單"; saveBtn.style.background = "var(--ly-blue)";

            }



            // 2. 抓取「所有成員」的登記紀錄 (為了呈現分支清單)

            const ordersSnap = await db.collection('group-orders').doc(orderId).collection('orders').get();

            

            let allOrdersByItem = {};

            let myOrdersMap = {};



            // 整理大家點的資料

            ordersSnap.forEach(oDoc => {

                const oData = oDoc.data();

                const uid = oDoc.id;

                const userItems = oData.items || {};

                

                if (uid === auth.currentUser.uid) myOrdersMap = userItems;



                for (let iId in userItems) {

                    if (!allOrdersByItem[iId]) allOrdersByItem[iId] = [];

                    

                    let qty = 0, note = "";

                    // 兼容舊資料 (純數字) 與新資料 (包含備註的物件)

                    if (typeof userItems[iId] === 'number') {

                        qty = userItems[iId];

                    } else {

                        qty = userItems[iId].qty || 0;

                        note = userItems[iId].note || "";

                    }



                    if (qty > 0) {

                        allOrdersByItem[iId].push({

                            userName: oData.userName || "成員",

                            qty: qty,

                            note: note

                        });

                    }

                }

            });



            // 3. 渲染商品、備註輸入框、以及大家的分支清單

            let html = '';

            currentGroupBuyItems.forEach(item => {

                // 解析我自己的訂單資料

                let myQty = 0, myNote = "";

                if (myOrdersMap[item.id]) {

                    if (typeof myOrdersMap[item.id] === 'number') myQty = myOrdersMap[item.id];

                    else { myQty = myOrdersMap[item.id].qty; myNote = myOrdersMap[item.id].note || ""; }

                }



                const imgSrc = item.imgUrl || 'https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true';

                

                // 組合大家的分支訂單清單 HTML

                let buyersHtml = '';

                let totalItemQty = 0;

                if (allOrdersByItem[item.id] && allOrdersByItem[item.id].length > 0) {

                    buyersHtml += `<div class="gb-buyers-wrapper">`;

                    allOrdersByItem[item.id].forEach(buyer => {

                        buyersHtml += `

                            <div class="gb-buyer-row">

                                <span class="gb-buyer-name">${buyer.userName}</span>

                                <span class="gb-buyer-note">${buyer.note ? `(備註：${buyer.note})` : ''}</span>

                                <span class="gb-buyer-qty">數量：${buyer.qty}</span>

                            </div>

                        `;

                        totalItemQty += buyer.qty;

                    });

                    // 如果有人買，最底下顯示總計

                    buyersHtml += `<div class="gb-buyer-row" style="background:#f1f3f5; font-weight:bold; justify-content:flex-end;">🔥 總計：${totalItemQty} 件</div>`;

                    buyersHtml += `</div>`;

                }



                html += `

                    <div class="gb-order-item" style="flex-direction: column; align-items: stretch; padding: 15px 0;">

                        <div style="display: flex; justify-content: space-between; align-items: center;">

                            <img src="${imgSrc}" onclick="if('${item.imgUrl}') openImageModal('${item.imgUrl}')" style="width:55px; height:55px; border-radius:8px; object-fit:cover; border:1px solid #ddd;">

                            <div class="gb-order-info" style="flex:1; padding:0 15px;">

                                <h4 style="margin:0 0 5px 0; font-size:1rem; color:var(--ly-blue); font-weight:800;">${item.name}</h4>

                                <p style="margin:0; font-size:0.9rem; color:#555; font-weight:600;">$${item.price} ${item.quantity ? `<span style="color:#d93025; font-size:12px;">(限量 ${item.quantity})</span>` : ''}</p>

                                ${item.note ? `<p style="font-size:0.75rem; color:#888; margin-top:4px; background:#f4f4f4; padding:2px 6px; border-radius:4px;">${item.note}</p>` : ''}

                            </div>

                            <div class="gb-qty-controls" style="display:flex; align-items:center; gap:12px;">

                                <button class="gb-qty-btn" onclick="changeGbQty('${item.id}', -1)" ${isExpired ? 'disabled' : ''}>-</button>

                                <span id="qty-${item.id}" style="font-weight:900; font-size:16px; min-width:20px; text-align:center;">${myQty}</span>

                                <button class="gb-qty-btn" onclick="changeGbQty('${item.id}', 1)" ${isExpired ? 'disabled' : ''}>+</button>

                            </div>

                        </div>

                        

                        <input type="text" id="note-${item.id}" class="gb-my-note-input" placeholder="✏️ LIMIT OVER 有多少買多少" value="${myNote}" style="display: ${myQty > 0 ? 'block' : 'none'};" ${isExpired ? 'disabled' : ''}>



                        ${buyersHtml}

                    </div>

                `;

            });

            

            if (currentGroupBuyItems.length === 0) html = '<div style="text-align:center;">無商品項目</div>';

            document.getElementById('vgbItemsList').innerHTML = html;



        } catch (e) {

            document.getElementById('vgbItemsList').innerHTML = '<div style="text-align:center; padding:20px; color:red;">讀取失敗，請檢查網路。</div>';

        }

    }



    function changeGbQty(itemId, delta) {

        const span = document.getElementById(`qty-${itemId}`);

        const noteInput = document.getElementById(`note-${itemId}`);

        

        let val = parseInt(span.innerText) + delta;

        if (val < 0) val = 0;

        span.innerText = val;

        

        // 【新增】：如果數量大於 0，彈出備註輸入框；若歸零則隱藏並清空

        if (val > 0) {

            noteInput.style.display = 'block';

        } else {

            noteInput.style.display = 'none';

            noteInput.value = '';

        }

    }



    async function saveGroupBuyOrder() {

        if (!currentGroupBuyId) return;

        const btn = document.getElementById('vgbSaveBtn');

        btn.disabled = true; 

        btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> 儲存中...';

        

        try {

            const myOrder = {};

            let totalQty = 0;

            currentGroupBuyItems.forEach(item => {

                const qty = parseInt(document.getElementById(`qty-${item.id}`).innerText);

                const note = document.getElementById(`note-${item.id}`).value.trim();

                

                // 【修改】：將數字改為「包含數量與備註的物件」儲存

                if (qty > 0) {

                    myOrder[item.id] = { qty: qty, note: note };

                    totalQty += qty;

                }

            });



            // 存入子集合

            const orderRef = db.collection('group-orders').doc(currentGroupBuyId).collection('orders').doc(auth.currentUser.uid);

            

            if (totalQty === 0) {

                // 若數量全為 0，直接刪除這份訂單紀錄

                await orderRef.delete(); 

            } else {

                await orderRef.set({

                    userName: document.getElementById('myName').innerText, 

                    items: myOrder,

                    updatedAt: firebase.firestore.FieldValue.serverTimestamp()

                });

            }

            

            alert("✅ 您的訂單已成功儲存！");

            closeModal('viewGroupBuyModal');

        } catch(e) {

            alert("儲存失敗，請重試。");

        } finally {

            btn.disabled = false; 

            btn.innerText = "儲存我的訂單";

        }

    }



    // ==========================================

    // 9. 團購系統：左滑互動與徹底刪除引擎

    // ==========================================

    let gbStartX = 0, gbCurrentX = 0, isGbSwiping = false, gbSwipedOpen = null;



    function gbSwipeStart(e, el) {

        if (gbSwipedOpen && gbSwipedOpen !== el) {

            gbSwipedOpen.style.transform = 'translateX(0px)';

            gbSwipedOpen = null;

        }

        gbStartX = e.type.includes('mouse') ? e.pageX : e.touches[0].clientX;

        gbCurrentX = 0;

        isGbSwiping = true;

        el.style.transition = 'none';

    }



    function gbSwipeMove(e, el) {

        if (!isGbSwiping) return;

        const x = e.type.includes('mouse') ? e.pageX : e.touches[0].clientX;

        gbCurrentX = x - gbStartX;

        // 限制只能向左滑動，且最多滑動 90px

        if (gbCurrentX < 0 && gbCurrentX > -90) {

            el.style.transform = `translateX(${gbCurrentX}px)`;

        }

    }



    function gbSwipeEnd(e, el, orderId, title, initiator, deadline) {

        if (!isGbSwiping) return;

        isGbSwiping = false;

        el.style.transition = 'transform 0.3s ease';

        

        if (gbCurrentX < -45) {

            // 滑動超過一半，將其固定在左側顯示垃圾桶 (-80px)

            el.style.transform = 'translateX(-80px)';

            gbSwipedOpen = el;

        } else {

            // 滑動太小，回彈歸位

            el.style.transform = 'translateX(0px)';

            if (gbSwipedOpen === el) gbSwipedOpen = null;

            

            // 如果幾乎沒滑動 (位移 < 10px)，就當作一般點擊處理 (傳送到聊天室)

            if (Math.abs(gbCurrentX) < 10 && orderId) {

                sendGroupBuyCard(orderId, title, initiator, deadline);

            }

        }

        gbCurrentX = 0;

    }



    // 執行徹底刪除 (主文件、子集合、圖片檔案)

    async function deleteGroupBuy(orderId) {

        if (!confirm("確定要刪除此團購單嗎？\n這將會清除所有成員的訂購紀錄與圖片，且無法復原！")) return;



        const btn = event.currentTarget;

        btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i>';

        btn.disabled = true;



        try {

            // 1. 取得這筆團購的資料，抓出所有的圖片網址進行刪除

            const docSnap = await db.collection("group-orders").doc(orderId).get();

            if (docSnap.exists) {

                const items = docSnap.data().items || [];

                for (let item of items) {

                    if (item.imgUrl) {

                        try { await storage.refFromURL(item.imgUrl).delete(); } 

                        catch(err) { console.warn("圖片已經不存在或刪除失敗"); }

                    }

                }

            }



            // 2. 刪除所有成員在裡面登記的訂單 (子集合 orders)

            const ordersSnap = await db.collection("group-orders").doc(orderId).collection("orders").get();

            if (!ordersSnap.empty) {

                const batch = db.batch();

                ordersSnap.forEach(doc => { batch.delete(doc.ref); });

                await batch.commit();

            }



            // 3. 刪除團購單主文件

            await db.collection("group-orders").doc(orderId).delete();



            alert("✅ 團購單與相關資料已徹底刪除！");

            

            // 重新載入團購清單畫面

            openGroupBuyListModal();

        } catch(e) {

            console.error(e);

            alert("刪除失敗，請檢查網路連線。");

            btn.innerHTML = '<i class="fas fa-trash-alt"></i>';

            btn.disabled = false;

        }

    }



    // ==========================================

    // 10. 多選收回 (Batch Revoke) 核心邏輯

    // ==========================================

    window.isSelectMode = false;

    window.selectedMsgIds = new Set();



    function enterSelectMode() {

        closeContextMenu();

        window.isSelectMode = true;

        window.selectedMsgIds.clear();

        

        document.body.classList.add('select-mode-active');

        document.querySelector('.input-area').style.display = 'none'; // 隱藏輸入框

        document.getElementById('selectActionBar').style.display = 'flex'; // 顯示底部操作列

        

        // 預設將剛剛長按的那一則訊息打勾

        if (activeMsgId) toggleMsgSelect(activeMsgId, true);



        // 【關鍵修復】：等待畫面排版更新後，自動捲動到底部，保證最後一則訊息不被擋住

        setTimeout(() => { scrollToBottom(true); }, 50);

    }



    function cancelSelectMode() {

        window.isSelectMode = false;

        window.selectedMsgIds.clear();

        

        document.body.classList.remove('select-mode-active');

        document.querySelector('.input-area').style.display = 'flex';

        document.getElementById('selectActionBar').style.display = 'none';

        

        // 移除畫面上所有打勾狀態

        document.querySelectorAll('.msg-checkbox').forEach(el => {

            el.classList.remove('checked');

            el.innerHTML = '<i class="far fa-circle"></i>';

        });

    }



    function toggleMsgSelect(msgId, isMine) {

        if (!window.isSelectMode || !isMine) return;

        

        const cb = document.getElementById(`cb-${msgId}`);

        if (!cb) return;



        if (window.selectedMsgIds.has(msgId)) {

            window.selectedMsgIds.delete(msgId);

            cb.classList.remove('checked');

            cb.innerHTML = '<i class="far fa-circle"></i>';

        } else {

            window.selectedMsgIds.add(msgId);

            cb.classList.add('checked');

            cb.innerHTML = '<i class="fas fa-check-circle"></i>';

        }

        

        document.getElementById('selectCountText').innerText = `已選擇 ${window.selectedMsgIds.size} 項`;

    }



    async function confirmMultiRevoke() {

        if (!window.selectedMsgIds || window.selectedMsgIds.size === 0) return alert("請先選擇要收回的訊息");

        if (!confirm(`確定要收回 ${window.selectedMsgIds.size} 則訊息？\n這將從所有人裝置中刪除。`)) return;



        const btn = document.querySelector('#selectActionBar button:last-child');

        btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> 處理中...';

        btn.disabled = true;



        try {

            const batch = db.batch();

            const chatRef = db.collection("chats").doc(currentChatId);

            

            // 抓取所有選擇的訊息，並同時清空 Storage 中的圖片/影片

            const promises = [];

            for (let msgId of window.selectedMsgIds) {

                promises.push(chatRef.collection("messages").doc(msgId).get());

            }

            const msgDocs = await Promise.all(promises);

            

            msgDocs.forEach(doc => {

                if (doc.exists) {

                    if (doc.data().fileUrl) storage.refFromURL(doc.data().fileUrl).delete().catch(()=>{});

                    batch.delete(doc.ref);

                }

            });



            // 重新計算最新的 lastMessage (排除掉被我們刪除的)

            const limitCount = window.selectedMsgIds.size + 1;

            const lastMsgSnap = await chatRef.collection("messages").orderBy("timestamp", "desc").limit(limitCount).get();

            

            let newLastData = null;

            for (let doc of lastMsgSnap.docs) {

                if (!window.selectedMsgIds.has(doc.id)) {

                    newLastData = doc.data();

                    break;

                }

            }



            if (newLastData) {

                let newLastMsg = newLastData.text || (newLastData.fileType === 'video' ? '[影片]' : '[圖片]');

                if (newLastMsg.includes('"type":"event_share"')) newLastMsg = "分享了一個活動";

                if (newLastMsg.includes('"type":"group_buy"')) newLastMsg = "發起了一個團購";

                batch.update(chatRef, { lastMessage: newLastMsg, lastSenderId: newLastData.senderId, updatedAt: newLastData.timestamp });

            } else {

                batch.update(chatRef, { lastMessage: "尚無訊息", lastSenderId: "" });

            }



            await batch.commit();

            cancelSelectMode();

        } catch (e) {

            alert("收回失敗: " + e.message);

        } finally {

            btn.innerHTML = '<i class="fas fa-trash-alt"></i> 收回';

            btn.disabled = false;

        }

    }

    </script>

</body>



    <div id="inAppNotification" style="display:none; position:fixed; top:-100px; left:50%; transform:translateX(-50%); width:90%; max-width:400px; background:rgba(0, 49, 83, 0.95); color:white; padding:15px; border-radius:12px; box-shadow:0 10px 25px rgba(0,0,0,0.2); z-index:999999; align-items:center; gap:15px; cursor:pointer; backdrop-filter:blur(10px); transition:top 0.4s ease, opacity 0.4s ease; opacity:0;">

        <i class="fas fa-bell" style="color: var(--ly-gold); font-size: 1.5rem;"></i>

        <div style="flex:1;">

            <div id="inAppNotifTitle" style="font-weight:bold; font-size:15px; margin-bottom:5px; color:var(--ly-gold);">標題</div>

            <div id="inAppNotifBody" style="font-size:13px; display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden;">內容</div>

        </div>

    </div>

    

</html>


```
