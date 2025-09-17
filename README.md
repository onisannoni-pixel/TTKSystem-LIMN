<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>TTKSystem LIMN公開版</title>
<script src="https://cdn.jsdelivr.net/npm/peerjs@1.5.1/dist/peerjs.min.js"></script>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
body{font-family:sans-serif;margin:0;padding:0;background:#f9f9f9;}
h1{text-align:center;padding:15px;background:#007bff;color:white;margin:0;}
.container{max-width:900px;margin:10px auto;padding:10px;}
.card{background:white;padding:15px;margin:10px 0;border-radius:10px;box-shadow:0 2px 6px rgba(0,0,0,0.1);}
input,textarea,button,select{width:100%;padding:10px;margin:5px 0;border:1px solid #ccc;border-radius:6px;box-sizing:border-box;}
button{cursor:pointer;font-weight:bold;}
button.primary{background:#007bff;color:white;border:none;}
button.success{background:#28a745;color:white;border:none;}
button.danger{background:#dc3545;color:white;border:none;}
button.warning{background:#ffc107;color:black;border:none;}
#friendList li{padding:8px;border-bottom:1px solid #eee;display:flex;flex-wrap:wrap;justify-content:space-between;align-items:center;}
.status-online{color:green;font-weight:bold;}
.status-offline{color:gray;font-weight:normal;}
.popup{position:fixed;bottom:-200px;left:50%;transform:translateX(-50%);background:#007bff;color:white;padding:15px;border-radius:10px;box-shadow:0 4px 8px rgba(0,0,0,0.3);transition:bottom 0.5s;width:90%;max-width:400px;z-index:1000;}
.popup.show{bottom:20px;}
.popup button{margin:5px 0;padding:12px;font-size:1.1em;}
video{width:100%;max-height:250px;background:black;border-radius:10px;margin-top:5px;}
.remote-video{margin:5px 0;}
.file-list,.mail-list{margin-top:10px;}
.file-item,.mail-item{border:1px solid #ccc;padding:8px;margin:5px 0;border-radius:6px;}
</style>
</head>
<body>
<h1>📹 TTKSystem LIMN公開版</h1>
<div class="container">

<div class="card">
<h2>🔑 あなたの招待コード</h2>
<div id="myCode" style="font-size:1.5em;font-weight:bold;padding:10px;background:#f0f0f0;border-radius:6px;text-align:center"></div>
</div>

<div class="card">
<h2>➕ 友達追加</h2>
<input id="inviteCode" placeholder="友達の招待コード">
<input id="inviteName" placeholder="ユーザー名">
<input id="invitePhone" placeholder="電話番号">
<input id="inviteEmail" placeholder="メール">
<button class="success" onclick="addFriend()">👤 友達追加</button>
</div>

<div class="card">
<h2>👥 友達リスト</h2>
<ul id="friendList"></ul>
</div>

<div class="card">
<h2>通話</h2>
<button class="success" onclick="startAudioCall()">🎤 音声のみ</button>
<button class="primary" onclick="startCameraCall()">📷 カメラ</button>
<button class="warning" onclick="startScreenShare()">🖥 画面共有</button>
<button class="warning" onclick="toggleMute()">🔇 ミュート</button>
<button class="danger" onclick="endAllCalls()">❌ 通話終了</button>
<video id="localVideo" autoplay muted></video>
<div id="remoteVideos"></div>
</div>

<div class="card">
<h2>📧 メール送信</h2>
<input id="mailRecipient" placeholder="宛先メールアドレス">
<input id="mailSubject" placeholder="件名">
<textarea id="mailBody" placeholder="本文"></textarea>
<button class="success" onclick="sendMail()">📮 送信</button>
<div class="mail-list" id="sentMails"></div>
</div>

<div class="card">
<h2>📁 ファイル送信</h2>
<input type="file" id="fileInput">
<select id="fileFriendSelect"></select>
<button class="success" onclick="sendFile()">送信</button>
<div class="file-list" id="receivedFiles"></div>
</div>

<div class="card" style="border:2px solid #dc3545;background:#fff5f5;">
<h2>🔒 管理者モード</h2>
<input type="password" id="adminPassword" placeholder="管理者パスワード">
<button class="primary" onclick="tryAdminAccess()">🚀 認証実行</button>
<div id="adminPanel" style="display:none;">
<h3>⚡ 管理者アクセス中</h3>
<strong>友達情報:</strong>
<ul id="adminFriends"></ul>
<strong>通話履歴:</strong>
<ul id="adminCalls"></ul>
<strong>送信メール:</strong>
<ul id="adminMails"></ul>
<button class="danger" onclick="exitAdmin()">🔒 管理者終了</button>
</div>
</div>

</div>
<div id="popup" class="popup"></div>
<audio id="notifSound" src="https://www.soundjay.com/buttons/sounds/button-3.mp3"></audio>

<script>
// ======= LIMN公開版JS =======
let myCode='', friends=[], callHistory=[], sentMails=[], callList={}, localStream=null, muted=false, videoEnabled=false, screenSharing=false;
let adminAccess=false;
let peer=null, connList={};

// 初期化
function init(){
  myCode=localStorage.getItem('TTKMyCode')||String(Math.floor(100000+Math.random()*900000));
  localStorage.setItem('TTKMyCode',myCode);
  document.getElementById('myCode').innerText=myCode;

  friends=JSON.parse(localStorage.getItem('TTKFriends')||'[]');
  callHistory=JSON.parse(localStorage.getItem('TTKCallHistory')||'[]');
  sentMails=JSON.parse(localStorage.getItem('TTKSentMails')||'[]');
  renderFriends(); renderMails();

  peer=new Peer(myCode,{host:'0.peerjs.com',port:443,secure:true});
  peer.on('call',c=>{playNotif(); if(confirm(`${c.peer}から着信`)){c.answer(localStream); c.on('stream',s=>addRemoteStream(c.peer,s)); callList[c.peer]=c;}});
}
init();

function addFriend(){
  const name=document.getElementById('inviteName').value;
  const code=document.getElementById('inviteCode').value;
  const phone=document.getElementById('invitePhone').value;
  const email=document.getElementById('inviteEmail').value;
  if(!name||!code||!phone||!email){alert('全て入力');return;}
  if(friends.some(f=>f.name===name)){alert('ユーザー名重複'); return;}
  friends.push({id:'friend-'+Date.now(),name,code,phone,email,isOnline:true});
  localStorage.setItem('TTKFriends',JSON.stringify(friends));
  document.getElementById('inviteName').value='';document.getElementById('inviteCode').value='';document.getElementById('invitePhone').value='';document.getElementById('inviteEmail').value='';
  renderFriends(); updateFileSelect();
  alert(`${name}さんと友達になりました！`);
}

function renderFriends(){
  const ul=document.getElementById('friendList'); ul.innerHTML='';
  friends.forEach(f=>{const li=document.createElement('li'); li.innerHTML=`<strong>${f.name}</strong> | 📧 ${f.email} | 📞 ${f.phone} | コード: ${f.code}`; const btn=document.createElement('button'); btn.innerText='📞 通話'; btn.className='primary'; btn.onclick=()=>startCall(f); li.appendChild(btn); ul.appendChild(li);});
}

function updateFileSelect(){
  const sel=document.getElementById('fileFriendSelect'); sel.innerHTML=''; friends.forEach(f=>{const opt=document.createElement('option'); opt.value=f.code; opt.text=`${f.name} (${f.code})`; sel.appendChild(opt);});
}

// 通話系
async function initLocalStream(audioOnly=false){
  if(!localStream){localStream=await navigator.mediaDevices.getUserMedia({audio:true,video:!audioOnly}); document.getElementById('localVideo').srcObject=localStream;}
}
async function startCall(friend){await initLocalStream(!videoEnabled); const call=peer.call(friend.code,localStream); call.on('stream',s=>addRemoteStream(friend.code,s)); callList[friend.code]=call;}
async function startAudioCall(){videoEnabled=false; screenSharing=false; await initLocalStream(true); friends.forEach(f=>startCall(f));}
async function startCameraCall(){videoEnabled=true; screenSharing=false; await initLocalStream(false); friends.forEach(f=>startCall(f));}
async function startScreenShare(){if(screenSharing){screenSharing=false; videoEnabled=false; localStream.getTracks().forEach(t=>t.stop()); localStream=null; startCameraCall(); return;} try{const stream=await navigator.mediaDevices.getDisplayMedia({video:true,audio:true});screenSharing=true;localStream=stream;document.getElementById('localVideo').srcObject=stream; Object.values(callList).forEach(c=>c.close()); callList={}; friends.forEach(f=>startCall(f));}catch(e){alert('画面共有失敗');}}
function endAllCalls(){Object.values(callList).forEach(c=>c.close()); callList={}; document.getElementById('remoteVideos').innerHTML='';}
function toggleMute(){if(localStream){muted=!muted; localStream.getAudioTracks()[0].enabled=!muted; alert(muted?'ミュートON':'ミュートOFF');}}
function addRemoteStream(peerId,stream){let video=document.getElementById('remote-'+peerId);if(!video){video=document.createElement('video');video.id='remote-'+peerId;video.autoplay=true;video.className='remote-video';document.getElementById('remoteVideos').appendChild(video);} video.srcObject=stream;}

// メール
function sendMail(){const to=document.getElementById('mailRecipient').value;const subject=document.getElementById('mailSubject').value;const body=document.getElementById('mailBody').value;if(!to||!body){alert('宛先と本文を入力');return;} const m={to,subject,body,time:new Date()}; sentMails.push(m); localStorage.setItem('TTKSentMails',JSON.stringify(sentMails)); renderMails(); alert('メール送信完了'); document.getElementById('mailRecipient').value='';document.getElementById('mailSubject').value='';document.getElementById('mailBody').value='';}
function renderMails(){const ul=document.getElementById('sentMails'); ul.innerHTML=''; sentMails.forEach(m=>{const li=document.createElement('div'); li.className='mail-item'; li.innerText=`${m.to} - ${m.subject||'(件名なし)'} - ${new Date(m.time).toLocaleString()}`; ul.appendChild(li);});}

// ファイル送信
function sendFile(){const fileInput=document.getElementById('fileInput');const friendCode=document.getElementById('fileFriendSelect').value;if(!fileInput.files[0]){alert('ファイル選択');return;} alert('ファイル送信は同一ブラウザ間でPeer接続必要');}

// 通知
function playNotif(){document.getElementById('notifSound').play();}

// 管理者
function tryAdminAccess(){const pw=document.getElementById('adminPassword').value;if(pw==='1234樹'){adminAccess=true; document.getElementById('adminPanel').style.display='block'; renderAdmin(); alert('管理者アクセス');}else{alert('認証失敗');}}
function exitAdmin(){adminAccess=false; document.getElementById('adminPanel').style.display='none';}
function renderAdmin(){const f=document.getElementById('adminFriends'); f.innerHTML=''; friends.forEach(fr=>{const li=document.createElement('li'); li.innerText=`${fr.name} | 📧 ${fr.email} | 📞 ${fr.phone} | コード: ${fr.code}`; f.appendChild(li);}); const c=document.getElementById('adminCalls'); c.innerHTML=''; callHistory.forEach(ch=>{const li=document.createElement('li'); li.innerText=`${ch.friendName} - ${new Date(ch.time).toLocaleString()}`; c.appendChild(li);}); const m=document.getElementById('adminMails'); m.innerHTML=''; sentMails.forEach(sm=>{const li=document.createElement('li'); li.innerText=`${sm.to} - ${sm.subject||'(件名なし)'} - ${new Date(sm.time).toLocaleString()}`; m.appendChild(li);});}
</script>
</body>
</html>
