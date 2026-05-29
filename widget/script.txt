// =====================================================
//   CHATBOX-IA — WhatsApp Widget · script.js
// =====================================================

const CONFIG = {
  maxMessages:     15,
  messageDuration: 0,
  theme:           'dark',
  streamName:      'Chat Twitch',
  avatarUrl:       '',
  fontSize:        14,
  showHeader:      true,
  showInputBar:    true,
  showBadges:      true,
  showTimestamp:   true,
  // Connexion Twitch directe
  twitchChannel:   '',
  // Sub goal
  subGoalEnabled:  false,
  subGoalCurrent:  0,
  subGoalTarget:   100,
  subGoalLabel:    'Objectif abonnements',
  subGoalDesc:     '',
};

const BADGE_META = {
  broadcaster: { emoji: '👑', color: '#E91916' },
  moderator:   { emoji: '⚔️', color: '#34AE1A' },
  subscriber:  { emoji: '⭐', color: '#9147FF' },
  'sub-gifter':{ emoji: '🎁', color: '#9147FF' },
  vip:         { emoji: '💎', color: '#E91E8C' },
  partner:     { emoji: '✅', color: '#9147FF' },
  premium:     { emoji: '💜', color: '#9147FF' },
  turbo:       { emoji: '⚡', color: '#FFD700' },
};

// Couleurs de secours pour les pseudos sans couleur Twitch
const USERNAME_COLORS = [
  '#FF6B6B','#FF9A3C','#FFD93D','#6BCB77',
  '#4D96FF','#C77DFF','#FF69B4','#00B4D8',
  '#F72585','#43AA8B','#277DA1','#F8961E',
];

let msgCount = 0;

// ── StreamElements ──────────────────────────────────

window.addEventListener('onWidgetLoad', ({ detail }) => {
  if (detail?.fieldData) applyFields(detail.fieldData);
  applyConfig();
});

window.addEventListener('onEventReceived', ({ detail }) => {
  if (!detail?.listener || !detail?.event) return;
  const { listener, event } = detail;

  if (listener === 'message') {
    addMessage(event.data);
  } else if (listener === 'subscriber-latest') {
    const { name, amount = 1, type = 'sub', gifted, sender, isCommunityGift } = event;
    if (gifted || isCommunityGift) {
      updateSubGoal(sender || name, +amount, 'giftsub');
    } else if (type === 'resub') {
      updateSubGoal(name, 1, 'resub');
    } else {
      updateSubGoal(name, 1, 'sub');
    }
  }
});

// ── Configuration ───────────────────────────────────

function applyFields(f) {
  if (f.maxMessages     !== undefined) CONFIG.maxMessages     = +f.maxMessages;
  if (f.messageDuration !== undefined) CONFIG.messageDuration = +f.messageDuration;
  if (f.theme)                          CONFIG.theme           = f.theme;
  if (f.streamName)                     CONFIG.streamName      = f.streamName;
  if (f.avatarUrl)                      CONFIG.avatarUrl       = f.avatarUrl;
  if (f.fontSize        !== undefined) CONFIG.fontSize         = +f.fontSize;
  if (f.showHeader      !== undefined) CONFIG.showHeader       = !!f.showHeader;
  if (f.showInputBar    !== undefined) CONFIG.showInputBar     = !!f.showInputBar;
  if (f.showBadges      !== undefined) CONFIG.showBadges       = !!f.showBadges;
  if (f.showTimestamp   !== undefined) CONFIG.showTimestamp    = !!f.showTimestamp;
  // Sub goal
  if (f.twitchChannel   !== undefined) CONFIG.twitchChannel   = f.twitchChannel;
  if (f.subGoalEnabled  !== undefined) CONFIG.subGoalEnabled  = !!f.subGoalEnabled;
  if (f.subGoalCurrent  !== undefined) CONFIG.subGoalCurrent  = +f.subGoalCurrent;
  if (f.subGoalTarget   !== undefined) CONFIG.subGoalTarget   = +f.subGoalTarget;
  if (f.subGoalLabel    !== undefined) CONFIG.subGoalLabel    = f.subGoalLabel;
  if (f.subGoalDesc     !== undefined) CONFIG.subGoalDesc     = f.subGoalDesc;
}

function applyConfig() {
  const root = document.getElementById('widget-container');
  if (!root) return;

  root.className = `theme-${CONFIG.theme}`;
  document.documentElement.style.setProperty('--font-size', `${CONFIG.fontSize}px`);

  const nameEl = document.getElementById('contact-name');
  if (nameEl) nameEl.textContent = CONFIG.streamName;

  if (CONFIG.avatarUrl) {
    const av = document.getElementById('wa-avatar');
    if (av) av.innerHTML = `<img src="${escHtml(CONFIG.avatarUrl)}" alt="avatar" />`;
  }

  const header   = document.getElementById('wa-header');
  const inputBar = document.getElementById('wa-input-bar');
  if (header)   header.style.display   = CONFIG.showHeader   ? '' : 'none';
  if (inputBar) inputBar.style.display = CONFIG.showInputBar ? '' : 'none';

  applySubGoal();

  // Connexion IRC directe (désactivée si StreamElements gère les events)
  if (!IS_SE && CONFIG.twitchChannel) {
    connectTwitch(CONFIG.twitchChannel);
  }
}

// ── Utilitaires ─────────────────────────────────────

function escHtml(t) {
  if (typeof t !== 'string') return '';
  return t.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

function formatTime() {
  return new Date().toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit' });
}

function getUserColor(username, twitchColor) {
  if (twitchColor && /^#[0-9A-Fa-f]{6}$/.test(twitchColor)) return twitchColor;
  let h = 0;
  const s = (username || '').toLowerCase();
  for (let i = 0; i < s.length; i++) h = s.charCodeAt(i) + ((h << 5) - h);
  return USERNAME_COLORS[Math.abs(h) % USERNAME_COLORS.length];
}

// ── Rendu des badges ────────────────────────────────

function renderBadges(badges) {
  if (!CONFIG.showBadges || !badges?.length) return '';
  return badges.map(b => {
    const meta = BADGE_META[b.type];
    if (!meta) return '';
    if (b.url) {
      return `<span class="badge" title="${escHtml(b.type)}"><img src="${escHtml(b.url)}" alt="${escHtml(b.type)}" /></span>`;
    }
    return `<span class="badge" style="background:${meta.color}" title="${escHtml(b.type)}">${meta.emoji}</span>`;
  }).join('');
}

// ── Rendu du texte / emotes ─────────────────────────

function renderText(text, emotes) {
  if (!text) return '';
  if (!emotes?.length) return escHtml(text);

  const sorted = [...emotes].sort((a, b) => (a.start ?? 0) - (b.start ?? 0));
  const parts  = [];
  let cursor   = 0;

  for (const em of sorted) {
    const start = em.start ?? 0;
    const end   = (em.end ?? em.start) + 1;
    if (start > cursor) parts.push(escHtml(text.slice(cursor, start)));

    const name = text.slice(start, end);
    const url  = em.urls?.['1'] || em.urls?.['1x'] || em.url || '';
    if (url) {
      parts.push(`<img class="emote" src="${escHtml(url)}" alt="${escHtml(name)}" title="${escHtml(name)}" loading="lazy" />`);
    } else {
      parts.push(escHtml(name));
    }
    cursor = end;
  }

  if (cursor < text.length) parts.push(escHtml(text.slice(cursor)));
  return parts.join('');
}

// ── Ajout d'un message ──────────────────────────────

function addMessage(data) {
  if (!data) return;
  const { displayName, username, text, emotes = [], badges = [], color } = data;

  const list = document.getElementById('messages-list');
  if (!list) return;

  const userColor  = getUserColor(username || displayName, color);
  const badgeHtml  = renderBadges(badges);
  const msgHtml    = renderText(text, emotes);
  const time       = CONFIG.showTimestamp ? formatTime() : '';

  const wrapper = document.createElement('div');
  wrapper.className  = 'message-wrapper';
  wrapper.dataset.id = ++msgCount;

  wrapper.innerHTML = `
    <div class="message-bubble">
      <div class="message-header">
        <span class="message-badges">${badgeHtml}</span>
        <span class="message-username" style="color:${userColor}">${escHtml(displayName || username || 'Anonyme')}</span>
      </div>
      <div class="message-text">${msgHtml}</div>
      ${time ? `<div class="message-footer">
        <span class="message-time">${time}</span>
        <span class="message-ticks">✓✓</span>
      </div>` : ''}
    </div>
  `;

  list.appendChild(wrapper);
  pruneMessages();

  if (CONFIG.messageDuration > 0) {
    setTimeout(() => removeMessage(wrapper), CONFIG.messageDuration * 1000);
  }
}

// ── Suppression avec animation ──────────────────────

function removeMessage(el) {
  if (!el || !el.parentNode) return;
  el.classList.add('removing');
  // Fallback si animationend ne se déclenche pas
  setTimeout(() => { if (el.parentNode) el.remove(); }, 350);
  el.addEventListener('animationend', () => el.remove(), { once: true });
}

// ── Écrêtage du nombre de messages ─────────────────

function pruneMessages() {
  const list = document.getElementById('messages-list');
  if (!list) return;
  const visible = [...list.querySelectorAll('.message-wrapper:not(.removing)')];
  const excess  = visible.length - CONFIG.maxMessages;
  for (let i = 0; i < excess; i++) visible[i].remove(); // silencieux (au-dessus du cadre)
}

// ── Connexion Twitch IRC (WebSocket anonyme) ─────────

let twitchWS            = null;
let twitchReconnectTimer = null;
let twitchReconnectDelay = 1500;
let twitchActiveChannel  = '';

function connectTwitch(channel) {
  const ch = channel.toLowerCase().replace(/^#/, '').trim();
  if (!ch) return;

  // Déjà connecté à ce channel
  if (ch === twitchActiveChannel && twitchWS?.readyState === WebSocket.OPEN) return;

  disconnectTwitch();
  twitchActiveChannel = ch;
  setConnectionStatus('connecting');

  const ws = new WebSocket('wss://irc-ws.chat.twitch.tv/');
  twitchWS = ws;

  ws.onopen = () => {
    ws.send('CAP REQ :twitch.tv/tags twitch.tv/commands');
    ws.send('PASS oauth:anonymous');
    ws.send(`NICK justinfan${Math.floor(Math.random() * 80000) + 10000}`);
    ws.send(`JOIN #${ch}`);
  };

  ws.onmessage = ({ data }) => {
    data.split('\r\n').forEach(line => { if (line) handleIRCLine(line, ch); });
  };

  ws.onerror = () => setConnectionStatus('error');

  ws.onclose = () => {
    if (twitchActiveChannel !== ch) return; // connexion volontairement changée
    setConnectionStatus('disconnected');
    twitchReconnectDelay = Math.min(twitchReconnectDelay * 2, 30000);
    twitchReconnectTimer = setTimeout(() => connectTwitch(ch), twitchReconnectDelay);
  };
}

function disconnectTwitch() {
  clearTimeout(twitchReconnectTimer);
  twitchActiveChannel = '';
  if (twitchWS) {
    twitchWS.onclose = null;
    twitchWS.close();
    twitchWS = null;
  }
  setConnectionStatus('disconnected');
}

function handleIRCLine(line, ch) {
  if (line.startsWith('PING')) {
    twitchWS?.send('PONG :tmi.twitch.tv');
    return;
  }

  // JOIN confirmé → connexion établie
  if (line.includes(`JOIN #${ch}`)) {
    twitchReconnectDelay = 1500;
    setConnectionStatus('connected');
    return;
  }

  // Regex générique : @tags :user!... COMMAND #channel :text
  const m = line.match(/^(?:@([^ ]+) )?:([^!]+)![^ ]+ ([A-Z]+) #\S+(?: :(.*))?$/);
  if (!m) return;

  const [, tagsStr = '', username, command, text = ''] = m;
  const tags = parseTags(tagsStr);

  if (command === 'PRIVMSG') {
    addMessage({
      displayName: tags['display-name'] || username,
      username,
      text,
      color:   tags['color'] || null,
      badges:  parseBadges(tags['badges']  || ''),
      emotes:  parseEmotes(tags['emotes']  || '', text),
    });
  } else if (command === 'USERNOTICE') {
    handleUserNotice(tags);
  }
}

function parseTags(str) {
  const t = {};
  str.split(';').forEach(p => {
    const i = p.indexOf('=');
    if (i !== -1) t[p.slice(0, i)] = p.slice(i + 1);
  });
  return t;
}

function parseBadges(str) {
  return str.split(',').filter(Boolean).map(b => ({ type: b.split('/')[0] }));
}

function parseEmotes(str, text) {
  if (!str) return [];
  const list = [];
  str.split('/').forEach(part => {
    if (!part) return;
    const [id, positions] = part.split(':');
    (positions || '').split(',').forEach(pos => {
      const [start, end] = pos.split('-').map(Number);
      if (!isNaN(start) && !isNaN(end)) {
        list.push({
          id, start, end,
          urls: {
            '1':  `https://static-cdn.jtvnw.net/emoticons/v2/${id}/default/dark/1.0`,
            '1x': `https://static-cdn.jtvnw.net/emoticons/v2/${id}/default/dark/1.0`,
          },
        });
      }
    });
  });
  return list;
}

function handleUserNotice(tags) {
  const msgId = tags['msg-id'] || '';
  const user  = tags['display-name'] || tags['login'] || 'Anonyme';
  if (msgId === 'sub')              updateSubGoal(user, 1, 'sub');
  else if (msgId === 'resub')       updateSubGoal(user, 1, 'resub');
  else if (msgId === 'subgift' || msgId === 'anonsubgift') updateSubGoal(user, 1, 'giftsub');
  else if (msgId === 'submysterygift') {
    updateSubGoal(user, parseInt(tags['msg-param-mass-gift-count'] || '1'), 'giftsub');
  }
}

function setConnectionStatus(status) {
  const dot = document.getElementById('connection-dot');
  if (dot) {
    dot.className  = `wa-online-dot conn-${status}`;
    dot.title = {
      connected:    `Connecté à #${twitchActiveChannel}`,
      connecting:   'Connexion en cours…',
      disconnected: 'Non connecté',
      error:        'Erreur de connexion',
    }[status] ?? '';
  }
  // Notifie le viewer parent (même origine)
  try { window.parent.postMessage({ type: 'twitchStatus', status, channel: twitchActiveChannel }, '*'); } catch (_) {}
}

// ── Objectif d'abonnements ──────────────────────────

function applySubGoal() {
  const el = document.getElementById('sub-goal');
  if (!el) return;

  if (!CONFIG.subGoalEnabled) {
    el.classList.remove('goal-visible');
    return;
  }

  el.classList.add('goal-visible');

  const pct = CONFIG.subGoalTarget > 0
    ? Math.min(100, (CONFIG.subGoalCurrent / CONFIG.subGoalTarget) * 100)
    : 0;
  const complete = CONFIG.subGoalCurrent >= CONFIG.subGoalTarget;

  document.getElementById('goal-label').textContent    = CONFIG.subGoalLabel || 'Objectif abonnements';
  document.getElementById('goal-fraction').textContent = `${CONFIG.subGoalCurrent} / ${CONFIG.subGoalTarget}`;
  document.getElementById('goal-fill').style.width     = `${pct}%`;

  const descEl = document.getElementById('goal-desc');
  descEl.textContent = CONFIG.subGoalDesc || '';
  descEl.style.display = CONFIG.subGoalDesc ? '' : 'none';

  el.classList.toggle('goal-complete', complete);
}

function updateSubGoal(username, amount = 1, type = 'sub') {
  if (!CONFIG.subGoalEnabled) return;

  const wasComplete = CONFIG.subGoalCurrent >= CONFIG.subGoalTarget;
  CONFIG.subGoalCurrent = Math.min(CONFIG.subGoalCurrent + amount, CONFIG.subGoalTarget);

  // Animation de pulse
  const el = document.getElementById('sub-goal');
  if (el) {
    el.classList.remove('sub-pulse');
    void el.offsetWidth;
    el.classList.add('sub-pulse');
    setTimeout(() => el.classList.remove('sub-pulse'), 500);
  }

  applySubGoal();

  // Message système dans le chat
  let chip;
  if (type === 'giftsub') {
    chip = amount > 1
      ? `🎁 ${escHtml(username)} offre ${amount} abonnements !`
      : `🎁 ${escHtml(username)} offre un abonnement !`;
  } else if (type === 'resub') {
    chip = `⭐ ${escHtml(username)} se réabonne !`;
  } else {
    chip = `⭐ ${escHtml(username)} vient de s'abonner !`;
  }
  addSystemMessage(chip);

  // Objectif atteint
  const nowComplete = CONFIG.subGoalCurrent >= CONFIG.subGoalTarget;
  if (!wasComplete && nowComplete) {
    setTimeout(() => {
      addSystemMessage(`🎉 Objectif atteint ! ${CONFIG.subGoalTarget} abonnements !`);
      spawnConfetti();
    }, 700);
  }
}

function addSystemMessage(text) {
  const list = document.getElementById('messages-list');
  if (!list) return;
  const wrapper = document.createElement('div');
  wrapper.className = 'message-wrapper system-msg';
  wrapper.innerHTML = `<div class="system-chip">${text}</div>`;
  list.appendChild(wrapper);
  pruneMessages();
}

function spawnConfetti() {
  const container = document.getElementById('messages-container');
  if (!container) return;
  const colors = ['#25D366','#00A884','#FFD700','#FF6B6B','#4D96FF','#C77DFF','#FF9A3C'];
  for (let i = 0; i < 14; i++) {
    setTimeout(() => {
      const c = document.createElement('div');
      c.className = 'confetti-piece';
      c.style.cssText = `
        left:${15 + Math.random() * 70}%;
        bottom:10px;
        background:${colors[Math.floor(Math.random() * colors.length)]};
        animation-delay:${Math.random() * .25}s;
        animation-duration:${.65 + Math.random() * .45}s;
      `;
      container.appendChild(c);
      c.addEventListener('animationend', () => c.remove(), { once: true });
    }, i * 55);
  }
}

// ── Mode test (hors StreamElements) ─────────────────

const IS_SE = typeof window.StreamElements !== 'undefined'
           || window.location.href.includes('streamelements.com');

// Toujours initialiser la config au chargement du DOM
// (onWidgetLoad refait applyConfig() avec les vraies valeurs en mode SE)
document.addEventListener('DOMContentLoaded', () => {
  // En accès direct (pas dans un iframe viewer, pas dans SE) :
  // on précharge des valeurs de démo pour que le sub goal soit visible
  const IS_IFRAME = window !== window.top;
  if (!IS_SE && !IS_IFRAME) {
    CONFIG.subGoalEnabled = true;
    CONFIG.subGoalCurrent = 45;
    CONFIG.subGoalTarget  = 100;
    CONFIG.subGoalDesc    = 'Facecam à 100 subs !';
  }
  applyConfig();
});

if (!IS_SE) {
  const SAMPLES = [
    { displayName: 'StreamFan42',   username: 'streamfan42',   text: 'Salut tout le monde ! 🎮',                     badges: [{ type: 'subscriber' }],                          color: '#FF6B6B' },
    { displayName: 'ModPuissant',   username: 'modpuissant',   text: 'Bienvenue dans le chat !',                      badges: [{ type: 'moderator' }],                           color: '#34AE1A' },
    { displayName: 'CasualViewer',  username: 'casualviewer',  text: 'Le jeu est vraiment stylé ce soir 🔥',          badges: [],                                                color: null     },
    { displayName: 'VIP_Legend',    username: 'vip_legend',    text: 'J\'adore ce stream, continuez comme ça !',      badges: [{ type: 'vip' }],                                 color: '#FF69B4' },
    { displayName: 'Broadcaster',   username: 'broadcaster',   text: 'Merci pour l\'abonnement ! <3',                 badges: [{ type: 'broadcaster' }],                         color: '#FFD700' },
    { displayName: 'SubGifter99',   username: 'subgifter99',   text: 'Je viens d\'offrir 5 abonnements !',            badges: [{ type: 'sub-gifter' }, { type: 'subscriber' }],  color: '#C77DFF' },
    { displayName: 'TwitchUser',    username: 'twitchuser',    text: 'Est-ce que quelqu\'un connaît ce jeu ?',        badges: [],                                                color: null     },
    { displayName: 'NightBot',      username: 'nightbot',      text: '!commandes pour voir les commandes disponibles',badges: [{ type: 'moderator' }],                           color: '#1B9AF5' },
  ];

  let idx = 0;
  const tick = () => {
    addMessage(SAMPLES[idx % SAMPLES.length]);
    idx++;
    setTimeout(tick, 1800 + Math.random() * 2200);
  };

  document.addEventListener('DOMContentLoaded', () => setTimeout(tick, 600));
}
