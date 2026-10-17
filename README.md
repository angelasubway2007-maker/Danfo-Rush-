# Danfo-Rush-
Drive , talk , hustle, from sango to lekki , every trip pay
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Lagos Hustle Driver</title>

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://unpkg.com/peerjs@1.5.3/dist/peerjs.min.js"></script>

<style>
*{margin:0;padding:0;box-sizing:border-box;font-family:system-ui}
body{background:#0a0a0a;color:white;overflow:hidden}
#top{position:absolute;top:0;left:0;width:100%;height:60px;background:rgba(0,0,0,.85);display:flex;justify-content:space-between;align-items:center;padding:0 12px;z-index:1000;border-bottom:2px solid #00e676}
#wallet{background:#00e676;color:#000;padding:6px 14px;border-radius:20px;font-weight:900}
#map{width:100vw;height:100vh}
#bottom{position:absolute;bottom:0;left:0;width:100%;background:rgba(0,0,0,.92);padding:10px;z-index:1000}
#passenger{background:#0d2a3a;border:1px solid #00e676;border-radius:14px;padding:10px;display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;gap:8px}
.btn{flex:1;padding:12px 5px;border-radius:12px;border:none;font-weight:800;cursor:pointer;touch-action:manipulation}
.btn-accept{background:#00e676;color:#000}
.btn-decline{background:#333;color:white}
.btn-voice{background:#1565c0;color:white}
.btn-voice.recording{background:#d32f2f;animation:pulse .8s infinite}
@keyframes pulse{50%{transform:scale(1.04)}}
#login{position:absolute;inset:0;background:#000;display:flex;flex-direction:column;align-items:center;justify-content:center;z-index:2000;padding:20px;text-align:center}
#login input,#voiceConnect input{padding:14px;width:100%;max-width:320px;border-radius:12px;border:none;margin:10px 0;font-size:16px}
#voiceConnect{margin-top:8px;display:flex;gap:5px;align-items:center}
#voiceConnect input{min-width:0;margin:0;padding:8px;font-size:12px}
#voiceConnect button{padding:9px;border:0;border-radius:8px;background:#1565c0;color:white;font-weight:bold}
#status{font-size:11px;color:#4fc3f7;margin-top:5px}
#speed{position:absolute;right:12px;bottom:145px;background:#000c;border:1px solid #00e676;border-radius:10px;padding:8px;z-index:900}
@media(max-width:400px){.btn{font-size:11px}#passenger{padding:8px}}
</style>
</head>

<body>

<div id="login">
  <h1 style="color:#00e676;font-size:36px">LAGOS<br>HUSTLE DRIVER</h1>
  <div style="background:#ffeb3b;color:#000;padding:8px 16px;border-radius:20px;font-weight:900;margin:10px">
    🎉 Start with ₦500 virtual cash!
  </div>
  <p style="opacity:.7;margin:10px">From Sango to Lekki — Lagos driving missions</p>
  <input id="username" maxlength="24" placeholder="Driver name e.g. Tunde">
  <button class="btn btn-accept" style="max-width:320px;width:100%" onclick="startGame()">
    START HUSTLE ▶
  </button>
</div>

<div id="top">
  <div><span id="driverName">Driver</span> • LV <span id="level">1</span></div>
  <div id="wallet">₦<span id="balance">500</span></div>
</div>

<div id="map"></div>

<div id="speed">🚕 <span id="speedValue">0</span> km/h</div>

<div id="bottom">
  <div id="passenger">
    <div style="min-width:0">
      <div style="font-weight:900">Passenger: <span id="pName">Amina</span></div>
      <div style="font-size:12px;opacity:.8">
        <span id="pFrom">Sango</span> → <span id="pTo">Lekki Phase 1</span>
      </div>
      <div style="color:#00e676;font-weight:800">
        Reward: ₦<span id="pReward">2500</span>
      </div>
      <div id="status">Ready for your next ride</div>
      <div id="voiceConnect">
        <input id="remoteId" placeholder="Other player's voice ID">
        <button onclick="connectVoice()">CALL</button>
      </div>
    </div>
    <div id="peerId" style="font-size:10px;opacity:.7;overflow-wrap:anywhere">Voice: offline</div>
  </div>

  <div style="display:flex;gap:7px">
    <button class="btn btn-decline" onclick="decline()">✕ DECLINE</button>
    <button class="btn btn-accept" onclick="acceptRide()">✓ ACCEPT</button>
    <button class="btn btn-voice" id="voiceBtn"
      onpointerdown="startVoice(event)"
      onpointerup="stopVoice(event)"
      onpointercancel="stopVoice(event)"
      onlostpointercapture="stopVoice(event)">🎙 TALK</button>
  </div>
</div>

<script>
'use strict';

const SANGO=[6.7072,3.2318];
const OSHODI=[6.5661,3.3525];
const LEKKI=[6.4474,3.4723];
const IKEJA=[6.6018,3.3515];

const SAVE_KEY='lagos_hustle_save_v2';
const BONUS_KEY='lagos_hustle_bonus_v2';

let balance=500;
let level=1;
let currentRide=null;
let map=null;
let driverMarker=null;
let routeLine=null;
let pickupMarker=null;
let dropMarker=null;

let peer=null;
let localStream=null;
let outgoingCall=null;
let voiceReady=false;
let voiceHeld=false;

let rideState='waiting';
let animationId=null;
let tripToken=0;

const passengers=[
  {name:'Amina',from:'Sango',to:'Lekki Phase 1',start:SANGO,end:LEKKI,reward:2500},
  {name:'Emeka',from:'Oshodi',to:'Sango',start:OSHODI,end:SANGO,reward:1800},
  {name:'Mama Nkechi',from:'Ikeja',to:'Oshodi',start:IKEJA,end:OSHODI,reward:1500},
  {name:'Tolu',from:'Lekki',to:'Sango',start:LEKKI,end:SANGO,reward:3000}
];

function setStatus(message){
  document.getElementById('status').textContent=message;
}

function updateBalance(){
  document.getElementById('balance').textContent=
    balance.toLocaleString('en-NG');
}

function saveGame(){
  try{
    localStorage.setItem(SAVE_KEY,JSON.stringify({balance,level}));
  }catch(error){
    console.warn('Progress could not be saved',error);
  }
}

function loadGame(){
  try{
    const saved=JSON.parse(localStorage.getItem(SAVE_KEY)||'null');
    if(saved){
      if(Number.isFinite(saved.balance)&&saved.balance>=0) balance=saved.balance;
      if(Number.isFinite(saved.level)&&saved.level>=1) level=saved.level;
    }else{
      balance=500;
      level=1;
      saveGame();
    }
  }catch(error){
    balance=500;
    level=1;
  }

  updateBalance();
  document.getElementById('level').textContent=level;
}

function startGame(){
  const name=document.getElementById('username').value.trim()||'Driver';

  document.getElementById('driverName').textContent=name;
  document.getElementById('login').style.display='none';

  loadGame();
  initMap();
  nextPassenger();
  initVoice();
}

function initMap(){
  if(map) return;

  map=L.map('map').setView([6.58,3.35],11);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{
    maxZoom:19,
    attribution:'&copy; OpenStreetMap contributors'
  }).addTo(map);

  L.polyline([SANGO,OSHODI,IKEJA,LEKKI],{
    color:'#00e676',
    weight:4,
    dashArray:'10,10'
  }).addTo(map);

  L.marker(SANGO).addTo(map).bindPopup('Sango');
  L.marker(OSHODI).addTo(map).bindPopup('Oshodi');
  L.marker(IKEJA).addTo(map).bindPopup('Ikeja');
  L.marker(LEKKI).addTo(map).bindPopup('Lekki');

  driverMarker=L.marker(SANGO).addTo(map).bindPopup('Your driver');

  // Start from the first pickup location.
  driverMarker.setLatLng(SANGO);
  map.setView(SANGO,12);
}

function clearRideMarkers(){
  if(routeLine){map.removeLayer(routeLine);routeLine=null;}
  if(pickupMarker){map.removeLayer(pickupMarker);pickupMarker=null;}
  if(dropMarker){map.removeLayer(dropMarker);dropMarker=null;}
}

function nextPassenger(){
  if(!map) return;

  tripToken++;
  if(animationId!==null){
    cancelAnimationFrame(animationId);
    animationId=null;
  }

  rideState='waiting';
  clearRideMarkers();

  currentRide=passengers[Math.floor(Math.random()*passengers.length)];

  document.getElementById('pName').textContent=currentRide.name;
  document.getElementById('pFrom').textContent=currentRide.from;
  document.getElementById('pTo').textContent=currentRide.to;
  document.getElementById('pReward').textContent=currentRide.reward.toLocaleString('en-NG');

  pickupMarker=L.circleMarker(currentRide.start,{
    radius:10,color:'#00e676',fillColor:'#00e676',fillOpacity:1
  }).addTo(map).bindPopup('Passenger pickup: '+currentRide.from);

  dropMarker=L.circleMarker(currentRide.end,{
    radius:10,color:'#4fc3f7',fillColor:'#4fc3f7',fillOpacity:1
  }).addTo(map).bindPopup('Destination: '+currentRide.to);

  routeLine=L.polyline([currentRide.start,currentRide.end],{
    color:'#00e676',weight:5,dashArray:'8,10'
  }).addTo(map);

  setStatus('Press ACCEPT to begin this trip.');
}

function decline(){
  if(rideState==='driving'){
    setStatus('Finish your current trip before changing passengers.');
    return;
  }
  nextPassenger();
}

function acceptRide(){
  if(!currentRide||rideState==='driving') return;

  if(rideState==='waiting'){
    rideState='driving';
    setStatus('Driving to pickup: '+currentRide.from);
    moveDriver(currentRide.start,'pickup');
  }
}

function moveDriver(destination,stage){
  if(!driverMarker||!map) return;

  const token=++tripToken;
  const start=driverMarker.getLatLng();
  const startLat=start.lat,startLng=start.lng;
  const endLat=destination[0],endLng=destination[1];
  const duration=9000;
  const startTime=performance.now();

  function animate(now){
    if(token!==tripToken) return;

    const progress=Math.min((now-startTime)/duration,1);
    // Smooth movement between the current position and destination.
    const eased=progress*progress*(3-2*progress);

    const lat=startLat+(endLat-startLat)*eased;
    const lng=startLng+(endLng-startLng)*eased;

    driverMarker.setLatLng([lat,lng]);
    map.panTo([lat,lng],{animate:false});

    document.getElementById('speedValue').textContent=
      progress<1?'35':'0';

    if(progress<1){
      animationId=requestAnimationFrame(animate);
    }else{
      animationId=null;
      document.getElementById('speedValue').textContent='0';

      if(stage==='pickup'){
        rideState='passenger';
        setStatus('Passenger picked up! Continue to '+currentRide.to+'.');
        alert('Passenger picked up! Now drive to '+currentRide.to+'.');
        moveDriver(currentRide.end,'dropoff');
      }else{
        finishRide();
      }
    }
  }

  animationId=requestAnimationFrame(animate);
}

function finishRide(){
  balance+=currentRide.reward;
  level=1+Math.floor(balance/10000);

  document.getElementById('level').textContent=level;
  updateBalance();
  saveGame();

  rideState='complete';
  setStatus('Trip complete! You earned ₦'+currentRide.reward.toLocaleString('en-NG'));

  alert('🎉 Trip complete! You earned ₦'+currentRide.reward.toLocaleString('en-NG'));

  nextPassenger();
}

// VOICE CHAT
async function initVoice(){
  if(!navigator.mediaDevices||!navigator.mediaDevices.getUserMedia){
    setStatus('Voice needs microphone access on a supported secure page.');
    return;
  }

  try{
    localStream=await navigator.mediaDevices.getUserMedia({
      audio:{echoCancellation:true,noiseSuppression:true}
    });

    // Keep microphone muted until the TALK button is held.
    localStream.getAudioTracks().forEach(track=>track.enabled=false);

    peer=new Peer();

    peer.on('open',id=>{
      voiceReady=true;
      document.getElementById('peerId').textContent='ID: '+id;
      setStatus('Voice ready. Share your ID or enter a friend’s ID.');
    });

    peer.on('call',call=>{
      call.answer(localStream);

      call.on('stream',remoteStream=>{
        playRemoteAudio(remoteStream);
      });

      call.on('close',()=>{
        setStatus('Voice call ended.');
      });

      call.on('error',()=>{
        setStatus('Voice connection error.');
      });

      outgoingCall=call;
    });

    peer.on('error',error=>{
      console.error('PeerJS error:',error);
      setStatus('Voice connection problem: '+error.type);
    });

  }catch(error){
    console.error(error);
    setStatus('Microphone permission denied or unavailable.');
  }
}

function playRemoteAudio(stream){
  let audio=document.getElementById('remoteAudio');

  if(!audio){
    audio=document.createElement('audio');
    audio.id='remoteAudio';
    audio.autoplay=true;
    audio.playsInline=true;
    document.body.appendChild(audio);
  }

  audio.srcObject=stream;
  audio.play().catch(error=>{
    console.warn('Audio playback requires user interaction',error);
  });
}

function connectVoice(){
  const remoteId=document.getElementById('remoteId').value.trim();

  if(!peer||!voiceReady||!localStream){
    setStatus('Voice is not ready yet. Allow microphone access first.');
    return;
  }

  if(!remoteId){
    setStatus('Enter the other player’s voice ID.');
    return;
  }

  if(remoteId===peer.id){
    setStatus('You cannot call your own ID.');
    return;
  }

  try{
    outgoingCall=peer.call(remoteId,localStream);

    if(!outgoingCall){
      setStatus('Could not start the call. Check the other player’s ID.');
      return;
    }

    outgoingCall.on('stream',playRemoteAudio);
    outgoingCall.on('error',()=>setStatus('Voice call failed.'));
    outgoingCall.on('close',()=>setStatus('Voice call ended.'));

    setStatus('Calling player…');
  }catch(error){
    console.error(error);
    setStatus('Unable to start voice call.');
  }
}

function startVoice(event){
  if(event) event.preventDefault();

  if(!localStream){
    setStatus('Allow microphone access before talking.');
    return;
  }

  voiceHeld=true;
  localStream.getAudioTracks().forEach(track=>track.enabled=true);
  document.getElementById('voiceBtn').classList.add('recording');
  setStatus('🎙 Microphone active — release to mute.');
}

function stopVoice(event){
  if(event) event.preventDefault();

  voiceHeld=false;

  if(localStream){
    localStream.getAudioTracks().forEach(track=>track.enabled=false);
  }

  document.getElementById('voiceBtn').classList.remove('recording');
  setStatus('Microphone muted.');
}

window.addEventListener('pagehide',()=>{
  if(localStream){
    localStream.getTracks().forEach(track=>track.stop());
  }
});

loadGame();
</script>

</body>
</html>
