import React, { useMemo, useState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { motion } from "framer-motion";

// --- تنظیمات اصلی بازی ---
const TRACK_LENGTH = 40; // مسیر ساده‌شده (حلقه)
const TOKENS_PER_PLAYER = 4;
const PLAYER_COLORS = [
  "#ef4444", // قرمز
  "#3b82f6", // آبی
  "#10b981", // سبز
  "#f59e0b", // زرد
];

const ANIMALS = [
  { id: "horse", label: "اسب", emoji: "🐴" },
  { id: "elephant", label: "فیل", emoji: "🐘" },
  { id: "dog", label: "سگ", emoji: "🐶" },
];

// خانه‌های امن: شروع هر بازیکن + یک خانه وسطی برای تنوع
const SAFE_CELLS = [0, 10, 20, 30, 5, 15, 25, 35];

// نقطه‌ی شروع هر بازیکن روی مسیر
const START_INDEX = [0, 10, 20, 30];

// ابزار کمکی
const inBase = (p) => p === -1; // داخل خانه
const inHome = (p) => p >= 100; // به خانه رسیده

function createInitialState(playerCount) {
  const players = Array.from({ length: playerCount }, (_, i) => ({
    id: i,
    color: PLAYER_COLORS[i],
    animalId: ANIMALS[i % ANIMALS.length].id,
    tokens: Array.from({ length: TOKENS_PER_PLAYER }, () => -1),
    finished: 0,
  }));
  return { players, current: 0, dice: null, canMove: false, winner: null, logs: [] };
}

export default function LudoAnimals() {
  const [playerCount, setPlayerCount] = useState(4);
  const [state, setState] = useState(() => createInitialState(4));

  const animalMap = useMemo(() => Object.fromEntries(ANIMALS.map(a => [a.id, a])), []);

  const reset = () => setState(createInitialState(playerCount));

  const rollDice = () => {
    if (state.winner) return;
    const v = Math.floor(Math.random() * 6) + 1;
    setState((s) => ({ ...s, dice: v, canMove: true }));
  };

  const nextPlayer = (forcedNext) => {
    setState((s) => ({
      ...s,
      current: typeof forcedNext === 'number' ? forcedNext : (s.current + 1) % s.players.length,
      dice: null,
      canMove: false,
    }));
  };

  const moveToken = (playerIdx, tokenIdx) => {
    setState((s) => {
      if (!s.canMove || s.winner) return s;
      const players = s.players.map(p => ({ ...p, tokens: [...p.tokens] }));
      const me = players[playerIdx];
      const roll = s.dice ?? 0;
      let pos = me.tokens[tokenIdx];

      if (inBase(pos)) {
        if (roll !== 6) return s; // نیاز به شش برای خروج
        pos = START_INDEX[playerIdx];
      } else if (!inHome(pos)) {
        pos = (pos + roll) % TRACK_LENGTH;
      }

      // ورود به خانه‌ی نهایی (قوانین ساده‌شده)
      // اگر دور کامل زد و دقیقاً روی خانه‌ی شروع خودش فرود آمد، می‌رود خانه‌ی نهایی
      if (!inBase(me.tokens[tokenIdx]) && !inHome(me.tokens[tokenIdx])) {
        const from = me.tokens[tokenIdx];
        const to = pos;
        const passedStart = (from < START_INDEX[playerIdx] && (to >= START_INDEX[playerIdx] || to < from))
          || (from >= START_INDEX[playerIdx] && to >= START_INDEX[playerIdx] && to < from);
        if (to === START_INDEX[playerIdx] || passedStart && roll !== 0) {
          // اگر عدد دقیق بود و روی خانه شروع خودش فرود آمد
          if (to === START_INDEX[playerIdx]) {
            pos = 100 + me.finished; // به خانه‌ی نهایی یکی اضافه می‌شود
          }
        }
      }

      // اعمال حرکت
      me.tokens[tokenIdx] = pos;

      // اگر روی خانه‌ی امن نیست و خانه‌ی مشترک با حریف شد: زدن مهره
      if (!inHome(pos) && !SAFE_CELLS.includes(pos)) {
        players.forEach((p, idx) => {
          if (idx === playerIdx) return;
          p.tokens = p.tokens.map(t => (t === pos ? -1 : t));
        });
      }

      // اگر به خانه‌ی نهایی رفت
      if (inHome(pos)) {
        me.finished = Math.min(TOKENS_PER_PLAYER, me.tokens.filter(inHome).length);
      }

      // برنده؟
      let winner = s.winner;
      if (me.finished === TOKENS_PER_PLAYER) {
        winner = playerIdx;
      }

      // حق حرکت مجدد در صورت شش
      const extraTurn = s.dice === 6;

      const logs = [
        `${emojiFor(me)} بازیکن ${playerIdx + 1} مهره ${tokenIdx + 1} را حرکت داد (${s.dice}).`,
        ...s.logs,
      ].slice(0, 12);

      return {
        ...s,
        players,
        canMove: false,
        winner,
        logs,
        current: extraTurn && !winner ? playerIdx : (playerIdx + 1) % s.players.length,
        dice: null,
      };
    });
  };

  const emojiFor = (p) => animalMap[p.animalId]?.emoji ?? "❓";

  const currentPlayer = state.players[state.current];

  // آیا بازیکن جاری حرکتی دارد؟
  const hasAnyMove = useMemo(() => {
    const roll = state.dice;
    if (!roll) return false;
    const p = currentPlayer;
    // اگر شش آمده و حداقل یک مهره در خانه است
    if (roll === 6 && p.tokens.some(inBase)) return true;
    // اگر مهره‌ای روی مسیر هست
    return p.tokens.some(t => !inBase(t) && !inHome(t));
  }, [state.dice, state.current, state.players]);

  // چیدمان مسیر به صورت 10x4 (ظاهر ساده و مرتب)
  const cells = useMemo(() => Array.from({ length: TRACK_LENGTH }, (_, i) => i), []);

  const cellContent = (idx) => {
    const stack = [];
    state.players.forEach((p, pi) => {
      p.tokens.forEach((t, ti) => {
        if (t === idx) {
          stack.push({ p: pi, t: ti, color: p.color, emoji: emojiFor(p) });
        }
      });
    });
    if (stack.length === 0) return null;
    return (
      <div className="flex -space-x-2 rtl:space-x-reverse">
        {stack.map((s, i) => (
          <span key={i} title={`P${s.p + 1}-T${i + 1}`} className="inline-flex items-center justify-center w-6 h-6 rounded-full border text-base shadow" style={{ background: s.color, borderColor: "rgba(0,0,0,0.2)" }}>
            {s.emoji}
          </span>
        ))}
      </div>
    );
  };

  const baseArea = (pi) => {
    const p = state.players[pi];
    const baseTokens = p.tokens
      .map((t, ti) => ({ t, ti }))
      .filter(x => inBase(x.t));
    return (
      <div className="grid grid-cols-2 gap-2">
        {baseTokens.map(({ ti }) => (
          <div key={ti} className="flex items-center justify-center w-10 h-10 rounded-2xl border shadow" style={{ borderColor: p.color }}>
            <span className="text-xl">{emojiFor(p)}</span>
          </div>
        ))}
      </div>
    );
  };

  const homeArea = (pi) => {
    const p = state.players[pi];
    const homeTokens = p.tokens.filter(inHome).length;
    return (
      <div className="flex gap-2">
        {Array.from({ length: TOKENS_PER_PLAYER }).map((_, i) => (
          <div key={i} className="w-8 h-8 rounded-xl border flex items-center justify-center" style={{ borderColor: p.color, opacity: i < homeTokens ? 1 : 0.25 }}>
            <span>{emojiFor(p)}</span>
          </div>
        ))}
      </div>
    );
  };

  const movableTokens = useMemo(() => {
    const p = currentPlayer;
    const roll = state.dice ?? 0;
    const list = [];
    p.tokens.forEach((pos, ti) => {
      if (inBase(pos)) {
        if (roll === 6) list.push({ ti, from: "خانه", to: START_INDEX[state.current] });
      } else if (!inHome(pos)) {
        list.push({ ti, from: pos, to: (pos + roll) % TRACK_LENGTH });
      }
    });
    return list;
  }, [state.dice, state.current, state.players]);

  return (
    <div className="p-4 md:p-6 max-w-6xl mx-auto">
      <motion.h1 initial={{ opacity: 0, y: -10 }} animate={{ opacity: 1, y: 0 }} className="text-2xl md:text-3xl font-bold mb-4">
        منچ حیوانات 🐴🐘🐶 (نمونه‌ی ساده‌شده)
      </motion.h1>

      <Card className="mb-4">
        <CardHeader className="pb-2">
          <CardTitle className="text-lg">تنظیمات سریع</CardTitle>
        </CardHeader>
        <CardContent className="grid grid-cols-1 md:grid-cols-5 gap-3 items-end">
          <div className="md:col-span-1">
            <label className="block mb-1 text-sm">تعداد بازیکن</label>
            <Select value={String(playerCount)} onValueChange={(v) => { setPlayerCount(Number(v)); setState(createInitialState(Number(v))); }}>
              {[2,3,4].map(n => (
                <SelectItem key={n} value={String(n)}>{n}</SelectItem>
              ))}
            </Select>
          </div>

          {state.players.map((p, i) => (
            <div key={i} className="flex items-end gap-2">
              <div className="flex-1">
                <label className="block mb-1 text-sm">بازیکن {i + 1} – حیوان</label>
                <Select value={p.animalId} onValueChange={(v) => setState(s => ({
                  ...s,
                  players: s.players.map((pp, idx) => idx === i ? { ...pp, animalId: v } : pp),
                }))}>
                  {ANIMALS.map(a => (
                    <SelectItem key={a.id} value={a.id}>{a.emoji} {a.label}</SelectItem>
                  ))}
                </Select>
              </div>
              <div className="w-10 h-10 rounded-xl border flex items-center justify-center" style={{ borderColor: p.color }}>
                <span className="text-xl">{animalMap[p.animalId].emoji}</span>
              </div>
            </div>
          ))}

          <div className="md:col-span-1 flex gap-2 md:justify-end">
            <Button variant="secondary" onClick={reset}>شروع مجدد</Button>
          </div>
        </CardContent>
      </Card>

      <div className="grid md:grid-cols-3 gap-4 items-start">
        <Card className="md:col-span-2">
          <CardHeader className="pb-2">
            <CardTitle className="text-lg">مسیر بازی (۴۰ خانه)</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-10 gap-1">
              {cells.map((idx) => (
                <div key={idx} className="relative aspect-square rounded-xl border flex items-center justify-center text-xs select-none" style={{
                  borderColor: SAFE_CELLS.includes(idx) ? "#94a3b8" : "#e5e7eb",
                  background: SAFE_CELLS.includes(idx) ? "#f1f5f9" : "white",
                }}>
                  <div className="absolute top-1 left-1 text-[10px] opacity-60">{idx}</div>
                  {cellContent(idx)}
                </div>
              ))}
            </div>
          </CardContent>
        </Card>

        <div className="space-y-4">
          <Card>
            <CardHeader className="pb-2">
              <CardTitle className="text-lg">نوبت: بازیکن {state.current + 1}</CardTitle>
            </CardHeader>
            <CardContent className="space-y-3">
              <div className="flex items-center gap-3">
                <Button onClick={rollDice} disabled={!!state.dice || !!state.winner}>تاس</Button>
                <div className="text-2xl font-bold min-w-[2ch] text-center">{state.dice ?? "–"}</div>
                {state.winner !== null && (
                  <div className="text-sm px-2 py-1 rounded-lg bg-green-50 border">🏆 برنده: بازیکن {state.winner + 1}</div>
                )}
              </div>

              <div>
                <div className="text-sm mb-2">مهره‌های قابل حرکت</div>
                <div className="flex flex-wrap gap-2">
                  {state.dice && hasAnyMove ? (
                    movableTokens.map(m => (
                      <Button key={m.ti} variant="outline" onClick={() => moveToken(state.current, m.ti)}>
                        مهره {m.ti + 1}
                      </Button>
                    ))
                  ) : (
                    <div className="text-xs opacity-70">ابتدا تاس بریزید. اگر حرکتی ندارید نوبت به بازیکن بعد می‌رسد.</div>
                  )}
              </div>

              {!hasAnyMove && state.dice && (
                <Button variant="secondary" onClick={() => nextPlayer()}>
                  حرکت ندارم – نوبت بعدی
                </Button>
              )}
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="pb-2">
              <CardTitle className="text-lg">وضعیت بازیکنان</CardTitle>
            </CardHeader>
            <CardContent className="space-y-3">
              {state.players.map((p, i) => (
                <div key={i} className="flex items-center justify-between p-2 rounded-xl border" style={{ borderColor: p.color }}>
                  <div className="flex items-center gap-2">
                    <span className="text-xl">{animalMap[p.animalId].emoji}</span>
                    <span className="font-medium">بازیکن {i + 1}</span>
                  </div>
                  <div className="flex items-center gap-3 text-sm">
                    <div className="flex items-center gap-1">
                      <span className="opacity-70">خانه:</span> {baseArea(i)}
                    </div>
                    <div className="flex items-center gap-1">
                      <span className="opacity-70">نهایی:</span> {homeArea(i)}
                    </div>
                  </div>
                </div>
              ))}
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="pb-2">
              <CardTitle className="text-lg">رویدادها</CardTitle>
            </CardHeader>
            <CardContent>
              <div className="text-xs space-y-1 max-h-48 overflow-auto">
                {state.logs.length === 0 ? (
                  <div className="opacity-60">هنوز رویدادی ثبت نشده است.</div>
                ) : (
                  state.logs.map((l, i) => (
                    <div key={i} className="p-1 rounded hover:bg-slate-50">{l}</div>
                  ))
                )}
              </div>
            </CardContent>
          </Card>
        </div>
      </div>

      <div className="mt-6 text-xs leading-6 opacity-80">
        <p>این یک نمونه‌ی ساده‌شده از منچ است تا سریع بازی کنید: برای خارج کردن مهره از خانه باید شش بیاورید، برخورد روی خانه‌ی امن باعث زدن نمی‌شود، و با رسیدن دقیق به خانه‌ی شروع خودتان، مهره وارد مقصد نهایی می‌شود. اگر شش بیاورید، نوبت اضافه دارید.</p>
        <p className="mt-2">اگر بخواهید، می‌توانم نسخه‌ی تخته‌ی کلاسیک منچ با مسیر ۵۲ خانه، رنگ‌بندی رسمی و قوانین دقیق‌ (خانه‌های رنگی، راهرو نهایی و ...) را هم اضافه کنم.</p>
      </div>
    </div>
  );
}
nice to meet you
let's play
how to play
play with me
come and play with me
