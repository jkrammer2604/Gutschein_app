<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Gutschein-App</title>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<style>
body { font-family: Arial, Helvetica, sans-serif; margin:0; min-height:100vh; display:flex; justify-content:center; align-items:flex-start; background:linear-gradient(to bottom,#a6dcef,#e7bed9);}
.box {background:white; padding:25px 30px; border-radius:12px; width:600px; box-shadow:0 6px 15px rgba(0,0,0,0.1); margin-top:40px;}
h2,h3 {margin-top:0;}
input, select, button {width:100%; padding:10px; margin:6px 0; box-sizing:border-box; border:2px solid #ccc; border-radius:6px; font-size:16px;}
button {background:#4CAF50; color:white; border:none; border-radius:6px; padding:12px 0; cursor:pointer; transition:background .2s;}
button:hover {background:#45a049;}
#logoutBtn {background:#f44336;}
#logoutBtn:hover {background:#d73833;}
.checkboxRow {display:flex; align-items:center; gap:8px; margin:6px 0;}
ul {list-style:none; padding-left:0; margin:0;}
li.entry {background:#e6f4ea; margin:6px 0; padding:10px; border-radius:6px; display:flex; justify-content:space-between; align-items:center;}
.smallBtn {background:#f44336; color:white; border:none; border-radius:5px; padding:6px 10px; cursor:pointer;}
.smallBtn:hover {background:#d73833;}
.flex {display:flex; gap:10px;}
.flex label {flex:1;}
.muted {color:#666; font-size:14px;}
</style>
</head>
<body>

<!-- LOGIN BOX -->
<div id="loginBox" class="box">
  <h2>Login</h2>
  <input id="loginName" placeholder="Benutzername">
  <input id="loginPass" type="password" placeholder="Passwort">
  <div id="loginError" class="muted"></div>
  <button onclick="login()">Einloggen</button>
</div>

<!-- APP BOX -->
<div id="appBox" class="box" style="display:none">
  <button id="logoutBtn" onclick="logout()">Logout</button>
  <h2>Hallo <span id="userName"></span></h2>
  <p>Guthaben: <span id="userMoney"></span> j€</p>

  <h3>Gutschein senden</h3>
  <input id="gText" placeholder="Gutschein Text">
  <select id="recipient"></select>
  <input id="amount" type="number" placeholder="Betrag optional">
  <button onclick="sendVoucher()">Senden</button>

  <h3>Gesendete Gutscheine</h3>
  <ul id="sentList"></ul>

  <h3>Empfangene Gutscheine</h3>
  <ul id="receivedList"></ul>

  <!-- ADMIN BOX -->
  <div id="adminBox" style="display:none;margin-top:20px">
    <h3>Admin – Benutzerverwaltung</h3>
    <input id="newName" placeholder="Neuer Benutzername">
    <input id="newPass" placeholder="Passwort">
    <div class="flex">
      <label><input type="checkbox" id="isAdminCheck"> Ist Admin</label>
      <label><input type="checkbox" id="canDeleteCheck"> Darf löschen</label>
    </div>
    <button onclick="createUser()">Benutzer erstellen</button>

    <h3>Alle Benutzer</h3>
    <ul id="userList"></ul>
  </div>
</div>

<script>
// ===== SUPABASE CONFIG =====
const SUPABASE_URL = "https://ukmrfjwgiurnlsysisoz.supabase.co";
const SUPABASE_ANON_KEY = "sb_publishable_4TgDm_iM1J52_QWlXcAgwQ_8BVoMTMl";
const supabase = Supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

let currentUser = null;

// ===== LOGIN =====
async function login(){
  const username = document.getElementById("loginName").value.trim();
  const password = document.getElementById("loginPass").value.trim();
  const errorEl = document.getElementById("loginError");
  errorEl.textContent="";

  if(!username || !password){ errorEl.textContent="Bitte alles ausfüllen"; return; }

  const { data:user, error } = await supabase.from("users")
    .select("*")
    .eq("username", username)
    .eq("password", password)
    .maybeSingle();

  if(error){ errorEl.textContent="Supabase Fehler: "+error.message; return; }
  if(!user){ errorEl.textContent="Benutzername oder Passwort falsch!"; return; }

  currentUser = user;
  document.getElementById("loginBox").style.display="none";
  document.getElementById("appBox").style.display="block";
  document.getElementById("userName").textContent = user.username;
  document.getElementById("userMoney").textContent = user.guthaben===999999?"∞":user.guthaben;
  document.getElementById("adminBox").style.display = user.is_admin ? "block" : "none";

  await updateUI();
}

// ===== LOGOUT =====
function logout(){
  currentUser=null;
  document.getElementById("loginBox").style.display="block";
  document.getElementById("appBox").style.display="none";
  document.getElementById("loginName").value="";
  document.getElementById("loginPass").value="";
}

// ===== SEND VOUCHER =====
async function sendVoucher(){
  const text = gText.value.trim();
  const recipientName = recipient.value;
  const amountNum = Number(amount.value||0);

  if(!text && amountNum<=0){ alert("Text oder Betrag nötig"); return; }

  const {data:rec} = await supabase.from("users").select("*").eq("username",recipientName).maybeSingle();
  if(!rec){ alert("Empfänger nicht gefunden"); return; }

  if(amountNum>0 && !currentUser.is_admin && currentUser.guthaben<amountNum){
    alert("Nicht genug Guthaben"); return;
  }

  const senderNew = currentUser.is_admin?currentUser.guthaben:currentUser.guthaben - amountNum;
  const recipientNew = rec.guthaben + amountNum;

  await supabase.from("users").update({guthaben:senderNew}).eq("id",currentUser.id);
  await supabase.from("users").update({guthaben:recipientNew}).eq("id",rec.id);

  await supabase.from("vouchers").insert({sender_id:currentUser.id, recipient_id:rec.id, text, amount:amountNum});

  gText.value=""; amount.value="";
  await updateUI();
}

// ===== ADMIN CREATE USER =====
async function createUser(){
  const name=newName.value.trim();
  const pass=newPass.value.trim();
  const isAdmin=isAdminCheck.checked;
  const canDelete=canDeleteCheck.checked;

  if(!name||!pass){ alert("Name & Passwort nötig"); return; }

  const {data:exists} = await supabase.from("users").select("*").eq("username",name);
  if(exists.length>0){ alert("Benutzername existiert"); return; }

  await supabase.from("users").insert({
    username:name,
    password:pass,
    is_admin:isAdmin,
    darf_loeschen:canDelete,
    guthaben:isAdmin?999999:0
  });

  newName.value=""; newPass.value="";
  isAdminCheck.checked=false; canDeleteCheck.checked=false;

  await updateUI();
}

// ===== UPDATE UI =====
async function updateUI(){
  const {data:usersData} = await supabase.from("users").select("*").neq("id",currentUser.id);
  recipient.innerHTML="";
  usersData.forEach(u=>{
    const opt=document.createElement('option'); opt.value=u.username; opt.textContent=u.username;
    recipient.appendChild(opt);
  });

  if(currentUser.is_admin){
    const {data:allUsers} = await supabase.from("users").select("*");
    userList.innerHTML="";
    allUsers.forEach(u=>{
      const li=document.createElement('li');
      li.textContent=`${u.username} ${u.is_admin?"(Admin)":" "} ${u.darf_loeschen?"(Darf löschen)":" "}`;
      if(currentUser.is_admin && !u.is_admin && u.id!==currentUser.id){
        const btn=document.createElement('button'); btn.textContent="Löschen"; btn.className="smallBtn";
        btn.onclick=async ()=>{
          if(confirm(`Benutzer ${u.username} löschen?`)){
            await supabase.from("users").delete().eq("id",u.id);
            await supabase.from("vouchers").delete().or(`sender_id.eq.${u.id},recipient_id.eq.${u.id}`);
            updateUI();
          }
        };
        li.appendChild(btn);
      }
      userList.appendChild(li);
    });
  }

  const {data:vouchers} = await supabase.from("vouchers")
    .select("*")
    .or(`sender_id.eq.${currentUser.id},recipient_id.eq.${currentUser.id}`);

  sentList.innerHTML=""; receivedList.innerHTML="";
  const {data:allUsers} = await supabase.from("users").select("*");
  vouchers.forEach(v=>{
    const sender = allUsers.find(u=>u.id===v.sender_id)?.username||"?";
    const recipientU = allUsers.find(u=>u.id===v.recipient_id)?.username||"?";
    if(v.sender_id===currentUser.id) sentList.innerHTML+=`<li>An ${recipientU}: ${v.text} (${v.amount} j€)</li>`;
    if(v.recipient_id===currentUser.id) receivedList.innerHTML+=`<li>Von ${sender}: ${v.text} (${v.amount} j€)</li>`;
  });

  const {data:user} = await supabase.from("users").select("*").eq("id",currentUser.id).maybeSingle();
  currentUser=user;
  userMoney.textContent=user.guthaben===999999?"∞":user.guthaben;
}
</script>
</body>
</html>
