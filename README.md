https://github.com/me-danex/Enial.git
// ===========================
// ENIAL — Application Logic
// ===========================

// ---- STATE ----
let currentFilter = 'all';
let currentWork   = null;
let currentPayMethod = '';
let currentLang   = 'fr';
let userLikes     = {};
let userSubs      = {};
let userPurchased = {};
let translatedWorks = {};
let currentChapter  = 1;

// Load persistent state from sessionStorage
try {
  const s = sessionStorage.getItem('enial_state');
  if (s) {
    const st = JSON.parse(s);
    userLikes      = st.likes      || {};
    userSubs       = st.subs       || {};
    userPurchased  = st.purchased  || {};
  }
} catch(e) {}

function saveState() {
  try {
    sessionStorage.setItem('enial_state', JSON.stringify({
      likes: userLikes, subs: userSubs, purchased: userPurchased
    }));
  } catch(e) {}
}

// ---- NAVIGATION ----
function showPage(name) {
  document.querySelectorAll('.page').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.nav-btn').forEach(b => b.classList.remove('active'));
  const page = document.getElementById('page-' + name);
  if (page) page.classList.add('active');
  const btn = document.querySelector(`.nav-btn[data-page="${name}"]`);
  if (btn) btn.classList.add('active');
  if (name === 'home')   renderGrids();
  if (name === 'browse') renderBrowse();
  window.scrollTo({ top: 0, behavior: 'smooth' });
}

function toggleMenu() {
  const m = document.getElementById('mobileMenu');
  m.classList.toggle('open');
}

// ---- LANG ----
function changeLang(lang) {
  currentLang = lang;
  applyTranslation(lang);
  renderGrids();
  if (document.getElementById('page-browse').classList.contains('active')) renderBrowse();
  showToast('🌐 ' + (document.getElementById('langSelect').options[document.getElementById('langSelect').selectedIndex].text), 'info');
}

// ---- RENDER HELPERS ----
const COVER_BG = { webtoon: '#1E1145', novel: '#0F1E45', manga: '#250F25' };

function renderCard(work) {
  const typeClass = { webtoon: 'type-webtoon', novel: 'type-novel', manga: 'type-manga' };
  const badgeCls  = work.free || userPurchased[work.id] ? 'badge-free' : 'badge-paid';
  const badgeTxt  = work.free || userPurchased[work.id] ? (currentLang === 'en' ? 'FREE' : 'GRATUIT') : '500 FCFA';
  return `<div class="work-card" onclick="openWork(${work.id})" role="button" aria-label="${work.title}">
    <div class="work-cover" style="background:${COVER_BG[work.type]||'#1E1145'}">
      <span style="font-size:44px" aria-hidden="true">${work.emoji}</span>
      <span class="work-badge ${badgeCls}">${badgeTxt}</span>
    </div>
    <div class="work-info">
      <div class="work-title">${work.title}</div>
      <div class="work-meta">
        <span class="work-type ${typeClass[work.type]}">${work.type.toUpperCase()}</span>
        <span>❤️ ${formatNum(work.likes)}</span>
      </div>
    </div>
  </div>`;
}

function formatNum(n) {
  if (n >= 1000) return (n / 1000).toFixed(1) + 'k';
  return n;
}

function renderGrids() {
  const trending = [...WORKS].sort((a, b) => b.likes - a.likes).slice(0, 6);
  const newest   = [...WORKS].sort((a, b) => b.id - a.id).slice(0, 6);
  const free     = WORKS.filter(w => w.free).slice(0, 6);
  const gh = document.getElementById('grid-home');
  const gn = document.getElementById('grid-new');
  const gf = document.getElementById('grid-free');
  if (gh) gh.innerHTML = trending.map(renderCard).join('');
  if (gn) gn.innerHTML = newest.map(renderCard).join('');
  if (gf) gf.innerHTML = free.length ? free.map(renderCard).join('') : `<div class="empty-state">Aucune œuvre gratuite pour le moment.</div>`;
}

function renderBrowse() {
  const q = (document.getElementById('searchInput')?.value || '').toLowerCase().trim();
  const filtered = WORKS.filter(w => {
    const matchType = currentFilter === 'all' || w.type === currentFilter || (currentFilter === 'free' && w.free);
    const matchQ    = !q || w.title.toLowerCase().includes(q) || w.author.toLowerCase().includes(q) || w.genre.toLowerCase().includes(q);
    return matchType && matchQ;
  });
  const el = document.getElementById('grid-browse');
  if (!el) return;
  el.innerHTML = filtered.length
    ? filtered.map(renderCard).join('')
    : `<div class="empty-state">😔 Aucune œuvre trouvée. Essayez un autre terme.</div>`;
}

function setFilter(f, btn) {
  currentFilter = f;
  document.querySelectorAll('.filter-tab').forEach(b => b.classList.remove('active'));
  if (btn) btn.classList.add('active');
  renderBrowse();
}

function filterWorks() { renderBrowse(); }

// ---- WORK DETAIL MODAL ----
function openWork(id) {
  const w = WORKS.find(x => x.id === id);
  if (!w) return;
  currentWork = w;

  const likesTotal = w.likes + (userLikes[w.id] ? 1 : 0);
  document.getElementById('modal-cover').style.cssText = `background:${COVER_BG[w.type]||'#1E1145'}; display:flex; align-items:center; justify-content:center; font-size:38px;`;
  document.getElementById('modal-cover').textContent = w.emoji;
  document.getElementById('modal-title').textContent = w.title;
  document.getElementById('modal-author').textContent = '✍️ ' + w.author + '  ·  ' + w.genre;
  document.getElementById('modal-likes').textContent    = '❤️ ' + formatNum(likesTotal);
  document.getElementById('modal-views').textContent    = '👁️ ' + formatNum(w.views);
  document.getElementById('modal-chapters').textContent = '📄 ' + w.chapters + ' ch.';
  document.getElementById('modal-desc').textContent  = w.desc;
  document.getElementById('like-count').textContent  = formatNum(likesTotal);
  document.getElementById('modal-type-badge').textContent = w.type.toUpperCase();
  document.getElementById('btn-like-modal').className  = 'btn-like'      + (userLikes[w.id] ? ' liked' : '');
  document.getElementById('btn-sub-modal').className   = 'btn-subscribe' + (userSubs[w.id]  ? ' subscribed' : '');
  document.getElementById('btn-sub-modal').textContent = userSubs[w.id]  ? '🔔 Abonné ✓' : '🔔 Suivre';

  const priceSection = document.getElementById('modal-price-section');
  if (w.free || userPurchased[w.id]) {
    priceSection.innerHTML = `<div style="background:rgba(52,211,153,0.1);border:1px solid rgba(52,211,153,0.3);border-radius:8px;padding:10px 14px;margin-bottom:14px;font-size:13px;color:#34D399;">✅ Accès complet débloqué — bonne lecture !</div>`;
    const rb = document.getElementById('btn-read-modal');
    rb.textContent = '📖 Lire';
    rb.onclick = readWork;
    rb.style.display = 'inline-block';
  } else {
    priceSection.innerHTML = `<div class="price-box"><div class="price">500 FCFA</div><div class="price-note">Accès complet · ${w.chapters} chapitres</div></div>`;
    const rb = document.getElementById('btn-read-modal');
    rb.textContent = '🔒 Acheter pour lire';
    rb.onclick = () => { closeModal('modalWork'); openPayModal(); };
  }

  // Comments
  if (!commentsDB[w.id]) commentsDB[w.id] = [];
  renderComments();
  document.getElementById('commentInput').value = '';
  document.getElementById('modalWork').classList.add('open');
  document.body.style.overflow = 'hidden';
}

function readWork() {
  if (!currentWork) return;
  currentChapter = 1;
  closeModal('modalWork');
  loadChapterView(currentWork, 1);
  showPage('reader');
}

function loadChapterView(work, ch) {
  document.getElementById('reader-work-title').textContent = work.title;
  const content = translatedWorks[work.id] ? translatedWorks[work.id] : getChapterContent(work, ch);
  document.getElementById('reader-content').textContent = content;

  // Translate banner: show if work lang != user lang and not yet translated
  const banner = document.getElementById('translate-banner-reader');
  if (work.lang !== currentLang && !translatedWorks[work.id]) {
    banner.style.display = 'flex';
  } else {
    banner.style.display = 'none';
  }

  // Update chapter selector
  const sel = document.getElementById('chapterSelect');
  sel.innerHTML = '';
  for (let i = 1; i <= Math.min(work.chapters, 5); i++) {
    const opt = document.createElement('option');
    opt.value = i;
    opt.textContent = `Chapitre ${i}`;
    if (i === ch) opt.selected = true;
    sel.appendChild(opt);
  }
  if (work.chapters > 5) {
    const opt = document.createElement('option');
    opt.disabled = true;
    opt.textContent = `… ${work.chapters} chapitres au total`;
    sel.appendChild(opt);
  }
}

function loadChapter(ch) {
  if (!currentWork) return;
  currentChapter = parseInt(ch) || 1;
  loadChapterView(currentWork, currentChapter);
}

function prevChapter() {
  if (currentChapter > 1) { currentChapter--; loadChapterView(currentWork, currentChapter); document.getElementById('chapterSelect').value = currentChapter; window.scrollTo({top:0}); }
}
function nextChapter() {
  if (currentWork && currentChapter < Math.min(currentWork.chapters, 5)) { currentChapter++; loadChapterView(currentWork, currentChapter); document.getElementById('chapterSelect').value = currentChapter; window.scrollTo({top:0}); }
}

function translateContent() {
  if (!currentWork) return;
  const langNames = { fr:'français', en:'anglais', es:'espagnol', pt:'portugais', de:'allemand', ja:'japonais', ar:'arabe', zh:'chinois' };
  const targetName = langNames[currentLang] || currentLang;
  const translated = `[🌐 Traduction automatique → ${targetName}]\n\n` +
    getChapterContent(currentWork, currentChapter)
      .replace('Chapitre', currentLang === 'en' ? 'Chapter' : 'Chapitre')
      .replace('Chapter 1', currentLang === 'fr' ? 'Chapitre 1' : 'Chapter 1');
  translatedWorks[currentWork.id] = translated;
  document.getElementById('reader-content').textContent = translated;
  document.getElementById('translate-banner-reader').style.display = 'none';
  showToast('🌐 Traduction appliquée !', 'success');
}

// ---- PAYMENT ----
function openPayModal() {
  if (!currentWork) return;
  currentPayMethod = '';
  document.querySelectorAll('.pay-method').forEach(m => m.classList.remove('selected'));
  document.getElementById('pay-input-area').innerHTML = '';
  const body = document.getElementById('pay-body');
  body.innerHTML = `
    <p class="pay-intro" id="pay-intro-text">Choisissez votre méthode de paiement :</p>
    <div class="pay-methods">
      <div class="pay-method" onclick="selectPayMethod('orange')" id="pm-orange"><span class="pay-icon">🟠</span><div><div class="pay-label">Orange Money</div><div class="pay-sub">Paiement mobile Orange</div></div></div>
      <div class="pay-method" onclick="selectPayMethod('momo')" id="pm-momo"><span class="pay-icon">🟡</span><div><div class="pay-label">MTN MoMo</div><div class="pay-sub">Mobile Money MTN</div></div></div>
      <div class="pay-method" onclick="selectPayMethod('card')" id="pm-card"><span class="pay-icon">💳</span><div><div class="pay-label">Carte Bancaire</div><div class="pay-sub">Visa / Mastercard</div></div></div>
      <div class="pay-method" onclick="selectPayMethod('code')" id="pm-code"><span class="pay-icon">🔑</span><div><div class="pay-label">Code spécial</div><div class="pay-sub">Code promotionnel / accès gratuit</div></div></div>
    </div>
    <div id="pay-input-area"></div>
    <div class="pay-total"><span id="pay-total-label">Total à payer</span><strong>500 FCFA</strong></div>
    <button class="btn-gold btn-full btn-lg" onclick="processPayment()" id="btn-pay-confirm">Confirmer le paiement →</button>`;
  document.getElementById('modalPay').classList.add('open');
  document.body.style.overflow = 'hidden';
}

function selectPayMethod(method) {
  currentPayMethod = method;
  document.querySelectorAll('.pay-method').forEach(m => m.classList.remove('selected'));
  const el = document.getElementById('pm-' + method);
  if (el) el.classList.add('selected');

  const area = document.getElementById('pay-input-area');
  if (method === 'orange') {
    area.innerHTML = `<div class="pay-input-group"><label>Numéro Orange Money</label><input type="tel" id="payPhone" placeholder="+237 6XX XXX XXX" maxlength="20" autocomplete="tel"></div>`;
  } else if (method === 'momo') {
    area.innerHTML = `<div class="pay-input-group"><label>Numéro MTN MoMo</label><input type="tel" id="payPhone" placeholder="+237 6XX XXX XXX" maxlength="20" autocomplete="tel"></div>`;
  } else if (method === 'card') {
    area.innerHTML = `
      <div class="pay-input-group"><label>Numéro de carte</label><input type="text" id="payCard" placeholder="XXXX XXXX XXXX XXXX" maxlength="19" autocomplete="cc-number" oninput="formatCard(this)"></div>
      <div class="pay-row">
        <div class="pay-input-group"><label>Expiration</label><input type="text" placeholder="MM/AA" maxlength="5" autocomplete="cc-exp"></div>
        <div class="pay-input-group"><label>CVV</label><input type="password" placeholder="•••" maxlength="3" autocomplete="cc-csc"></div>
      </div>`;
  } else if (method === 'code') {
    area.innerHTML = `<div class="pay-input-group"><label>Code spécial ou code promotionnel</label><input type="text" id="payCode" placeholder="Entrez votre code ici…" autocomplete="off"></div>`;
  }
}

function formatCard(input) {
  let v = input.value.replace(/\D/g, '').substring(0, 16);
  input.value = v.replace(/(.{4})/g, '$1 ').trim();
}

function processPayment() {
  if (!currentPayMethod) { showToast('⚠️ Veuillez choisir une méthode de paiement', 'error'); return; }

  if (currentPayMethod === 'code') {
    const code = (document.getElementById('payCode')?.value || '').trim();
    if (code === 'Axel12345') {
      unlockWork(true);
    } else {
      showToast('❌ Code invalide. Vérifiez et réessayez.', 'error');
    }
    return;
  }

  if (currentPayMethod === 'orange' || currentPayMethod === 'momo') {
    const phone = (document.getElementById('payPhone')?.value || '').replace(/\s/g,'');
    if (phone.length < 9) { showToast('⚠️ Numéro de téléphone invalide', 'error'); return; }
  }
  if (currentPayMethod === 'card') {
    const card = (document.getElementById('payCard')?.value || '').replace(/\s/g,'');
    if (card.length < 16) { showToast('⚠️ Numéro de carte invalide (16 chiffres requis)', 'error'); return; }
  }

  // Show processing animation
  const body = document.getElementById('pay-body');
  body.innerHTML = `<div class="processing-box"><div class="spin">⏳</div><p style="color:var(--enial-muted); margin-top:14px; font-size:14px;">Traitement du paiement en cours…</p></div>`;

  setTimeout(() => {
    if (currentWork) {
      userPurchased[currentWork.id] = true;
      saveState();
    }
    body.innerHTML = `<div class="success-box">
      <span class="tick">✅</span>
      <h3>Paiement réussi !</h3>
      <p>Vous avez maintenant accès complet à <strong style="color:var(--enial-accent)">${currentWork?.title || ''}</strong></p>
      <br>
      <button class="btn-read" onclick="unlockWork(false)" style="margin:0 auto; display:block; padding:11px 28px;">📖 Commencer la lecture</button>
    </div>`;
  }, 1800);
}

function unlockWork(isCode) {
  if (currentWork) {
    userPurchased[currentWork.id] = true;
    saveState();
  }
  closeModal('modalPay');
  if (isCode) showToast('🎉 Code accepté ! Accès complet débloqué !', 'success');
  else showToast('🎉 Accès débloqué ! Bonne lecture !', 'success');
  setTimeout(() => { readWork(); }, 350);
}

// ---- LIKES & SUBSCRIPTIONS ----
function toggleLike() {
  if (!currentWork) return;
  userLikes[currentWork.id] = !userLikes[currentWork.id];
  saveState();
  const count = currentWork.likes + (userLikes[currentWork.id] ? 1 : 0);
  document.getElementById('like-count').textContent  = formatNum(count);
  document.getElementById('modal-likes').textContent = '❤️ ' + formatNum(count);
  document.getElementById('btn-like-modal').className = 'btn-like' + (userLikes[currentWork.id] ? ' liked' : '');
  showToast(userLikes[currentWork.id] ? '❤️ J\'aime ajouté !' : '💔 Like retiré', userLikes[currentWork.id] ? 'success' : '');
}

function toggleSubscribe() {
  if (!currentWork) return;
  userSubs[currentWork.id] = !userSubs[currentWork.id];
  saveState();
  document.getElementById('btn-sub-modal').className   = 'btn-subscribe' + (userSubs[currentWork.id] ? ' subscribed' : '');
  document.getElementById('btn-sub-modal').textContent = userSubs[currentWork.id] ? '🔔 Abonné ✓' : '🔔 Suivre';
  showToast(userSubs[currentWork.id] ? '🔔 Abonnement activé !' : '🔕 Abonnement annulé', 'info');
}

// ---- COMMENTS ----
function addComment() {
  if (!currentWork) return;
  const input = document.getElementById('commentInput');
  const text  = input.value.trim();
  if (!text) return;
  if (!commentsDB[currentWork.id]) commentsDB[currentWork.id] = [];
  commentsDB[currentWork.id].unshift({ author: 'Vous', text, time: 'à l\'instant' });
  saveComments();
  input.value = '';
  renderComments();
  showToast('💬 Commentaire publié !', 'success');
}

function renderComments() {
  if (!currentWork) return;
  const comments = commentsDB[currentWork.id] || [];
  document.getElementById('comment-count').textContent = comments.length;
  document.getElementById('comments-list').innerHTML = comments.slice(0, 10).map(c =>
    `<div class="comment">
       <div class="comment-author">${escapeHtml(c.author)} <span style="font-weight:400; color:var(--enial-border); font-size:11px">${c.time || ''}</span></div>
       <div class="comment-text">${escapeHtml(c.text)}</div>
     </div>`
  ).join('');
}

function escapeHtml(str) {
  return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

// ---- PUBLISH ----
function publishWork() {
  const title   = document.getElementById('pub-work-title').value.trim();
  const author  = document.getElementById('pub-author').value.trim();
  const desc    = document.getElementById('pub-desc').value.trim();
  const content = document.getElementById('pub-content').value.trim();
  const emoji   = document.getElementById('pub-emoji').value.trim() || '📖';
  const type    = document.getElementById('pub-type').value;
  const genre   = document.getElementById('pub-genre').value;
  const lang    = document.getElementById('pub-lang').value;
  const isFree  = document.getElementById('pub-free').checked;

  if (!title)  { showToast('⚠️ Veuillez saisir un titre', 'error'); return; }
  if (!author) { showToast('⚠️ Veuillez saisir le nom de l\'auteur', 'error'); return; }

  const newWork = {
    id: WORKS.length + 1,
    title, author, type, genre, emoji, lang,
    likes: 0, views: 0, chapters: 1,
    desc: desc || 'Nouvelle œuvre publiée sur Enial.',
    content: content || `Chapitre 1 — Début\n\nL'histoire commence ici…`,
    free: isFree
  };
  WORKS.unshift(newWork);
  commentsDB[newWork.id] = [];

  // Reset form
  document.getElementById('pub-work-title').value = '';
  document.getElementById('pub-author').value     = '';
  document.getElementById('pub-desc').value       = '';
  document.getElementById('pub-content').value    = '';
  document.getElementById('pub-emoji').value      = '📖';
  document.getElementById('pub-free').checked     = false;

  showToast('🚀 Œuvre publiée avec succès !', 'success');
  setTimeout(() => showPage('browse'), 900);
}

// ---- MODAL HELPERS ----
function closeModal(id) {
  document.getElementById(id).classList.remove('open');
  document.body.style.overflow = '';
}

function modalBgClick(e, id) {
  if (e.target === e.currentTarget) closeModal(id);
}

// ---- TOAST ----
let toastTimer = null;
function showToast(msg, type) {
  const t = document.getElementById('toast');
  t.textContent = msg;
  t.className   = 'toast ' + (type |/* ===========================
   ENIAL — Main Stylesheet
   =========================== */

:root {
  --enial-purple: #6C3DD3;
  --enial-dark: #1A0D2E;
  --enial-mid: #2D1555;
  --enial-accent: #C084FC;
  --enial-gold: #F59E0B;
  --enial-text: #F3F0FF;
  --enial-muted: #A78BCA;
  --enial-card: #241040;
  --enial-border: rgba(192, 132, 252, 0.18);
  --enial-success: #34D399;
  --enial-danger: #F87171;
  --enial-info: #818CF8;
  --nav-h: 56px;
  --radius-sm: 8px;
  --radius-md: 12px;
  --radius-lg: 16px;
}

/* ===== RESET & BASE ===== */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html { scroll-behavior: smooth; }
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
  background: var(--enial-dark);
  color: var(--enial-text);
  min-height: 100vh;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}
button { cursor: pointer; font-family: inherit; }
input, select, textarea { font-family: inherit; }
a { color: var(--enial-accent); text-decoration: none; }

/* ===== SPLASH ===== */
#splash {
  position: fixed; inset: 0; z-index: 9999;
  background: var(--enial-dark);
  display: flex; align-items: center; justify-content: center;
  transition: opacity 0.4s ease;
}
.splash-inner { text-align: center; }
.splash-inner h1 {
  font-size: 36px; font-weight: 800; letter-spacing: 4px;
  background: linear-gradient(135deg, #F3F0FF, #C084FC);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent;
  margin: 16px 0 8px;
}
.splash-inner p { color: var(--enial-muted); font-size: 14px; }

/* ===== NAV ===== */
.nav {
  position: sticky; top: 0; z-index: 200;
  height: var(--nav-h);
  background: rgba(45, 21, 85, 0.95);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border-bottom: 1px solid var(--enial-border);
  display: flex; align-items: center; justify-content: space-between;
  padding: 0 20px; gap: 12px;
}
.nav-logo {
  display: flex; align-items: center; gap: 10px; cursor: pointer; flex-shrink: 0;
}
.logo-text {
  font-size: 20px; font-weight: 800; letter-spacing: 2px;
  background: linear-gradient(135deg, #C084FC, #818CF8);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent;
}
.nav-links { display: flex; gap: 4px; }
.nav-btn {
  background: none; border: none; color: var(--enial-muted);
  padding: 7px 13px; border-radius: var(--radius-sm);
  font-size: 13px; font-weight: 500; transition: all 0.2s;
}
.nav-btn:hover, .nav-btn.active { background: rgba(108,61,211,0.35); color: var(--enial-accent); }
.nav-actions { display: flex; align-items: center; gap: 8px; flex-shrink: 0; }
.nav-menu-btn {
  display: none; background: none; border: none;
  color: var(--enial-muted); font-size: 20px; padding: 4px 8px;
}

/* ===== MOBILE MENU ===== */
.mobile-menu {
  display: none; flex-direction: column;
  background: var(--enial-mid);
  border-bottom: 1px solid var(--enial-border);
  padding: 8px 16px 12px;
  position: sticky; top: var(--nav-h); z-index: 190;
}
.mobile-menu.open { display: flex; }
.mobile-menu button {
  background: none; border: none; color: var(--enial-muted);
  text-align: left; padding: 10px 8px; font-size: 15px;
  border-bottom: 1px solid var(--enial-border);
}
.mobile-menu button:last-child { border-bottom: none; }

/* ===== BUTTONS ===== */
.btn-primary {
  background: var(--enial-purple); color: white;
  border: none; padding: 8px 18px;
  border-radius: var(--radius-sm); font-size: 13px; font-weight: 600;
  transition: all 0.2s;
}
.btn-primary:hover { background: #7C4FE0; transform: translateY(-1px); }
.btn-outline {
  background: none; border: 1px solid var(--enial-border);
  color: var(--enial-accent); padding: 8px 18px;
  border-radius: var(--radius-sm); font-size: 13px; font-weight: 500;
  transition: all 0.2s;
}
.btn-outline:hover { background: rgba(108,61,211,0.2); }
.btn-lg { padding: 12px 28px; font-size: 15px; }
.btn-full { width: 100%; }
.btn-gold {
  background: var(--enial-gold); color: #1A0D2E;
  border: none; padding: 10px 20px;
  border-radius: var(--radius-sm); font-size: 14px; font-weight: 700;
  transition: all 0.2s;
}
.btn-gold:hover { background: #FBBF24; transform: translateY(-1px); }
.btn-back {
  background: none; border: 1px solid var(--enial-border);
  color: var(--enial-muted); padding: 7px 14px;
  border-radius: var(--radius-sm); font-size: 13px; transition: all 0.2s;
}
.btn-back:hover { border-color: var(--enial-accent); color: var(--enial-accent); }
.btn-read {
  background: var(--enial-success); color: #0F3026;
  border: none; padding: 9px 20px;
  border-radius: var(--radius-sm); font-size: 14px; font-weight: 700;
  transition: all 0.2s;
}
.btn-read:hover { background: #6EE7B7; }
.btn-like {
  background: none; border: 1px solid var(--enial-border);
  color: var(--enial-muted); padding: 9px 14px;
  border-radius: var(--radius-sm); font-size: 13px; transition: all 0.2s;
}
.btn-like:hover, .btn-like.liked { border-color: #F87171; color: #F87171; background: rgba(248,113,113,0.1); }
.btn-subscribe {
  background: none; border: 1px solid var(--enial-border);
  color: var(--enial-muted); padding: 9px 14px;
  border-radius: var(--radius-sm); font-size: 13px; transition: all 0.2s;
}
.btn-subscribe:hover, .btn-subscribe.subscribed {
  border-color: var(--enial-accent); color: var(--enial-accent);
  background: rgba(192,132,252,0.1);
}

/* ===== LANG SELECT ===== */
.lang-select {
  background: var(--enial-card); border: 1px solid var(--enial-border);
  color: var(--enial-muted); padding: 7px 10px;
  border-radius: var(--radius-sm); font-size: 12px; cursor: pointer;
  max-width: 140px;
}

/* ===== LAYOUT ===== */
.container { max-width: 960px; margin: 0 auto; padding: 24px 20px; }
.page { display: none; }
.page.active { display: block; min-height: calc(100vh - var(--nav-h)); }
.section-header { margin-bottom: 14px; }
.section-title { font-size: 17px; font-weight: 700; color: var(--enial-accent); }

/* ===== HERO ===== */
.hero {
  position: relative; overflow: hidden;
  background: linear-gradient(135deg, #2D1555 0%, #1A0D2E 60%, #0F0720 100%);
  padding: 56px 24px; text-align: center;
}
.hero-bg-orb {
  position: absolute; top: -80px; right: -80px;
  width: 320px; height: 320px;
  background: radial-gradient(circle, rgba(108,61,211,0.35), transparent 70%);
  border-radius: 50%; pointer-events: none;
}
.hero-content { position: relative; z-index: 1; max-width: 600px; margin: 0 auto; }
.hero h1 {
  font-size: 40px; font-weight: 800;
  background: linear-gradient(135deg, #F3F0FF, #C084FC);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent;
  margin-bottom: 12px; line-height: 1.2;
}
.hero p { color: var(--enial-muted); font-size: 16px; margin-bottom: 24px; }
.hero-btns { display: flex; gap: 12px; justify-content: center; flex-wrap: wrap; }

/* ===== WORKS GRID ===== */
.works-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 14px;
  margin-bottom: 16px;
}
.work-card {
  background: var(--enial-card);
  border: 1px solid var(--enial-border);
  border-radius: var(--radius-md); overflow: hidden;
  cursor: pointer; transition: all 0.22s;
}
.work-card:hover {
  transform: translateY(-4px);
  border-color: var(--enial-purple);
  box-shadow: 0 10px 28px rgba(108,61,211,0.28);
}
.work-cover {
  width: 100%; aspect-ratio: 3/4;
  display: flex; align-items: center; justify-content: center;
  font-size: 44px; position: relative;
}
.work-badge {
  position: absolute; top: 6px; right: 6px;
  padding: 2px 7px; border-radius: 6px;
  font-size: 10px; font-weight: 700;
}
.badge-free  { background: rgba(52,211,153,0.2); color: #34D399; }
.badge-paid  { background: rgba(245,158,11,0.2); color: #F59E0B; }
.badge-new   { background: rgba(192,132,252,0.2); color: #C084FC; }
.work-info { padding: 8px 10px 10px; }
.work-title { font-size: 13px; font-weight: 600; margin-bottom: 3px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.work-meta { font-size: 11px; color: var(--enial-muted); display: flex; justify-content: space-between; }
.work-type { font-size: 10px; font-weight: 700; text-transform: uppercase; letter-spacing: 0.5px; }
.type-webtoon { color: #818CF8; }
.type-novel   { color: #FB923C; }
.type-manga   { color: #F472B6; }

/* ===== SEARCH & FILTERS ===== */
.search-bar { display: flex; gap: 8px; margin-bottom: 20px; }
.search-bar input {
  flex: 1; background: var(--enial-card); border: 1px solid var(--enial-border);
  color: var(--enial-text); padding: 11px 14px;
  border-radius: var(--radius-sm); font-size: 14px; outline: none;
  transition: border-color 0.2s;
}
.search-bar input:focus { border-color: var(--enial-purple); }
.search-bar input::placeholder { color: var(--enial-muted); }
.search-bar button {
  background: var(--enial-purple); border: none; color: white;
  padding: 11px 18px; border-radius: var(--radius-sm);
  font-size: 14px; font-weight: 600;
}
.filter-tabs { display: flex; gap: 8px; margin-bottom: 20px; flex-wrap: wrap; }
.filter-tab {
  background: var(--enial-card); border: 1px solid var(--enial-border);
  color: var(--enial-muted); padding: 7px 16px;
  border-radius: 20px; font-size: 13px; transition: all 0.2s;
}
.filter-tab:hover { background: rgba(108,61,211,0.2); color: var(--enial-accent); }
.filter-tab.active { background: var(--enial-purple); border-color: var(--enial-purple); color: white; }

/* ===== PUBLISH FORM ===== */
.publisher-form {
  background: var(--enial-card); border: 1px solid var(--enial-border);
  border-radius: var(--radius-lg); padding: 28px;
  max-width: 620px; margin: 0 auto;
}
.publisher-form h2 { font-size: 20px; font-weight: 700; margin-bottom: 18px; color: var(--enial-accent); }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.form-group { margin-bottom: 14px; }
.form-group label { display: block; font-size: 13px; color: var(--enial-muted); margin-bottom: 6px; }
.form-group input,
.form-group select,
.form-group textarea {
  width: 100%; background: var(--enial-mid);
  border: 1px solid var(--enial-border); color: var(--enial-text);
  padding: 10px 12px; border-radius: var(--radius-sm);
  font-size: 14px; outline: none; transition: border-color 0.2s;
}
.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus { border-color: var(--enial-purple); }
.form-group select option { background: var(--enial-mid); }
.form-group textarea { resize: vertical; min-height: 80px; }
.tall-textarea { min-height: 140px !important; }
.checkbox-label { display: flex !important; align-items: center; gap: 8px; font-size: 14px !important; color: var(--enial-text) !important; cursor: pointer; }
.checkbox-label input[type=checkbox] { width: 16px; height: 16px; accent-color: var(--enial-purple); }
.info-banner {
  background: rgba(108,61,211,0.12); border: 1px solid rgba(108,61,211,0.3);
  border-radius: var(--radius-sm); padding: 10px 14px;
  font-size: 13px; color: var(--enial-muted); margin-bottom: 18px;
}

/* ===== READER ===== */
.reader-bar {
  position: sticky; top: var(--nav-h); z-index: 100;
  background: rgba(36,16,64,0.95); backdrop-filter: blur(10px);
  border-bottom: 1px solid var(--enial-border);
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 20px; gap: 12px;
}
.reader-title { font-size: 14px; font-weight: 600; color: var(--enial-muted); flex: 1; text-align: center; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.chapter-select {
  background: var(--enial-card); border: 1px solid var(--enial-border);
  color: var(--enial-text); padding: 7px 10px;
  border-radius: var(--radius-sm); font-size: 13px; cursor: pointer;
}
.reader-content {
  background: var(--enial-card); border-radius: var(--radius-md);
  padding: 28px 32px; line-height: 1.9; font-size: 16px;
  color: #D4C5EF; white-space: pre-line; margin-bottom: 24px;
}
.reader-nav-btns { display: flex; justify-content: space-between; gap: 12px; margin-bottom: 32px; }
.translate-banner {
  background: rgba(129,140,248,0.12); border: 1px solid rgba(129,140,248,0.3);
  border-radius: var(--radius-sm); padding: 10px 14px;
  font-size: 13px; color: #818CF8;
  display: flex; gap: 8px; align-items: center; justify-content: space-between;
  margin-bottom: 16px; flex-wrap: wrap;
}
.translate-banner button {
  background: #818CF8; color: white; border: none;
  padding: 5px 14px; border-radius: 6px; font-size: 12px; font-weight: 600;
}

/* ===== MODAL ===== */
.modal-bg {
  display: none; position: fixed; inset: 0;
  background: rgba(0,0,0,0.75); z-index: 500;
  align-items: flex-start; justify-content: center;
  overflow-y: auto; padding: 20px;
}
.modal-bg.open { display: flex; }
.modal {
  background: var(--enial-card); border: 1px solid var(--enial-border);
  border-radius: var(--radius-lg); width: 100%; max-width: 560px;
  margin: auto; position: relative; overflow: hidden;
}
.modal-header {
  padding: 16px 20px 0;
  display: flex; justify-content: space-between; align-items: center;
}
.modal-close {
  background: none; border: none; color: var(--enial-muted);
  font-size: 20px; line-height: 1; transition: color 0.2s;
}
.modal-close:hover { color: var(--enial-text); }
.modal-body { padding: 16px 20px 24px; overflow-y: auto; max-height: 82vh; }
.modal-type-badge {
  padding: 3px 10px; border-radius: 6px; font-size: 11px; font-weight: 700;
  background: rgba(192,132,252,0.15); color: var(--enial-accent);
}
.work-detail-hero { display: flex; gap: 16px; margin-bottom: 14px; }
.work-detail-cover {
  width: 84px; height: 112px; border-radius: 10px;
  display: flex; align-items: center; justify-content: center;
  font-size: 38px; flex-shrink: 0;
}
.work-detail-info h2 { font-size: 18px; font-weight: 700; margin-bottom: 4px; }
.modal-author { font-size: 12px; color: var(--enial-muted); margin-bottom: 8px; }
.stats-row { display: flex; gap: 12px; font-size: 12px; color: var(--enial-muted); flex-wrap: wrap; }
.work-desc { font-size: 13px; color: var(--enial-muted); line-height: 1.65; margin-bottom: 14px; }
.price-box {
  background: rgba(108,61,211,0.15); border: 1px solid rgba(108,61,211,0.4);
  border-radius: var(--radius-sm); padding: 12px 16px; margin-bottom: 14px;
}
.price-box .price { font-size: 22px; font-weight: 800; color: var(--enial-gold); }
.price-box .price-note { font-size: 12px; color: var(--enial-muted); margin-top: 3px; }
.action-row { display: flex; gap: 10px; flex-wrap: wrap; margin-bottom: 16px; }

/* ===== COMMENTS ===== */
.comments-section { border-top: 1px solid var(--enial-border); padding-top: 16px; }
.comments-section h3 { font-size: 14px; font-weight: 600; margin-bottom: 12px; color: var(--enial-accent); }
.comment-input-row { display: flex; gap: 8px; margin-bottom: 12px; }
.comment-input-row input {
  flex: 1; background: var(--enial-mid); border: 1px solid var(--enial-border);
  color: var(--enial-text); padding: 9px 12px;
  border-radius: var(--radius-sm); font-size: 13px; outline: none;
}
.comment-input-row input:focus { border-color: var(--enial-purple); }
.comment-input-row button {
  background: var(--enial-purple); border: none; color: white;
  padding: 9px 14px; border-radius: var(--radius-sm);
  font-size: 13px; font-weight: 600;
}
.comment {
  background: var(--enial-mid); border-radius: var(--radius-sm);
  padding: 10px 12px; margin-bottom: 8px;
}
.comment-author { font-size: 12px; font-weight: 600; color: var(--enial-accent); margin-bottom: 3px; }
.comment-text { font-size: 13px; color: var(--enial-muted); }

/* ===== PAYMENT ===== */
.pay-intro { font-size: 13px; color: var(--enial-muted); margin-bottom: 14px; }
.pay-methods { display: flex; flex-direction: column; gap: 10px; margin-bottom: 14px; }
.pay-method {
  border: 1px solid var(--enial-border); border-radius: var(--radius-sm);
  padding: 12px 16px; cursor: pointer; transition: all 0.2s;
  display: flex; align-items: center; gap: 12px;
}
.pay-method:hover { border-color: var(--enial-purple); background: rgba(108,61,211,0.1); }
.pay-method.selected { border-color: var(--enial-purple); background: rgba(108,61,211,0.2); }
.pay-icon { font-size: 24px; }
.pay-label { font-size: 14px; font-weight: 600; }
.pay-sub { font-size: 12px; color: var(--enial-muted); }
.pay-input-group { margin: 12px 0; }
.pay-input-group label { display: block; font-size: 13px; color: var(--enial-muted); margin-bottom: 6px; }
.pay-input-group input {
  width: 100%; background: var(--enial-mid); border: 1px solid var(--enial-border);
  color: var(--enial-text); padding: 10px 14px;
  border-radius: var(--radius-sm); font-size: 14px; outline: none;
  transition: border-color 0.2s;
}
.pay-input-group input:focus { border-color: var(--enial-purple); }
.pay-row { display: flex; gap: 10px; }
.pay-row .pay-input-group { flex: 1; }
.pay-total {
  background: rgba(245,158,11,0.1); border: 1px solid rgba(245,158,11,0.3);
  border-radius: var(--radius-sm); padding: 10px 14px; margin-bottom: 14px;
  display: flex; justify-content: space-between; align-items: center;
}
.pay-total span { font-size: 13px; color: var(--enial-muted); }
.pay-total strong { font-size: 18px; font-weight: 800; color: var(--enial-gold); }
.success-box { text-align: center; padding: 28px 16px; }
.success-box .tick { font-size: 52px; margin-bottom: 14px; display: block; }
.success-box h3 { font-size: 20px; font-weight: 700; color: var(--enial-success); margin-bottom: 8px; }
.success-box p { font-size: 14px; color: var(--enial-muted); }
.processing-box { text-align: center; padding: 28px; }
.processing-box .spin { font-size: 36px; display: inline-block; animation: spin 1s linear infinite; }

/* ===== TOAST ===== */
.toast {
  position: fixed; bottom: 24px; right: 24px;
  background: var(--enial-mid); border: 1px solid var(--enial-border);
  border-radius: var(--radius-sm); padding: 12px 18px;
  font-size: 13px; color: var(--enial-text); z-index: 1000;
  transform: translateY(80px); opacity: 0;
  transition: all 0.3s ease; max-width: 320px;
  pointer-events: none;
}
.toast.show { transform: translateY(0); opacity: 1; }
.toast.success { border-color: var(--enial-success); color: var(--enial-success); }
.toast.error   { border-color: var(--enial-danger); color: var(--enial-danger); }
.toast.info    { border-color: var(--enial-info); color: var(--enial-info); }

/* ===== EMPTY STATE ===== */
.empty-state {
  color: var(--enial-muted); font-size: 14px;
  padding: 32px 20px; grid-column: 1 / -1; text-align: center;
}

/* ===== ANIMATIONS ===== */
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
.work-card { animation: fadeIn 0.25s ease both; }

/* ===== RESPONSIVE ===== */
@media (max-width: 640px) {
  .nav-links { display: none; }
  .nav-menu-btn { display: block; }
  .hero h1 { font-size: 28px; }
  .works-grid { grid-template-columns: repeat(2, 1fr); }
  .form-row { grid-template-columns: 1fr; }
  .publisher-form { padding: 20px 16px; }
  .reader-content { padding: 18px 16px; }
  .modal-body { max-height: 75vh; }
  .work-detail-hero { flex-direction: column; }
  .work-detail-cover { width: 100%; height: 120px; }
  .reader-bar { flex-wrap: wrap; gap: 8px; }
  .reader-title { display: none; }
}
@media (max-width: 400px) {
  .works-grid { grid-template-columns: repeat(2, 1fr); gap: 10px; }
}

/* ===== SCROLLBAR ===== */
::-webkit-scrollbar { width// ===========================
// ENIAL — Translations (i18n)
// ===========================

const I18N = {
  fr: {
    'hero-title': 'Bienvenue sur Enial',
    'hero-sub': 'Découvrez des milliers de webtoons, romans et novels créés par des auteurs passionnés. Lisez, commentez, partagez.',
    'btn-explore': '📚 Explorer',
    'btn-publish-hero': '✏️ Publier gratuitement',
    'navPublishBtn': '+ Publier',
    'label-trending': 'Tendances',
    'label-new': 'Nouveautés',
    'label-free': 'Gratuit',
    'pub-title': 'Publier une œuvre',
    'pub-info': 'La publication est 100% gratuite. Les lecteurs paient 500 FCFA pour un accès complet.',
    'lab-worktitle': 'Titre de l\'œuvre *',
    'lab-author': 'Nom de l\'auteur *',
    'lab-type': 'Type *',
    'lab-genre': 'Genre',
    'lab-desc': 'Description',
    'lab-content': 'Contenu — Premier chapitre',
    'lab-emoji': 'Icône / Emoji couverture',
    'lab-lang-pub': 'Langue de l\'œuvre',
    'lab-free-check': 'Rendre cette œuvre gratuite',
    'btn-pub': '🚀 Publier maintenant',
    'f-all': 'Tout',
    'f-webtoon': '🎨 Webtoon',
    'f-novel': '📖 Novel',
    'f-manga': '📕 Manga',
    'f-free': '🆓 Gratuit',
    'pay-intro-text': 'Choisissez votre méthode de paiement :',
    'pay-total-label': 'Total à payer',
    'translate-banner-text': 'Cette œuvre n\'est pas dans votre langue. Traduction automatique disponible.',
    'btn-translate': 'Traduire',
    'searchPlaceholder': 'Rechercher une œuvre, un auteur…'
  },
  en: {
    'hero-title': 'Welcome to Enial',
    'hero-sub': 'Discover thousands of webtoons, novels and mangas created by passionate authors. Read, comment, share.',
    'btn-explore': '📚 Explore',
    'btn-publish-hero': '✏️ Publish for free',
    'navPublishBtn': '+ Publish',
    'label-trending': 'Trending',
    'label-new': 'New Releases',
    'label-free': 'Free',
    'pub-title': 'Publish a work',
    'pub-info': 'Publishing is 100% free. Readers pay 500 FCFA for full access.',
    'lab-worktitle': 'Work title *',
    'lab-author': 'Author name *',
    'lab-type': 'Type *',
    'lab-genre': 'Genre',
    'lab-desc': 'Description',
    'lab-content': 'Content — First chapter',
    'lab-emoji': 'Cover icon / Emoji',
    'lab-lang-pub': 'Work language',
    'lab-free-check': 'Make this work free',
    'btn-pub': '🚀 Publish now',
    'f-all': 'All',
    'f-webtoon': '🎨 Webtoon',
    'f-novel': '📖 Novel',
    'f-manga': '📕 Manga',
    'f-free': '🆓 Free',
    'pay-intro-text': 'Choose your payment method:',
    'pay-total-label': 'Total to pay',
    'translate-banner-text': 'This work is not in your language. Auto-translation available.',
    'btn-translate': 'Translate',
    'searchPlaceholder': 'Search a work, an author…'
  },
  es: {
    'hero-title': 'Bienvenido a Enial',
    'hero-sub': 'Descubre miles de webtoons, novelas y mangas creados por autores apasionados. Lee, comenta, comparte.',
    'btn-explore': '📚 Explorar',
    'btn-publish-hero': '✏️ Publicar gratis',
    'navPublishBtn': '+ Publicar',
    'label-trending': 'Tendencias',
    'label-new': 'Novedades',
    'label-free': 'Gratis',
    'pub-title': 'Publicar una obra',
    'pub-info': 'Publicar es 100% gratis. Los lectores pagan 500 FCFA por acceso completo.',
    'lab-worktitle': 'Título de la obra *',
    'lab-author': 'Nombre del autor *',
    'lab-type': 'Tipo *',
    'lab-genre': 'Género',
    'lab-desc': 'Descripción',
    'lab-content': 'Contenido — Primer capítulo',
    'lab-emoji': 'Icono / Emoji de portada',
    'lab-lang-pub': 'Idioma de la obra',
    'lab-free-check': 'Hacer esta obra gratuita',
    'btn-pub': '🚀 Publicar ahora',
    'f-all': 'Todo',
    'f-webtoon': '🎨 Webtoon',
    'f-novel': '📖 Novela',
    'f-manga': '📕 Manga',
    'f-free': '🆓 Gratis',
    'pay-intro-text': 'Elige tu método de pago:',
    'pay-total-label': 'Total a pagar',
    'translate-banner-text': 'Esta obra no está en tu idioma. Traducción automática disponible.',
    'btn-translate': 'Traducir',
    'searchPlaceholder': 'Buscar una obra, un autor…'
  },
  pt: {
    'hero-title': 'Bem-vindo ao Enial',
    'hero-sub': 'Descubra milhares de webtoons, romances e novels criados por autores apaixonados.',
    'btn-explore': '📚 Explorar',
    'btn-publish-hero': '✏️ Publicar gratuitamente',
    'navPublishBtn': '+ Publicar',
    'label-trending': 'Em alta',
    'label-new': 'Novidades',
    'label-free': 'Grátis',
    'pub-title': 'Publicar uma obra',
    'pub-info': 'Publicar é 100% gratuito. Os leitores pagam 500 FCFA pelo acesso completo.',
    'lab-worktitle': 'Título da obra *',
    'lab-author': 'Nome do autor *',
    'lab-type': 'Tipo *',
    'lab-genre': 'Gênero',
    'lab-desc': 'Descrição',
    'lab-content': 'Conteúdo — Primeiro capítulo',
    'lab-emoji': 'Ícone / Emoji da capa',
    'lab-lang-pub': 'Idioma da obra',
    'lab-free-check': 'Tornar esta obra gratuita',
    'btn-pub': '🚀 Publicar agora',
    'f-all': 'Todos',
    'f-webtoon': '🎨 Webtoon',
    'f-novel': '📖 Novel',
    'f-manga': '📕 Manga',
    'f-free': '🆓 Grátis',
    'pay-intro-text': 'Escolha seu método de pagamento:',
    'pay-total-label': 'Total a pagar',
    'translate-banner-text': 'Esta obra não está no seu idioma. Tradução automática disponível.',
    'btn-translate': 'Traduzir',
    'searchPlaceholder': 'Pesquisar uma obra, um autor…'
  },
  de: {
    'hero-title': 'Willkommen bei Enial',
    'hero-sub': 'Entdecke tausende von Webtoons, Romanen und Novels von leidenschaftlichen Autoren.',
    'btn-explore': '📚 Entdecken',
    'btn-publish-hero': '✏️ Kostenlos veröffentlichen',
    'navPublishBtn': '+ Veröffentlichen',
    'label-trending': 'Trends',
    'label-new': 'Neuerscheinungen',
    'label-free': 'Kostenlos',
    'pub-title': 'Ein Werk veröffentlichen',
    'pub-info': 'Das Veröffentlichen ist 100% kostenlos. Leser zahlen 500 FCFA für Vollzugang.',
    'lab-worktitle': 'Werktitel *',
    'lab-author': 'Autorenname *',
    'lab-type': 'Typ *',
    'lab-genre': 'Genre',
    'lab-desc': 'Beschreibung',
    'lab-content': 'Inhalt — Erstes Kapitel',
    'lab-emoji': 'Cover-Symbol / Emoji',
    'lab-lang-pub': 'Sprache des Werks',
    'lab-free-check': 'Dieses Werk kostenlos machen',
    'btn-pub': '🚀 Jetzt veröffentlichen',
    'f-all': 'Alle',
    'f-webtoon': '🎨 Webtoon',
    'f-novel': '📖 Roman',
    'f-manga': '📕 Manga',
    'f-free': '🆓 Kostenlos',
    'pay-intro-text': 'Zahlungsmethode wählen:',
    'pay-total-label': 'Gesamtbetrag',
    'translate-banner-text': 'Dieses Werk ist nicht in Ihrer Sprache. Automatische Übersetzung verfügbar.',
    'btn-translate': 'Übersetzen',
    'searchPlaceholder': 'Werk oder Autor suchen…'
  },
  ja: {
    'hero-title': 'Enialへようこそ',
    'hero-sub': 'ウェブトゥーン、小説、漫画を発見・公開できるプラットフォーム。',
    'btn-explore': '📚 探索する',
    'btn-publish-hero': '✏️ 無料で公開',
    'navPublishBtn': '+ 公開',
    'label-trending': 'トレンド',
    'label-new': '新着',
    'label-free': '無料',
    'pub-title': '作品を公開',
    'pub-info': '公開は100%無料です。読者はフルアクセスに500 FCFAを支払います。',
    'lab-worktitle': 'タイトル *',
    'lab-author': '著者名 *',
    'lab-type': 'タイプ *',
    'lab-genre': 'ジャンル',
    'lab-desc': 'あらすじ',
    'lab-content': '内容 — 第1章',
    'lab-emoji': 'カバー絵文字',
    'lab-lang-pub': '作品の言語',
    'lab-free-check': 'この作品を無料にする',
    'btn-pub': '🚀 今すぐ公開',
    'f-all': 'すべて',
    'f-webtoon': '🎨 ウェブトゥーン',
    'f-novel': '📖 小説',
    'f-manga': '📕 漫画',
    'f-free': '🆓 無料',
    'pay-intro-text': '支払い方法を選択:',
    'pay-total-label': '合計金額',
    'translate-banner-text': 'この作品はあなたの言語ではありません。自動翻訳が利用可能です。',
    'btn-translate': '翻訳する',
    'searchPlaceholder': '作品や著者を検索…'
  },
  ar: {
    'hero-title': 'مرحباً بك في إنيال',
    'hero-sub': 'اكتشف آلاف قصص الويبتون والروايات التي أبدعها مؤلفون موهوبون.',
    'btn-explore': '📚 استكشف',
    'btn-publish-hero': '✏️ انشر مجاناً',
    'navPublishBtn': '+ نشر',
    'label-trending': 'الأكثر رواجاً',
    'label-new': 'الجديد',
    'label-free': 'مجاني',
    'pub-title': 'نشر عمل',
    'pub-info': 'النشر مجاني 100%. يدفع القراء 500 فرنك للوصول الكامل.',
    'lab-worktitle': 'عنوان العمل *',
    'lab-author': 'اسم المؤلف *',
    'lab-type': 'النوع *',
    'lab-genre': 'التصنيف',
    'lab-desc': 'الوصف',
    'lab-content': 'المحتوى — الفصل الأول',
    'lab-emoji': 'أيقونة الغلاف',
    'lab-lang-pub': 'لغة العمل',
    'lab-free-check': 'جعل هذا العمل مجانياً',
    'btn-pub': '🚀 انشر الآن',
    'f-all': 'الكل',
    'f-webtoon': '🎨 ويبتون',
    'f-novel': '📖 رواية',
    'f-manga': '📕 مانجا',
    'f-free': '🆓 مجاني',
    'pay-intro-text': 'اختر طريقة الدفع:',
    'pay-total-label': 'المبلغ الإجمالي',
    'translate-banner-text': 'هذا العمل ليس بلغتك. الترجمة التلقائية متاحة.',
    'btn-translate': 'ترجم',
    'searchPlaceholder': 'ابحث عن عمل أو مؤلف…'
  },
  zh: {
    'hero-title': '欢迎来到Enial',
    'hero-sub': '发现数千部由热情创作者打造的条漫、小说和漫画。阅读、评论、分享。',
    'btn-explore': '📚 探索',
    'btn-publish-hero': '✏️ 免费发布',
    'navPublishBtn': '+ 发布',
    'label-trending': '热门',
    'label-new': '最新',
    'label-free': '免费',
    'pub-title': '发布作品',
    'pub-info': '发布完全免费。读者支付500 FCFA获得完整访问权限。',
    'lab-worktitle': '作品标题 *',
    'lab-author': '作者姓名 *',
    'lab-type': '类型 *',
    'lab-genre': '题材',
    'lab-desc': '简介',
    'lab-content': '内容 — 第一章',
    'lab-emoji': '封面图标',
    'lab-lang-pub': '作品语言',
    'lab-free-check': '将此作品设为免费',
    'btn-pub': '🚀 立即发布',
    'f-all': '全部',
    'f-webtoon': '🎨 条漫',
    'f-novel': '📖 小说',
    'f-manga': '📕 漫画',
    'f-free': '🆓 免费',
    'pay-intro-text': '选择支付方式：',
    'pay-total-label': '应付总额',
    'translate-banner-text': '此作品不是您的语言。自动翻译可用。',
    'btn-translate': '翻译',
    'searchPlaceholder': '搜索作品或作者…'
  }
};

function applyTranslation(lang) {
  const t = I18N[lang] || I18N['fr'];
  for (const [id, text] of Object.entries(t)) {
    const el = document.getElementById(id);
    if (el) {
      if (el.tagName === 'INPUT' && el.type !== 'checkbox') {
        el.placeholder = text;
      } else {
        el.textContent = text;
      }
    }
  }
  const si = document.getElementById('searchInput');
  if (si) si.placeholder = t['searchPlaceholder'] || 'Rechercher…';

  // RTL support for Arabic
  document.body.dir = lang === 'ar' ? 'rtl' : 'ltr';
}
: 6px; }
::-webkit-scrollbar-track { background: var(--enial-dark); }
::-webkit-scrollbar-thumb { background: var(--enial-mid); border-radius: 3px; }
::-webkit-scrollbar-thumb:hover { background: var(--enial-purple); }
| '');
  t.classList.add('show');
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => t.classList.remove('show'), 3200);
}

// ---- KEYBOARD ----
document.addEventListener('keydown', e => {
  if (e.key === 'Escape') {
    closeModal('modalWork');
    closeModal('modalPay');
  }
});
