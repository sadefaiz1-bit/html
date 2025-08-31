<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Rekap Pemasukan & Pengeluaran — Toko</title>
  <style>
    :root{--bg:#f7f9fc;--card:#ffffff;--muted:#6b7280;--accent:#0ea5a4}
    *{box-sizing:border-box;font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto,'Helvetica Neue',Arial}
    body{margin:0;background:var(--bg);padding:28px}
    .container{max-width:980px;margin:0 auto}
    header{display:flex;align-items:center;justify-content:space-between;margin-bottom:18px}
    h1{font-size:20px;margin:0}
    small{color:var(--muted)}
    .card{background:var(--card);border-radius:12px;padding:16px;box-shadow:0 6px 18px rgba(16,24,40,0.06);}
    form{display:grid;grid-template-columns:repeat(12,1fr);gap:10px;align-items:end}
    label{display:block;font-size:12px;color:var(--muted);margin-bottom:6px}
    input,select,textarea{width:100%;padding:8px;border:1px solid #e6edf3;border-radius:8px}
    .col-3{grid-column:span 3}
    .col-4{grid-column:span 4}
    .col-2{grid-column:span 2}
    .col-12{grid-column:span 12}
    .btn{display:inline-block;padding:10px 14px;border-radius:10px;border:0;background:var(--accent);color:white;cursor:pointer}
    .btn.ghost{background:transparent;border:1px solid #e6edf3;color:var(--muted)}
    .summary{display:flex;gap:10px;margin-top:12px}
    .tile{flex:1;padding:12px;border-radius:10px;background:linear-gradient(180deg,#fff,#fbfeff);text-align:center}
    .tile h3{margin:0;font-size:18px}
    .tile p{margin:6px 0 0;color:var(--muted)}
    table{width:100%;border-collapse:collapse;margin-top:12px}
    th,td{padding:10px;border-bottom:1px solid #f0f3f7;text-align:left}
    th{font-size:13px;color:var(--muted)}
    tfoot td{font-weight:700}
    .right{text-align:right}
    .controls{display:flex;gap:8px;flex-wrap:wrap}
    .filter-row{display:flex;gap:8px;margin-top:12px}
    @media(max-width:800px){form{grid-template-columns:repeat(6,1fr)}.col-3{grid-column:span 6}.col-4{grid-column:span 6}.col-2{grid-column:span 3}}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div>
        <h1>Rekap Pemasukan & Pengeluaran</h1>
        <h2>FAUZAN STORE</h2>
        <small>Catat transaksi toko, lihat ringkasan, ekspor CSV. Data disimpan di browser.</small>
      </div>
      <div>
        <button id="exportBtn" class="btn">Ekspor CSV</button>
      </div>
    </header>

    <section class="card">
      <form id="txForm">
        <div class="col-3">
          <label for="date">Tanggal</label>
          <input type="date" id="date" required>
        </div>
        <div class="col-4">
          <label for="desc">Keterangan</label>
          <input id="desc" placeholder="Contoh: Penjualan kasir A / Beli stok" required>
        </div>
        <div class="col-3">
          <label for="type">Jenis</label>
          <select id="type">
            <option value="income">Pemasukan</option>
            <option value="expense">Pengeluaran</option>
          </select>
        </div>
        <div class="col-2">
          <label for="amount">Nominal (Rp)</label>
          <input id="amount" type="number" min="0" step="1" required>
        </div>

        <div class="col-12" style="display:flex;gap:8px;margin-top:6px">
          <button type="submit" class="btn">Tambah / Simpan</button>
          <button type="button" id="clearForm" class="btn ghost">Bersihkan</button>
          <div style="margin-left:auto" class="controls">
            <select id="quickRange">
              <option value="all">Semua</option>
              <option value="today">Hari ini</option>
              <option value="this_month">Bulan ini</option>
            </select>
            <input type="month" id="monthFilter">
            <input id="search" placeholder="Cari keterangan...">
          </div>
        </div>
      </form>

      <div class="summary">
        <div class="tile">
          <h3 id="totalIncome">Rp 0</h3>
          <p>Pemasukan</p>
        </div>
        <div class="tile">
          <h3 id="totalExpense">Rp 0</h3>
          <p>Pengeluaran</p>
        </div>
        <div class="tile">
          <h3 id="balance">Rp 0</h3>
          <p>Saldo Bersih</p>
        </div>
      </div>

      <div style="margin-top:12px;overflow:auto">
        <table id="txTable">
          <thead>
            <tr>
              <th>Tgl</th><th>Keterangan</th><th>Jenis</th><th class="right">Nominal (Rp)</th><th>Aksi</th>
            </tr>
          </thead>
          <tbody></tbody>
          <tfoot>
            <tr><td colspan="3">Total</td><td class="right" id="tfootTotal">Rp 0</td><td></td></tr>
          </tfoot>
        </table>
      </div>
    </section>

    <p style="margin-top:12px;color:var(--muted);font-size:13px">Tip: halaman ini menyimpan data di browser (localStorage). Untuk backup gunakan tombol Ekspor CSV.</p>
  </div>

<script>
// Simple Rekap Transaksi — single-file, no backend.
// Data model: {id, date, desc, type: 'income'|'expense', amount}

const STORAGE_KEY = 'rekap_toko_v1';
let transactions = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');

// helpers
const fmt = (n) => {
  const x = Number(n) || 0;
  return 'Rp ' + x.toLocaleString('id-ID');
};
const byId = (id) => document.getElementById(id);

// UI elements
const txForm = byId('txForm');
const tbody = document.querySelector('#txTable tbody');
const totalIncomeEl = byId('totalIncome');
const totalExpenseEl = byId('totalExpense');
const balanceEl = byId('balance');
const tfootTotal = byId('tfootTotal');
const exportBtn = byId('exportBtn');
const clearFormBtn = byId('clearForm');
const quickRange = byId('quickRange');
const monthFilter = byId('monthFilter');
const searchInput = byId('search');

// initialize default date to today
byId('date').value = new Date().toISOString().slice(0,10);

function save(){
  localStorage.setItem(STORAGE_KEY, JSON.stringify(transactions));
}

function addTransaction(tx){
  transactions.push(tx);
  transactions.sort((a,b)=> new Date(b.date)-new Date(a.date));
  save();
  render();
}

function deleteTx(id){
  if(!confirm('Hapus transaksi ini?')) return;
  transactions = transactions.filter(t=>t.id!==id);
  save(); render();
}

function editTx(id){
  const tx = transactions.find(t=>t.id===id);
  if(!tx) return;
  // populate form for quick edit
  byId('date').value = tx.date;
  byId('desc').value = tx.desc;
  byId('type').value = tx.type;
  byId('amount').value = tx.amount;
  // remove old and let user save (creating new id)
  deleteTx(id);
}

function calcSummary(list){
  const income = list.filter(t=>t.type==='income').reduce((s,t)=>s+Number(t.amount),0);
  const expense = list.filter(t=>t.type==='expense').reduce((s,t)=>s+Number(t.amount),0);
  return {income,expense,balance:income-expense};
}

function renderTable(list){
  tbody.innerHTML = '';
  if(list.length===0){
    tbody.innerHTML = '<tr><td colspan="5" style="text-align:center;color:#8892a6">Belum ada transaksi</td></tr>';
    tfootTotal.textContent = fmt(0);
    return;
  }
  list.forEach(tx=>{
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td>${tx.date}</td>
      <td>${escapeHtml(tx.desc)}</td>
      <td>${tx.type==='income'? 'Pemasukan':'Pengeluaran'}</td>
      <td class="right">${fmt(tx.amount)}</td>
      <td><button class="btn ghost" data-id="${tx.id}">Edit</button> <button class="btn ghost" data-del="${tx.id}">Hapus</button></td>
    `;
    tbody.appendChild(tr);
  });
  const totals = calcSummary(list);
  tfootTotal.textContent = fmt(totals.balance);
}

function escapeHtml(s){return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":"&#39;"})[c]);}

function applyFilters(){
  let list = [...transactions];
  // quickRange
  const qr = quickRange.value;
  const today = new Date();
  if(qr==='today'){
    const d = today.toISOString().slice(0,10);
    list = list.filter(t=>t.date===d);
  } else if(qr==='this_month'){
    const mm = today.toISOString().slice(0,7);
    list = list.filter(t=>t.date.slice(0,7)===mm);
  }
  // monthFilter
  if(monthFilter.value){
    list = list.filter(t=>t.date.slice(0,7)===monthFilter.value);
  }
  // search
  const q = searchInput.value.trim().toLowerCase();
  if(q) list = list.filter(t=>t.desc.toLowerCase().includes(q));

  renderTable(list);
  const sums = calcSummary(list);
  totalIncomeEl.textContent = fmt(sums.income);
  totalExpenseEl.textContent = fmt(sums.expense);
  balanceEl.textContent = fmt(sums.balance);
}

// events
txForm.addEventListener('submit', e=>{
  e.preventDefault();
  const d = byId('date').value;
  const desc = byId('desc').value.trim();
  const type = byId('type').value;
  const amount = Number(byId('amount').value) || 0;
  if(!d || !desc || amount<=0){ alert('Isi semua field dengan benar (nominal > 0)'); return; }
  const tx = {id: 'tx_' + Date.now(), date: d, desc, type, amount};
  addTransaction(tx);
  txForm.reset();
  byId('date').value = new Date().toISOString().slice(0,10);
});

clearFormBtn.addEventListener('click', ()=>{txForm.reset(); byId('date').value=new Date().toISOString().slice(0,10)});

tbody.addEventListener('click', e=>{
  const id = e.target.getAttribute('data-id');
  const del = e.target.getAttribute('data-del');
  if(id) editTx(id);
  if(del) deleteTx(del);
});

[quickRange, monthFilter, searchInput].forEach(el=>el.addEventListener('input', applyFilters));

exportBtn.addEventListener('click', ()=>{
  if(transactions.length===0){ alert('Tidak ada data untuk diekspor'); return; }
  const rows = [['Tanggal','Keterangan','Jenis','Nominal']];
  transactions.forEach(t=> rows.push([t.date,t.desc,t.type,t.amount]));
  const csv = rows.map(r => r.map(c => '"'+String(c).replace(/"/g,'""')+'"').join(',')).join('\n');
  const blob = new Blob([csv], {type:'text/csv'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'rekap_transaksi.csv'; document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url);
});

// render on load
function render(){ applyFilters(); }

render();

</script>
</body>
</html>
