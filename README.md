<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1">
<title>QED Commitment Ledger</title>
<style>
@import url('https://fonts.googleapis.com/css2?family=Spectral:wght@500;600&family=IBM+Plex+Mono:wght@400;500;600&family=Inter:wght@400;500&display=swap');
:root{
  --ink:#12161c; --panel:#181d25; --panel2:#1e2530; --line:#2b3341; --line-soft:#232a35;
  --text:#e7e4da; --text-dim:#8d94a1; --text-faint:#5c6472;
  --gold:#c9a24a; --gold-dim:#8c7538;
  --met:#4f9d69; --met-bg:#16261c;
  --pending:#d4a53f; --pending-bg:#2a2214;
  --overdue:#c0553f; --overdue-bg:#2a1a15;
}
*{box-sizing:border-box;}
html,body{margin:0;padding:0;background:var(--ink);}
.qed{font-family:'Inter',sans-serif;color:var(--text);background:var(--ink);padding:0;}
.slab{font-family:'Spectral',serif;}
.mono{font-family:'IBM Plex Mono',monospace;}

/* --- mobile layout --- */
.row-inner{display:flex;justify-content:space-between;align-items:flex-start;gap:10px;}
.row-actions{display:flex;flex-direction:column;align-items:flex-end;gap:6px;flex-shrink:0;}
.add-grid{display:grid;grid-template-columns:1.4fr 0.7fr 0.7fr;gap:8px;margin-bottom:8px;}
input, select, textarea, button{font-size:16px;} /* prevents iOS auto-zoom on focus */

@media (max-width:560px){
  .header-wrap{padding:16px 14px 14px !important;}
  .header-title{font-size:19px !important;}
  .rows-wrap{padding:6px 14px !important;}
  .add-wrap{padding:12px 14px 20px !important;}
  .add-grid{grid-template-columns:1fr !important;}
  .row-card{padding:12px !important;}
  .row-inner{flex-direction:column;}
  .row-actions{flex-direction:row;align-items:center;width:100%;justify-content:space-between;margin-top:8px;padding-top:8px;border-top:1px solid var(--line-soft);}
  .status-sel{flex:1;padding:8px 10px !important;font-size:13px;}
  .del-btn{padding:8px 14px !important;}
}
</style>
</head>
<body>
<div class="qed" style="max-width:100%;">

  <div class="header-wrap" style="border-bottom:1px solid var(--line);padding:20px 22px 16px;display:flex;justify-content:space-between;align-items:flex-end;flex-wrap:wrap;gap:10px;">
    <div>
      <div class="mono" style="font-size:11px;letter-spacing:.12em;color:var(--gold-dim);text-transform:uppercase;">AIM: QED</div>
      <div class="slab header-title" style="font-size:22px;font-weight:600;margin-top:2px;">Commitment ledger</div>
      <div style="font-size:13px;color:var(--text-dim);margin-top:2px;">Quadrise plc — dated promises to check against future RNS</div>
    </div>
    <div style="text-align:right;">
      <div class="mono" style="font-size:11px;color:var(--text-faint);">entries</div>
      <div class="slab" id="count" style="font-size:20px;font-weight:600;">0</div>
    </div>
  </div>

  <div id="rows" class="rows-wrap" style="padding:8px 22px;"></div>

  <div class="add-wrap" style="padding:14px 22px 22px;border-top:1px solid var(--line-soft);margin-top:8px;">
    <div class="mono" style="font-size:11px;letter-spacing:.08em;color:var(--text-faint);text-transform:uppercase;margin-bottom:10px;">Add entry</div>
    <div class="add-grid">
      <input id="in-title" placeholder="e.g. OCP OEM approval received" style="background:var(--panel);border:1px solid var(--line);color:var(--text);border-radius:6px;padding:10px;">
      <input id="in-date" type="date" style="background:var(--panel);border:1px solid var(--line);color:var(--text);border-radius:6px;padding:10px;">
      <select id="in-status" style="background:var(--panel);border:1px solid var(--line);color:var(--text);border-radius:6px;padding:10px;">
        <option value="pending">Pending</option>
        <option value="met">Met</option>
        <option value="overdue">Overdue / slipped</option>
      </select>
    </div>
    <textarea id="in-note" placeholder="Notes — quote or paraphrase the RNS wording" style="width:100%;background:var(--panel);border:1px solid var(--line);color:var(--text);border-radius:6px;padding:10px;min-height:60px;resize:vertical;margin-bottom:10px;"></textarea>
    <button id="add-btn" style="background:var(--gold);color:#221a05;border:none;border-radius:6px;padding:12px 18px;font-weight:600;cursor:pointer;width:100%;max-width:220px;">Add to ledger</button>
    <div><span id="save-status" class="mono" style="font-size:11px;color:var(--text-faint);">&nbsp;</span></div>
  </div>
</div>
</body>
</html>

<script>
(function(){
  const KEY = 'qed-ledger-entries';
  const rowsEl = document.getElementById('rows');
  const countEl = document.getElementById('count');
  const saveStatus = document.getElementById('save-status');

  const DEFAULTS_VERSION = '2026-11-08b';
  const VERSION_KEY = 'qed-ledger-version';
  const REMOVED_KEY = 'qed-ledger-removed';
  const RETIRED = ['d2'];

  const defaults = [
    {id:'d1', title:'Morocco: OEM approval for OCP trial at Jorf Lasfar (revised kiln/line)', date:'2026-10-26', status:'met', note:'Received per 26 Oct 2026 results RNS; trial equipment is on site. Context: relocation was disclosed in Q1 2025, OEM approval was awaited from mid-2025 (Q3 2025 expectation) and only arrived by Oct 2026.'},
    {id:'d18', title:'Morocco: OCP trial agreement signed (30-day paid trial)', date:'2026-06-30', status:'overdue', note:'Mar 2026: revised plan "expected to be signed shortly" / Q2 2026. 26 Oct 2026 RNS: "working to conclude the trial agreement"; going-concern note says an amendment is "pending". Trial cannot start until it is signed.'},
    {id:'d3', title:'Morocco: signed commercial MSAR supply agreement (Maamar warrants deadline)', date:'2026-12-31', status:'pending', note:'12m warrants for the Morocco representative vest only if a signed Commercial MSAR Fuel Supply Agreement is announced by 31 Dec 2026 (parties to meet week before 15 Dec to reset if missed). The 30-day trial has not started and the trial agreement is unsigned, so this looks unlikely on the RNS timetable.'},
    {id:'d4', title:'MSC / Cargill: bilateral agreements signed', date:'', status:'overdue', note:'"Near final form" in June 2025; still unsigned at 26 Oct 2026. New open items: original bunker barge operator reallocated its barge (alternative barge owners being sought) and firm glycerine prices. Board has set signed contracts as the milestone the Executive is measured against.'},
    {id:'d5', title:'MSC / Cargill: Antwerp trial on MSC Leandra V commences', date:'2027-03-31', status:'pending', note:'On-site preparation expected Q4 2026, trial to commence Q1 2027 (26 Oct 2026). Earlier targets missed: Q1 2025, Q3 2025, "early 2026". Plan: 2-3 months proof of concept then c.4,000 hours bioMSAR for engine-maker (Wartsila) LONO; CEO wants a supply agreement signed during the LONO phase.'},
    {id:'d6', title:'Cash runway funded through', date:'2027-09-30', status:'pending', note:'Cash £2.1m at 30 Jun 2026 plus c.£1.4m net raised July 2026 (raise was £1.6m gross vs up to £2.4m sought). Operating outflow c.£0.31m/month. Runway is "assuming delivery in line with the Board’s expectations". No binding commitments for further funding; auditors flag a material uncertainty on going concern. Earlier guidance: 30 Jun 2027.'},
    {id:'d7', title:'Sustainable positive cashflow forecast to commence', date:'2028-08-31', status:'pending', note:'Now "August 2028" (26 Oct 2026). Gap between runway end (Sep 2027) and this date is about 11 months of burn, i.e. roughly £3m+ of further funding at the current rate (my arithmetic, not a company figure).'},
    {id:'d8', title:'Panama: supply partner secured for Sparkle Power', date:'', status:'pending', note:'Trial completed July 2025. 26 Oct 2026: fuel permitting progressed, "securing a suitable supply partner the immediate priority". No fuel supply agreement. Over 13 months since the trial.'},
    {id:'d9', title:'Valkor: US$0.3m licence fee instalment (now "early November 2026")', date:'', status:'overdue', note:'Originally due 31 Mar 2026; deferred. Agreed 14 Oct 2026 to be paid 31 Oct, and repeated in the 15 Oct and 26 Oct RNSs. RNS 2 Nov 2026: Valkor "now expects to pay" the US$300k, "that was due by 31 October 2026", in early November 2026. Missed within days of the results RNS. Watch for a receipt announcement.'},
    {id:'d10', title:'Valkor: US$0.65m balance of licence fee', date:'', status:'overdue', note:'Originally due 30 Jun 2026 (then "by 30 Sep 2026" per the Jun 2026 RNS, then "by end Oct" per Sep 2026). Now payable "upon closure of Valkor’s ongoing fundraising" with no date; 2 Nov 2026 RNS: "upon completion of Valkor’s ongoing fundraising process in Q4 2026".'},
    {id:'d11', title:'Valkor: 600bpd MMU delivered to Utah (US$0.2m payment)', date:'2026-12-31', status:'pending', note:'Originally by 30 Jun 2026, then Q3 2026. Now scheduled "following receipt of the outstanding fee"; CEO report says Q4 2026 once the fee is received. 2 Nov 2026 RNS: shipment and mobilisation "planned following receipt of the outstanding licence fee balance" (unclear whether that means the US$0.3m or the full US$0.95m). A further US$0.3m falls due on a later full-size 6,000bpd unit.'},
    {id:'d12', title:'Valkor: US$75k quarterly support fee payments begin', date:'2026-07-01', status:'pending', note:'Under the Sep 2025 addendum, payable quarterly from 1 Jul 2026 for at least two years. The 26 Oct 2026 RNS does not mention these; check whether any have been received.'},
    {id:'d13', title:'Valkor: 500bpd oil-sands pilot plant operational', date:'2026-12-31', status:'overdue', note:'Q2 2026 (Nov 2025) -> Q4 2026 (Mar 2026) -> Q4 2026 commissioning in the Oct 2026 CEO report, but the 16 Sep 2026 RNS said production "expected to commence in Q1 2027". Watch which date is used next.'},
    {id:'d14', title:'Valkor receipts assumed in cash forecasts (about US$1.5m)', date:'2026-12-31', status:'pending', note:'FY26 accounts recognise a £245k (US$325k) impairment provision on the Valkor receivable and describe the deferral. FY25 report had already flagged the risk that these sums are "not received, reduced in amount, or significantly delayed".'},
    {id:'d15', title:'TRACKED SLIP: positive cashflow date', date:'2028-08-31', status:'overdue', note:'Q1 2027 (FY25 report, Sep 2025) -> FY2028-29 implied (Jul 2026 placing doc) -> August 2028 (Oct 2026). Further movement would be the third change in about 13 months.'},
    {id:'d16', title:'MSC: commercial-scale bioMSAR trial aboard Leandra V', date:'2026-03-31', status:'overdue', note:'"Planned for early 2026" (FY25 report, Sep 2025). CEO report 26 Oct 2026: "We did not start the MSC and Cargill trial". Now targeted Q1 2027 (see separate entry).'},
    {id:'d17', title:'Refinery: at least one commercial production agreement', date:'2027-10-26', status:'pending', note:'Target is "in the year ahead" (26 Oct 2026). Active discussions with four refineries near bunkering hubs; no agreement. Supported by Quadrise modelling of US$15-25m potential annual margin improvement for a mid-sized refinery, which is a company estimate, not a licence fee. Earlier SE Asia refinery trial was anticipated Q1 2025 (Nov 2024 update); I have not seen an agreement.'},
    {id:'d19', title:'Valkor: representative oil sample received and MSAR/bioMSAR formulated', date:'2026-10-15', status:'met', note:'Reported 15 Oct 2026 and repeated in the 26 Oct RNS. Promised since Apr 2024 (marketing "before end Q2 2024 once samples tested"); Aug 2024 sample "being sent" for Q3 2024; Jun 2025 earlier samples "unrepresentative"; Q1 2026 -> Q2 2026 -> received Oct 2026. Only a formulation result so far, not commercial-grade test data.'},
    {id:'d20', title:'Valkor: further formulation/optimisation work completed', date:'2026-12-31', status:'pending', note:'"Further optimisation work planned during Q4 2026". US marketing and MMU deployment were conditioned on testing results.'},
    {id:'d21', title:'Strategic Review: progress update to shareholders', date:'2027-02-15', status:'pending', note:'"No later than mid-February 2027". It is an update on progress, not a completion date, and "there can be no assurance that any transaction will result". Options: master licence, private JV, non-controlling strategic investor, acquisition of related private business(es). Company says it is not seeking offers and no offer period has begun.'},
    {id:'d22', title:'AGM: 27 Nov 2026, 12 noon (Salisbury House, 114 London Wall)', date:'2026-11-27', status:'pending', note:'Submit questions via Investor Meet Company by 09:00 GMT on the day or during the live presentation. Proxy forms are separate.'},
    {id:'d23', title:'FY27 corporate scorecard (30m performance options at 3.5p)', date:'2027-06-30', status:'pending', note:'Granted 21 Sep 2026 incl. 8m to the CEO and 7m to the CTO; vest on FY27 scorecard (25% or 50% per anniversary). The 15.6m options issued Oct 2025 lapsed in full as targets were missed. Ask for the scorecard contents.'}
  ];

  function clone(o){ return JSON.parse(JSON.stringify(o)); }

  function readJSON(key, fallback){
    try{
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : fallback;
    }catch(e){ return fallback; }
  }

  let entries = [];

  async function init(){
    let saved = readJSON(KEY, null);
    const removed = readJSON(REMOVED_KEY, []);
    let savedVersion = null;
    try{ savedVersion = localStorage.getItem(VERSION_KEY); }catch(e){}

    if(!Array.isArray(saved)){
      entries = clone(defaults).filter(function(d){ return removed.indexOf(d.id) === -1; });
    } else if(savedVersion === DEFAULTS_VERSION){
      entries = saved;
    } else {
      // New built-in entries/wording: merge by id, keep the user's own entries and any status they set themselves
      const defById = {};
      defaults.forEach(function(d){ defById[d.id] = d; });
      const seen = {};
      entries = [];
      saved.forEach(function(e){
        if(RETIRED.indexOf(e.id) !== -1) return;
        if(defById[e.id]){
          const d = clone(defById[e.id]);
          if(e.userSet){ d.status = e.status; d.userSet = true; }
          entries.push(d);
          seen[e.id] = true;
        } else {
          entries.push(e);
        }
      });
      defaults.forEach(function(d){
        if(!seen[d.id] && removed.indexOf(d.id) === -1) entries.push(clone(d));
      });
    }
    try{
      localStorage.setItem(KEY, JSON.stringify(entries));
      localStorage.setItem(VERSION_KEY, DEFAULTS_VERSION);
    }catch(e){}
    render();
  }

  async function persist(){
    saveStatus.textContent = 'saving…';
    try{
      localStorage.setItem(KEY, JSON.stringify(entries));
      saveStatus.textContent = 'saved';
      setTimeout(()=>{ saveStatus.textContent=''; }, 1200);
    }catch(e){
      saveStatus.textContent = 'save failed';
    }
  }

  function statusMeta(s){
    if(s==='met') return {label:'Met', color:'var(--met)', bg:'var(--met-bg)'};
    if(s==='overdue') return {label:'Overdue', color:'var(--overdue)', bg:'var(--overdue-bg)'};
    return {label:'Pending', color:'var(--pending)', bg:'var(--pending-bg)'};
  }

  function fmtDate(d){
    if(!d) return 'no fixed date';
    const dt = new Date(d+'T00:00:00');
    if(isNaN(dt)) return d;
    return dt.toLocaleDateString('en-GB',{day:'numeric',month:'short',year:'numeric'});
  }

  function render(){
    rowsEl.innerHTML = '';
    countEl.textContent = entries.length;
    entries.forEach((e,idx)=>{
      const meta = statusMeta(e.status);
      const row = document.createElement('div');
      row.className = 'row-card';
      row.style.cssText = 'border:1px solid var(--line-soft);border-radius:8px;padding:12px 14px;margin-bottom:8px;background:var(--panel);';
      row.innerHTML = `
        <div class="row-inner">
          <div style="flex:1;min-width:0;">
            <div style="font-size:14px;font-weight:500;line-height:1.4;">${escapeHtml(e.title)}</div>
            <div class="mono" style="font-size:11px;color:var(--text-faint);margin-top:4px;">target: ${fmtDate(e.date)}</div>
            ${e.note ? `<div style="font-size:12.5px;color:var(--text-dim);margin-top:6px;line-height:1.5;">${escapeHtml(e.note)}</div>` : ''}
          </div>
          <div class="row-actions">
            <select data-idx="${idx}" class="status-sel mono" style="background:${meta.bg};color:${meta.color};border:1px solid ${meta.color};border-radius:5px;padding:4px 8px;font-size:11px;font-weight:600;cursor:pointer;">
              <option value="pending" ${e.status==='pending'?'selected':''}>Pending</option>
              <option value="met" ${e.status==='met'?'selected':''}>Met</option>
              <option value="overdue" ${e.status==='overdue'?'selected':''}>Overdue</option>
            </select>
            <button data-idx="${idx}" class="del-btn mono" style="background:none;border:1px solid var(--line);color:var(--text-faint);border-radius:5px;padding:3px 8px;font-size:11px;cursor:pointer;">remove</button>
          </div>
        </div>`;
      rowsEl.appendChild(row);
    });

    rowsEl.querySelectorAll('.status-sel').forEach(sel=>{
      sel.addEventListener('change', (ev)=>{
        const i = parseInt(ev.target.getAttribute('data-idx'));
        entries[i].status = ev.target.value;
        entries[i].userSet = true;
        persist();
        render();
      });
    });
    rowsEl.querySelectorAll('.del-btn').forEach(btn=>{
      btn.addEventListener('click', (ev)=>{
        const i = parseInt(ev.target.getAttribute('data-idx'));
        const gone = entries.splice(i,1)[0];
        if(gone && /^d\d+$/.test(gone.id)){
          const removed = readJSON(REMOVED_KEY, []);
          if(removed.indexOf(gone.id) === -1) removed.push(gone.id);
          try{ localStorage.setItem(REMOVED_KEY, JSON.stringify(removed)); }catch(e){}
        }
        persist();
        render();
      });
    });
  }

  function escapeHtml(s){
    const d = document.createElement('div');
    d.textContent = s;
    return d.innerHTML;
  }

  document.getElementById('add-btn').addEventListener('click', ()=>{
    const title = document.getElementById('in-title').value.trim();
    const date = document.getElementById('in-date').value;
    const status = document.getElementById('in-status').value;
    const note = document.getElementById('in-note').value.trim();
    if(!title){
      document.getElementById('in-title').style.borderColor = 'var(--overdue)';
      return;
    }
    entries.unshift({id:'e'+Date.now(), title, date, status, note});
    document.getElementById('in-title').value = '';
    document.getElementById('in-date').value = '';
    document.getElementById('in-note').value = '';
    document.getElementById('in-title').style.borderColor = 'var(--line)';
    persist();
    render();
  });

  init();
})();
</script>
