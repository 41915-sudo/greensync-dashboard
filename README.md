<!doctype html>
<html lang="th">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>🌿 GreenSync — Realtime Dashboard</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&family=Noto+Sans+Thai:wght@400;600&display=swap" rel="stylesheet">
  <style>
    :root{
      --bg:#f5f7fa; --text:#1f2430; --muted:#667085;
      --card:#fff; --border:#e5e7eb;
      --green1:#22a16f; --green2:#a6f2c3;
      --blue1:#4285f4; --blue2:#cfe2ff;
      --red1:#ef4444; --red2:#fecaca;
      --amber1:#f59e0b; --amber2:#fde68a;
    }
    body{margin:16px;background:var(--bg);color:var(--text);
         font-family:"Noto Sans Thai","Inter",system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Arial,sans-serif;}
    h1{font-size:22px;margin:0;font-weight:700}
    header{display:flex;flex-wrap:wrap;justify-content:space-between;align-items:center;gap:10px;margin-bottom:10px}
    .controls{display:flex;flex-wrap:wrap;gap:8px;align-items:center}
    input,select,button{padding:7px 9px;border-radius:8px;border:1px solid var(--border);font:inherit}
    input:focus,select:focus{border-color:var(--green1);outline:none;box-shadow:0 0 0 2px var(--green2)}
    button{background:var(--green1);color:#fff;border:none;font-weight:600;cursor:pointer;padding:8px 12px}
    button:hover{filter:brightness(0.95)}
    button.ghost{background:#fff;color:var(--green1);border:1px solid var(--green1)}
    .cards{display:grid;grid-template-columns:repeat(auto-fit,minmax(160px,1fr));gap:10px;margin:12px 0 16px}
    .card{background:var(--card);border-radius:14px;box-shadow:0 3px 12px rgba(0,0,0,.06);
          padding:12px 10px;border:1px solid var(--border);position:relative;overflow:hidden;}
    .card::before{
      content:"";position:absolute;inset:0;border-radius:inherit;z-index:0;
      background:linear-gradient(135deg,var(--c1),var(--c2));
      opacity:.25;
    }
    .card .label{font-size:12px;color:var(--muted);position:relative;z-index:1;}
    .card .value{font-size:22px;font-weight:700;font-family:"Inter","Noto Sans Thai";position:relative;z-index:1;}
    /* เฉดสีเฉพาะหมวด */
    .soil,.light{--c1:var(--green1);--c2:var(--green2);}
    .humid,.temp{--c1:var(--blue1);--c2:var(--blue2);}
    .pm{--c1:var(--red1);--c2:var(--red2);}
    .npk{--c1:var(--amber1);--c2:var(--amber2);}
    .time{--c1:#64748b;--c2:#e2e8f0;}

    .chart-wrap{background:var(--card);border-radius:16px;border:1px solid var(--border);
                box-shadow:0 4px 16px rgba(0,0,0,.06);padding:10px;margin:16px 0;overflow:hidden}
    .chart-wrap canvas{width:100%!important;height:100%!important}
    .table-wrap{overflow:auto;background:var(--card);border-radius:12px;border:1px solid var(--border);
                box-shadow:0 3px 12px rgba(0,0,0,.04);padding:6px}
    table{border-collapse:collapse;width:100%;min-width:640px}
    th,td{border-bottom:1px solid #f0f1f5;padding:5px 6px;text-align:left;font-size:12px;line-height:1.3}
    th{background:#f9fafb;position:sticky;top:0}
    tr:hover td{background:#f9fffa}
    .footer{color:var(--muted);font-size:11px;margin-top:6px}
    /* Chart sizes */
    body.chart-normal .chart-wrap{height:300px;}
    body.chart-large  .chart-wrap{height:480px;}
    body.chart-full   .chart-wrap{height:80vh;}
  </style>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="chart-large">
  <header>
    <h1>🌿 GreenSync — Realtime Dashboard</h1>
    <div class="controls">
      <label>Sensor:<input id="sensor" value="esp32" /></label>
      <label>Rows:<select id="limit"><option>20</option><option selected>100</option></select></label>
      <label>Chart:
        <select id="chartSize">
          <option value="normal">Normal</option>
          <option value="large" selected>Large</option>
          <option value="full">Full</option>
        </select>
      </label>
      <button id="start">Start</button>
      <button id="stop" class="ghost">Stop</button>
      <span id="status" class="footer"></span>
    </div>
  </header>

  <section class="cards" id="cards"></section>

  <section class="chart-wrap">
    <canvas id="lineChart"></canvas>
  </section>

  <section class="table-wrap">
    <table id="dataTable"><thead></thead><tbody></tbody></table>
    <div class="footer" id="info"></div>
  </section>

<script>
const API_URL="https://script.google.com/macros/s/AKfycbyEMeOfF9wqrsoduyFCIlxid7lFzzOqXsIN_GxO9V6QgUY_OXx5atJKZd0fkMlg7P9mbw/exec";
const SENSOR_NAME_DEFAULT="esp32";
const POLL_MS=2000;
const $=id=>document.getElementById(id);
let chart,pollTimer,lastRowIndex=1,header=[],labels=[],temps=[],humids=[],pm25s=[],queueRows=[];

const shadowLinePlugin={id:'shadowLine',beforeDatasetDraw(c,a){const{ctx}=c;const d=c.config.data.datasets[a.index];if(!d)return;ctx.save();ctx.shadowColor=d.borderColor;ctx.shadowBlur=10;ctx.shadowOffsetY=3;},afterDatasetDraw(c){c.ctx.restore();}};
function makeGrad(ctx,rgba){const g=ctx.createLinearGradient(0,0,0,260);g.addColorStop(0,rgba.replace('1)','0.3)'));g.addColorStop(1,rgba.replace('1)','0.05)'));return g;}

async function apiExport({sensor,limit,order="asc",after=0}){
  const u=new URL(API_URL);
  u.searchParams.set("action","export");u.searchParams.set("sensor",sensor);u.searchParams.set("order",order);u.searchParams.set("_t",Date.now());
  after>0?u.searchParams.set("after",after):u.searchParams.set("limit",limit);
  const r=await fetch(u);if(!r.ok)throw new Error(`HTTP ${r.status}`);return await r.json();
}

function renderCards(v){
  const c=$("cards");c.innerHTML="";
  const list=[
    ["Time",v["Timestamp (Asia/Bangkok)"],"time"],
    ["Soil",v["Soil Moisture (%)"]+" %","soil"],
    ["Light",v["Light (%)"]+" %","light"],
    ["Humid",v["Air Humidity (%)"]+" %","humid"],
    ["Temp",v["Temperature (°C)"]+" °C","temp"],
    ["PM1.0",v["PM1.0 (µg/m³)"],"pm"],
    ["PM2.5",v["PM2.5 (µg/m³)"],"pm"],
    ["PM10",v["PM10 (µg/m³)"],"pm"],
    ["N",v["N (mg/kg)"]+" mg/kg","npk"],
    ["P",v["P (mg/kg)"]+" mg/kg","npk"],
    ["K",v["K (mg/kg)"]+" mg/kg","npk"],
  ];
  list.forEach(([label,val,cls])=>{
    const d=document.createElement("div");
    d.className=`card ${cls}`;
    d.innerHTML=`<div class="label">${label}</div><div class="value">${val??"-"}</div>`;
    c.appendChild(d);
  });
}

function initTable(h){header=h;$("dataTable").querySelector("thead").innerHTML="<tr>"+h.map(x=>`<th>${x}</th>`).join("")+"</tr>";$("dataTable").querySelector("tbody").innerHTML="";queueRows=[];}
function updateTable(){const tb=$("dataTable").querySelector("tbody");tb.innerHTML=queueRows.map(r=>"<tr>"+header.map(h=>`<td>${r[h]??""}</td>`).join("")+"</tr>").join("");}

function ensureChart(){
  if(chart)return;
  const ctx=$("lineChart").getContext("2d");
  chart=new Chart(ctx,{type:"line",plugins:[shadowLinePlugin],
    data:{labels,datasets:[
      {label:"Temp °C",data:temps,borderColor:"#22a16f",backgroundColor:makeGrad(ctx,"rgba(34,161,111,1)"),fill:true,tension:.35,borderWidth:2,pointRadius:0},
      {label:"Humidity %",data:humids,borderColor:"#4285f4",backgroundColor:makeGrad(ctx,"rgba(66,133,244,1)"),fill:true,tension:.35,borderWidth:2,pointRadius:0},
      {label:"PM2.5",data:pm25s,borderColor:"#ef4444",backgroundColor:makeGrad(ctx,"rgba(239,68,68,1)"),fill:true,tension:.35,borderWidth:2,pointRadius:0}
    ]},
    options:{responsive:true,maintainAspectRatio:false,
      plugins:{legend:{position:'top'},title:{display:true,text:'Environment (Realtime)'}},
      scales:{x:{ticks:{maxRotation:0,autoSkip:true,maxTicksLimit:8}},y:{grace:'10%'}}
    }});
}
function appendToSeries(rows){
  rows.forEach(r=>{labels.push(r["Timestamp (Asia/Bangkok)"]);temps.push(+r["Temperature (°C)"]||0);humids.push(+r["Air Humidity (%)"]||0);pm25s.push(+r["PM2.5 (µg/m³)"]||0);});
  if(labels.length>50){const cut=labels.length-50;labels.splice(0,cut);temps.splice(0,cut);humids.splice(0,cut);pm25s.splice(0,cut);}
  chart&&chart.update('none');
}

async function initialLoad(){
  const s=$("sensor").value.trim()||SENSOR_NAME_DEFAULT;
  const l=parseInt($("limit").value)||100;
  $("status").textContent="Loading...";
  const d=await apiExport({sensor:s,limit:l});
  initTable(d.header);
  labels.length=temps.length=humids.length=pm25s.length=0;
  queueRows=d.rows.slice(-5);
  updateTable();appendToSeries(d.rows);ensureChart();
  if(d.rows.length)renderCards(d.rows.at(-1));
  $("info").textContent=`Sheet: ${d.sheet} • Showing last 5 rows`;
  lastRowIndex=d.lastRow||1;
}
async function pollNewRows(){
  const s=$("sensor").value.trim()||SENSOR_NAME_DEFAULT;
  const d=await apiExport({sensor:s,order:"asc",after:lastRowIndex});
  if(!d.rows?.length)return;
  d.rows.forEach(r=>{queueRows.push(r);if(queueRows.length>5)queueRows.shift();});
  updateTable();appendToSeries(d.rows);renderCards(queueRows.at(-1));
  $("info").textContent=`Sheet: ${d.sheet} • Showing last 5 rows`;lastRowIndex=d.lastRow||lastRowIndex;
}
function startRealtime(){if(pollTimer)return;pollTimer=setInterval(pollNewRows,POLL_MS);$("status").textContent="Realtime: ON";}
function stopRealtime(){if(!pollTimer)return;clearInterval(pollTimer);pollTimer=null;$("status").textContent="Realtime: OFF";}

/* Chart size toggle */
const chartSel=$("chartSize");
chartSel.addEventListener("change",()=>{document.body.classList.remove("chart-normal","chart-large","chart-full");document.body.classList.add(`chart-${chartSel.value}`);chart&&chart.resize();});
new ResizeObserver(()=>{chart&&chart.resize();}).observe(document.querySelector(".chart-wrap"));

$("start").onclick=async()=>{stopRealtime();await initialLoad();startRealtime();};
$("stop").onclick=stopRealtime;
window.onload=async()=>{await initialLoad();startRealtime();};
</script>
</body>
</html>
