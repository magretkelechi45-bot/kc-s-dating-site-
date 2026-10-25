# kc-s-dating-site-
A fun, interactive date invitation website where visitors can accept or decline a date, choose from available dates, locations, and times, and receive booking confirmations. Features include a waiting list, secure payment integration through Paystack, and a private admin dashboard.
Flask>=3.0,<4
# Date Invitation Website 💌

## Run locally
1. Install Python 3.10+.
2. `pip install -r requirements.txt`
3. Set environment variables `ADMIN_PASSWORD` (a strong password) and `SESSION_SECRET` (random long string).
4. Optional: set `PAYSTACK_SECRET_KEY` to your Paystack **test** secret key for testing.
5. Run `python app.py` and visit `http://localhost:5000`.

## Deploy
Deploy this Flask app on a Python-capable host with HTTPS, persistent storage for `date_site.db`, and the environment variables above. A platform that deletes local files on restart will lose bookings; use a persistent disk or migrate to managed PostgreSQL for production. Run behind a production WSGI server such as gunicorn (`gunicorn app:app`) rather than Flask's development server.

## Important
- Dates are preloaded as **examples**; edit them through Admin before sharing.
- Payment is **voluntary**. A visitor may enter an optional account number, but it is not transmitted or used for payment. The site never collects bank PINs. Paystack securely handles checkout authentication.
- Paystack payment requires your own secret key configured on the server. Never paste it into the HTML.
- Payment verification checks the reference, currency, amount, and Paystack success status.
- Dashboard polls every eight seconds; no push/email notifications are configured.
- Dashboard counts YES/NO clicks in the last 100 events; these are not unique people.
- Bookings use a database-backed global queue, not separate queues per date.
- Visitor identity is self-entered; anonymous clicks cannot identify the visitor.
- Production hardening needed before public launch: CSRF protection, rate limits, stronger admin auth, privacy notice, and database backups.

## Payment flow clarification
Visitors provide an email to initiate optional Paystack checkout. This website does not request or store their bank account numbers, bank PINs, or OTPs. Paystack handles payment credentials on its hosted page.

import os, sqlite3, secrets, json, urllib.request, urllib.error
from datetime import datetime, timezone
from functools import wraps
from flask import Flask, request, jsonify, session, send_from_directory
from werkzeug.security import generate_password_hash, check_password_hash

app=Flask(__name__,static_folder='static')
app.secret_key=os.getenv('SESSION_SECRET',secrets.token_hex(32))
DB=os.getenv('DATABASE_PATH','date_site.db')
PAYSTACK_SECRET=os.getenv('PAYSTACK_SECRET_KEY','')
ADMIN_PASSWORD=os.getenv('ADMIN_PASSWORD','')
def conn():
    db=sqlite3.connect(DB,timeout=15)
    db.row_factory=sqlite3.Row
    db.execute('PRAGMA journal_mode=WAL')
    return db
def init():
    with conn() as db:
        db.executescript("""CREATE TABLE IF NOT EXISTS slots(id INTEGER PRIMARY KEY,day TEXT NOT NULL,place TEXT NOT NULL,time TEXT NOT NULL,active INTEGER DEFAULT 1);
        CREATE TABLE IF NOT EXISTS bookings(id INTEGER PRIMARY KEY,ref TEXT UNIQUE NOT NULL,name TEXT NOT NULL,slot_id INTEGER NOT NULL,created TEXT NOT NULL,status TEXT DEFAULT 'confirmed');
        CREATE TABLE IF NOT EXISTS events(id INTEGER PRIMARY KEY,type TEXT NOT NULL,detail TEXT NOT NULL,created TEXT NOT NULL);
        CREATE TABLE IF NOT EXISTS payments(ref TEXT PRIMARY KEY,email TEXT,amount INTEGER,status TEXT DEFAULT 'pending',created TEXT NOT NULL);""")
        if not db.execute('SELECT 1 FROM slots').fetchone():
            db.executemany('INSERT INTO slots(day,place,time) VALUES(?,?,?)',[
                ('2026-11-07','Location to be announced','5:00 PM'),
                ('2026-11-14','Location to be announced','4:00 PM'),
                ('2026-11-21','Location to be announced','6:00 PM')])
                <!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>One Little Question 💌</title><style>
:root{font-family:system-ui,Arial,sans-serif;color:#271921;background:#fff5f8}*{box-sizing:border-box}body{margin:0;min-height:100vh;background:radial-gradient(circle at 10% 0%,#ffc9db,transparent 35%),linear-gradient(150deg,#fff6f8,#ffe6ef)}main{max-width:720px;margin:5vh auto;padding:24px}.panel{background:#fffdfef2;border:1px solid #ffd1e0;border-radius:28px;box-shadow:0 20px 60px #a8214920;padding:clamp(22px,5vw,44px)}h1{font-size:clamp(2rem,7vw,3.4rem);line-height:1.1;margin:12px 0}h2{margin-top:0}p{color:#6e5360;line-height:1.6}.tag{color:#d82c67;letter-spacing:2px;font-size:.8rem;font-weight:800;text-transform:uppercase}.buttons{display:flex;flex-wrap:wrap;gap:12px;margin-top:24px}button{background:#e93e79;border:0;border-radius:14px;padding:14px 22px;color:white;font-weight:750;font-size:1rem;cursor:pointer}button.alt{background:#281c25}button.ghost{background:#ffe4ed;color:#a82050}button:disabled{opacity:.5;cursor:default}input{width:100%;padding:14px;border:1px solid #e9bac9;border-radius:12px;margin:8px 0 14px;font-size:1rem}label{font-weight:700}.slot{display:block;width:100%;text-align:left;background:white;color:#38202b;border:2px solid #f2c4d3;margin:10px 0;padding:17px}.slot.selected{border-color:#e93e79;background:#fff0f5}.muted{font-size:.88rem;color:#826976}.hide{display:none!important}.receipt{border:2px dashed #e93e79;background:#fff0f5;border-radius:18px;padding:18px;margin:20px 0}.big{font-size:2rem;font-weight:900;color:#d72f67}.top{display:flex;justify-content:space-between;align-items:center;margin-bottom:16px}.top button{font-size:.8rem;padding:9px 12px}.item{padding:12px;border-bottom:1px solid #f5d6e1;overflow-wrap:anywhere}.admin-section{margin-top:24px;padding-top:20px;border-top:1px solid #f3cad7}.status{margin-top:14px;color:#b41d51;font-weight:650}.heart{font-size:3rem;animation:float 2.4s ease-in-out infinite}@keyframes float{50%{transform:translateY(-9px)}}small{color:#755d68}a{color:#c62560}
</style></head><body><main><div class="top"><strong>💌 one little question</strong><button class="ghost" onclick="showAdmin()">Admin 🔒</button></div>
<section class="panel" id="welcome"><div class="heart">💗</div><div class="tag">A very important invitation</div><h1>Will you go on a date with me?</h1><p>Choose carefully. One answer comes with butterflies. The other comes with a very dramatic invoice. 👀</p><div class="buttons"><button onclick="choose('yes')">YES, of course 💕</button><button class="alt" onclick="choose('no')">NO 🙃</button></div></section>
<section class="panel hide" id="select"><div class="tag">Your next adventure</div><h2>Pick our date ✨</h2><p>Choose from these available plans.</p><div id="slots"></div><label>Your first name<input id="visitorName" maxlength="80" placeholder="What should I call you?"></label><div class="buttons"><button onclick="book()">Confirm my date 💘</button><button class="ghost" onclick="go('welcome')">Back</button></div><div id="bookErr" class="status"></div></section>
<section class="panel hide" id="confirmed"><div class="heart">💖</div><h1>It's a date!</h1><div id="ticket" class="receipt"></div><p id="queue"></p><small>Keep your booking reference to check your place later. Queue positions are based on real confirmed bookings.</small></section>
<section class="panel hide" id="rejected"><div class="heart">💔</div><div class="tag">Heartbreak department</div><h1>Rejection received.</h1><p>You've unlocked the most dramatic invoice in history. This is a joke, and paying is completely voluntary. You can leave without paying.</p><div class="receipt"><div>Optional heartbreak fee</div><div class="big">₦5,000</div></div><p class="muted">You do not need to enter a bank account number or PIN on this website. Paystack will handle payment details and any required authentication securely.</p><label>Email for optional Paystack checkout<input id="payerEmail" type="email" placeholder="you@example.com"></label><button onclick="pay()">Pay ₦5,000 securely</button><p class="muted">Payment is optional. Payment status is shown only after Paystack verification.</p><div id="payMsg" class="status"></div></section>
<section class="panel hide" id="adminLogin"><h2>Private dashboard 🔒</h2><label>Admin password<input id="adminPassword" type="password"></label><div class="buttons"><button onclick="login()">Sign in</button><button class="ghost" onclick="go('welcome')">Close</button></div><div id="loginErr" class="status"></div></section>
<section class="panel hide" id="admin"><div class="top"><h2>Dashboard 💌</h2><button class="ghost" onclick="logout()">Log out</button></div><p class="muted">Refreshes every 8 seconds while open. Never share your password.</p><div id="overview"></div><div class="admin-section"><h3>Add available date</h3><label>Date<input type="date" id="newDay"></label><label>Location<input id="newPlace" placeholder="e.g. Café in Abuja"></label><label>Time<input id="newTime" placeholder="e.g. 5:00 PM"></label><button onclick="addSlot()">Add date</button><div id="adminMsg" class="status"></div></div><div class="admin-section"><h3>Manage dates</h3><div id="manageSlots"></div></div><div class="admin-section"><h3>Bookings and queue</h3><div id="manageBookings"></div></div><div class="admin-section"><h3>Payments</h3><div id="managePayments"></div></div><div class="admin-section"><h3>Recent notifications</h3><div id="events"></div></div></section>
</main><script>
let selected=null,interval=null,lockedNo=false;const ids=['welcome','select','confirmed','rejected','adminLogin','admin'];function go(id){if(lockedNo&&id==='welcome')return;ids.forEach(x=>document.getElementById(x).classList.toggle('hide',x!==id));if(id!=='admin'&&interval){clearInterval(interval);interval=null}window.scrollTo(0,0)}function msg(id,s){document.getElementById(id).textContent=s}async function api(path,opts={}){let r=await fetch(path,{headers:{'Content-Type':'application/json'},...opts});let d=await r.json();if(!r.ok)throw Error(d.error||'Something went wrong');return d}const post=(path,d)=>api(path,{method:'POST',body:JSON.stringify(d)});function esc(s){return String(s??'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]))}
async function choose(v){try{await post('/api/choice',{choice:v})}catch(e){alert(e.message);return}if(v==='no'){lockedNo=true;go('rejected')}else{go('select');loadSlots()}}
async function loadSlots(){try{let s=await api('/api/slots');document.getElementById('slots').innerHTML=s.length?s.map(x=>`<button class="slot ${selected===x.id?'selected':''}" onclick="selected=${x.id};loadSlots()">📅 ${esc(x.day)}<br>📍 ${esc(x.place)}<br>🕒 ${esc(x.time)}</button>`).join(''):'No dates available yet.'}catch(e){msg('bookErr',e.message)}}
async function book(){if(!selected)return msg('bookErr','Choose a date first');try{let d=await post('/api/book',{slot_id:selected,name:document.getElementById('visitorName').value});sessionStorage.setItem('bookingRef',d.ref);go('confirmed');showTicket(d.ref)}catch(e){msg('bookErr',e.message)}}
async function showTicket(ref){try{let d=await api('/api/booking/'+encodeURIComponent(ref));document.getElementById('ticket').innerHTML=`<strong>${esc(d.name)}</strong><p>📅 ${esc(d.day)}<br>📍 ${esc(d.place)}<br>🕒 ${esc(d.time)}</p><strong>Reference: ${esc(d.ref)}</strong>`;document.getElementById('queue').textContent=`You're number ${d.position} in line. ${d.ahead} people ahead of you!`}catch(e){msg('queue',e.message)}}
async function pay(){try{let d=await post('/api/paystack/start',{email:document.getElementById('payerEmail').value});window.location.href=d.url}catch(e){msg('payMsg',e.message)}}
async function showAdmin(){go('adminLogin')}
async function login(){try{await post('/api/admin/login',{password:document.getElementById('adminPassword').value});go('admin');refresh();interval=setInterval(refresh,8000)}catch(e){msg('loginErr',e.message)}}
async function logout(){await post('/api/admin/logout',{});go(lockedNo?'rejected':'welcome')}
async function refresh(){if(document.getElementById('admin').classList.contains('hide'))return;try{let d=await api('/api/admin/data');document.getElementById('overview').innerHTML=`<div class="receipt">💗 ${d.events.filter(x=>x.type==='yes').length} recent YES clicks · 💔 ${d.events.filter(x=>x.type==='no').length} recent NO clicks · 🎟️ ${d.bookings.filter(x=>x.status==='confirmed').length} active bookings</div>`;document.getElementById('manageSlots').innerHTML=d.slots.map(x=>`<div class="item"><b>${esc(x.day)}</b> · ${esc(x.place)} · ${esc(x.time)} · ${x.active?'Available':'Hidden'}<br><button class="ghost" onclick="editSlot(${x.id})">Edit</button> <button class="alt" onclick="toggleSlot(${x.id},${x.active?0:1})">${x.active?'Hide':'Show'}</button></div>`).join('');document.getElementById('manageBookings').innerHTML=d.bookings.map(x=>`<div class="item"><b>${esc(x.name)}</b> · ${esc(x.ref)}<br>${esc(x.day)} · ${esc(x.place)} · ${esc(x.time)}<br>Status: ${esc(x.status)} <button class="ghost" onclick="setBooking(${x.id},'${x.status==='confirmed'?'cancelled':'confirmed'}')">${x.status==='confirmed'?'Cancel':'Restore'}</button></div>`).join('')||'No bookings yet';document.getElementById('managePayments').innerHTML=d.payments.map(x=>`<div class="item">${esc(x.ref)} · ${esc(x.email)} · ${esc(x.status)} · ₦${(x.amount/100).toLocaleString()}</div>`).join('')||'No payments yet';document.getElementById('events').innerHTML=d.events.map(x=>`<div class="item"><b>${esc(x.type)}</b> · ${esc(x.detail)}<br><small>${esc(x.created)}</small></div>`).join('')||'No activity yet'}catch(e){msg('adminMsg',e.message)}}
async function addSlot(){try{await post('/api/admin/slots',{day:document.getElementById('newDay').value,place:document.getElementById('newPlace').value,time:document.getElementById('newTime').value});msg('adminMsg','Date added');refresh()}catch(e){msg('adminMsg',e.message)}}
async function toggleSlot(id,active){await api('/api/admin/slots/'+id,{method:'PATCH',body:JSON.stringify({active})});refresh()}
async function editSlot(id){let day=prompt('New date (YYYY-MM-DD), leave blank to keep');if(day===null)return;let place=prompt('New location, leave blank to keep');if(place===null)return;let time=prompt('New time, leave blank to keep');if(time===null)return;let d={};if(day)d.day=day;if(place)d.place=place;if(time)d.time= time;await api('/api/admin/slots/'+id,{method:'PATCH',body:JSON.stringify(d)});refresh()}
async function setBooking(id,status){await api('/api/admin/bookings/'+id,{method:'PATCH',body:JSON.stringify({status})});refresh()}
const qs=new URLSearchParams(location.search);if(qs.has('payment_ref')){lockedNo=true;go('rejected');api('/api/paystack/verify/'+encodeURIComponent(qs.get('payment_ref'))).then(d=>msg('payMsg',d.paid?'Payment verified successfully! 💗':'Payment not confirmed. Status: '+d.status)).catch(e=>msg('payMsg',e.message));history.replaceState({},'',location.pathname)}
</script></body></html>
init()
def now():return datetime.now(timezone.utc).isoformat()
def event(db,typ,detail):db.execute('INSERT INTO events(type,detail,created) VALUES(?,?,?)',(typ,detail,now()))
