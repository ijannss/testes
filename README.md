"use client";
import { useState, useEffect, useRef, useCallback } from "react";

const MODES = {
  easy:   { label: "EASY",   duration: 2500, color: "#00ff88", spawnRate: 1200, desc: "2.5s window" },
  medium: { label: "MEDIUM", duration: 1400, color: "#ffcc00", spawnRate: 900,  desc: "1.4s window" },
  hard:   { label: "HARD",   duration: 700,  color: "#ff3355", spawnRate: 650,  desc: "0.7s window" },
};

const GAME_DURATION = 30;
const DOT_SIZE = 48;
const PADDING = 60;

function getGrade(acc) {
  if (acc >= 95) return { label: "S+", color: "#00ff88" };
  if (acc >= 85) return { label: "S",  color: "#00ddff" };
  if (acc >= 70) return { label: "A",  color: "#aaff00" };
  if (acc >= 55) return { label: "B",  color: "#ffcc00" };
  if (acc >= 40) return { label: "C",  color: "#ff8800" };
  return { label: "D", color: "#ff3355" };
}

let dotIdCounter = 0;

export default function AimTrainer() {
  const [screen, setScreen] = useState("menu"); // menu | game | result
  const [mode, setMode] = useState("medium");
  const [dots, setDots] = useState([]);
  const [score, setScore] = useState(0);
  const [misses, setMisses] = useState(0);
  const [timeLeft, setTimeLeft] = useState(GAME_DURATION);
  const [hits, setHits] = useState(0);
  const [reactions, setReactions] = useState([]);
  const [ripples, setRipples] = useState([]);
  const [flashType, setFlashType] = useState(null);
  const [finalStats, setFinalStats] = useState(null);

  const arenaRef = useRef(null);
  const spawnTimer = useRef(null);
  const gameTimer = useRef(null);
  const dotTimers = useRef({});
  const activeMode = MODES[mode];

  // Cleanup all timers
  const clearAll = useCallback(() => {
    clearInterval(spawnTimer.current);
    clearInterval(gameTimer.current);
    Object.values(dotTimers.current).forEach(clearTimeout);
    dotTimers.current = {};
  }, []);

  const endGame = useCallback((currentHits, currentMisses, currentScore, currentReactions) => {
    clearAll();
    const total = currentHits + currentMisses;
    const acc = total > 0 ? Math.round((currentHits / total) * 100) : 0;
    const avgReaction = currentReactions.length > 0
      ? Math.round(currentReactions.reduce((a, b) => a + b, 0) / currentReactions.length)
      : 0;
    setFinalStats({ score: currentScore, hits: currentHits, misses: currentMisses, acc, avgReaction, total });
    setDots([]);
    setScreen("result");
  }, [clearAll]);

  const spawnDot = useCallback(() => {
    const arena = arenaRef.current;
    if (!arena) return;
    const rect = arena.getBoundingClientRect();
    const w = rect.width  || window.innerWidth;
    const h = rect.height || window.innerHeight;
    const x = PADDING + Math.random() * (w  - DOT_SIZE - PADDING * 2);
    const y = PADDING + Math.random() * (h - DOT_SIZE - PADDING * 2);
    const id = ++dotIdCounter;
    const spawnedAt = Date.now();

    setDots(prev => [...prev, { id, x, y, spawnedAt }]);

    // Auto-expire dot
    dotTimers.current[id] = setTimeout(() => {
      setDots(prev => prev.filter(d => d.id !== id));
      setMisses(prev => {
        const newMisses = prev + 1;
        setHits(h2 => {
          setScore(s2 => {
            setReactions(r2 => {
              if (h2 + newMisses >= 999) endGame(h2, newMisses, s2, r2);
              return r2;
            });
            return s2;
          });
          return h2;
        });
        return newMisses;
      });
      setFlashType("miss");
      setTimeout(() => setFlashType(null), 180);
      delete dotTimers.current[id];
    }, MODES[mode].duration);
  }, [mode, endGame]);

  const startGame = useCallback(() => {
    clearAll();
    dotIdCounter = 0;
    setDots([]);
    setScore(0);
    setHits(0);
    setMisses(0);
    setReactions([]);
    setRipples([]);
    setFlashType(null);
    setTimeLeft(GAME_DURATION);
    setScreen("game");
  }, [clearAll]);

  // Start spawning & countdown after screen = game
  useEffect(() => {
    if (screen !== "game") return;
    const cfg = MODES[mode];

    spawnTimer.current = setInterval(() => spawnDot(), cfg.spawnRate);
    spawnDot();

    let remaining = GAME_DURATION;
    gameTimer.current = setInterval(() => {
      remaining -= 1;
      setTimeLeft(remaining);
      if (remaining <= 0) {
        clearInterval(gameTimer.current);
        clearInterval(spawnTimer.current);
        setHits(h => {
          setMisses(m => {
            setScore(s => {
              setReactions(r => {
                endGame(h, m, s, r);
                return r;
              });
              return s;
            });
            return m;
          });
          return h;
        });
      }
    }, 1000);

    return () => clearAll();
  }, [screen]);

  const handleDotClick = useCallback((e, dot) => {
    e.stopPropagation();
    const reaction = Date.now() - dot.spawnedAt;
    clearTimeout(dotTimers.current[dot.id]);
    delete dotTimers.current[dot.id];

    setDots(prev => prev.filter(d => d.id !== dot.id));
    setHits(prev => prev + 1);
    setScore(prev => prev + 100);
    setReactions(prev => [...prev, reaction]);
    setFlashType("hit");
    setTimeout(() => setFlashType(null), 150);

    const rippleId = Date.now() + Math.random();
    setRipples(prev => [...prev, { id: rippleId, x: dot.x + DOT_SIZE / 2, y: dot.y + DOT_SIZE / 2, color: MODES[mode].color }]);
    setTimeout(() => setRipples(prev => prev.filter(r => r.id !== rippleId)), 500);
  }, [mode]);

  const handleArenaClick = useCallback(() => {
    setFlashType("miss");
    setMisses(prev => prev + 1);
    setTimeout(() => setFlashType(null), 150);
  }, []);

  const total = hits + misses;
  const acc = total > 0 ? Math.round((hits / total) * 100) : 100;
  const timerPct = (timeLeft / GAME_DURATION) * 100;

  // ─── MENU ───────────────────────────────────────────────────────────────────
  if (screen === "menu") return (
    <div style={{ minHeight: "100vh", background: "#0a0a0f", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", fontFamily: "'Courier New', monospace", padding: "2rem", position: "relative", overflow: "hidden" }}>
      {/* bg grid */}
      <div style={{ position: "absolute", inset: 0, backgroundImage: "linear-gradient(rgba(255,255,255,0.025) 1px, transparent 1px), linear-gradient(90deg, rgba(255,255,255,0.025) 1px, transparent 1px)", backgroundSize: "40px 40px", pointerEvents: "none" }} />
      <div style={{ position: "relative", zIndex: 1, textAlign: "center", maxWidth: 520 }}>
        {/* crosshair icon */}
        <div style={{ display: "flex", justifyContent: "center", marginBottom: "2rem" }}>
          <svg width="64" height="64" viewBox="0 0 64 64" fill="none">
            <circle cx="32" cy="32" r="20" stroke="#ff3355" strokeWidth="2"/>
            <circle cx="32" cy="32" r="4" fill="#ff3355"/>
            <line x1="32" y1="4" x2="32" y2="20" stroke="#ff3355" strokeWidth="2"/>
            <line x1="32" y1="44" x2="32" y2="60" stroke="#ff3355" strokeWidth="2"/>
            <line x1="4" y1="32" x2="20" y2="32" stroke="#ff3355" strokeWidth="2"/>
            <line x1="44" y1="32" x2="60" y2="32" stroke="#ff3355" strokeWidth="2"/>
          </svg>
        </div>
        <h1 style={{ color: "#ffffff", fontSize: "clamp(2.2rem, 6vw, 3.5rem)", fontWeight: 900, letterSpacing: "0.2em", margin: 0, textTransform: "uppercase" }}>AIM<span style={{ color: "#ff3355" }}>TRAINER</span></h1>
        <p style={{ color: "#555", letterSpacing: "0.15em", marginTop: "0.5rem", fontSize: "0.75rem" }}>CLICK THE DOTS BEFORE THEY VANISH</p>

        <div style={{ marginTop: "3rem" }}>
          <p style={{ color: "#666", fontSize: "0.65rem", letterSpacing: "0.2em", marginBottom: "1rem" }}>SELECT DIFFICULTY</p>
          <div style={{ display: "flex", gap: "1rem", justifyContent: "center", flexWrap: "wrap" }}>
            {Object.entries(MODES).map(([key, cfg]) => (
              <button key={key} onClick={() => setMode(key)}
                style={{ padding: "1rem 1.5rem", border: `2px solid ${mode === key ? cfg.color : "#222"}`, background: mode === key ? cfg.color + "18" : "transparent", color: mode === key ? cfg.color : "#444", cursor: "pointer", fontFamily: "inherit", letterSpacing: "0.15em", fontSize: "0.75rem", fontWeight: 700, textTransform: "uppercase", transition: "all 0.2s", minWidth: 120, borderRadius: 2, outline: "none" }}>
                <div style={{ fontSize: "1rem", fontWeight: 900 }}>{cfg.label}</div>
                <div style={{ fontSize: "0.6rem", marginTop: 4, opacity: 0.7 }}>{cfg.desc}</div>
              </button>
            ))}
          </div>
        </div>

        <button onClick={startGame}
          style={{ marginTop: "2.5rem", padding: "1rem 3rem", background: "#ff3355", border: "none", color: "#fff", fontFamily: "inherit", fontSize: "0.9rem", fontWeight: 900, letterSpacing: "0.25em", textTransform: "uppercase", cursor: "pointer", transition: "all 0.2s", borderRadius: 2 }}
          onMouseEnter={e => e.target.style.background = "#ff5577"}
          onMouseLeave={e => e.target.style.background = "#ff3355"}>
          START — {GAME_DURATION}s
        </button>

        <p style={{ color: "#333", fontSize: "0.6rem", marginTop: "2rem", letterSpacing: "0.1em" }}>MISCLICKING COUNTS AS A MISS</p>
      </div>
    </div>
  );

  // ─── RESULT ──────────────────────────────────────────────────────────────────
  if (screen === "result" && finalStats) {
    const grade = getGrade(finalStats.acc);
    return (
      <div style={{ minHeight: "100vh", background: "#0a0a0f", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", fontFamily: "'Courier New', monospace", padding: "2rem" }}>
        <div style={{ position: "absolute", inset: 0, backgroundImage: "linear-gradient(rgba(255,255,255,0.025) 1px, transparent 1px), linear-gradient(90deg, rgba(255,255,255,0.025) 1px, transparent 1px)", backgroundSize: "40px 40px", pointerEvents: "none" }} />
        <div style={{ position: "relative", zIndex: 1, textAlign: "center", maxWidth: 480, width: "100%" }}>
          <p style={{ color: "#444", fontSize: "0.65rem", letterSpacing: "0.25em", marginBottom: "0.5rem" }}>SESSION COMPLETE</p>
          <h1 style={{ color: "#fff", fontSize: "clamp(2rem, 6vw, 3rem)", fontWeight: 900, margin: 0, letterSpacing: "0.15em" }}>RESULTS</h1>

          {/* Grade */}
          <div style={{ margin: "2rem auto", width: 100, height: 100, border: `3px solid ${grade.color}`, display: "flex", alignItems: "center", justifyContent: "center", borderRadius: 4 }}>
            <span style={{ color: grade.color, fontSize: "3rem", fontWeight: 900 }}>{grade.label}</span>
          </div>

          {/* Stats grid */}
          <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "1px", background: "#1a1a1a", border: "1px solid #1a1a1a", borderRadius: 4, overflow: "hidden" }}>
            {[
              { label: "SCORE",     value: finalStats.score },
              { label: "ACCURACY",  value: `${finalStats.acc}%` },
              { label: "HITS",      value: finalStats.hits,   color: "#00ff88" },
              { label: "MISSES",    value: finalStats.misses, color: "#ff3355" },
              { label: "AVG REACT", value: finalStats.avgReaction ? `${finalStats.avgReaction}ms` : "N/A" },
              { label: "MODE",      value: MODES[mode].label, color: activeMode.color },
            ].map(s => (
              <div key={s.label} style={{ background: "#0d0d12", padding: "1.2rem 1rem" }}>
                <div style={{ color: "#444", fontSize: "0.55rem", letterSpacing: "0.2em", marginBottom: 6 }}>{s.label}</div>
                <div style={{ color: s.color || "#ffffff", fontSize: "1.4rem", fontWeight: 900 }}>{s.value}</div>
              </div>
            ))}
          </div>

          <div style={{ display: "flex", gap: "1rem", marginTop: "2rem", justifyContent: "center" }}>
            <button onClick={startGame}
              style={{ padding: "0.9rem 2rem", background: "#ff3355", border: "none", color: "#fff", fontFamily: "inherit", fontSize: "0.8rem", fontWeight: 900, letterSpacing: "0.2em", cursor: "pointer", borderRadius: 2 }}>
              PLAY AGAIN
            </button>
            <button onClick={() => setScreen("menu")}
              style={{ padding: "0.9rem 2rem", background: "transparent", border: "2px solid #333", color: "#666", fontFamily: "inherit", fontSize: "0.8rem", fontWeight: 700, letterSpacing: "0.2em", cursor: "pointer", borderRadius: 2 }}>
              MENU
            </button>
          </div>
        </div>
      </div>
    );
  }

  // ─── GAME ────────────────────────────────────────────────────────────────────
  return (
    <div style={{ position: "relative", width: "100vw", height: "100vh", background: "#0a0a0f", overflow: "hidden", cursor: "crosshair", userSelect: "none" }}
      onClick={handleArenaClick} ref={arenaRef}>

      {/* bg grid */}
      <div style={{ position: "absolute", inset: 0, backgroundImage: "linear-gradient(rgba(255,255,255,0.02) 1px, transparent 1px), linear-gradient(90deg, rgba(255,255,255,0.02) 1px, transparent 1px)", backgroundSize: "40px 40px", pointerEvents: "none" }} />

      {/* Flash overlay */}
      {flashType && (
        <div style={{ position: "absolute", inset: 0, background: flashType === "hit" ? `${activeMode.color}18` : "#ff335522", pointerEvents: "none", zIndex: 50 }} />
      )}

      {/* HUD — top bar */}
      <div style={{ position: "absolute", top: 0, left: 0, right: 0, zIndex: 40, display: "flex", alignItems: "center", gap: "2rem", padding: "1rem 1.5rem", background: "linear-gradient(#0a0a0fcc, transparent)" }}>
        {/* Timer bar */}
        <div style={{ flex: 1, height: 3, background: "#1a1a1a", borderRadius: 2, overflow: "hidden" }}>
          <div style={{ height: "100%", width: `${timerPct}%`, background: timerPct > 40 ? activeMode.color : timerPct > 20 ? "#ffcc00" : "#ff3355", transition: "width 1s linear, background 0.5s" }} />
        </div>
        <span style={{ color: timerPct <= 20 ? "#ff3355" : "#fff", fontFamily: "'Courier New', monospace", fontWeight: 900, fontSize: "1.1rem", minWidth: 32, textAlign: "right" }}>{timeLeft}</span>
      </div>

      {/* HUD — stats row */}
      <div style={{ position: "absolute", top: "3rem", left: 0, right: 0, zIndex: 40, display: "flex", justifyContent: "center", gap: "2.5rem", padding: "0.5rem" }}>
        {[
          { label: "SCORE", value: score, color: "#fff" },
          { label: "HITS",  value: hits,  color: activeMode.color },
          { label: "ACC",   value: `${acc}%`, color: acc >= 70 ? "#00ff88" : acc >= 50 ? "#ffcc00" : "#ff3355" },
          { label: "MISS",  value: misses, color: "#ff3355" },
        ].map(s => (
          <div key={s.label} style={{ textAlign: "center", fontFamily: "'Courier New', monospace" }}>
            <div style={{ color: "#333", fontSize: "0.5rem", letterSpacing: "0.2em" }}>{s.label}</div>
            <div style={{ color: s.color, fontSize: "1rem", fontWeight: 900 }}>{s.value}</div>
          </div>
        ))}
      </div>

      {/* Mode badge */}
      <div style={{ position: "absolute", bottom: "1.2rem", right: "1.5rem", zIndex: 40, fontFamily: "'Courier New', monospace", color: activeMode.color, fontSize: "0.6rem", letterSpacing: "0.2em", opacity: 0.6 }}>{activeMode.label} — {activeMode.desc}</div>

      {/* Ripples */}
      {ripples.map(r => (
        <div key={r.id} style={{ position: "absolute", left: r.x - 24, top: r.y - 24, width: 48, height: 48, borderRadius: "50%", border: `2px solid ${r.color}`, pointerEvents: "none", zIndex: 30, animation: "rippleOut 0.5s ease-out forwards" }} />
      ))}

      {/* Dots */}
      {dots.map(dot => (
        <button key={dot.id} onClick={e => handleDotClick(e, dot)}
          style={{ position: "absolute", left: dot.x, top: dot.y, width: DOT_SIZE, height: DOT_SIZE, borderRadius: "50%", background: `radial-gradient(circle at 35% 35%, ${activeMode.color}ff, ${activeMode.color}88)`, border: `2px solid ${activeMode.color}`, cursor: "crosshair", padding: 0, outline: "none", zIndex: 20, boxShadow: `0 0 14px ${activeMode.color}88, 0 0 30px ${activeMode.color}33`, animation: "dotPop 0.12s ease-out" }}>
          {/* shrink timer ring */}
          <svg style={{ position: "absolute", inset: -10, width: "calc(100% + 20px)", height: "calc(100% + 20px)", pointerEvents: "none" }}
            viewBox="0 0 68 68">
            <circle cx="34" cy="34" r="32" fill="none" stroke={activeMode.color} strokeWidth="1.5" opacity="0.3"
              style={{ animation: `shrinkRing ${MODES[mode].duration}ms linear forwards`, strokeDasharray: "201", strokeDashoffset: "0", transformOrigin: "34px 34px" }} />
          </svg>
        </button>
      ))}

      <style>{`
        @keyframes dotPop {
          0%   { transform: scale(0.3); opacity: 0; }
          70%  { transform: scale(1.15); }
          100% { transform: scale(1); opacity: 1; }
        }
        @keyframes rippleOut {
          0%   { transform: scale(1); opacity: 0.8; }
          100% { transform: scale(2.5); opacity: 0; }
        }
        @keyframes shrinkRing {
          0%   { r: 32; opacity: 0.35; }
          80%  { opacity: 0.55; }
          100% { r: 17; opacity: 0; }
        }
      `}</style>
    </div>
  );
}
