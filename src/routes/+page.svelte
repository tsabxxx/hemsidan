<script>
  import { onMount } from 'svelte';

  // ─── Constants ───────────────────────────────────────────
  const START_HOUR = 0;
  const END_HOUR   = 24;
  const PX_PER_HR  = 48;
  const STORAGE_KEY = 'planner-events';

  const DAYS_FULL = ['Måndag', 'Tisdag', 'Onsdag', 'Torsdag', 'Fredag'];
  const DAYS_ABBR = ['Må', 'Ti', 'On', 'To', 'Fr'];

  const PASTELS = [
    '#a8d8ea', '#aa96da', '#fcbad3', '#ffffba', '#b5ead7',
    '#ffdac1', '#e2f0cb', '#c7ceea', '#f0b3c7', '#d4f1f4',
  ];

  const HOUR_MARKS = Array.from({ length: END_HOUR - START_HOUR + 1 }, (_, i) => START_HOUR + i);

  // ─── State ───────────────────────────────────────────────
  let weekOffset = $state(0);
  let now        = $state(new Date());
  let events     = $state(/** @type {any[]} */([]));

  onMount(() => {
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (stored) events = JSON.parse(stored);
    } catch {}
    const tick = setInterval(() => { now = new Date(); }, 30_000);
    return () => clearInterval(tick);
  });

  function persist(next) {
    events = next;
    localStorage.setItem(STORAGE_KEY, JSON.stringify(next));
  }

  const addEvent    = (ev)     => persist([...events, { ...ev, id: crypto.randomUUID() }]);
  const editEvent   = (id, ev) => persist(events.map(e => e.id === id ? { ...e, ...ev } : e));
  const removeEvent = (id)     => persist(events.filter(e => e.id !== id));

  // ─── Derived ─────────────────────────────────────────────
  let monday = $derived.by(() => {
    const d = new Date();
    d.setHours(0, 0, 0, 0);
    const dow = d.getDay();
    d.setDate(d.getDate() + (dow === 0 ? -6 : 1 - dow) + weekOffset * 7);
    return d;
  });

  let weekDays = $derived(
    Array.from({ length: 5 }, (_, i) => {
      const d = new Date(monday);
      d.setDate(monday.getDate() + i);
      return d;
    })
  );

  let weekNum = $derived.by(() => {
    const d = new Date(Date.UTC(monday.getFullYear(), monday.getMonth(), monday.getDate()));
    const dow = d.getUTCDay() || 7;
    d.setUTCDate(d.getUTCDate() + 4 - dow);
    const y0 = new Date(Date.UTC(d.getUTCFullYear(), 0, 1));
    return Math.ceil((((d - y0) / 86400000) + 1) / 7);
  });

  let nowTop = $derived.by(() => {
    const m = now.getHours() * 60 + now.getMinutes();
    return (m - START_HOUR * 60) / 60 * PX_PER_HR;
  });

  let nowVisible = $derived.by(() => {
    const m = now.getHours() * 60 + now.getMinutes();
    return m >= START_HOUR * 60 && m < END_HOUR * 60;
  });

  // ─── Helpers ─────────────────────────────────────────────
  const toMins   = (t) => { const [h, m] = t.split(':').map(Number); return h * 60 + m; };
  const fromMins = (m) => `${String(Math.floor(m / 60)).padStart(2, '0')}:${String(m % 60).padStart(2, '0')}`;
  const dateFmt  = (d) => `${d.getDate()}/${d.getMonth() + 1}`;
  const dateKey  = (d) =>
    `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;

  const isToday = (d) =>
    d.getDate()     === now.getDate()     &&
    d.getMonth()    === now.getMonth()    &&
    d.getFullYear() === now.getFullYear();

  const evTop = (st)      => (toMins(st) - START_HOUR * 60) / 60 * PX_PER_HR;
  const evH   = (st, et)  => (toMins(et) - toMins(st))      / 60 * PX_PER_HR;

  const dayEvents = (i) => {
    const key = dateKey(weekDays[i]);
    const starting = events.filter(e => e.date === key);

    // Overnight events from the previous day that spill into this day
    const prev = new Date(weekDays[i]);
    prev.setDate(prev.getDate() - 1);
    const prevKey = dateKey(prev);
    const spilling = events
      .filter(e => e.date === prevKey && e.overnight)
      .map(e => ({ ...e, _cont: true }));

    return [...starting, ...spilling];
  };

  // ─── Modal ───────────────────────────────────────────────
  let modal = $state(/** @type {{ mode: string; ev: any } | null} */(null));

  function openAdd(dayIndex, clickY) {
    const raw  = Math.round((clickY / PX_PER_HR * 60 + START_HOUR * 60) / 15) * 15;
    const sMin = Math.max(START_HOUR * 60, Math.min(raw, END_HOUR * 60 - 30));
    const eMin = Math.min(sMin + 60, END_HOUR * 60);
    modal = {
      mode: 'add',
      ev: {
        date: dateKey(weekDays[dayIndex]),
        startTime: fromMins(sMin),
        endTime:   fromMins(eMin % (END_HOUR * 60)),
        overnight: false,
        title: '', subtitle: '', room: '', color: PASTELS[0],
      },
    };
  }

  function openEdit(event) {
    modal = { mode: 'edit', ev: { ...event } };
  }

  const closeModal = () => { modal = null; };

  function save() {
    const overnight = toMins(modal.ev.startTime) >= toMins(modal.ev.endTime);
    const ev = { ...modal.ev, overnight };
    if (modal.mode === 'add') addEvent(ev);
    else                      editEvent(ev.id, ev);
    closeModal();
  }

  function del() {
    removeEvent(modal.ev.id);
    closeModal();
  }

  function onColumnClick(e, i) {
    if (/** @type {HTMLElement} */(e.target).closest('.ev')) return;
    openAdd(i, e.clientY - e.currentTarget.getBoundingClientRect().top);
  }

  function onKey(e) {
    if (e.key === 'Escape') closeModal();
  }
</script>

<svelte:window onkeydown={onKey} />

<!-- ════════════════════════════════════════════════════════ -->
<!--  Planner                                                 -->
<!-- ════════════════════════════════════════════════════════ -->
<div class="planner">

  <!-- Header -->
  <header>
    <div class="hdr-left">
      <span class="wk-num">V.{weekNum}</span>
      <div class="nav">
        <button onclick={() => weekOffset--} title="Föregående vecka">‹</button>
        <button class="today-btn" onclick={() => weekOffset = 0}>Idag</button>
        <button onclick={() => weekOffset++} title="Nästa vecka">›</button>
      </div>
    </div>

    <div class="day-tabs">
      {#each weekDays as day, i}
        <div class="day-tab" class:today={isToday(day)}>
          <span class="abbr">{DAYS_ABBR[i]}</span>
          <span class="date-lbl">{dateFmt(day)}</span>
        </div>
      {/each}
    </div>
  </header>

  <!-- Grid -->
  <div class="grid-wrap">

    <!-- Time gutter -->
    <div class="gutter">
      {#each HOUR_MARKS as h}
        <span class="hr-lbl" style:top="{(h - START_HOUR) * PX_PER_HR}px">
          {String(h).padStart(2, '0')}:00
        </span>
      {/each}
    </div>

    <!-- Day columns -->
    <div class="cols">
      {#each weekDays as day, i}
        <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
        <div
          class="col"
          class:today={isToday(day)}
          onclick={(e) => onColumnClick(e, i)}
        >
          <!-- Hour and half-hour lines -->
          {#each HOUR_MARKS.slice(0, -1) as h}
            <div class="hr-line"   style:top="{(h - START_HOUR) * PX_PER_HR}px"></div>
            <div class="half-line" style:top="{(h - START_HOUR) * PX_PER_HR + PX_PER_HR / 2}px"></div>
          {/each}
          <!-- Bottom border of last hour -->
          <div class="hr-line" style:top="{(END_HOUR - START_HOUR) * PX_PER_HR}px"></div>

          <!-- Current-time indicator -->
          {#if isToday(day) && nowVisible}
            <div class="now-line" style:top="{nowTop}px"></div>
          {/if}

          <!-- Events -->
          {#each dayEvents(i) as ev (ev._cont ? ev.id + '_c' : ev.id)}
            {@const top    = ev._cont ? 0 : evTop(ev.startTime)}
            {@const height = ev._cont
              ? Math.max(evH('00:00', ev.endTime), 8)
              : ev.overnight
                ? Math.max((END_HOUR - START_HOUR) * PX_PER_HR - evTop(ev.startTime), 8)
                : Math.max(evH(ev.startTime, ev.endTime), 20)}
            <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
            <div
              class="ev"
              class:ev-cont={ev._cont}
              style:top="{top}px"
              style:height="{height}px"
              style:background={ev.color}
              onclick={(e) => { e.stopPropagation(); openEdit(ev); }}
            >
              <span class="ev-header">
                <b class="ev-title">{ev.title || '–'}</b>
                <time class="ev-time">
                  {ev._cont ? '00:00' : ev.startTime}–{ev.overnight && !ev._cont ? '→' : ev.endTime}
                </time>
              </span>
              {#if ev.room}<small class="ev-room">{ev.room}</small>{/if}
            </div>
          {/each}
        </div>
      {/each}
    </div>
  </div>
</div>

<!-- ════════════════════════════════════════════════════════ -->
<!--  Modal                                                   -->
<!-- ════════════════════════════════════════════════════════ -->
{#if modal}
  <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
  <div class="backdrop" onclick={closeModal}>
    <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
    <div class="modal" role="dialog" aria-modal="true" onclick={(e) => e.stopPropagation()}>
      <h2>{modal.mode === 'add' ? 'Ny händelse' : 'Redigera händelse'}</h2>

      <label class="field">
        <span>Dag</span>
        <select bind:value={modal.ev.date}>
          {#each weekDays as d, i}
            <option value={dateKey(d)}>{DAYS_FULL[i]} {dateFmt(d)}</option>
          {/each}
        </select>
      </label>

      <div class="row2">
        <label class="field">
          <span>Starttid</span>
          <input type="time" bind:value={modal.ev.startTime} step="300" />
        </label>
        <label class="field">
          <span>Sluttid {toMins(modal.ev.startTime) >= toMins(modal.ev.endTime) ? '(nästa dag)' : ''}</span>
          <input type="time" bind:value={modal.ev.endTime} step="300" />
        </label>
      </div>

      <label class="field">
        <span>Aktivitet</span>
        <input type="text" bind:value={modal.ev.title} placeholder="" />
      </label>
      <label class="field">
        <span>Plats</span>
        <input type="text" bind:value={modal.ev.room} placeholder="" />
      </label>

      <p class="field-lbl">Färg</p>
      <div class="swatches">
        {#each PASTELS as c}
          <button
            class="swatch"
            class:sel={modal.ev.color === c}
            style:background={c}
            onclick={() => { modal.ev.color = c; }}
            aria-label="Välj färg"
          ></button>
        {/each}
      </div>

      <div class="actions">
        {#if modal.mode === 'edit'}
          <button class="btn-del" onclick={del}>Radera</button>
        {/if}
        <button class="btn-cancel" onclick={closeModal}>Avbryt</button>
        <button class="btn-save"   onclick={save}>Spara</button>
      </div>
    </div>
  </div>
{/if}

<!-- ════════════════════════════════════════════════════════ -->
<!--  Styles                                                  -->
<!-- ════════════════════════════════════════════════════════ -->
<style>
  @import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500&family=IBM+Plex+Sans:wght@400;500;600&display=swap');

  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) { overflow: hidden; }

  /* ── Planner shell ─────────────────────────────────────── */
  .planner {
    font-family: 'IBM Plex Sans', sans-serif;
    background: #fff;
    user-select: none;
    min-width: 560px;
    height: 100dvh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  /* ── Header ────────────────────────────────────────────── */
  header {
    display: flex;
    background: #1b4332;
    color: #fff;
  }

  .hdr-left {
    width: 80px;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 12px 6px;
    border-right: 1px solid rgba(255, 255, 255, .12);
  }

  .wk-num {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 13px;
    font-weight: 500;
    color: #a7f3d0;
    letter-spacing: .05em;
  }

  .nav {
    display: flex;
    flex-direction: column;
    gap: 3px;
    width: 100%;
  }

  .nav button {
    background: rgba(255, 255, 255, .10);
    border: none;
    color: #fff;
    border-radius: 5px;
    cursor: pointer;
    font-size: 16px;
    padding: 2px 4px;
    width: 100%;
    line-height: 1.4;
    transition: background .15s;
  }

  .nav button:hover { background: rgba(255, 255, 255, .22); }

  .today-btn {
    font-family: 'IBM Plex Sans', sans-serif !important;
    font-size: 11px !important;
    padding: 3px 4px !important;
    font-weight: 500 !important;
  }

  .day-tabs { display: flex; flex: 1; }

  .day-tab {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 12px 4px;
    gap: 3px;
    border-right: 1px solid rgba(255, 255, 255, .09);
    position: relative;
    transition: background .15s;
  }
  .day-tab:last-child { border-right: none; }

  .day-tab.today { background: rgba(255, 255, 255, .14); }
  .day-tab.today::after {
    content: '';
    position: absolute;
    bottom: 0; left: 22%; right: 22%;
    height: 3px;
    background: #86efac;
    border-radius: 2px 2px 0 0;
  }

  .abbr {
    font-size: 13px;
    font-weight: 600;
    letter-spacing: .06em;
    text-transform: uppercase;
  }

  .date-lbl {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    opacity: .68;
  }

  /* ── Grid wrapper ──────────────────────────────────────── */
  .grid-wrap {
    display: flex;
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    overflow-x: hidden;
  }

  .gutter {
    width: 80px;
    flex-shrink: 0;
    position: relative;
    height: 1152px;
    border-right: 1px solid #e5e7eb;
    background: #fafafa;
  }

  .hr-lbl {
    position: absolute;
    right: 8px;
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    color: #9ca3af;
    transform: translateY(-50%);
    white-space: nowrap;
    pointer-events: none;
  }

  /* ── Day columns ───────────────────────────────────────── */
  .cols { display: flex; flex: 1; }

  .col {
    flex: 1;
    position: relative;
    height: 1152px;
    border-right: 1px solid #e5e7eb;
    cursor: crosshair;
    transition: background .1s;
  }
  .col:last-child { border-right: none; }
  .col:not(.today):hover { background: rgba(0, 0, 0, .018); }
  .col.today            { background: #f0fdf4; }
  .col.today:hover      { background: #ecfdf5; }

  .hr-line   { position: absolute; inset-inline: 0; height: 1px; background: #e5e7eb; pointer-events: none; }
  .half-line { position: absolute; inset-inline: 0; height: 1px; background: #f3f4f6; pointer-events: none; }

  /* ── Now indicator ─────────────────────────────────────── */
  .now-line {
    position: absolute;
    inset-inline: 0;
    height: 2px;
    background: #ef4444;
    z-index: 10;
    pointer-events: none;
  }
  .now-line::before {
    content: '';
    position: absolute;
    left: -4px; top: -4px;
    width: 10px; height: 10px;
    border-radius: 50%;
    background: #ef4444;
  }

  /* ── Events ────────────────────────────────────────────── */
  .ev {
    position: absolute;
    left: 3px; right: 3px;
    border-radius: 5px;
    padding: 2px 5px;
    cursor: pointer;
    overflow: hidden;
    z-index: 5;
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 1px;
    border: 1px solid rgba(0, 0, 0, .08);
    transition: box-shadow .15s, filter .12s;
  }
  .ev:hover {
    box-shadow: 0 3px 10px rgba(0, 0, 0, .18);
    filter: brightness(.94);
  }
  .ev-cont {
    border-top: 2px dashed rgba(0,0,0,.25);
    border-radius: 0 0 5px 5px;
  }

  .ev-header {
    display: flex;
    align-items: baseline;
    gap: 5px;
    overflow: hidden;
  }

  .ev-title {
    font-size: 11px;
    font-weight: 600;
    color: #1f2937;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex-shrink: 1;
    min-width: 0;
  }
  .ev-room {
    font-size: 10px;
    color: #6b7280;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .ev-time {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    color: #4b5563;
    white-space: nowrap;
    flex-shrink: 0;
  }

  /* ── Modal ─────────────────────────────────────────────── */
  .backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, .45);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 200;
    backdrop-filter: blur(3px);
  }

  .modal {
    background: #fff;
    border-radius: 14px;
    padding: 24px 24px 20px;
    width: 348px;
    max-width: 95vw;
    box-shadow: 0 24px 64px rgba(0, 0, 0, .22);
  }

  .modal h2 {
    font-size: 15px;
    font-weight: 600;
    color: #1b4332;
    margin-bottom: 16px;
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 10px;
  }

  .field > span,
  .field-lbl {
    font-size: 11px;
    font-weight: 500;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: .07em;
  }

  .field-lbl { margin-bottom: 7px; }

  .modal input,
  .modal select {
    font-family: 'IBM Plex Sans', sans-serif;
    font-size: 14px;
    border: 1px solid #e5e7eb;
    border-radius: 7px;
    padding: 7px 10px;
    color: #1f2937;
    outline: none;
    transition: border-color .15s;
    width: 100%;
  }
  .modal input:focus,
  .modal select:focus { border-color: #1b4332; }

  .row2 {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
  }

  /* ── Colour swatches ───────────────────────────────────── */
  .swatches {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 18px;
  }

  .swatch {
    width: 28px; height: 28px;
    border-radius: 50%;
    border: 3px solid transparent;
    cursor: pointer;
    transition: transform .12s, border-color .12s;
  }
  .swatch:hover { transform: scale(1.18); }
  .swatch.sel   { border-color: #1b4332; transform: scale(1.12); }

  /* ── Modal actions ─────────────────────────────────────── */
  .actions {
    display: flex;
    gap: 8px;
    justify-content: flex-end;
  }

  .actions button {
    font-family: 'IBM Plex Sans', sans-serif;
    font-size: 13px;
    font-weight: 500;
    border: none;
    border-radius: 7px;
    padding: 8px 16px;
    cursor: pointer;
    transition: background .15s;
  }

  .btn-del    { background: #fee2e2; color: #dc2626; margin-right: auto; }
  .btn-del:hover { background: #fecaca; }

  .btn-cancel { background: #f3f4f6; color: #6b7280; }
  .btn-cancel:hover { background: #e5e7eb; }

  .btn-save   { background: #1b4332; color: #fff; }
  .btn-save:hover { background: #2d6a4f; }
</style>
