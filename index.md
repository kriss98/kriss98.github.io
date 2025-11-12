<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Work‑Week Progress</title>
  <meta name="description" content="Real‑time progress bar for the work week (Mon–Fri, 09:00–18:00), auto‑resets weekly." />
  <style>
    :root{
      --bg: #0b0d10;
      --fg: #e6eef7;
      --muted:#9bb0c9;
      --bar-bg:#1b2230;
      --bar-fill:#4aa3ff;
      --accent:#8dd59b;
      --card:#12161d;
    }
    @media (prefers-color-scheme: light) {
      :root{
        --bg:#f7f8fb; --fg:#0b1220; --muted:#445064; --bar-bg:#dfe7f3; --bar-fill:#2d6cdf; --accent:#137a37; --card:#ffffff;
      }
    }

    *{box-sizing:border-box}
    html,body{height:100%}
    body{
      margin:0; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Inter, "Helvetica Neue", Arial, "Apple Color Emoji", "Segoe UI Emoji";
      background: radial-gradient(1200px 600px at 50% -10%, rgba(74,163,255,.15), transparent), var(--bg);
      color:var(--fg);
      display:grid; place-items:center; padding:24px;
    }

    .wrap{width:min(900px, 92vw)}

    .card{
      background:var(--card);
      border:1px solid rgba(100,120,150,.15);
      border-radius:20px; padding:24px 24px 18px; box-shadow: 0 10px 30px rgba(0,0,0,.15);
    }

    h1{ margin:0 0 6px; font-size: clamp(22px, 3vw, 34px); letter-spacing:.2px }
    .sub{color:var(--muted); font-size:14px; margin-bottom:22px}

    .bar{
      width:100%; height:24px; background:var(--bar-bg); border-radius:999px; overflow:hidden; position:relative;
    }
    .fill{
      height:100%; width:0%; background:linear-gradient(90deg, var(--bar-fill), #7cd4ff); border-radius:999px; transition: width 200ms ease-out;
    }

    .row{display:flex; gap:16px; align-items:center; margin-top:12px; flex-wrap:wrap}
    .kpi{flex:1 1 180px; background:linear-gradient(180deg, rgba(255,255,255,.03), rgba(255,255,255,0)); border:1px solid rgba(100,120,150,.14); padding:12px 14px; border-radius:14px}
    .kpi .lab{color:var(--muted); font-size:12px; letter-spacing:.2px}
    .kpi .val{font-weight:700; font-size:18px; margin-top:2px}

    footer{margin-top:20px; color:var(--muted); font-size:12px}
    code {background: rgba(100,120,150,.12); padding:.2em .45em; border-radius:6px}

    .sr-only{position:absolute; width:1px; height:1px; padding:0; margin:-1px; overflow:hidden; clip:rect(0,0,0,0); border:0}
  </style>
</head>
<body>
  <main class="wrap">
    <section class="card" role="group" aria-labelledby="title">
      <h1 id="title">Work‑Week Progress</h1>
      <div class="sub">Mon–Fri · 09:00–18:00 · local time</div>

      <div class="bar" aria-label="Work week progress" role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="0">
        <div id="fill" class="fill"></div>
        <span class="sr-only" id="sr">0%</span>
      </div>
      <div class="row">
        <div class="kpi"><div class="lab">Completion</div><div class="val" id="pct">0%</div></div>
        <div class="kpi"><div class="lab">Worked this week</div><div class="val" id="worked">0h 00m</div></div>
        <div class="kpi"><div class="lab">Remaining</div><div class="val" id="remain">45h 00m</div></div>
      </div>

      <footer>
        Drop this file as <code>index.html</code> into a GitHub Pages repo. No dependencies. Updates in real time.
      </footer>
    </section>
  </main>

  <script>
    const START_HOUR = 9;
    const END_HOUR   = 18;
    const WORK_DAYS = [1,2,3,4,5];

    function startOfWeekMonday0900(d = new Date()){
      const dt = new Date(d);
      const day = dt.getDay();
      const diffToMonday = (day + 6) % 7;
      dt.setHours(START_HOUR,0,0,0);
      dt.setDate(dt.getDate() - diffToMonday);
      return dt;
    }

    function intervalsForWeek(d = new Date()){
      const mon = startOfWeekMonday0900(d);
      const spans = [];
      for(let i=0;i<5;i++){
        const start = new Date(mon);
        start.setDate(start.getDate()+i);
        const end = new Date(start);
        end.setHours(END_HOUR,0,0,0);
        spans.push({start, end});
      }
      return spans;
    }

    function msToHM(ms){
      ms = Math.max(0, Math.floor(ms));
      const h = Math.floor(ms / 3_600_000);
      const m = Math.floor((ms % 3_600_000) / 60_000);
      return `${h}h ${m.toString().padStart(2,'0')}m`;
    }

    function computeProgress(now = new Date()){
      const spans = intervalsForWeek(now);
      const total = spans.reduce((acc,s)=> acc + (s.end - s.start), 0);
      let done = 0;
      for(const s of spans){
        if(now <= s.start) break;
        if(now >= s.end){ done += (s.end - s.start); continue; }
        done += (now - s.start);
        break;
      }
      const lastEnd = spans[spans.length-1].end;
      if(now >= lastEnd) done = total;
      if(now < spans[0].start) done = 0;
      const pct = total === 0 ? 0 : (done / total) * 100;
      const remainMs = Math.max(0, total - done);
      return { pct: Math.max(0, Math.min(100, pct)), doneMs: done, remainMs };
    }

    const fillEl = document.getElementById('fill');
    const pctEl = document.getElementById('pct');
    const workedEl = document.getElementById('worked');
    const remainEl = document.getElementById('remain');
    const barEl = document.querySelector('.bar');
    const srEl = document.getElementById('sr');

    function update(){
      const now = new Date();
      const {pct, doneMs, remainMs} = computeProgress(now);
      const pctRounded = Math.floor(pct * 100) / 100;
      fillEl.style.width = pct.toFixed(6) + '%';
      barEl.setAttribute('aria-valuenow', pctRounded.toString());
      pctEl.textContent = pctRounded.toFixed(pctRounded % 1 === 0 ? 0 : 2) + '%';
      workedEl.textContent = msToHM(doneMs);
      remainEl.textContent = msToHM(remainMs);
      srEl.textContent = pctRounded + '%';
    }

    update();
    setInterval(update, 1000);
  </script>
</body>
</html>
