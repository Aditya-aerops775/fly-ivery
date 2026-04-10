[api.js](https://github.com/user-attachments/files/26638734/api.js)
/**
 * Fly-Ivery — API Client
 * Connects the frontend to the Django REST backend.
 *
 * Base URL: change to your deployed server address in production.
 */

const API_BASE = 'http://127.0.0.1:8000/api';

// ─────────────────────────────────────────────────────────────────────────────
// Utility
// ─────────────────────────────────────────────────────────────────────────────

async function apiPost(endpoint, body) {
  const res = await fetch(`${API_BASE}${endpoint}`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(body),
  });
  return res.json();
}

async function apiGet(endpoint) {
  const res = await fetch(`${API_BASE}${endpoint}`);
  return res.json();
}

function showToast(msg, type = 'success') {
  const toast = document.getElementById('toast');
  if (!toast) return;
  toast.textContent = msg;
  toast.className   = `toast toast-${type} show`;
  setTimeout(() => toast.classList.remove('show'), 3500);
}

// ─────────────────────────────────────────────────────────────────────────────
// Page router (same as original)
// ─────────────────────────────────────────────────────────────────────────────

function showPage(id) {
  document.querySelectorAll('.page').forEach(p => p.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  window.scrollTo(0, 0);
}

function switchTab(el) {
  el.closest('.tab-bar').querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  el.classList.add('active');
}

// ─────────────────────────────────────────────────────────────────────────────
// Quote — calls GET /api/quote/ with form data
// ─────────────────────────────────────────────────────────────────────────────

const weightMap = {
  'Under 0.5 kg': '0.5',
  '0.5 – 2 kg':   '2',
  '2 – 5 kg':     '5',
  '5 – 10 kg':    '10',
  '10 – 25 kg':   '25',
};

async function updateQuote() {
  const weightSel = document.querySelector('select.form-input');
  const weight    = weightSel ? (weightMap[weightSel.value] || '2') : '2';

  try {
    const data = await apiPost('/quote/', { weight });
    document.getElementById('quotePrice').textContent = '₹' + data.total_price;
    document.getElementById('quoteDist').textContent  = data.distance_km + ' km';
    const payBtn = document.getElementById('payBtn');
    if (payBtn) payBtn.textContent = '₹' + data.total_price;
    if (document.getElementById('etaText'))
      document.getElementById('etaText').textContent = data.eta_minutes + '–' + (data.eta_minutes + 4) + ' min';
  } catch (_) {
    // Fail silently — network might not be up during prototype preview
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// Drone status panel (visual only — no API call needed)
// ─────────────────────────────────────────────────────────────────────────────

function updateDroneStatus() {
  const pickup = document.getElementById('pickupInput').value.trim();
  const drop   = document.getElementById('dropInput').value.trim();
  const panel  = document.getElementById('dronePanel');
  if (pickup && drop) {
    document.getElementById('dronePickup').textContent = pickup;
    document.getElementById('droneDrop').textContent   = drop;
    panel.style.display = 'block';
    updateQuote();
  } else {
    panel.style.display = 'none';
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// Confirm Booking — POST /api/book/
// ─────────────────────────────────────────────────────────────────────────────

async function confirmBooking() {
  const pickup  = document.getElementById('pickupInput').value.trim();
  const drop    = document.getElementById('dropInput').value.trim();
  const nameEl  = document.querySelector('input[placeholder="Full name"]');
  const phoneEl = document.querySelector('input[placeholder="+91 XXXXX XXXXX"]');
  const weightEl= document.querySelector('select.form-input');
  const serviceEl = document.querySelectorAll('select.form-input')[1];

  const senderName = nameEl  ? nameEl.value.trim()  : '';
  const phone      = phoneEl ? phoneEl.value.trim()  : '';

  // Validation
  if (!pickup || !drop)        { showToast('Please enter pickup and drop locations.', 'error'); return; }
  if (!senderName)             { showToast('Please enter sender name.', 'error');   return; }
  if (!phone)                  { showToast('Please enter phone number.', 'error');  return; }

  const btn = document.getElementById('confirmBtn');
  if (btn) { btn.disabled = true; btn.textContent = 'Booking…'; }

  const weight      = weightEl  ? (weightMap[weightEl.value]  || '2')       : '2';
  const serviceRaw  = serviceEl ? serviceEl.value : 'Express (under 30 min)';
  const serviceMap  = {
    'Express (under 30 min)': 'express',
    'Standard (1–2 hrs)':     'standard',
    'Medical grade':          'medical',
    'Scheduled slot':         'scheduled',
  };
  const service_type = serviceMap[serviceRaw] || 'express';

  try {
    const data = await apiPost('/book/', {
      pickup, drop,
      sender_name: senderName,
      phone, weight, service_type,
    });

    if (data.success) {
      showToast(`✅ Order ${data.order_id} placed! Drone ${data.drone_id} dispatched.`);
      // Auto-track the new order
      setTimeout(() => {
        trackOrder(data.order_id);
        showPage('track');
      }, 800);
    } else {
      showToast(data.error || 'Booking failed. Try again.', 'error');
    }
  } catch (err) {
    showToast('Server unreachable. Is Django running?', 'error');
  } finally {
    if (btn) { btn.disabled = false; btn.textContent = 'Confirm & Pay'; }
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// Track Order — GET /api/track/<id>/  or  /api/track/phone/<phone>/
// ─────────────────────────────────────────────────────────────────────────────

async function loadTracking() {
  const raw = document.getElementById('trackInput').value.trim();
  if (!raw) { showToast('Enter an Order ID or phone number.', 'error'); return; }

  const isPhone   = /^\+?[\d\s-]{10,}$/.test(raw);
  const endpoint  = isPhone
    ? `/track/phone/${encodeURIComponent(raw)}/`
    : `/track/${encodeURIComponent(raw.toUpperCase())}/`;

  const btn = document.getElementById('trackBtn');
  if (btn) { btn.disabled = true; btn.textContent = 'Searching…'; }

  try {
    const data = await apiGet(endpoint);
    if (data.success) {
      renderTrackCard(data.order);
    } else {
      showToast(data.error || 'Order not found.', 'error');
    }
  } catch (_) {
    showToast('Server unreachable. Is Django running?', 'error');
  } finally {
    if (btn) { btn.disabled = false; btn.textContent = 'Track Package'; }
  }
}

async function trackOrder(orderId) {
  try {
    const data = await apiGet(`/track/${orderId}/`);
    if (data.success) renderTrackCard(data.order);
  } catch (_) {}
}

function renderTrackCard(order) {
  const STATUS_STEPS = ['confirmed', 'picked_up', 'dispatched', 'in_flight', 'delivered'];
  const STEP_LABELS  = { confirmed: 'Confirmed', picked_up: 'Picked Up', dispatched: 'Dispatched', in_flight: 'In Flight', delivered: 'Delivered' };
  const STEP_ICONS   = { confirmed: '✓', picked_up: '✓', dispatched: '✓', in_flight: '🚁', delivered: '📦' };

  const currentIdx = STATUS_STEPS.indexOf(order.status);

  const stepsHTML = STATUS_STEPS.map((step, idx) => {
    let cls  = 'step-pending';
    if (idx <  currentIdx) cls = 'step-done';
    if (idx === currentIdx) cls = 'step-active';
    const icon = idx <= currentIdx ? STEP_ICONS[step] : '';
    return `
      <div class="step ${cls}">
        <div class="step-icon" style="font-size:12px;">${icon}</div>
        <div class="step-name">${STEP_LABELS[step]}</div>
      </div>`;
  }).join('');

  const eventsHTML = order.events.map(e =>
    `<div style="font-size:12px;color:var(--muted);padding:4px 0;border-bottom:0.5px solid var(--border);">
       <span style="color:var(--text);">${e.time}</span> — ${e.message}
     </div>`
  ).join('');

  const card = document.getElementById('trackCard');
  if (!card) return;
  card.style.display = 'block';
  card.innerHTML = `
    <div class="track-header">
      <div>
        <div style="font-size:12px;color:var(--muted);margin-bottom:4px;">Order ID</div>
        <div class="track-id">${order.order_id}</div>
        <div style="font-size:12px;color:var(--muted);margin-top:4px;">Placed ${order.placed_at} · From ${order.customer}</div>
      </div>
      <div class="status-badge"><div class="badge-dot"></div> ${order.status_display}</div>
    </div>

    <div class="map-placeholder">
      <div class="map-grid"></div>
      <div class="map-dot" style="left:22%;top:60%;"></div>
      <div class="map-dot" style="left:78%;top:45%;width:8px;height:8px;background:var(--warm);animation:none;"></div>
      <div class="map-drone">🚁</div>
      <div style="position:absolute;bottom:12px;left:16px;font-size:11px;color:var(--muted);">${order.pickup} → ${order.drop}</div>
      <div style="position:absolute;bottom:12px;right:16px;font-size:11px;color:var(--accent);font-weight:500;">ETA: ${order.eta_minutes} min</div>
      <div style="position:absolute;top:12px;left:16px;font-size:10px;color:var(--muted);background:var(--surface2);padding:4px 10px;border-radius:20px;border:0.5px solid var(--border);">Live map • ${order.distance_km} km route</div>
    </div>

    <div class="track-progress">
      <div class="progress-bar"><div class="progress-fill" style="width:${order.progress_pct}%"></div></div>
      <div class="progress-label"><span>Order placed</span><span style="color:var(--accent)">${order.progress_pct}% complete</span></div>
    </div>

    <div class="track-steps">${stepsHTML}</div>

    <div style="margin-top:1.5rem;padding-top:1.5rem;border-top:0.5px solid var(--border);display:grid;grid-template-columns:repeat(auto-fit,minmax(100px,1fr));gap:1rem;">
      <div><div style="font-size:11px;color:var(--muted);margin-bottom:4px;">Drone ID</div><div style="font-size:13px;font-weight:500;">${order.drone_id || '—'}</div></div>
      <div><div style="font-size:11px;color:var(--muted);margin-bottom:4px;">Altitude</div><div style="font-size:13px;font-weight:500;color:var(--accent);">${order.drone_altitude ? order.drone_altitude + ' m AGL' : '—'}</div></div>
      <div><div style="font-size:11px;color:var(--muted);margin-bottom:4px;">Battery</div><div style="font-size:13px;font-weight:500;color:#4ade80;">${order.drone_battery ? order.drone_battery + '%' : '—'}</div></div>
      <div><div style="font-size:11px;color:var(--muted);margin-bottom:4px;">Total Paid</div><div style="font-size:13px;font-weight:500;">₹${order.total_price}</div></div>
    </div>

    ${eventsHTML ? `<div style="margin-top:1rem;"><div style="font-size:12px;font-weight:600;margin-bottom:8px;">Timeline</div>${eventsHTML}</div>` : ''}
  `;
}

