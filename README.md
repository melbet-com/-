<!DOCTYPE html>
<html lang="mn">
<head>
<meta charset="UTF-8">
<title>Покер – Бүх Хувилбартай</title>
<style>
  body {font-family: Arial; background: #0a3d0a; color: #eee; padding: 20px; margin: 0;}
  h1 {text-align: center; color: #ffeb3b;}
  input, button, select {margin: 5px; padding: 8px; font-size: 14px;}
  .card {display: inline-block; width: 40px; height: 56px; line-height: 56px;
         text-align: center; background: #fff; color: #000; border-radius: 6px;
         margin: 2px; font-weight: bold; font-size: 18px;}
  #log, #chat-log {max-height: 200px; overflow: auto; background: #111; padding: 10px; margin-top: 10px; border-radius: 6px;}
  .player {margin: 5px 0; font-size:89px; padding: 8px; background: #1a1a1a; border-radius: 6px;}
  .stats {font-size: 11px; color: #aaa;}
  .bot {color: #ff9800;}
  .upcard {border: 2px solid #0f0;}
  #game {max-width: 900px; margin: 0 auto;}
  #controls {text-align: center; margin: 15px 0;}
  #chat-input {width: 70%;}
</style>
</head>
<body>
<div id="game">
  <h1>Покер – Бүх Хувилбартай</h1>

  <!-- Тоглоом сонгох -->
  <div id="game-select" style="text-align:center; margin:20px 0">
    <label>Тоглоом: </label>
    <select id="game-type">
      <option value="holdem">Texas Hold'em</option>
      <option value="omaha">Pot Limit Omaha</option>
      <option value="stud">7-Card Stud</option>
      <option value="razz">Razz (Lowball)</option>
      <option value="badugi">Badugi</option>
    </select>
    <button id="change-game">Сонгох</button>
  </div>

  <!-- Нэгдэх -->
  <div id="join" style="text-align:center">
    Нэр: <input id="name" placeholder="Тоглогч" value="Тоглогч">
    <button id="join">Нэгдэх</button>
  </div>

  <!-- Тоглоомын хэсэг -->
  <div id="game-area" style="display:none">
    <button id="start" style="width:100%; padding:12px; font-size:18px; margin:10px 0">Гар эхлүүлэх</button>
    <hr>

    <!-- Тоглогчид -->
    <div id="players"></div>

    <!-- Таны гар -->
    <div id="hand" style="text-align:center; margin:15px 0; font-weight:bold"></div>

    <!-- Нийтийн карт / Up cards -->
    <div id="community" style="text-align:center; margin:15px 0"></div>

    <!-- Сав, бооцоо -->
    <div id="pot" style="text-align:center; font-size:18px; margin:10px 0"></div>

    <!-- Бооцоо хийх -->
    <div id="bet-controls" style="text-align:center; margin:15px 0"></div>

    <!-- Чат -->
    <div style="margin-top:20px">
      <input id="chat-input" placeholder="Чат бичих..." style="width:70%">
      <button id="send-chat">Илгээх</button>
    </div>
    <div id="chat-log"></div>

    <!-- Лог -->
    <div id="log"></div>
  </div>
</div>

<script src="/socket.io/socket.io.js"></script>
<script>
const socket = io();
const $ = id => document.getElementById(id);

const nameIn = $('name'), joinBtn = $('join'),
      startBtn = $('start'),
      playersDiv = $('players'), handDiv = $('hand'),
      communityDiv = $('community'), potDiv = $('pot'),
      logDiv = $('log'), betDiv = $('bet-controls'),
      chatIn = $('chat-input'), sendChat = $('send-chat'),
      chatLog = $('chat-log'), gameDiv = $('game-area'),
      gameSelect = $('game-type'), changeGameBtn = $('change-game');

let currentGame = 'holdem';

function log(msg) {
  const d = document.createElement('div'); d.textContent = msg; logDiv.prepend(d);
}

// Тоглоом сонгох
changeGameBtn.onclick = () => {
  currentGame = gameSelect.value;
  log(`Тоглоом сонгогдлоо: ${currentGame.toUpperCase()}`);
};

// Нэгдэх
joinBtn.onclick = () => {
  const name = nameIn.value.trim() || 'Тоглогч';
  socket.emit('join', { name, gameType: currentGame }, res => {
    if (res.error) return alert(res.error);
    $('join').style.display = 'none';
    gameDiv.style.display = 'block';
    log(`${name} нэгдлээ (${currentGame.toUpperCase()})`);
  });
};

// Гар эхлүүлэх
startBtn.onclick = () => socket.emit('start', {}, res => res.error ? alert(res.error) : log('Гар эхэллээ'));

// Чат илгээх
sendChat.onclick = () => {
  const msg = chatIn.value.trim();
  if (msg) socket.emit('chat', { msg }, () => { chatIn.value = ''; });
};
chatIn.addEventListener('keypress', e => { if (e.key === 'Enter') sendChat.click(); });

// Таны гар
socket.on('hand', ({ hand }) => {
  handDiv.innerHTML = 'Таны гар: ' + hand.map((c, i) => 
    `<span class="card ${i < 2 ? 'upcard' : ''}">${c}</span>`
  ).join('');
});

// Ширээний байдал
socket.on('table_update', data => {
  playersDiv.innerHTML = data.players.map(p => `
    <div class="player ${p.isBot ? 'bot' : ''}">
      <strong>${p.name}</strong> – ${p.chips} чип ${p.bet ? `(бооцоо ${p.bet})` : ''} ${p.folded ? '[фолд]' : ''}
      ${p.stats ? `<div class="stats">VPIP: ${p.stats.vpip}% | PFR: ${p.stats.pfr}% | Agg: ${p.stats.agg}%</div>` : ''}
      ${p.upCards ? `<div>Нээлттэй: ${p.upCards.map(c => `<span class="card upcard">${c}</span>`).join('')}</div>` : ''}
    </div>
  `).join('');

  // Нийтийн карт / улирал
  if (data.community.length > 0) {
    communityDiv.innerHTML = `<strong>${data.stage.toUpperCase()}:</strong> ` +
      data.community.map(c => `<span class="card">${c}</span>`).join(' ');
  } else {
    communityDiv.innerHTML = '';
  }

  potDiv.textContent = `Сав: ${data.pot}`;

  // Чат
  chatLog.innerHTML = data.chat.map(m => `<div><small>${m.time}</small> <strong>${m.name}:</strong> ${m.msg}</div>`).join('');
  chatLog.scrollTop = chatLog.scrollHeight;

  // Бооцоо
  betDiv.innerHTML = '';
  if (data.currentPlayer === socket.id && !['showdown', 'waiting'].includes(data.stage)) {
    const callAmt = data.lastBet - (data.players.find(p => p.id === socket.id)?.bet || 0);
    const minRaise = data.minRaise;

    const btn = (txt, act, amt = 0) => {
      const b = document.createElement('button');
      b.textContent = txt;
      b.onclick = () => socket.emit(act, amt ? { amount: amt } : null, res => res.error ? alert(res.error) : 0);
      return b;
    };

    betDiv.appendChild(btn('Фолд', 'fold'));
    betDiv.appendChild(btn(callAmt > 0 ? `Калл ${callAmt}` : 'Чек', 'call'));

    const raiseIn = document.createElement('input');
    raiseIn.type = 'number'; raiseIn.min = minRaise; raiseIn.value = minRaise; raiseIn.style.width = '70px';
    const raiseBtn = btn('Өсгөх', 'raise');
    raiseBtn.onclick = () => {
      const val = parseInt(raiseIn.value);
      if (isNaN(val) || val < minRaise) return alert('Буруу өсгөлт');
      socket.emit('raise', { amount: val });
    };
    betDiv.appendChild(raiseIn);
    betDiv.appendChild(raiseBtn);
  }
});

// Бот үйлдэл
socket.on('bot_action', data => {
  log(`[Бот] ${data.name}: ${data.action}${data.amount ? ` ${data.amount}` : ''}`);
});

// Чат
socket.on('chat', msg => {
  const d = document.createElement('div');
  d.innerHTML = `<small>${msg.time}</small> <strong>${msg.name}:</strong> ${msg.msg}`;
  chatLog.appendChild(d);
  chatLog.scrollTop = chatLog.scrollHeight;
});

// Шоудаун
socket.on('showdown', data => {
  const gameNames = {holdem: 'Hold\'em', omaha: 'Omaha', stud: 'Stud', razz: 'Razz', badugi: 'Badugi'};
  log(`*** ЯЛАГЧ: ${data.winners.map(id => TABLE.players?.find(p=>p.id===id)?.name).join(', ')} – ${data.winAmt} чип (${gameNames[data.gameType]}) ***`);
});
</script>
</body>
</html>
