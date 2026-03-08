+++
date = '2026-03-08T08:00:00+00:00'
draft = false
title = 'How Long Until Your First Million? An Interactive Calculator'
+++

This post is an experimentation with Claude Code. Below this paragraph all content is AI generated:

The math behind compound interest is simple, but the numbers it produces are consistently surprising. Small, regular contributions — given enough time and a decent return — compound into sums that feel disproportionate to the effort. This calculator lets you explore that relationship directly.

Move any slider. The others adjust automatically to keep the end result at exactly **€1,000,000**:

- Drag **Starting Sum**, **Annual Return**, or **Years** → Monthly Investment recalculates
- Drag **Monthly Investment** → Years recalculates

<div id="inv-calc">
<style>
#inv-calc {
  max-width: 680px;
  margin: 2.5rem auto;
  font-size: 0.95rem;
}
.slider-group {
  margin-bottom: 0.6rem;
}
.slider-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.2rem;
}
.slider-label { color: #999; }
.slider-value { font-weight: 600; color: #e2e2e2; }
input[type=range] {
  width: 100%;
  height: 4px;
  accent-color: #5b9cf6;
  cursor: pointer;
}
.summary {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.75rem;
  margin: 1rem 0 1rem;
  text-align: center;
}
.summary-item {
  background: rgba(255,255,255,0.04);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 8px;
  padding: 0.9rem 0.5rem;
}
.summary-item .s-label {
  font-size: 0.7rem;
  color: #777;
  text-transform: uppercase;
  letter-spacing: 0.07em;
}
.summary-item .s-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #5b9cf6;
  margin-top: 0.3rem;
}
.summary-item.highlight .s-value { color: #6ee7b7; }
.chart-wrap { position: relative; }
.calc-note {
  font-size: 0.75rem;
  color: #555;
  margin-top: 0.9rem;
  line-height: 1.5;
}
</style>

<div class="slider-group">
  <div class="slider-header">
    <span class="slider-label">Starting Sum</span>
    <span class="slider-value" id="disp-s">€0</span>
  </div>
  <input type="range" id="sl-s" min="0" max="50000" step="50" value="0">
</div>

<div class="slider-group">
  <div class="slider-header">
    <span class="slider-label">Monthly Investment</span>
    <span class="slider-value" id="disp-m">—</span>
  </div>
  <input type="range" id="sl-m" min="0" max="5000" step="10" value="500">
</div>

<div class="slider-group">
  <div class="slider-header">
    <span class="slider-label">Annual Return</span>
    <span class="slider-value" id="disp-r">7.0%</span>
  </div>
  <input type="range" id="sl-r" min="1" max="20" step="0.5" value="7">
</div>

<div class="slider-group">
  <div class="slider-header">
    <span class="slider-label">Years</span>
    <span class="slider-value" id="disp-y">20 years</span>
  </div>
  <input type="range" id="sl-y" min="1" max="60" step="1" value="20">
</div>

<div class="summary">
  <div class="summary-item">
    <div class="s-label">Total Invested</div>
    <div class="s-value" id="sum-invested">—</div>
  </div>
  <div class="summary-item">
    <div class="s-label">Returns</div>
    <div class="s-value" id="sum-returns">—</div>
  </div>
  <div class="summary-item highlight">
    <div class="s-label">Final Value</div>
    <div class="s-value" id="sum-final">€1,000,000</div>
  </div>
</div>

<div class="chart-wrap">
  <canvas id="inv-chart"></canvas>
</div>

<p class="calc-note">Assumes a constant annual return compounded monthly. Does not account for taxes, fees, or inflation. Past market performance does not guarantee future results.</p>

<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script>
(function () {
  const GOAL = 1000000;

  const slS = document.getElementById('sl-s');
  const slM = document.getElementById('sl-m');
  const slR = document.getElementById('sl-r');
  const slY = document.getElementById('sl-y');

  const fmt = v => '€' + Math.round(v).toLocaleString('en-US');
  const fmtR = v => (+v).toFixed(1) + '%';

  function fv(S, M, r, n) {
    if (r < 1e-10) return S + M * n;
    const g = Math.pow(1 + r, n);
    return S * g + M * (g - 1) / r;
  }

  function solveM(S, R, Y) {
    const r = R / 100 / 12;
    const n = Y * 12;
    const g = Math.pow(1 + r, n);
    if (r < 1e-10) return Math.max(0, (GOAL - S) / n);
    return Math.max(0, (GOAL - S * g) * r / (g - 1));
  }

  function solveY(S, M, R) {
    const r = R / 100 / 12;
    if (fv(S, M, r, 720) < GOAL) return 60;
    if (S >= GOAL) return 0;
    let lo = 0, hi = 720;
    for (let i = 0; i < 64; i++) {
      const mid = (lo + hi) / 2;
      fv(S, M, r, mid) < GOAL ? (lo = mid) : (hi = mid);
    }
    return Math.min(60, Math.max(1, Math.ceil(hi / 12)));
  }

  function chartData(S, M, R, Y) {
    const r = R / 100 / 12;
    const labels = [], portfolio = [], invested = [];
    for (let yr = 0; yr <= Y; yr++) {
      labels.push(yr === 0 ? 'Now' : yr + 'y');
      portfolio.push(Math.round(fv(S, M, r, yr * 12)));
      invested.push(Math.round(S + M * yr * 12));
    }
    return { labels, portfolio, invested };
  }

  const ctx = document.getElementById('inv-chart').getContext('2d');
  const chart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: [],
      datasets: [
        {
          label: 'Portfolio Value',
          data: [],
          borderColor: '#5b9cf6',
          backgroundColor: 'rgba(91,156,246,0.12)',
          fill: true,
          tension: 0.35,
          pointRadius: 0,
          borderWidth: 2,
        },
        {
          label: 'Amount Invested',
          data: [],
          borderColor: '#666',
          backgroundColor: 'transparent',
          borderDash: [5, 4],
          tension: 0.1,
          pointRadius: 0,
          borderWidth: 1.5,
        },
        {
          label: '€1M Goal',
          data: [],
          borderColor: '#6ee7b7',
          backgroundColor: 'transparent',
          borderDash: [3, 5],
          tension: 0,
          pointRadius: 0,
          borderWidth: 1.5,
        },
      ],
    },
    options: {
      responsive: true,
      interaction: { mode: 'index', intersect: false },
      plugins: {
        legend: {
          labels: { color: '#888', boxWidth: 12, font: { size: 12 } },
        },
        tooltip: {
          callbacks: {
            label: c => c.dataset.label + ': ' + fmt(c.parsed.y),
          },
        },
      },
      scales: {
        x: {
          ticks: { color: '#666', maxTicksLimit: 10 },
          grid: { color: 'rgba(255,255,255,0.05)' },
        },
        y: {
          ticks: {
            color: '#666',
            callback: v => v >= 1000000 ? '€' + (v/1000000).toFixed(1) + 'M' : '€' + (v/1000).toFixed(0) + 'k',
          },
          grid: { color: 'rgba(255,255,255,0.05)' },
        },
      },
    },
  });

  function update(changed) {
    let S = +slS.value;
    let M = +slM.value;
    let R = +slR.value;
    let Y = +slY.value;

    if (changed === 'M') {
      Y = solveY(S, M, R);
      slY.value = Y;
    } else {
      M = solveM(S, R, Y);
      // Dynamically expand slider range so the thumb position stays meaningful
      const needed = Math.max(2000, Math.ceil(M * 2 / 100) * 100);
      if (needed !== +slM.max) {
        slM.max = needed;
        slM.step = needed > 50000 ? 500 : needed > 10000 ? 100 : needed > 2000 ? 50 : 10;
      }
      slM.value = M;
    }

    document.getElementById('disp-s').textContent = fmt(S);
    const mLabel = Math.round(M);
    const mAnnual = Math.round(M * 12);
    document.getElementById('disp-m').textContent = fmt(mLabel) + '/mo  (' + fmt(mAnnual) + '/yr)';
    document.getElementById('disp-r').textContent = fmtR(R);
    document.getElementById('disp-y').textContent = Y + (Y >= 60 ? '+ years' : ' years');

    const totalInvested = S + M * Y * 12;
    const finalVal = fv(S, M, R / 100 / 12, Y * 12);
    const returns = finalVal - totalInvested;

    document.getElementById('sum-invested').textContent = fmt(totalInvested);
    document.getElementById('sum-returns').textContent = fmt(Math.max(0, returns));
    document.getElementById('sum-final').textContent = fmt(finalVal);

    const d = chartData(S, M, R, Y);
    chart.data.labels = d.labels;
    chart.data.datasets[0].data = d.portfolio;
    chart.data.datasets[1].data = d.invested;
    chart.data.datasets[2].data = d.labels.map(() => GOAL);
    chart.update('none');
  }

  slS.addEventListener('input', () => update('S'));
  slM.addEventListener('input', () => update('M'));
  slR.addEventListener('input', () => update('R'));
  slY.addEventListener('input', () => update('Y'));

  update('S');
})();
</script>
</div>

A few things worth noting when reading these numbers. The return rate matters far more than the starting sum — a 10% annual return over 25 years does most of the heavy lifting even from zero. The starting sum helps most in the early years when compounding hasn't had time to build momentum. And time, predictably, is the most powerful variable: cutting 10 years off your timeline typically requires doubling or tripling your monthly contribution to compensate.

The S&P 500 has historically returned around 7–10% annually after inflation is excluded. A 7% real return is a reasonable central estimate, though any individual period can deviate significantly. Use this to build intuition, not a retirement plan.
