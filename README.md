# Sample Code Files from TeleTask

This document contains five key source files extracted from the TeleTask codebase:
1. `src/pages/Editor.jsx` â€” Post editor UI with rich blocks, preview, and channel selection.
2. `src/pages/Channels.jsx` â€” Channel management and connection list.
3. `src/pages/Posts.jsx` â€” Overview of drafts, scheduled, and published posts by channel.
4. `base44/functions/telegramPublishPost/entry.ts` â€” Backend function handling immediate and scheduled post delivery to Telegram.
5. `base44/entities/Post.jsonc` â€” Schema definition for post entities.

---

## `schema.js`

```js
// Rich Bot database schema — Telegram Serverless (sdk/db DSL).
import { table, integer, text, boolean, json, sql } from 'sdk/db';

// One row per Telegram user: the in-progress editor session.
export const richSessions = table('rich_sessions', {
  userId: integer('user_id').primaryKey(),
  state: json('state'),
  updatedAt: text('updated_at'),
});

// Saved pages ("My Pages").
export const richPages = table('rich_pages', {
  id: text('id').primaryKey(),
  userId: integer('user_id').notNull(),
  title: text('title'),
  isRtl: boolean('is_rtl'),
  blocks: json('blocks'),
  buttons: json('buttons'),
  createdAt: integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
  updatedAt: text('updated_at'),
});

// Alert/toast texts for {Label - alert: ...} buttons inside sent messages.
export const richAlerts = table('rich_alerts', {
  id: text('id').primaryKey(),
  text: text('text'),
  showAlert: boolean('show_alert'),
});

```

---



## `lib/bot.py`

```js
// Rich Bot — the full editor flow, ported 1:1 from the Lovable version.
import * as kb from "./rich/keyboards.js";
import { buildTextBlock, parseButtons } from "./rich/parse.js";
import { renderHtml } from "./rich/render.js";
import { pagesRichBlocks, toWireBlocks } from "./rich/wire.js";
import { getSession, newDraft, openPage, syncDraft, clone } from "./rich/state.js";
import { getAlert, loadSession, saveSession } from "./db.js";
import { BLOCK_LABELS, PROMPTS } from "./rich/types.js";
import { tgCall } from "./telegram.js";

const esc = (s) => s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

const CAPTION_TYPES = ["photo", "video", "animation", "audio", "voice_note", "document", "collage", "slideshow"];
function captionState(b) {
  if (!CAPTION_TYPES.includes(b.type)) return undefined;
  return b.caption ? "has" : "none";
}

const MEDIA_TYPES = ["photo", "video", "animation", "audio", "voice_note", "document"];

/** Sends text, or edits the given message in place when possible. */
async function show(chatId, text, markup, editId) {
  if (editId) {
    try {
      await tgCall("editMessageText", { chat_id: chatId, message_id: editId, text, parse_mode: "HTML", reply_markup: markup });
      return;
    } catch {
      /* fall through to a fresh message */
    }
  }
  await tgCall("sendMessage", { chat_id: chatId, text, parse_mode: "HTML", reply_markup: markup });
}

function editorText(s, note = "") {
  const d = s.draft;
  const btnCount = d.buttons.flat().length;
  return `${note}<b>📝 ${esc(d.title)}</b>\nBlocks: ${d.blocks.length} · Buttons: ${btnCount}\n\n` +
    (d.blocks.length ? "Tap a block to move, edit or delete it." : "Your page is empty. Tap ➕ Add Block to start.\n\n<i>Tip: forward any rich message here to import it.</i>");
}

async function showEditor(chatId, s, editId, note = "") {
  await show(chatId, editorText(s, note), kb.editorKeyboard(s), editId);
}

/** Root blocks, or the open Details block's inner blocks when `inner`. */
function targetArr(s, inner) {
  if (!inner) return s.draft.blocks;
  const det = s.draft.blocks[s.detailsIndex ?? -1];
  return det && det.type === "details" ? det.inner_blocks : [];
}

async function showDetails(chatId, s, editId, note = "") {
  const det = s.draft.blocks[s.detailsIndex];
  if (!det || det.type !== "details") {
    s.detailsIndex = undefined;
    return showEditor(chatId, s, editId);
  }
  await show(
    chatId,
    `${note}<b>📁 Details:</b> ${esc(det.summary)}\nInner blocks: ${det.inner_blocks.length}\n\nAdd the content that appears when the reader expands it.`,
    kb.detailsKeyboard(s),
    editId,
  );
}

function toMarkup(rows) {
  return rows.length ? { inline_keyboard: rows } : undefined;
}

const DEMO_BLOCKS = [
  { type: "section_heading", text: "✨ Rich Message Showcase" },
  { type: "paragraph", text: "This message is built from native Telegram blocks. The buttons below sit in the middle of the message, not at the end." },
  { type: "buttons", buttons: [[
    { text: "Open Docs", url: "https://core.telegram.org/bots/api", style: "primary" },
    { text: "Tap me", callback_data: "demo_tap", style: "success" },
  ]] },
  { type: "pull_quotation", text: "Buttons can live anywhere in the flow.", credit: "Rich Bot" },
  { type: "mathematical_expression", expression: "E = mc^2" },
  { type: "details", summary: "Tap to expand", inner_blocks: [
    { type: "paragraph", text: "Hidden content — with its own button inside:" },
    { type: "buttons", buttons: [[{ text: "Inside details", callback_data: "demo_tap", style: "danger" }]] },
  ] },
  { type: "footer", text: "Sent with sendRichMessage" },
];

/** Sends a rich message via sendRichMessage, falling back to HTML sendMessage. */
async function sendRich(chatId, blocks, isRtl, buttons) {
  const reply_markup = toMarkup(buttons);
  try {
    await tgCall("sendRichMessage", { chat_id: chatId, rich_message: { blocks: toWireBlocks(blocks), is_rtl: isRtl }, reply_markup });
    return "rich";
  } catch (error) {
    console.error("sendRichMessage failed, using HTML fallback:", error);
    const html = renderHtml(blocks) || "(empty)";
    await tgCall("sendMessage", { chat_id: chatId, text: html.slice(0, 4096), parse_mode: "HTML", reply_markup });
    return "fallback";
  }
}

/** My Pages as a rich message, with an HTML/keyboard fallback. */
async function showPagesRich(chatId, s) {
  try {
    await tgCall("sendRichMessage", { chat_id: chatId, rich_message: { blocks: pagesRichBlocks(s.pages), is_rtl: false } });
  } catch (e) {
    console.error("sendRichMessage (pages) failed, using fallback:", e);
    await show(chatId, s.pages.length ? "📚 <b>Your saved pages</b>" : "📚 No saved pages yet. Tap 🔄 Sync in the editor to save one.", kb.pagesKeyboard(s, 0));
  }
}


function addBlock(s, block, inner) {
  if (inner && s.detailsIndex !== undefined) {
    const det = s.draft.blocks[s.detailsIndex];
    if (det && det.type === "details") det.inner_blocks.push(block);
  } else {
    s.draft.blocks.push(block);
  }
  s.draft.dirty = true;
}

async function afterAdd(chatId, s, inner, label) {
  s.pending = undefined;
  if (inner) await showDetails(chatId, s, undefined, `✅ ${label} added.\n\n`);
  else await showEditor(chatId, s, undefined, `✅ ${label} added.\n\n`);
}

async function startBlock(chatId, s, type, inner, editId) {
  if (type === "divider") {
    addBlock(s, { type: "divider" }, inner);
    return inner ? showDetails(chatId, s, editId, "✅ Divider added.\n\n") : showEditor(chatId, s, editId, "✅ Divider added.\n\n");
  }
  if (type === "details") {
    s.pending = { kind: "details_summary", inner: false };
  } else if (type === "collage" || type === "slideshow") {
    s.pending = { kind: "media_group", type, inner, items: [] };
    return show(chatId, `<b>${BLOCK_LABELS[type]}</b>\n${PROMPTS[type]}`, kb.mediaDoneKeyboard(), editId);
  } else {
    s.pending = { kind: "block", type, inner };
  }
  await show(chatId, `<b>${BLOCK_LABELS[type]}</b>\n${PROMPTS[type]}`, kb.cancelKeyboard(), editId);
}

function mediaFromMessage(type, m) {
  const caption = m.caption;
  const photo = m.photo && m.photo[m.photo.length - 1] ? m.photo[m.photo.length - 1].file_id : undefined;
  switch (type) {
    case "photo": return photo ? { type, photo, caption } : null;
    case "video": return m.video ? { type, video: m.video.file_id, caption } : null;
    case "animation": return m.animation ? { type, animation: m.animation.file_id, caption } : null;
    case "audio": return m.audio ? { type, audio: m.audio.file_id, caption } : null;
    case "voice_note": return m.voice ? { type, voice: m.voice.file_id, caption } : null;
    case "document": return m.document ? { type, document: m.document.file_id, caption } : null;
    case "map": return m.location ? { type, latitude: m.location.latitude, longitude: m.location.longitude, zoom: 15 } : null;
    default: return null;
  }
}

async function handlePending(m, s) {
  const p = s.pending;
  if (!p) return false;
  const chatId = m.chat.id;
  const text = m.text ?? "";

  switch (p.kind) {
    case "block": {
      let block = (MEDIA_TYPES.includes(p.type) || (p.type === "map" && m.location)) ? mediaFromMessage(p.type, m) : null;
      if (!block) block = text ? buildTextBlock(p.type, text) : `Please send ${MEDIA_TYPES.includes(p.type) ? "a file" : "text"}.`;
      if (typeof block === "string") {
        await show(chatId, `⚠️ ${esc(block)}`, kb.cancelKeyboard());
        return true;
      }
      addBlock(s, block, p.inner);
      await afterAdd(chatId, s, p.inner, BLOCK_LABELS[p.type]);
      return true;
    }
    case "details_summary": {
      if (!text) return true;
      s.draft.blocks.push({ type: "details", summary: text.slice(0, 256), inner_blocks: [] });
      s.draft.dirty = true;
      s.detailsIndex = s.draft.blocks.length - 1;
      s.pending = undefined;
      await showDetails(chatId, s);
      return true;
    }
    case "media_group": {
      const photo = m.photo && m.photo[m.photo.length - 1] ? m.photo[m.photo.length - 1].file_id : undefined;
      if (photo) p.items.push({ type: "photo", media: photo });
      else if (m.video) p.items.push({ type: "video", media: m.video.file_id });
      else {
        await show(chatId, "Send a photo or video, or tap ✅ Done.", kb.mediaDoneKeyboard());
        return true;
      }
      await show(chatId, `Added item ${p.items.length}. Send more or tap ✅ Done.`, kb.mediaDoneKeyboard());
      return true;
    }
    case "edit_block": {
      const arr = targetArr(s, p.inner);
      const old = arr[p.index];
      if (!old) break;
      const block = mediaFromMessage(old.type, m) ?? (text ? buildTextBlock(old.type, text) : "Send new content.");
      if (typeof block === "string") {
        await show(chatId, `⚠️ ${esc(block)}`, kb.cancelKeyboard());
        return true;
      }
      arr[p.index] = block;
      s.draft.dirty = true;
      s.pending = undefined;
      if (p.inner) await showDetails(chatId, s, undefined, "✅ Inner block updated.\n\n");
      else await showEditor(chatId, s, undefined, "✅ Block updated.\n\n");
      return true;
    }
    case "caption": {
      const blk = targetArr(s, p.inner)[p.index];
      if (!blk || !text) return true;
      blk.caption = text.slice(0, 1024);
      s.draft.dirty = true;
      s.pending = undefined;
      if (p.inner) await showDetails(chatId, s, undefined, "✅ Caption saved.\n\n");
      else await showEditor(chatId, s, undefined, "✅ Caption saved.\n\n");
      return true;
    }
    case "rename_page":
    case "rename_draft": {
      if (!text) return true;
      const name = text.trim().slice(0, 64);
      if (p.kind === "rename_page") {
        const pg = s.pages.find((x) => x.id === p.pageId);
        if (pg) pg.title = name;
        if (s.draft.savedId === p.pageId) s.draft.title = name;
        s.pending = undefined;
        await showPagesRich(chatId, s);
      } else {
        s.draft.title = name;
        s.draft.dirty = true;
        s.pending = undefined;
        await showEditor(chatId, s, undefined, "✅ Renamed.\n\n");
      }
      return true;
    }
    case "buttons": {
      const rows = text.trim() === "-" ? [] : parseButtons(text);
      if (text.trim() !== "-" && !rows.length) {
        await show(chatId, "⚠️ No buttons found. Use <code>{Label - https://link #primary}</code>, or send <code>-</code> to clear.", kb.cancelKeyboard());
        return true;
      }
      s.draft.buttons = rows;
      s.draft.dirty = true;
      s.pending = undefined;
      await showEditor(chatId, s, undefined, `✅ ${rows.flat().length} message button(s) set.\n\n`);
      return true;
    }
    case "import_json": {
      try {
        const data = JSON.parse(text);
        const blocks = Array.isArray(data) ? data : data.blocks;
        if (!Array.isArray(blocks)) throw new Error("no blocks array");
        s.draft.blocks = blocks;
        if (typeof data.is_rtl === "boolean") s.draft.is_rtl = data.is_rtl;
        if (Array.isArray(data.buttons)) s.draft.buttons = data.buttons;
        if (typeof data.title === "string") s.draft.title = data.title.slice(0, 64);
        s.draft.dirty = true;
        s.pending = undefined;
        await showEditor(chatId, s, undefined, "📥 Imported.\n\n");
      } catch (e) {
        await show(chatId, `⚠️ Invalid JSON: ${esc(String(e))}`, kb.cancelKeyboard());
      }
      return true;
    }
    case "channel": {
      const ch = text.trim();
      if (!/^(@[A-Za-z0-9_]{4,}|-?\d+)$/.test(ch)) {
        await show(chatId, "⚠️ Send a channel like <code>@mychannel</code> or a numeric chat ID.", kb.cancelKeyboard());
        return true;
      }
      s.channel = ch;
      s.pending = undefined;
      await show(chatId, `✅ Channel set to ${esc(ch)}. Make sure the bot is an admin there.`, kb.toolsKeyboard(s));
      return true;
    }
  }
  return false;
}

async function handleMessage(m) {
  const chatId = m.chat.id;
  const userId = (m.from && m.from.id) ?? chatId;
  const s = getSession(userId);
  const text = (m.text ?? "").trim();
  const command = text.startsWith("/") ? (((text.split(/\s+/)[0] ?? "").split("@")[0]) ?? "").toLowerCase() : "";

  if (command) s.pending = undefined;

  switch (command) {
    case "/start":
      return show(chatId,
        "<b>Welcome to Rich Bot!</b>\n\nBuild native Telegram Rich Messages block by block: headings, tables, formulas, details, galleries and more.\n\n/editor — open the page builder\n/pages — your saved pages\n/new — start a new page\n/help — how it works",
        kb.homeKeyboard());
    case "/help":
      return show(chatId,
        "<b>How it works</b>\n1. /editor and tap ➕ Add Block\n2. Pick a block type and send its content\n3. 👁 Preview sends it with <code>sendRichMessage</code>\n4. 🔄 Sync saves it to 📚 My Pages\n5. 🛠 More tools: rename, buttons, JSON export/import, send to a channel\n\n<b>Button syntax</b>\n<code>{Open site - https://example.com #primary}</code>\n\n<b>Import</b>: forward any rich message here.",
        kb.homeKeyboard());
    case "/editor":
      return showEditor(chatId, s);
    case "/pages":
      return showPagesRich(chatId, s);
    case "/new":
      s.draft = newDraft();
      s.detailsIndex = undefined;
      return showEditor(chatId, s, undefined, "🧹 New page.\n\n");
    case "/cancel":
      return showEditor(chatId, s, undefined, "Cancelled.\n\n");
    case "/rich":
    case "/demo": {
      const r = await sendRich(chatId, DEMO_BLOCKS, false, []);
      if (r === "fallback") await show(chatId, "⚠️ Telegram rejected the rich message — showed a plain version instead.", kb.homeKeyboard());
      return;
    }
  }

  if (await handlePending(m, s)) return;

  // Direct add: media sent or forwarded is appended automatically as its block type.
  if (s.directAdd) {
    const type = m.video ? "video" : m.document ? "document" : (m.photo && m.photo.length) ? "photo"
      : m.animation ? "animation" : m.audio ? "audio" : m.voice ? "voice_note" : m.location ? "map" : null;
    const block = type ? mediaFromMessage(type, m) : null;
    if (block && type) {
      const inner = s.detailsIndex !== undefined;
      addBlock(s, block, inner);
      if (inner) return showDetails(chatId, s, undefined, `✅ ${BLOCK_LABELS[type]} added inside details.\n\n`);
      return showEditor(chatId, s, undefined, `✅ ${BLOCK_LABELS[type]} added automatically.\n\n`);
    }
  }

  // Forwarded rich message → JSON import is paused for now.

  await show(chatId, "Use /editor to build a rich page, or /help.", kb.homeKeyboard());
}

async function handleCallback(cb) {
  const msg = cb.message;
  if (!msg) return;
  const chatId = msg.chat.id;
  const mid = msg.message_id;
  const s = getSession(cb.from.id);
  const parts = (cb.data ?? "").split(/:(.*)/s);
  const action = parts[0];
  const arg = parts[1] ?? "";
  const idx = Number(arg);
  let toast;
  let alert = false;

  switch (action) {
    case "noop": break;
    case "fwd_edit":
    case "fwd_save": {
      if (!s.forwarded) { toast = "Forward the message again."; break; }
      s.draft = { ...newDraft(), title: "Imported Page", blocks: clone(s.forwarded.blocks), is_rtl: s.forwarded.is_rtl, dirty: true };
      s.detailsIndex = undefined;
      if (action === "fwd_save") { syncDraft(s); toast = "💾 Saved to My Pages"; }
      await showEditor(chatId, s, undefined, action === "fwd_save" ? "💾 Saved.\n\n" : "📥 Imported.\n\n");
      break;
    }
    case "demo": {
      const r = await sendRich(chatId, DEMO_BLOCKS, false, []);
      toast = r === "rich" ? "Sent!" : "Rich message rejected — see logs.";
      break;
    }
    case "demo_tap":
      toast = "👋 You tapped a button from inside the message!";
      alert = true;
      break;
    case "editor":
      s.pending = undefined;
      s.detailsIndex = undefined;
      await showEditor(chatId, s, mid);
      break;
    case "cancel":
      s.pending = undefined;
      if (s.detailsIndex !== undefined) await showDetails(chatId, s, mid);
      else await showEditor(chatId, s, mid);
      break;
    case "add":
      await show(chatId, "<b>Choose a block type</b>\n\n<i>Direct add ON: just send or forward a video, document, photo, audio or location and it's added instantly.</i>", kb.pickerKeyboard(false, !!s.directAdd), mid);
      break;
    case "direct":
      s.directAdd = !s.directAdd;
      toast = s.directAdd ? "Direct add ON" : "Direct add OFF";
      await show(chatId, "<b>Choose a block type</b>\n\n<i>Direct add ON: just send or forward a video, document, photo, audio or location and it's added instantly.</i>", kb.pickerKeyboard(false, s.directAdd), mid);
      break;
    case "pick":
      s.detailsIndex = undefined;
      await startBlock(chatId, s, arg, false, mid);
      break;
    case "ipick":
      await startBlock(chatId, s, arg, true, mid);
      break;
    case "iadd":
      await show(chatId, "<b>Choose an inner block</b>", kb.pickerKeyboard(true), mid);
      break;
    case "details":
      s.pending = undefined;
      if (s.detailsIndex !== undefined && arg === "back") {
        const back = s.detailsIndex;
        s.detailsIndex = undefined;
        if (cb.data === "details:back" && ((cb.message && cb.message.text) ?? "").startsWith("Choose an inner")) {
          s.detailsIndex = back;
          await showDetails(chatId, s, mid);
          break;
        }
      }
      await showEditor(chatId, s, mid);
      break;
    case "dopen":
      s.detailsIndex = idx;
      await showDetails(chatId, s, mid);
      break;
    case "dtoggle": {
      const det = s.draft.blocks[s.detailsIndex ?? -1];
      if (det && det.type === "details") { det.is_open = !det.is_open; s.draft.dirty = true; }
      await showDetails(chatId, s, mid);
      break;
    }
    case "idel": {
      const det = s.draft.blocks[s.detailsIndex ?? -1];
      if (det && det.type === "details") { det.inner_blocks.splice(idx, 1); s.draft.dirty = true; }
      await showDetails(chatId, s, mid);
      break;
    }
    case "iblk": {
      const arr = targetArr(s, true);
      const blk = arr[idx];
      if (!blk) break;
      await show(chatId, `📁 › <b>${BLOCK_LABELS[blk.type]} #${idx + 1}</b>\n<pre>${esc(JSON.stringify(blk, null, 1).slice(0, 3000))}</pre>`,
        kb.innerBlockKeyboard(idx, arr.length, captionState(blk)), mid);
      break;
    }
    case "iup":
    case "idown": {
      const arr = targetArr(s, true);
      const j = action === "iup" ? idx - 1 : idx + 1;
      if (arr[idx] && arr[j]) { [arr[idx], arr[j]] = [arr[j], arr[idx]]; s.draft.dirty = true; }
      await showDetails(chatId, s, mid);
      break;
    }
    case "idup": {
      const arr = targetArr(s, true);
      if (arr[idx]) { arr.splice(idx + 1, 0, clone(arr[idx])); s.draft.dirty = true; }
      await showDetails(chatId, s, mid);
      break;
    }
    case "iedit": {
      const blk = targetArr(s, true)[idx];
      if (!blk) break;
      s.pending = { kind: "edit_block", index: idx, inner: true };
      await show(chatId, `✏️ <b>Replace inner ${BLOCK_LABELS[blk.type]} #${idx + 1}</b>\n${PROMPTS[blk.type]}`, kb.cancelKeyboard(), mid);
      break;
    }
    case "icap": {
      const blk = targetArr(s, true)[idx];
      if (!blk || !captionState(blk)) break;
      s.pending = { kind: "caption", index: idx, inner: true };
      await show(chatId, `💬 <b>Send the caption for inner ${BLOCK_LABELS[blk.type]} #${idx + 1}</b>`, kb.cancelKeyboard(), mid);
      break;
    }
    case "icapdel": {
      const blk = targetArr(s, true)[idx];
      if (!blk) break;
      delete blk.caption;
      s.draft.dirty = true;
      toast = "Caption removed";
      await showDetails(chatId, s, mid);
      break;
    }
    case "iback":
      await showDetails(chatId, s, mid);
      break;
    case "cbd": {
      const pg = s.pages.find((x) => x.id === arg);
      toast = pg ? `Page: ${pg.title}` : "That page isn't available.";
      break;
    }
    case "mdone": {
      const p = s.pending;
      if (p && p.kind === "media_group") {
        if (p.items.length < 2) { toast = "Add at least 2 items."; alert = true; break; }
        addBlock(s, { type: p.type, items: p.items }, p.inner);
        await afterAdd(chatId, s, p.inner, BLOCK_LABELS[p.type]);
      }
      break;
    }
    case "blk": {
      const blk = s.draft.blocks[idx];
      if (!blk) break;
      await show(chatId, `<b>${BLOCK_LABELS[blk.type]} #${idx + 1}</b>\n<pre>${esc(JSON.stringify(blk, null, 1).slice(0, 3000))}</pre>`,
        kb.blockKeyboard(idx, s.draft.blocks.length, blk.type === "details", captionState(blk)), mid);
      break;
    }
    case "up":
    case "down": {
      const j = action === "up" ? idx - 1 : idx + 1;
      const arr = s.draft.blocks;
      if (arr[idx] && arr[j]) { [arr[idx], arr[j]] = [arr[j], arr[idx]]; s.draft.dirty = true; }
      await showEditor(chatId, s, mid);
      break;
    }
    case "dup":
      if (s.draft.blocks[idx]) { s.draft.blocks.splice(idx + 1, 0, clone(s.draft.blocks[idx])); s.draft.dirty = true; }
      await showEditor(chatId, s, mid);
      break;
    case "del":
      s.draft.blocks.splice(idx, 1);
      s.draft.dirty = true;
      await showEditor(chatId, s, mid, "🗑 Block deleted.\n\n");
      break;
    case "edit": {
      const blk = s.draft.blocks[idx];
      if (!blk) break;
      s.pending = { kind: "edit_block", index: idx };
      await show(chatId, `✏️ <b>Replace ${BLOCK_LABELS[blk.type]} #${idx + 1}</b>\n${PROMPTS[blk.type]}`, kb.cancelKeyboard(), mid);
      break;
    }
    case "cap": {
      const blk = s.draft.blocks[idx];
      if (!blk || !captionState(blk)) break;
      s.pending = { kind: "caption", index: idx };
      await show(chatId, `💬 <b>Send the caption for ${BLOCK_LABELS[blk.type]} #${idx + 1}</b>`, kb.cancelKeyboard(), mid);
      break;
    }
    case "capdel": {
      const blk = s.draft.blocks[idx];
      if (!blk) break;
      delete blk.caption;
      s.draft.dirty = true;
      toast = "Caption removed";
      await showEditor(chatId, s, mid);
      break;
    }
    case "dir":
      s.draft.is_rtl = !s.draft.is_rtl;
      s.draft.dirty = true;
      await showEditor(chatId, s, mid);
      break;
    case "sync":
      if (!s.draft.blocks.length) { toast = "Add at least one block first!"; alert = true; break; }
      syncDraft(s);
      toast = "Page saved";
      await showEditor(chatId, s, mid);
      break;
    case "preview": {
      if (!s.draft.blocks.length) { toast = "Please add at least one block first!"; alert = true; break; }
      const mode = await sendRich(chatId, s.draft.blocks, s.draft.is_rtl, s.draft.buttons);
      await showEditor(chatId, s, undefined, mode === "rich" ? "✅ Preview is ready.\n\n" : "⚠️ sendRichMessage was rejected — showed an HTML preview instead.\n\n");
      break;
    }
    case "tools":
      await show(chatId, "🛠 <b>More tools</b>", kb.toolsKeyboard(s), mid);
      break;
    case "rename":
      s.pending = { kind: "rename_draft" };
      await show(chatId, `Send a new name for “${esc(s.draft.title)}”; maximum 64 characters.`, kb.cancelKeyboard(), mid);
      break;
    case "msgbtn":
      s.pending = { kind: "buttons" };
      await show(chatId, `🔘 <b>Message buttons</b>\n${PROMPTS.buttons}\nSend <code>-</code> to remove all.`, kb.cancelKeyboard(), mid);
      break;
    case "export":
      await tgCall("sendMessage", {
        chat_id: chatId,
        text: `<pre><code class="language-json">${esc(JSON.stringify({ title: s.draft.title, is_rtl: s.draft.is_rtl, blocks: s.draft.blocks, buttons: s.draft.buttons }, null, 2)).slice(0, 4000)}</code></pre>`,
        parse_mode: "HTML",
      });
      break;
    case "import":
      s.pending = { kind: "import_json" };
      await show(chatId, "📥 Paste the page JSON (from Copy code).", kb.cancelKeyboard(), mid);
      break;
    case "channel":
      s.pending = { kind: "channel" };
      await show(chatId, "📢 Send the channel username (e.g. <code>@mychannel</code>) or chat ID. The bot must be an admin there.", kb.cancelKeyboard(), mid);
      break;
    case "broadcast":
      if (!s.channel || !s.draft.blocks.length) { toast = "Nothing to send."; alert = true; break; }
      try {
        await sendRich(s.channel, s.draft.blocks, s.draft.is_rtl, s.draft.buttons);
        toast = `Sent to ${s.channel}`;
      } catch (e) {
        toast = `Failed: ${String(e).slice(0, 150)}`;
        alert = true;
      }
      break;
    case "new":
      s.draft = newDraft();
      s.detailsIndex = undefined;
      await showEditor(chatId, s, mid, "🧹 New page.\n\n");
      break;
    case "pages":
      if (msg.text && msg.text.startsWith("Delete")) await tgCall("deleteMessage", { chat_id: chatId, message_id: mid }).catch(() => {});
      await showPagesRich(chatId, s);
      break;
    case "popen":
      if (openPage(s, arg)) await showEditor(chatId, s, mid, "✏️ Page opened.\n\n");
      break;
    case "pren": {
      const pg = s.pages.find((p) => p.id === arg);
      if (!pg) break;
      s.pending = { kind: "rename_page", pageId: arg };
      await show(chatId, `Send a new name for “${esc(pg.title)}”; maximum 64 characters.`, kb.cancelKeyboard(), mid);
      break;
    }
    case "pcode": {
      const pg = s.pages.find((p) => p.id === arg);
      if (pg) await tgCall("sendMessage", { chat_id: chatId, text: `<pre><code class="language-json">${esc(JSON.stringify(pg, null, 2)).slice(0, 4000)}</code></pre>`, parse_mode: "HTML" });
      break;
    }
    case "psend": {
      const pg = s.pages.find((p) => p.id === arg);
      if (pg) await sendRich(chatId, pg.blocks, pg.is_rtl, pg.buttons);
      break;
    }
    case "pdel": {
      const pg = s.pages.find((p) => p.id === arg);
      if (pg) await tgCall("sendMessage", { chat_id: chatId, text: `Delete “${esc(pg.title)}”?`, parse_mode: "HTML", reply_markup: kb.confirmDeleteKeyboard(arg) });
      break;
    }
    case "pdelok":
      s.pages = s.pages.filter((p) => p.id !== arg);
      if (s.draft.savedId === arg) s.draft.savedId = undefined;
      await tgCall("deleteMessage", { chat_id: chatId, message_id: mid }).catch(() => {});
      await showPagesRich(chatId, s);
      break;
    case "pcancel":
      await tgCall("deleteMessage", { chat_id: chatId, message_id: mid }).catch(() => {});
      await showPagesRich(chatId, s);
      break;
    case "u":
      toast = `You tapped: ${arg}`;
      break;
  }

  await tgCall("answerCallbackQuery", { callback_query_id: cb.id, text: toast, show_alert: alert }).catch(() => {});
}

async function handleInline(q) {
  const s = await loadSession(q.from.id);
  const term = (q.query ?? "").trim().toLowerCase();
  const pages = s.pages
    .filter((p) => !term || p.title.toLowerCase().includes(term))
    .sort((a, b) => b.updatedAt - a.updatedAt)
    .slice(0, 20);
  const results = pages.map((p) => ({
    type: "article",
    id: p.id.slice(0, 64),
    title: p.title,
    description: `${p.blocks.length} blocks`,
    input_message_content: {
      rich_message: {
        blocks: toWireBlocks(p.blocks),
        ...(p.is_rtl ? { is_rtl: true } : {}),
      },
    },
    ...(p.buttons.length ? { reply_markup: { inline_keyboard: p.buttons } } : {}),
  }));
  await tgCall("answerInlineQuery", {
    inline_query_id: q.id, results, cache_time: 0, is_personal: true,
    ...(results.length ? {} : { button: { text: "No pages yet — open the editor", start_parameter: "editor" } }),
  }).catch((e) => console.error("answerInlineQuery", e));
}




/** Entry point — each handler file calls this with its piece of the Update. */
export async function runUpdate(update) {
  if (update.inline_query) return handleInline(update.inline_query);
  const cb = update.callback_query;
  // Alert/toast buttons work anywhere (channels too) without touching the user's session.
  if (cb && cb.data && cb.data.startsWith("al:")) {
    const a = await getAlert(cb.data.slice(3));
    await tgCall("answerCallbackQuery", {
      callback_query_id: cb.id, text: (a && a.text) ?? "This button has expired.", show_alert: a ? a.show_alert : true, cache_time: 3600,
    }).catch(() => {});
    return;
  }
  const message = update.message ?? update.edited_message;
  const userId = message ? ((message.from && message.from.id) ?? message.chat.id) : cb ? cb.from.id : undefined;
  if (!userId) return;
  await loadSession(userId);
  const before = new Map(getSession(userId).pages.map((p) => [p.id, JSON.stringify(p)]));
  try {
    if (message) await handleMessage(message);
    else if (cb) await handleCallback(cb);
  } finally {
    await saveSession(userId, getSession(userId), before);
  }
}

```

---


## `schema.js`

```js

```

---



## `schema.js`

```js

```

---
