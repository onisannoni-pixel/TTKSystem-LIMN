<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>TTKシステム 最強版</title>
<script src="https://cdn.jsdelivr.net/npm/peerjs@1.4.7/dist/peerjs.min.js"></script>
<style>
body{font-family:sans-serif;padding:20px;max-width:900px;margin:auto;}
h1{color:#007bff;}
.box{border:1px solid #ccc;border-radius:8px;padding:15px;margin:20px 0;}
input,textarea,select{width:100%;padding:8px;margin:5px 0;}
button{padding:8px 15px;border:none;border-radius:6px;cursor:pointer;}
button.add{background:#28a745;color:#fff;}
button.call{background:#007bff;color:#fff;}
button.send{background:#17a2b8;color:#fff;}
ul{list-style:none;padding:0;}
li{border-bottom:1px solid #eee;padding:8px;}
audio,video{display:block;margin:5px 0;max-width:100%;}
</style>
</head>
<body>

<h1>📞 TTKシステム 最強版</h1>

<div class="box">
<h2>🔑 あなたの招待コード</h2>
<div id="myCode" style="font-size:1.5em;font-weight:bold;padding:10px;background:#f0f0f0;border-radius:6px;text-align:center;"></div>
</div>

<div class="box">
<h2>➕ 友達を追加</h2>
<input id="inviteCode" placeholder="友達の招待コード"/>
<input id="inviteName" placeholder="ユーザー名"/>
<button class="add" onclick="addFriend()">👤 友達追加</button>
</div>

<div class="box">
<h2>👥 友達リスト</h2>
<ul id="friendList"></ul>
</div>

<div class="box">
<h2>💬 チャット</h2>
<select id="chatSelect" onchange="loadChat()"><option value=''>友達を選択</option></select>
<div id="chatArea" style="border:1px solid #ddd; padding:10px; height:200px; overflow-y:scroll; background:#fafafa;"></div>
<input id="chatMessage" placeholder="メッセージを入力"/>
<button class="send" onclick="sendMessage()">📮 送信</button>
</div>

<div class="box">
<h2>📂 ファイル送信</h2>
<input type="file" id="fileInput"/>
<button class="send" onclick="sendFile()">📤 送信</button>
<ul id="fileHistory"></ul>
</div>

<div class="box">
<h2>🎤 ビデオ通話（複数人対応）</h2>
<select id="callSelect"><option value=''>友達を選択</option></select>
<button onclick="startCallPeer()">通話に参加</button>
<div id="videoArea"></div>
<ul id="callParticipants"></ul>
</div>

<div class="box">
<h2>🗝 管理者モード</h2>
<input type="password" id="adminPassword" placeholder="管理者パスワード"/>
<button onclick="tryAdmin()">🔒 管理者モード</button>
<div id="adminPanel" style="display:none;">
<h3>全友達履歴</h3>
<div id="adminHistory"></div>
<button onclick="exitAdmin()">🔓 管理者終了</button>
</div>
</div>

<script>
// ===== 初期設定 =====
let myCode=localStorage.getItem("TTKMyCode");
if(!myCode){myCode=String(Math.floor(100000+Math.random()*900000));localStorage.setItem("TTKMyCode",myCode);}
document.getElementById("myCode").innerText=myCode;

let friends=JSON.parse(localStorage.getItem("TTKFriends")||"[]");
renderFriends(); renderChatSelect(); renderCallSelect();

let localStream=null;
let peers={};
const peer=new Peer(myCode);

navigator.mediaDevices.getUserMedia({audio:true,video:true}).then(stream=>{localStream=stream;});

// ===== 友達管理 =====
function addFriend(){
  const code=document.getElementById("inviteCode").value.trim();
  const name=document.getElementById("inviteName").value.trim();
  if(!code||!name){alert("招待コードと名前を入力してください");return;}
  if(friends.some(f=>f.name===name)){alert("そのユーザー名は既に使われています");return;}
  friends.push({id:Date.now(),code,name});
  localStorage.setItem("TTKFriends",JSON.stringify(friends));
  renderFriends(); renderChatSelect(); renderCallSelect();
  document.getElementById("inviteCode").value="";
  document.getElementById("inviteName").value="";
  alert(`${name} さんを追加しました`);
}

function renderFriends(){
  const list=document.getElementById("friendList");
  list.innerHTML="";
  if(friends.length===0){list.innerHTML="<li>まだ友達はいません</li>";return;}
  friends.forEach(f=>{
    const li=document.createElement("li");
    li.innerHTML=`<strong>${f.name}</strong> (コード:${f.code}) <button class="call" onclick="startCallPeer('${f.code}')">📞 通話</button>`;
    list.appendChild(li);
  });
}

function renderChatSelect(){
  const select=document.getElementById("chatSelect");
  select.innerHTML="<option value=''>友達を選択</option>";
  friends.forEach(f=>{select.innerHTML+=`<option value='${f.code}'>${f.name}</option>`});
}

function renderCallSelect(){
  const select=document.getElementById("callSelect");
  select.innerHTML="<option value=''>友達を選択</option>";
  friends.forEach(f=>{select.innerHTML+=`<option value='${f.code}'>${f.name}</option>`});
}

// ===== チャット =====
function loadChat(){
  const friendCode=document.getElementById("chatSelect").value;
  const chatArea=document.getElementById("chatArea");
  if(!friendCode){chatArea.innerHTML="";return;}
  let history=JSON.parse(localStorage.getItem("TTKChat_"+friendCode)||"[]");
  chatArea.innerHTML="";
  history.forEach(c=>{chatArea.innerHTML+=`[${c.time}] ${c.text}<br>`;});
}

function sendMessage(){
  const friendCode=document.getElementById("chatSelect").value;
  const msg=document.getElementById("chatMessage").value.trim();
  if(!friendCode){alert("友達を選択してください");return;}
  if(!msg)return;
  const history=JSON.parse(localStorage.getItem("TTKChat_"+friendCode)||"[]");
  const entry={text:msg,time:new Date().toLocaleString()};
  history.push(entry);
  localStorage.setItem("TTKChat_"+friendCode,JSON.stringify(history));
  document.getElementById("chatMessage").value="";
  loadChat();
  notify("新しいメッセージ",msg);
  if(peers[friendCode]){peers[friendCode].send(JSON.stringify({type:"chat",msg,from:myCode}))}
}

// ===== ファイル送信 =====
function sendFile(){
  const friendCode=document.getElementById("chatSelect").value;
  const fileInput=document.getElementById("fileInput");
  if(!friendCode){alert("友達を選択してください");return;}
  if(!fileInput.files.length){alert("ファイルを選択してください");return;}
  const file=fileInput.files[0];
  const reader=new FileReader();
  reader.onload=function(e){
    const data=e.target.result;
    if(peers[friendCode]){peers[friendCode].send(JSON.stringify({type:"file",name:file.name,data}))}
  }
  reader.readAsDataURL(file);
  const history=JSON.parse(localStorage.getItem("TTKFile_"+friendCode)||"[]");
  history.push({name:file.name,time:new Date().toLocaleString()});
  localStorage.setItem("TTKFile_"+friendCode,JSON.stringify(history));
  fileInput.value="";
  renderFileHistory(friendCode);
  notify("ファイル送信完了",file.name);
}

function renderFileHistory(friendCode){
  const list=document.getElementById("fileHistory");
  list.innerHTML="";
  const history=JSON.parse(localStorage.getItem("TTKFile_"+friendCode)||"[]");
  history.forEach(f=>{
    const li=document.createElement("li");
    const link=document.createElement("a");
    link.href=f.data||"#";
    link.download=f.name;
    link.innerText=`[${f.time}] ${f.name}`;
    li.appendChild(link);
    list.appendChild(li);
  });
}

// ===== 通知 =====
function notify(title,body){
  if(Notification.permission==="granted"){new Notification(title,{body});}
  else if(Notification.permission!=="denied"){Notification.requestPermission().then(p=>{if(p==="granted")new Notification(title,{body});});}
}

// ===== 管理者モード =====
const ADMIN_PASS="1234樹";
function tryAdmin(){
  const pw=document.getElementById("adminPassword").value;
  if(pw!==ADMIN_PASS){alert("認証失敗");return;}
  document.getElementById("adminPanel").style.display="block";
  renderAdmin();
}
function exitAdmin(){document.getElementById("adminPanel").style.display="none";}

function renderAdmin(){
  const div=document.getElementById("adminHistory");
  div.innerHTML="";
  friends.forEach(f=>{
    div.innerHTML+=`<h4>${f.name} (コード:${f.code})</h4>`;
    let chat=JSON.parse(localStorage.getItem("TTKChat_"+f.code)||"[]");
    let files=JSON.parse(localStorage.getItem("TTKFile_"+f.code)||"[]");
    let calls=JSON.parse(localStorage.getItem("TTKCall_"+f.code)||"[]");
    div.innerHTML+="<b>チャット:</b><ul>"+chat.map(c=>`<li>[${c.time}] ${c.text}</li>`).join("")+"</ul>";
    div.innerHTML+="<b>ファイル:</b><ul>"+files.map(f=>`<li>[${f.time}] ${f.name}</li>`).join("")+"</ul>";
    div.innerHTML+="<b>通話:</b><ul>"+calls.map(c=>`<li>[${c.time}] ${c.name}</li>`).join("")+"</ul>";
  });
}

// ===== WebRTC ビデオ通話 =====
const videoArea=document.getElementById("videoArea");

function startCallPeer(){
  const friendCode=document.getElementById("callSelect").value;
  if(!friendCode){alert("友達を選択してください");return;}
  const call=peer.call(friendCode,localStream);
  call.on('stream',stream=>{addVideo(stream,friendCode)});
  peers[friendCode]=call;
  saveCallHistory(friendCode);
  updateParticipants();
  notify("通話開始",`あなたが ${friendCode} と接続`);
}

peer.on('call', call=>{
  call.answer(localStream);
  call.on('stream',stream=>{addVideo(stream,call.peer)});
  peers[call.peer]=call;
  saveCallHistory(call.peer);
  updateParticipants();
});

function addVideo(stream,name){
  const video=document.createElement("video");
  video.srcObject=stream;
  video.autoplay=true;
  video.controls=true;
  video.dataset.name=name;
  videoArea.appendChild(video);
  updateParticipants();
}

function updateParticipants(){
  const ul=document.getElementById("callParticipants");
  ul.innerHTML="";
  const vids=videoArea.querySelectorAll("video");
  vids.forEach(v=>{const li=document.createElement("li");li.innerText=v.dataset.name;ul.appendChild(li);});
}

// ===== 通話履歴自動保存 =====
function saveCallHistory(friendCode){
  const history=JSON.parse(localStorage.getItem("TTKCall_"+friendCode)||"[]");
  history.push({time:new Date().toLocaleString(),name:friendCode});
  localStorage.setItem("TTKCall_"+friendCode,
