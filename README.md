<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>🌑 Plasma & Shadow Effects</title>
<style>
html,body{margin:0;overflow:hidden;background:#00030a;font-family:system-ui}
canvas{display:block}
.hud{
  position:absolute;left:16px;top:16px;
  padding:14px 18px;border-radius:14px;
  background:rgba(255,255,255,0.06);
  backdrop-filter:blur(10px);
  color:#d9f3ff;
  font-size:13px;
  min-width:240px;
}
.ood{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

// Earth
const earth={x:w/2,y:h/2,r:70,mu:9000};

// Ground Station
const gs={angle:Math.PI/4,x:0,y:0};

// Satellite
const sat={x:earth.x,y:earth.y-220,vx:2.3,vy:0};

// Sun direction (for shadow)
const sun={angle:0};

// Radio constants
const txPower=120;
const maxDist=360;

// Physics
function gravity(){
  const dx=earth.x-sat.x;
  const dy=earth.y-sat.y;
  const d=Math.hypot(dx,dy);
  const f=earth.mu/(d*d);
  sat.vx+=f*dx/d;
  sat.vy+=f*dy/d;
}

function updateGS(){
  gs.angle+=0.0008;
  gs.x=earth.x+Math.cos(gs.angle)*earth.r;
  gs.y=earth.y+Math.sin(gs.angle)*earth.r;
}

// Line of sight
function hasLOS(){
  const dx=sat.x-gs.x, dy=sat.y-gs.y;
  const d=Math.hypot(dx,dy);
  if(d>maxDist) return false;

  const t=((earth.x-gs.x)*dx+(earth.y-gs.y)*dy)/(d*d);
  const px=gs.x+t*dx;
  const py=gs.y+t*dy;
  return Math.hypot(px-earth.x,py-earth.y)>earth.r;
}

// Shadow check (satellite in eclipse)
function inShadow(){
  const lx=Math.cos(sun.angle);
  const ly=Math.sin(sun.angle);

  const dx=sat.x-earth.x;
  const dy=sat.y-earth.y;
  const proj=dx*lx+dy*ly;
  if(proj<0) return false;

  const perp=Math.abs(-dx*ly+dy*lx);
  return perp<earth.r;
}

// Plasma model
function plasmaNoise(dist){
  // plasma density fluctuates
  const turbulence=Math.sin(Date.now()*0.002+dist*0.05);
  return 0.04 + Math.abs(turbulence)*0.12;
}

// Signal model
function signalModel(){
  if(!hasLOS()) return null;

  const dx=sat.x-gs.x;
  const dy=sat.y-gs.y;
  const dist=Math.hypot(dx,dy);

  let signal=txPower/(dist*dist);

  // Shadow attenuation
  const shadow=inShadow();
  if(shadow) signal*=0.25;

  // Plasma noise
  const noise=plasmaNoise(dist);
  const snr=signal/noise;

  return {signal,noise,snr,shadow};
}

function update(){
  gravity();
  sat.x+=sat.vx;
  sat.y+=sat.vy;
  updateGS();
  sun.angle+=0.0003;
}

function draw(){
  ctx.fillStyle="rgba(0,3,10,0.35)";
  ctx.fillRect(0,0,w,h);

  update();

  // Earth
  ctx.beginPath();
  ctx.arc(earth.x,earth.y,earth.r,0,Math.PI*2);
  ctx.fillStyle="#0b3d91";
  ctx.fill();

  // Ground station
  ctx.beginPath();
  ctx.arc(gs.x,gs.y,4,0,Math.PI*2);
  ctx.fillStyle="#00ffcc";
  ctx.fill();

  // Satellite
  ctx.beginPath();
  ctx.arc(sat.x,sat.y,4,0,Math.PI*2);
  ctx.fillStyle="#e6f7ff";
  ctx.fill();

  const data=signalModel();
  let hud="🌍 Ground Station<br>🛰️ Satellite<br>";

  if(data){
    let cls="good";
    if(data.snr<6) cls="bad";
    else if(data.snr<12) cls="mid";

    ctx.strokeStyle=
      data.snr>12 ? "rgba(120,220,255,0.9)" :
      data.snr>6  ? "rgba(255,210,120,0.7)" :
                    "rgba(255,120,120,0.6)";
    ctx.lineWidth=1.6;
    ctx.beginPath();
    ctx.moveTo(gs.x,gs.y);
    ctx.lineTo(sat.x,sat.y);
    ctx.stroke();

    hud+=`
📶 Signal: ${data.signal.toFixed(4)}<br>
⚡ Plasma Noise: ${data.noise.toFixed(3)}<br>
📊 SNR: <span class="${cls}">${data.snr.toFixed(1)}</span><br>
🌑 Shadow: ${data.shadow?"YES":"NO"}<br>
📡 Link: <span class="${cls}">
${data.snr>4?"CONNECTED":"UNUSABLE"}
</span>`;
  }else{
    hud+=`📡 Link: <span class="bad">NO SIGNAL</span>`;
  }

  document.getElementById("hud").innerHTML=hud;
  requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
