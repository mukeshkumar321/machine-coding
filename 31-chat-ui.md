# Chat UI — Machine Coding

> **Difficulty:** Hard | **Time:** 60 min | **Frequency:** ⭐⭐⭐

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Requirements](#2-requirements)
3. [Clarifying Questions to Ask](#3-clarifying-questions-to-ask)
4. [Approach & Component Design](#4-approach--component-design)
5. [Reference Solution](#5-reference-solution)
6. [How It Works](#6-how-it-works)
7. [Edge Cases](#7-edge-cases)
8. [Accessibility](#8-accessibility)
9. [Performance Considerations](#9-performance-considerations)
10. [Follow-up Questions](#10-follow-up-questions)
11. [Common Mistakes](#11-common-mistakes)

---

## 1. Problem Statement

Build a **chat window** (WhatsApp / Slack style) with:

- a scrollable **message list** grouped by **day** (`Today`, `Yesterday`, `Mon, Nov 3`) and by **sender** (consecutive messages from one person share one name label / timestamp),
- **smart auto-scroll** — new messages pull the view to the bottom *only if the user is already near the bottom*; otherwise a **"↓ 2 new messages"** pill appears,
- a **typing indicator** ("Ava is typing…"),
- **optimistic send** — your message appears instantly as `Sending…`, becomes `✓ Sent` on ack, or `Failed to send` with a **Retry** button,
- **load older messages** when scrolling to the top, **without the viewport jumping**,
- a **mock event-emitter "socket"** that pushes incoming messages and typing events, with proper subscribe / unsubscribe cleanup.

---

## 2. Requirements

### Functional

- Initial page of the latest 20 messages; the list opens **scrolled to the bottom**.
- Incoming socket message:
  - user within `80px` of the bottom → scroll to the new message,
  - user scrolled up → **don't move**; show a pill with an unread counter; clicking it jumps to the bottom and clears the counter.
- Sending your own message always scrolls to the bottom.
- Optimistic send: `sending → sent | failed`. A failed message shows **Retry**; retry re-sends the *same* message (same position, same `clientId`).
- Scrolling near the top (or clicking "Load earlier messages") fetches the previous page and **prepends** it; the visible messages must stay at exactly the same screen position.
- Typing indicator shown on `typing` events; auto-hides on stop, when that person's message arrives, or after a TTL (lost "stop" event).
- Grouping: day dividers; **first message of a sender run** shows the name; **last** shows the time; a run breaks on sender change, day change, or a gap > 5 min.
- Enter sends, Shift+Enter inserts a newline; empty / whitespace-only messages are ignored.
- Server echo / duplicate delivery must **not** create duplicate bubbles (dedupe by `id` and `clientId`).

### Non-functional

- Pure **reducer** for message state, pure **`groupMessages`** helper — both unit-testable without React.
- All socket listeners and timers are cleaned up on unmount (React 18 StrictMode safe).
- Scroll logic isolated in one hook (`useChatScroll`); no layout reads outside event handlers / layout effects.
- Plain CSS, no libraries.

---

## 3. Clarifying Questions to Ask

- Transport: WebSocket / SSE / polling? Do we need **reconnect + catch-up** of missed messages?
- Is message **ordering** by server timestamp / sequence number, or by arrival?
- How large can a conversation get — do we need **virtualization**?
- Does the server **echo** our own messages over the socket, and is there an **idempotency key** (`clientId`)?
- Are there **read receipts**, "delivered" ticks, reactions, attachments, images (variable heights)?
- Pagination: cursor (`before=<id>`) or offset? Page size?
- Should the draft persist per conversation? Multiple conversations / threads?
- Time zone & locale for day boundaries? Rendering of very old dates?
- Should the pill show a **count** or an "unread" divider line at the first unread message?

---

## 4. Approach & Component Design

**Message model**

```js
{ id: 'm42',          // server id (== clientId until the ack arrives)
  key: 'c-17…-1',     // STABLE React key — never changes, even when id changes on ack
  clientId: 'c-17…-1',
  senderId, senderName, text, ts,
  status: 'sending' | 'sent' | 'failed' }   // only on my own messages
```

The separate **`key`** is the trick that makes optimistic UI smooth: the optimistic bubble and the acknowledged bubble are *the same React element*, so no remount, no flicker, no scroll jump.

**State machine of one outgoing message**

```
          send()                ack ok
  (none) ───────► sending ───────────────► sent
                     │  ▲
            ack fail │  │ retry()
                     ▼  │
                   failed
```

**Reducer actions**

| Action | Effect |
|---|---|
| `INIT` | newest page; merged with anything the socket delivered meanwhile |
| `RECEIVE` | append (dedupe by `id`; if it carries my `clientId`, *merge* into the optimistic bubble) |
| `SEND` / `SEND_OK` / `SEND_FAIL` / `RETRY` | optimistic append / patch status by `key` |
| `OLDER_START` / `PREPEND` / `OLDER_FAIL` | pagination state; prepend deduped page |

**Smart auto-scroll — the core algorithm**

```
 onScroll:   stuck = (scrollHeight - scrollTop - clientHeight) <= 80      ← measured BEFORE content grows
 on new tail message (layout effect, after DOM update, before paint):
      stuck || message is mine  → scrollTop = scrollHeight ; unread = 0
      else                      → unread += 1   (pill)
 on prepend (first key changed, last key same):
      scrollTop += scrollHeight_now - scrollHeight_before                 ← scroll anchoring by hand
```

`stuck` is stored in a **ref updated by the scroll event**, not recomputed after the new message renders — once the new bubble is in the DOM, `scrollHeight` has grown and the "am I near the bottom?" question would always answer *no*.

**Prepend without jumping** — adding N px of content above the viewport pushes everything down by N px. Adding the *height delta* to `scrollTop` in a **layout effect** (before paint) cancels that exactly, so the user sees nothing move. We also set `overflow-anchor: none` so the browser's own anchoring doesn't apply a second correction.

**Component tree**

```
App                         owns socket + api (created once), me
 ├─ header (title, "simulate incoming" demo button)
 ├─ MessageList             useChatScroll · groupMessages
 │    ├─ "Load earlier messages" / loading / retry
 │    ├─ day dividers + <Message> (memo) × n  → status, Retry
 │    ├─ TypingIndicator
 │    └─ "↓ N new messages" pill
 ├─ visually-hidden live region (announces incoming text)
 └─ Composer                textarea · Enter / Shift+Enter · Send

useChat(socket, api, me)    reducer + socket subscriptions + send/retry/loadOlder
useChatScroll(...)          stuck ref · unread · prepend restore · jump
groupMessages(messages)     flat list of { day | msg(first,last) } items
createMockSocket()          on / off / emit / connect / disconnect (interval "peer")
createMockApi()             fetchMessages({before}) · sendMessage({clientId,text})
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`socket.js`** (mock event emitter; reusable after `disconnect()` so StrictMode's mount → unmount → mount works)

```jsx
export function createMockSocket({ intervalMs = 8000, typingMs = 2000 } = {}) {
  const handlers = new Map();
  let timers = [];
  let interval = null;
  let seq = 0;

  const SCRIPT = [
    'Did you see the new design?',
    'I pushed the fix an hour ago',
    'Can we sync at 4?',
    'Looks good to me 👍',
  ];

  const off = (event, fn) => handlers.get(event)?.delete(fn);

  const on = (event, fn) => {
    if (!handlers.has(event)) handlers.set(event, new Set());
    handlers.get(event).add(fn);
    return () => off(event, fn);
  };

  const emit = (event, payload) => {
    [...(handlers.get(event) ?? [])].forEach((fn) => fn(payload));
  };

  const peerSays = (text) => {
    emit('message', {
      id: `p${++seq}`,
      senderId: 'ava',
      senderName: 'Ava',
      text: text ?? SCRIPT[seq % SCRIPT.length],
      ts: Date.now(),
    });
  };

  const connect = () => {
    if (interval) return;
    interval = setInterval(() => {
      emit('typing', { userId: 'ava', name: 'Ava', isTyping: true });
      timers.push(setTimeout(() => peerSays(), typingMs));
    }, intervalMs);
  };

  const disconnect = () => {
    clearInterval(interval);
    interval = null;
    timers.forEach(clearTimeout);
    timers = [];
  };

  const listenerCount = (event) => handlers.get(event)?.size ?? 0;

  return { on, off, emit, connect, disconnect, peerSays, listenerCount };
}
```

**`api.js`** (mock REST: 75 messages spread over ~2 days; sends containing "fail" fail once per `clientId`)

```jsx
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

export function createMockApi({ total = 75, latency = 400 } = {}) {
  const now = Date.now();
  const failedOnce = new Set();
  let serverSeq = 0;

  const PATTERN = [0, 0, 1, 0, 1, 1, 1, 0];
  const all = Array.from({ length: total }, (_, i) => {
    const mine = PATTERN[i % PATTERN.length] === 1;
    return {
      id: `m${i + 1}`,
      senderId: mine ? 'me' : 'ava',
      senderName: mine ? 'You' : 'Ava',
      text: `Message #${i + 1}`,
      ts: now - (total - i) * 40 * 60 * 1000,
    };
  });

  async function fetchMessages({ before, limit = 20 } = {}) {
    await wait(latency);
    const end = before ? all.findIndex((m) => m.id === before) : all.length;
    const start = Math.max(0, end - limit);
    return { messages: all.slice(start, end), hasMore: start > 0 };
  }

  async function sendMessage({ clientId, text }) {
    await wait(latency);
    if (/fail/i.test(text) && !failedOnce.has(clientId)) {
      failedOnce.add(clientId);
      throw new Error('Network error');
    }
    return { id: `s${++serverSeq}`, clientId, ts: Date.now() };
  }

  return { fetchMessages, sendMessage };
}
```

**`chatReducer.js`**

```jsx
export const initialState = {
  messages: [],
  hasMore: false,
  loadingOlder: false,
  olderError: false,
};

const withKey = (m) => ({ ...m, key: m.key ?? m.id });

const patch = (state, key, changes) => ({
  ...state,
  messages: state.messages.map((m) => (m.key === key ? { ...m, ...changes } : m)),
});

export function chatReducer(state, action) {
  switch (action.type) {
    case 'INIT': {
      const page = action.messages.map(withKey);
      const ids = new Set(page.map((m) => m.id));
      return {
        ...state,
        messages: [...page, ...state.messages.filter((m) => !ids.has(m.id))],
        hasMore: action.hasMore,
      };
    }
    case 'RECEIVE': {
      const incoming = withKey(action.message);
      const i = state.messages.findIndex(
        (m) => m.id === incoming.id || (incoming.clientId && m.key === incoming.clientId)
      );
      if (i === -1) return { ...state, messages: [...state.messages, incoming] };
      if (!incoming.clientId) return state;
      const messages = state.messages.slice();
      messages[i] = { ...messages[i], ...incoming, key: messages[i].key, status: 'sent' };
      return { ...state, messages };
    }
    case 'SEND':
      return { ...state, messages: [...state.messages, action.message] };
    case 'SEND_OK':
      return {
        ...state,
        messages: state.messages.map((m) =>
          m.key === action.key ? { ...m, ...action.saved, key: m.key, status: 'sent' } : m
        ),
      };
    case 'SEND_FAIL':
      return patch(state, action.key, { status: 'failed' });
    case 'RETRY':
      return patch(state, action.key, { status: 'sending' });
    case 'OLDER_START':
      return { ...state, loadingOlder: true, olderError: false };
    case 'PREPEND': {
      const ids = new Set(state.messages.map((m) => m.id));
      const fresh = action.messages.filter((m) => !ids.has(m.id)).map(withKey);
      return {
        ...state,
        messages: [...fresh, ...state.messages],
        hasMore: action.hasMore,
        loadingOlder: false,
      };
    }
    case 'OLDER_FAIL':
      return { ...state, loadingOlder: false, olderError: true };
    default:
      return state;
  }
}
```

**`groupMessages.js`**

```jsx
const GAP_MS = 5 * 60 * 1000;

const dayKey = (ts) => {
  const d = new Date(ts);
  return `${d.getFullYear()}-${d.getMonth()}-${d.getDate()}`;
};

export function dayLabel(ts, now = Date.now()) {
  if (dayKey(ts) === dayKey(now)) return 'Today';
  const yesterday = new Date(now);
  yesterday.setDate(yesterday.getDate() - 1);
  if (dayKey(ts) === dayKey(yesterday.getTime())) return 'Yesterday';
  return new Date(ts).toLocaleDateString(undefined, {
    weekday: 'short',
    month: 'short',
    day: 'numeric',
  });
}

export function groupMessages(messages, now = Date.now()) {
  const items = [];
  messages.forEach((m, i) => {
    const prev = messages[i - 1];
    const next = messages[i + 1];
    const day = dayKey(m.ts);
    const newDay = !prev || dayKey(prev.ts) !== day;

    if (newDay) items.push({ type: 'day', key: `day-${day}`, label: dayLabel(m.ts, now) });

    const first = newDay || prev.senderId !== m.senderId || m.ts - prev.ts > GAP_MS;
    const last =
      !next || dayKey(next.ts) !== day || next.senderId !== m.senderId || next.ts - m.ts > GAP_MS;

    items.push({ type: 'msg', key: m.key, message: m, first, last });
  });
  return items;
}
```

**`useChat.js`**

```jsx
import { useCallback, useEffect, useReducer, useRef, useState } from 'react';
import { chatReducer, initialState } from './chatReducer';

const TYPING_TTL_MS = 3000;

export function useChat({ socket, api, me }) {
  const [state, dispatch] = useReducer(chatReducer, initialState);
  const [typing, setTyping] = useState(null);
  const [announce, setAnnounce] = useState('');
  const stateRef = useRef(state);
  stateRef.current = state;
  const loadingRef = useRef(false);
  const typingTimer = useRef(null);
  const seq = useRef(0);

  useEffect(() => {
    let cancelled = false;
    api.fetchMessages({ limit: 20 }).then(({ messages, hasMore }) => {
      if (!cancelled) dispatch({ type: 'INIT', messages, hasMore });
    });
    return () => {
      cancelled = true;
    };
  }, [api]);

  useEffect(() => {
    const offMessage = socket.on('message', (message) => {
      dispatch({ type: 'RECEIVE', message });
      if (message.senderId !== me.id) {
        clearTimeout(typingTimer.current);
        setTyping(null);
        setAnnounce(`${message.senderName}: ${message.text}`);
      }
    });
    const offTyping = socket.on('typing', ({ name, isTyping }) => {
      clearTimeout(typingTimer.current);
      if (!isTyping) {
        setTyping(null);
        return;
      }
      setTyping(name);
      typingTimer.current = setTimeout(() => setTyping(null), TYPING_TTL_MS);
    });
    socket.connect();
    return () => {
      offMessage();
      offTyping();
      socket.disconnect();
      clearTimeout(typingTimer.current);
    };
  }, [socket, me.id]);

  const deliver = useCallback(
    async (message) => {
      try {
        const saved = await api.sendMessage({ clientId: message.key, text: message.text });
        dispatch({ type: 'SEND_OK', key: message.key, saved });
      } catch {
        dispatch({ type: 'SEND_FAIL', key: message.key });
      }
    },
    [api]
  );

  const send = useCallback(
    (text) => {
      const clientId = `c-${Date.now()}-${++seq.current}`;
      const message = {
        id: clientId,
        key: clientId,
        clientId,
        senderId: me.id,
        senderName: me.name,
        text,
        ts: Date.now(),
        status: 'sending',
      };
      dispatch({ type: 'SEND', message });
      deliver(message);
    },
    [deliver, me.id, me.name]
  );

  const retry = useCallback(
    (message) => {
      dispatch({ type: 'RETRY', key: message.key });
      deliver(message);
    },
    [deliver]
  );

  const loadOlder = useCallback(async () => {
    const { messages, hasMore } = stateRef.current;
    if (loadingRef.current || !hasMore || messages.length === 0) return;
    loadingRef.current = true;
    dispatch({ type: 'OLDER_START' });
    try {
      const page = await api.fetchMessages({ before: messages[0].id, limit: 20 });
      dispatch({ type: 'PREPEND', messages: page.messages, hasMore: page.hasMore });
    } catch {
      dispatch({ type: 'OLDER_FAIL' });
    } finally {
      loadingRef.current = false;
    }
  }, [api]);

  return { ...state, typing, announce, send, retry, loadOlder };
}
```

**`useChatScroll.js`**

```jsx
import { useCallback, useLayoutEffect, useRef, useState } from 'react';

const NEAR_BOTTOM_PX = 80;
const NEAR_TOP_PX = 60;

export function useChatScroll({ messages, typing, meId, onReachTop }) {
  const scrollerRef = useRef(null);
  const stuck = useRef(true);
  const heightRef = useRef(0);
  const seen = useRef({ first: undefined, last: undefined });
  const reachTop = useRef(onReachTop);
  reachTop.current = onReachTop;
  const [unread, setUnread] = useState(0);

  const jumpToBottom = useCallback(() => {
    const el = scrollerRef.current;
    if (!el) return;
    el.scrollTop = el.scrollHeight;
    heightRef.current = el.scrollHeight;
    stuck.current = true;
    setUnread(0);
  }, []);

  const onScroll = useCallback(() => {
    const el = scrollerRef.current;
    stuck.current = el.scrollHeight - el.scrollTop - el.clientHeight <= NEAR_BOTTOM_PX;
    heightRef.current = el.scrollHeight;
    if (stuck.current) setUnread(0);
    if (el.scrollTop <= NEAR_TOP_PX) reachTop.current?.();
  }, []);

  useLayoutEffect(() => {
    const el = scrollerRef.current;
    if (!el) return;
    const first = messages[0]?.key;
    const last = messages[messages.length - 1]?.key;
    const prev = seen.current;
    seen.current = { first, last };

    if (last !== undefined && prev.last === undefined) {
      jumpToBottom();
      return;
    }

    if (prev.first !== undefined && first !== prev.first) {
      el.scrollTop += el.scrollHeight - heightRef.current;
    }

    if (last !== prev.last) {
      const lastMsg = messages[messages.length - 1];
      const oldIndex = messages.findIndex((m) => m.key === prev.last);
      const added = oldIndex === -1 ? [lastMsg] : messages.slice(oldIndex + 1);
      if (stuck.current || lastMsg.senderId === meId) {
        jumpToBottom();
        return;
      }
      setUnread((n) => n + added.filter((m) => m.senderId !== meId).length);
    }
    heightRef.current = el.scrollHeight;
  }, [messages, meId, jumpToBottom]);

  useLayoutEffect(() => {
    if (typing && stuck.current) jumpToBottom();
    else if (scrollerRef.current) heightRef.current = scrollerRef.current.scrollHeight;
  }, [typing, jumpToBottom]);

  return { scrollerRef, onScroll, unread, jumpToBottom };
}
```

**`MessageList.jsx`**

```jsx
import { memo, useMemo } from 'react';
import { groupMessages } from './groupMessages';
import { useChatScroll } from './useChatScroll';

const fmtTime = (ts) =>
  new Date(ts).toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' });

const Message = memo(function Message({ message, mine, first, last, onRetry }) {
  const { status } = message;
  const showMeta = last || status === 'sending' || status === 'failed';
  return (
    <li className={`msg ${mine ? 'mine' : 'theirs'}${first ? ' first' : ''}`} data-status={status}>
      {first && !mine && <span className="sender">{message.senderName}</span>}
      <div className="bubble">{message.text}</div>
      {showMeta && (
        <div className="meta">
          <time dateTime={new Date(message.ts).toISOString()}>{fmtTime(message.ts)}</time>
          {mine && status === 'sending' && <span> · Sending…</span>}
          {mine && status === 'sent' && <span> · ✓ Sent</span>}
          {mine && status === 'failed' && (
            <>
              <span className="error"> · Failed to send</span>
              <button type="button" className="retry" onClick={() => onRetry(message)}>
                Retry
              </button>
            </>
          )}
        </div>
      )}
    </li>
  );
});

function TypingIndicator({ name }) {
  return (
    <div className="typing" role="status">
      <span className="dots" aria-hidden="true">
        <i />
        <i />
        <i />
      </span>
      {name} is typing…
    </div>
  );
}

export default function MessageList({
  messages,
  me,
  hasMore,
  loadingOlder,
  olderError,
  typing,
  onLoadOlder,
  onRetry,
}) {
  const { scrollerRef, onScroll, unread, jumpToBottom } = useChatScroll({
    messages,
    typing,
    meId: me.id,
    onReachTop: () => {
      if (hasMore && !loadingOlder && !olderError) onLoadOlder();
    },
  });
  const items = useMemo(() => groupMessages(messages), [messages]);

  return (
    <div className="chat-body">
      <div
        ref={scrollerRef}
        className="chat-scroller"
        role="log"
        aria-live="off"
        aria-label="Conversation"
        tabIndex={0}
        onScroll={onScroll}
      >
        {hasMore && (
          <div className="load-older">
            {loadingOlder ? (
              <span role="status">Loading earlier messages…</span>
            ) : (
              <button type="button" onClick={onLoadOlder}>
                {olderError ? 'Failed to load — retry' : 'Load earlier messages'}
              </button>
            )}
          </div>
        )}
        <ul className="chat-list">
          {items.map((item) =>
            item.type === 'day' ? (
              <li key={item.key} className="day-divider">
                <span>{item.label}</span>
              </li>
            ) : (
              <Message
                key={item.key}
                message={item.message}
                mine={item.message.senderId === me.id}
                first={item.first}
                last={item.last}
                onRetry={onRetry}
              />
            )
          )}
        </ul>
        {typing && <TypingIndicator name={typing} />}
      </div>
      {unread > 0 && (
        <button type="button" className="new-pill" onClick={jumpToBottom}>
          ↓ {unread} new message{unread > 1 ? 's' : ''}
        </button>
      )}
    </div>
  );
}
```

**`Composer.jsx`**

```jsx
import { useId, useState } from 'react';

export default function Composer({ onSend }) {
  const [text, setText] = useState('');
  const id = useId();

  const submit = (e) => {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    onSend(trimmed);
    setText('');
  };

  const onKeyDown = (e) => {
    if (e.key === 'Enter' && !e.shiftKey && !e.nativeEvent.isComposing) submit(e);
  };

  return (
    <form className="composer" onSubmit={submit}>
      <label htmlFor={id} className="sr-only">
        Message
      </label>
      <textarea
        id={id}
        rows={1}
        value={text}
        placeholder="Type a message… (include “fail” to see Retry)"
        onChange={(e) => setText(e.target.value)}
        onKeyDown={onKeyDown}
      />
      <button type="submit" disabled={!text.trim()}>
        Send
      </button>
    </form>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import { createMockApi } from './api';
import { createMockSocket } from './socket';
import { useChat } from './useChat';
import MessageList from './MessageList';
import Composer from './Composer';
import './chat.css';

const ME = { id: 'me', name: 'You' };

export default function App({ socket: socketProp, api: apiProp }) {
  const [socket] = useState(() => socketProp ?? createMockSocket());
  const [api] = useState(() => apiProp ?? createMockApi());
  const chat = useChat({ socket, api, me: ME });

  return (
    <div className="chat">
      <header className="chat-header">
        <h1>Ava</h1>
        <button type="button" onClick={() => socket.peerSays('Hello from the socket!')}>
          Simulate incoming
        </button>
      </header>
      <MessageList
        messages={chat.messages}
        me={ME}
        hasMore={chat.hasMore}
        loadingOlder={chat.loadingOlder}
        olderError={chat.olderError}
        typing={chat.typing}
        onLoadOlder={chat.loadOlder}
        onRetry={chat.retry}
      />
      <div className="sr-only" aria-live="polite">
        {chat.announce}
      </div>
      <Composer onSend={chat.send} />
    </div>
  );
}
```

**`chat.css`**

```css
.chat {
  display: flex;
  flex-direction: column;
  width: min(420px, 100%);
  height: min(80vh, 640px);
  margin: 24px auto;
  border: 1px solid #d0d7de;
  border-radius: 12px;
  background: #fff;
  font-family: system-ui, sans-serif;
  overflow: hidden;
}
.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  border-bottom: 1px solid #e5e7eb;
}
.chat-header h1 { margin: 0; font-size: 16px; }

.chat-body { position: relative; flex: 1; min-height: 0; }
.chat-scroller {
  height: 100%;
  overflow-y: auto;
  overflow-anchor: none;
  overscroll-behavior: contain;
  padding: 8px 12px;
}
.chat-scroller:focus-visible { outline: 2px solid #2563eb; outline-offset: -2px; }
.chat-list { list-style: none; margin: 0; padding: 0; }

.load-older { text-align: center; padding: 6px 0; font-size: 13px; color: #6b7280; }
.load-older button, .retry {
  border: 0;
  background: none;
  color: #2563eb;
  cursor: pointer;
  font: inherit;
  text-decoration: underline;
}

.day-divider { display: flex; justify-content: center; margin: 12px 0 6px; }
.day-divider span {
  padding: 2px 10px;
  border-radius: 999px;
  background: #f1f5f9;
  color: #475569;
  font-size: 12px;
}

.msg { display: flex; flex-direction: column; margin-top: 2px; max-width: 78%; }
.msg.first { margin-top: 10px; }
.msg.mine { align-items: flex-end; margin-left: auto; }
.msg.theirs { align-items: flex-start; }
.sender { margin-bottom: 2px; font-size: 12px; font-weight: 600; color: #475569; }
.bubble {
  padding: 7px 11px;
  border-radius: 14px;
  background: #f1f5f9;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
}
.mine .bubble { background: #2563eb; color: #fff; }
.msg[data-status='sending'] .bubble { opacity: 0.6; }
.msg[data-status='failed'] .bubble { background: #fee2e2; color: #991b1b; }
.meta { margin-top: 2px; font-size: 11px; color: #6b7280; }
.meta .error { color: #b91c1c; }

.typing { display: flex; align-items: center; gap: 6px; margin-top: 8px; font-size: 12px; color: #6b7280; }
.dots { display: inline-flex; gap: 3px; }
.dots i {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: #9ca3af;
  animation: blink 1.2s infinite ease-in-out;
}
.dots i:nth-child(2) { animation-delay: 0.2s; }
.dots i:nth-child(3) { animation-delay: 0.4s; }
@keyframes blink { 0%, 80%, 100% { opacity: 0.25; } 40% { opacity: 1; } }
@media (prefers-reduced-motion: reduce) { .dots i { animation: none; } }

.new-pill {
  position: absolute;
  bottom: 10px;
  left: 50%;
  transform: translateX(-50%);
  padding: 6px 14px;
  border: 0;
  border-radius: 999px;
  background: #111827;
  color: #fff;
  font-size: 13px;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.25);
}

.composer { display: flex; gap: 8px; padding: 10px; border-top: 1px solid #e5e7eb; }
.composer textarea {
  flex: 1;
  resize: none;
  padding: 8px 10px;
  border: 1px solid #d0d7de;
  border-radius: 8px;
  font: inherit;
}
.composer button {
  padding: 0 16px;
  border: 0;
  border-radius: 8px;
  background: #2563eb;
  color: #fff;
  cursor: pointer;
}
.composer button:disabled { opacity: 0.5; cursor: not-allowed; }

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}
```

</details>

---

## 6. How It Works

- **One source of truth.** `chatReducer` holds the ordered `messages`. Optimistic send, ack, failure, socket receive, and pagination are all actions on it, so every transition is testable without a DOM.
- **Stable `key` vs changing `id`.** The optimistic bubble starts with `id = key = clientId`. On ack, `SEND_OK` swaps in the server `id`/`ts` but **keeps `key`**, so React updates the same `<li>` in place (`Sending…` → `✓ Sent`) instead of unmounting and remounting it.
- **Dedupe.** `RECEIVE` ignores a message whose `id` already exists, and if an echo carries my `clientId` it **merges into the optimistic bubble** (marking it `sent`) instead of appending a twin — so it doesn't matter whether the echo or the REST ack arrives first. `INIT` merges with any socket message that landed while the first page was loading.
- **`stuck` is measured in `onScroll`.** The scroll handler stores `distanceFromBottom <= 80` in a ref. When a message arrives, the layout effect *reads that ref* — it does not re-measure, because the DOM already grew. `useLayoutEffect` (not `useEffect`) runs after DOM mutation but **before paint**, so the user never sees a frame where the new message is below the fold and then jumps.
- **New tail message → decide.** `stuck || mine` → `scrollTop = scrollHeight`. Otherwise `unread += (non-mine messages added)` and the pill appears. Scrolling back to the bottom (or clicking the pill) resets `unread`.
- **Prepend restore.** `heightRef` remembers `scrollHeight` as of the last scroll / layout pass. When the **first** key changes but the **last** doesn't, older content was inserted above; `scrollTop += scrollHeight - heightRef` pushes the viewport down by exactly the inserted height. `overflow-anchor: none` disables the browser's own anchoring so the correction isn't applied twice.
- **Pagination trigger.** `onScroll` calls `onReachTop` when `scrollTop <= 60`; `loadOlder` has a **ref guard** (`loadingRef`) so a burst of scroll events fires one request, and the cursor is `messages[0].id`. A visible **button** is the fallback for short lists with no scrollbar and for keyboard users; after a failure auto-loading stops and the button becomes "retry" (no infinite error loop).
- **Typing indicator** is `{ name | null }` state driven by the socket. A **TTL timer** clears it if the "stopped" event is lost; a message from that sender clears it immediately; the timer is cleared on unmount. If you're stuck to the bottom, showing the indicator also scrolls so it isn't hidden.
- **Grouping** (`groupMessages`) turns the flat array into `{day}` and `{msg, first, last}` items in a single O(n) pass: a run breaks on sender change, a new calendar day, or a gap over 5 minutes. `first` shows the name, `last` shows the time; sending/failed bubbles always show their status line.
- **Socket lifecycle.** `socket.on()` returns an unsubscribe function. The effect subscribes, calls `connect()`, and its cleanup unsubscribes, `disconnect()`s (clearing the interval and pending peer timers) and clears the typing timer. The mock is reusable after `disconnect()`, so StrictMode's simulated unmount/remount doesn't leave it dead or double-subscribed.
- **Announcing without noise.** The scroller is `aria-live="off"` (otherwise prepending 20 old messages would be read aloud); a separate visually-hidden live region announces only *incoming* messages.

---

## 7. Edge Cases

- **List shorter than the viewport** → no scrollbar, no scroll events → the "Load earlier messages" button is the only trigger (and keeps keyboard/AT users unblocked).
- **Rapid scroll events at the top** → `loadingRef` guard; only one request in flight.
- **Load-older fails** → `olderError`; auto-trigger disabled, button reads "retry"; no request storm.
- **Prepended page overlaps existing messages** (new messages shifted the cursor) → deduped by `id`.
- **User scrolls up while the older page is loading** → restore uses the *current* `scrollTop` plus the height delta, not a stale snapshot.
- **Own message while scrolled up** → always jump to bottom (the user just acted).
- **Echo before ack / ack before echo / duplicate delivery** → single bubble thanks to `id` + `clientId` matching.
- **Retry of a failed message** → same `key`, same `ts`, so it stays in place instead of jumping to the end; the server must treat `clientId` as an **idempotency key**.
- **Socket message arrives before the first page** → `INIT` merges instead of overwriting.
- **Typing "stop" event lost** → TTL clears it.
- **Unmount while a request is in flight** → the late `dispatch` is a harmless no-op in React 18; the initial-load effect uses a `cancelled` flag.
- **Several messages arrive while scrolled up** → counter adds the number of non-mine messages in the batch.
- **Day boundary / DST** → grouping uses local calendar keys; "Yesterday" is computed with `setDate(-1)`, not `now - 24h`.
- **Long unbroken text / URLs** → `overflow-wrap: anywhere`.
- **IME composition** → Enter during composition doesn't send.
- **Messages with identical timestamps** → order is array order (stable); sort by server `seq` in a real app.
- **Images / embeds that load late** change the height *after* we scrolled → need a `ResizeObserver` (see follow-ups).

---

## 8. Accessibility

- The scroller is `role="log"` with a label and `tabIndex=0`, so keyboard users can focus it and scroll with arrows / PageUp / PageDown / Home / End.
- `aria-live="off"` on the log + one **visually-hidden polite live region** that announces only new *incoming* messages ("Ava: Can we sync at 4?"). Own messages and prepended history are not announced.
- The typing indicator is `role="status"` with real text ("Ava is typing…"); the animated dots are `aria-hidden`. Dynamically inserted live regions are announced inconsistently — for stricter control render a persistent empty `role="status"` and fill it.
- Message status is **text**, not just color/opacity (`Sending…`, `✓ Sent`, `Failed to send`); Retry is a real `<button>`.
- The new-messages pill and "Load earlier messages" are real `<button>`s (focusable, Enter/Space).
- Composer textarea has an associated (visually-hidden) `<label>`; Enter sends, Shift+Enter newline, IME-safe.
- `<time dateTime>` gives machine-readable timestamps; day dividers give structure.
- `prefers-reduced-motion` stops the typing animation; contrast of failed/meta text meets AA.
- After "Send" focus stays in the textarea so typing continues uninterrupted.

---

## 9. Performance Considerations

- **`React.memo` on `Message`** with stable props (`onRetry` is `useCallback`, message objects keep identity unless patched) → an incoming message re-renders only the new row and the previous tail (whose `last` flag changed), not all n rows.
- Typing events change `typing` state in `useChat`, which re-renders `MessageList`; because `items` is `useMemo`'d on `messages` and rows are memoized, that's cheap. For very busy rooms, isolate the indicator into its own subscribed component.
- `groupMessages` is O(n) and memoized on `messages`.
- Layout reads (`scrollHeight`, `scrollTop`) happen only in scroll handlers and layout effects, each once per pass — no read/write interleaving loops (layout thrashing).
- The scroll handler is cheap (a few reads + a ref write; `setUnread(0)` bails out when unchanged), so no throttling is needed. Pagination is protected by the `loadingRef` guard rather than by debouncing.
- **Long histories (thousands of rows):** virtualize with variable-height rows (e.g. `react-virtuoso` in "chat" mode / `react-window` + measured sizes) or cap the in-memory window and drop far-away pages. Virtualization changes the anchoring math — the library must own the prepend correction.
- Fetch pages in a cursor (`before=id`) form; offset pagination breaks when new messages shift positions.
- `content-visibility: auto` on old rows (with a `contain-intrinsic-size`) can cut layout/paint cost without a virtualization library.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you make a real WebSocket connection robust?</strong></summary>

Wrap it in the same `on/emit/connect/disconnect` shape. Add **reconnect with exponential backoff + jitter**, heartbeats (ping/pong) to detect half-open connections, and on reconnect ask the server for **messages since the last `seq`/id** (catch-up) and merge them through the existing deduping `RECEIVE`/`INIT` logic. Show a "Reconnecting…" banner, and queue outgoing messages as `sending` (or fail them after a timeout so Retry appears). Pause reconnecting when the tab is hidden / offline and resume on `online` / `visibilitychange`.

</details>

<details>
<summary><strong>2. Why a separate `key` and `clientId` — what goes wrong with a single id?</strong></summary>

If the React key is the server id, the optimistic bubble (key = temp id) is **unmounted and a new element mounted** when the ack arrives — flicker, lost focus/selection, restarted animations, and a possible one-frame layout shift that can break scroll position. With a stable `key` the element persists. `clientId` doubles as the **idempotency key** sent to the server so a retry after a timeout can't create two messages, and it lets us match the server's echo to the optimistic bubble.

</details>

<details>
<summary><strong>3. What is the `flex-direction: column-reverse` trick and why not use it?</strong></summary>

With a reversed flex container, the "bottom" is scroll origin 0, so the browser naturally keeps you pinned to the newest message and prepending older content (at the end of the DOM) doesn't move the viewport — no JS anchoring needed. Downsides: DOM order is reversed relative to visual order (you must render the array reversed or reason about it carefully), tab/reading order and some assistive tech behaviours are confusing, `scrollTop` is negative/inconsistent in older browsers, Safari has had bugs, and you still need the "near bottom?" logic for the pill. The explicit approach is more portable and easier to explain.

</details>

<details>
<summary><strong>4. How do you keep the view pinned when images or embeds load after render?</strong></summary>

Heights change *without* a new message, so layout-effect logic won't run. Observe the content wrapper with a **`ResizeObserver`**; when its height grows and `stuck.current` is true, set `scrollTop = scrollHeight`. Also reserve space up front (`width`/`height` or `aspect-ratio` on `<img>`) to avoid the shift. The same observer helps when the mobile keyboard opens and `clientHeight` shrinks.

</details>

<details>
<summary><strong>5. How would you add an "unread messages" divider and read receipts?</strong></summary>

Store `lastReadId` (server-side too). When the user opens the chat, render a `— N new messages —` item *before* the first message after `lastReadId` (add it as another item type in `groupMessages`) and scroll to it instead of the bottom. Advance `lastReadId` when the bottom is reached while the tab is visible (use `IntersectionObserver` on the last message + `document.visibilityState`) and send a debounced `read` event; render ✓✓ when the peer's `read` event covers that message.

</details>

<details>
<summary><strong>6. How would you implement the sender's own "typing…" event?</strong></summary>

On textarea `onChange`, **throttle** an `emit('typing', { isTyping: true })` (e.g. once every 2–3 s while keys are being pressed) and schedule an idle timer (~3 s) that emits `isTyping: false`; also emit `false` on send / blur / unmount. Receivers rely on a TTL anyway (as here), so a missed "stop" self-heals. Don't send an event per keystroke.

</details>

<details>
<summary><strong>7. How do you guarantee message ordering?</strong></summary>

Arrival order isn't ordering — two clients' messages can cross on the wire. Use a **server-assigned monotonically increasing `seq`** (or server timestamp + id tiebreak) and insert into the array sorted by it; optimistic messages sit at the end with a provisional position and are re-slotted on ack. Binary-insert for O(log n) search. Groups/day dividers are derived, so they stay correct automatically.

</details>

<details>
<summary><strong>8. How would you test this?</strong></summary>

Unit-test `chatReducer` (echo + ack in both orders → one bubble, `INIT` merge, prepend dedupe) and `groupMessages` (day change, sender change, 5-min gap). For the component, jsdom has no layout, so **mock `scrollHeight` / `clientHeight` / `scrollTop`** on `HTMLElement.prototype` (e.g. derive `scrollHeight` from the number of rows), use fake timers for the mock API/socket latency, and assert: near-bottom message → `scrollTop === scrollHeight`; scrolled-up message → unchanged `scrollTop` + pill count; prepend → `scrollTop` increased by the added height; optimistic send → `Sending…` → `✓ Sent`; failing send → Retry; unmount → `socket.listenerCount()` is 0 and no timers left.

</details>

---

## 11. Common Mistakes

- Measuring "am I at the bottom?" **after** the new message rendered (always false) instead of tracking it from scroll events.
- Using `useEffect` instead of `useLayoutEffect` for scroll correction → visible jump for one frame.
- Prepending older messages without compensating `scrollTop` → the viewport jumps to the top of the new page and immediately triggers loading the next one (runaway pagination).
- Letting the browser's scroll anchoring and manual correction both run (no `overflow-anchor: none`).
- Always auto-scrolling on new messages → yanks the user away while they read history.
- Using the temp id as the React key and swapping it for the server id → remount/flicker; or generating a new key per render (`Math.random()`).
- Not deduping the server echo → every sent message appears twice.
- Retry that appends a *new* message (new `ts`, new `clientId`) → duplicates if the first request actually succeeded, and the message jumps to the end.
- Firing `loadOlder` on every scroll event with no in-flight guard; no error state so a failing request loops forever.
- Forgetting to unsubscribe the socket listener / clear typing and interval timers → duplicate messages after remount (StrictMode exposes this), leaks, updates after unmount.
- Typing indicator that never clears (no TTL, not cleared when the message arrives).
- `aria-live` on the whole log → prepended history and own messages are read aloud.
- Grouping with `now - 24h` for "Yesterday" or UTC dates for day boundaries → wrong labels around midnight / DST.
- Re-rendering every row on each incoming message (no `memo`, unstable callbacks).
- Sending whitespace-only messages, or sending on Enter during IME composition.
