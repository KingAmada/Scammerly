/*
  SCAMENOT TEMPORARY HOURLY METRICS TEST
  --------------------------------------
  Purpose:
  - Writes ONE scripted metric event to Supabase per clock hour while the site is being loaded/used.
  - Displays REAL database totals + TEMP scripted metric totals with no visual test badge.
  - Does NOT create scam reports, identities, evidence, or risk-profile allegations.
  - Remove this file + its <script> tag when testing is finished.
*/

(() => {
  'use strict';

  const CONFIG = {
    endpoint: 'https://rwvngisohnflfofikspy.supabase.co/functions/v1/scamenot-demo-metrics-api',
    publishableKey: 'sb_publishable_V3B4qEnO8hPcHofizy8zoA_TDANaLlx',

    // Random scripted increment written once per hour.
    // Change these ranges to tune how quickly the temporary counters grow.
    hourlyRange: {
      checks_today: [12, 28],
      verified_reports_today: [0, 3],
      identity_checks: [20, 45],
      high_risk_matches: [0, 4],
      incident_reports: [1, 6]
    },

    // Existing page calls loadStats() every minute and can overwrite the DOM.
    // Re-sync combined DB + scripted totals every minute, and repaint from cache in between.
    syncEveryMs: 60_000,
    repaintEveryMs: 5_000
  };

  let latestCombined = null;
  let latestResponse = null;

  function randomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }

  function makeHourlyIncrement() {
    const out = {};
    for (const [key, range] of Object.entries(CONFIG.hourlyRange)) {
      out[key] = randomInt(range[0], range[1]);
    }
    return out;
  }

  async function request(action, increments) {
    const response = await fetch(CONFIG.endpoint, {
      method: 'POST',
      headers: {
        apikey: CONFIG.publishableKey,
        'content-type': 'application/json'
      },
      body: JSON.stringify({ action, ...(increments ? { increments } : {}) })
    });

    const data = await response.json().catch(() => ({ error: 'Invalid server response' }));
    if (!response.ok) throw new Error(data.error || `Demo metrics request failed (${response.status})`);
    return data;
  }

  function formatNumber(value) {
    return Number(value || 0).toLocaleString();
  }

  function setText(id, value) {
    const el = document.getElementById(id);
    if (el) el.textContent = formatNumber(value);
  }

  function renderCombined(combined) {
    if (!combined) return;

    setText('checksCount', combined.checks_today);
    setText('reportsReviewedCount', combined.verified_reports_today);
    setText('networkChecks', combined.identity_checks);
    setText('highRiskMatches', combined.high_risk_matches);
    setText('networkReports', combined.incident_reports);
  }

  function accept(data) {
    latestResponse = data;
    latestCombined = data?.combined || null;
    renderCombined(latestCombined);
    return data;
  }

  async function tick() {
    try {
      const increments = makeHourlyIncrement();
      const data = accept(await request('tick', increments));

      if (data.wrote_new_hour) {
        console.info('[Scamenot metrics test] New hourly database increment written:', data.increment);
      } else {
        console.info('[Scamenot metrics test] This hour already exists; using stored database totals.');
      }
      console.table({ real: data.real, scripted: data.demo, combined: data.combined });
      return data;
    } catch (error) {
      console.warn('[Scamenot metrics test] Tick failed:', error);
      return null;
    }
  }

  async function sync() {
    try {
      return accept(await request('get'));
    } catch (error) {
      console.warn('[Scamenot metrics test] Sync failed:', error);
      return null;
    }
  }

  function scheduleTopOfHour() {
    const now = Date.now();
    const nextHour = Math.floor(now / 3_600_000) * 3_600_000 + 3_600_000 + 1_500;
    const delay = Math.max(1_000, nextHour - now);

    setTimeout(() => {
      tick();
      setInterval(tick, 3_600_000);
    }, delay);
  }

  function start() {

    // On first page load: creates this hour if it doesn't exist, otherwise just reads it.
    tick();

    // Pull combined totals regularly because the main page refreshes its real stats every minute.
    setInterval(sync, CONFIG.syncEveryMs);

    // Keep our combined values visible if the original page repaints the real-only counters.
    setInterval(() => renderCombined(latestCombined), CONFIG.repaintEveryMs);

    // Then create at most one new scripted database event per clock hour.
    scheduleTopOfHour();

    // If a sleeping/background tab wakes up, make sure the current hour is represented.
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'visible') tick();
    });
  }

  // Small console API for testing/debugging.
  window.ScamenotDemoMetrics = {
    tick,
    sync,
    get latest() { return latestResponse; },
    config: CONFIG
  };

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', start, { once: true });
  } else {
    start();
  }
})();
