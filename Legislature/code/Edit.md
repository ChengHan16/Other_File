### 在 Messenger.html 網頁版加上建立時間 <br> 為了讓網頁版也能同步看到建立時間，請在 Messenger.html 裡面找到 async function openGroupBuyDetail(orderId) 這個函式，然後將解析與渲染資料的部份替換如下：

找到這段程式碼：

JavaScript
```js
            const data = doc.data();
            document.getElementById('vgbTitle').innerText = data.title;
            document.getElementById('vgbInitiator').innerText = data.initiatorName;
            document.getElementById('vgbDeadline').innerText = data.deadline.replace('T', ' ');
            currentGroupBuyItems = data.items || [];
```
替換成以下這段加上建立時間的版本：

JavaScript
```js
            const data = doc.data();
            document.getElementById('vgbTitle').innerText = data.title;
            document.getElementById('vgbInitiator').innerText = data.initiatorName;
            
            // 👉 抓取建立時間並格式化
            const createdAtStr = data.createdAt ? new Date(data.createdAt.toDate()).toLocaleString('zh-TW', {year:'numeric',month:'2-digit',day:'2-digit',hour:'2-digit',minute:'2-digit',hour12:false}).replace(/\//g,'-') : '未知時間';
            
            // 👉 重新覆寫標頭內容，插入建立時間與紅色的截止時間
            const headerContainer = document.getElementById('vgbInitiator').parentElement.parentElement;
            headerContainer.innerHTML = `
                <div style="margin-bottom: 5px;"><strong><i class="fas fa-user-tag" style="color:var(--ly-gold);"></i> 代購成員：</strong> <span id="vgbInitiator" style="font-weight:bold; color:var(--ly-blue);">${data.initiatorName}</span></div>
                <div style="margin-bottom: 5px;"><strong><i class="far fa-calendar-plus" style="color:var(--ly-gold);"></i> 建立時間：</strong> <span style="font-weight:bold; color:#555;">${createdAtStr}</span></div>
                <div><strong><i class="far fa-clock" style="color:red;"></i> <span style="color:red;">截止時間：</span></strong> <span id="vgbDeadline" style="font-weight:bold; color:#d93025;">${data.deadline.replace('T', ' ')}</span></div>
            `;
            
            currentGroupBuyItems = data.items || [];
```
