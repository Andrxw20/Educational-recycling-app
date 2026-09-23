import React, { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import {
    Animated,
    Dimensions,
    Easing,
    LayoutChangeEvent,
    LayoutRectangle,
    Pressable,
    StyleSheet,
    Text,
    View,
} from 'react-native';

// ==============================
// Conveyor Sorter – TypeScript (index.tsx)
// ==============================

const TYPES = ['paper', 'plastic', 'glass', 'organic'] as const;
type BinType = typeof TYPES[number];

type Item = {
  id: string;
  type: BinType;
  emoji: string;
  size: number;
  pos: Animated.ValueXY;
  x: number;
  y: number;
  flying: boolean;
  listenerId?: string;
};

type BinsLayout = Record<BinType, LayoutRectangle | null>;

const EMOJI_VARIANTS: Record<BinType, string[]> = {
  paper:   ['📄', 
    '📦',],
  plastic: ['🧴', 
    '🥤',
  ],
  glass:   ['🫙', 
            '🍾', '🥛', '🍼'
  ],
  organic: ['🍌', '🍎', '🥕', '🍞', '🥚', '🍅', '🌽', '🍇', '🍊'
  ],
};

const COLORS: Record<BinType, string> = {
  paper: '#FFDD57',
  plastic: '#4CC9F0',
  glass: '#7BE495',
  organic: '#C49A6C',
};

const BIN_LABEL: Record<BinType, string> = {
  paper: 'Paper',
  plastic: 'Plastic',
  glass: 'Glass',
  organic: 'Organic',
};

const TIPS = [
  'Rinse bottles before recycling!',
  'Glass can be recycled endlessly.',
  'Flatten cardboard to save space.',
  'Banana peels belong in organic!',
  'Caps on? Check local rules.',
  'No greasy pizza boxes in paper.',
];

// --- Layout guards so playfield never shifts ---
const HUD_HEIGHT = 112;  // fixed header height
const TIP_HEIGHT = 36;   // fixed tip bar height (shown or empty)

function choose<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)];
}
function uid(): string {
  return Math.random().toString(36).slice(2);
}

export default function App() {
  const [items, setItems] = useState<Item[]>([]);
  const itemsRef = useRef<Item[]>([]);
  useEffect(() => { itemsRef.current = items; }, [items]);

  const [binsLayout, setBinsLayout] = useState<BinsLayout>({
    paper: null,
    plastic: null,
    glass: null,
    organic: null,
  });
  const [binsRowLayout, setBinsRowLayout] = useState<LayoutRectangle | null>(null);
  const [playfield, setPlayfield] = useState<LayoutRectangle | null>(null);


    const recomputeGeometry = useCallback(() => {
  if (!playfield) return;

  const y = Math.round(Math.max(120, Math.min(playfield.height * 0.4, playfield.height - 220)));
  setBeltY(y);

  const w = Math.max(140, Math.min(260, playfield.width * 0.5));
  const h = 120;
  const x = Math.round((playfield.width - w) / 2);
  const zy = Math.round(y - h / 2);

  setZoneRect({ x, y: zy, w: Math.round(w), h: Math.round(h) });

  // lock after recompute so it stays fixed
  beltLocked.current = true;
  zoneLocked.current = true;
}, [playfield]);

  // Locked overlay geometry
  const [beltY, setBeltY] = useState<number>(220);
  const [zoneRect, setZoneRect] = useState<{ x: number; y: number; w: number; h: number } | null>(null);
  const beltLocked = useRef(false);
  const zoneLocked = useRef(false);

  const [activeId, setActiveId] = useState<string | null>(null);
  const [score, setScore] = useState(0);
  const [lives, setLives] = useState(3);
  const [streak, setStreak] = useState(0);
  const [paused, setPaused] = useState(false);
  const [level, setLevel] = useState(1);
  const [tip, setTip] = useState<string | null>(null);

  // start with 3 or 4 items; +1 every 5 levels
  const [baseConcurrent, setBaseConcurrent] = useState<number>(() => (Math.random() < 0.5 ? 3 : 4));

  const spawnTimer = useRef<ReturnType<typeof setInterval> | null>(null);
  const rafRef = useRef<number>(0);
  const binKeys = useMemo<BinType[]>(() => TYPES.slice() as BinType[], []);

  // Measure & lock belt once
  useEffect(() => {
    if (!playfield || beltLocked.current) return;
    const y = Math.max(120, Math.min(playfield.height * 0.4, playfield.height - 220));
    setBeltY(Math.round(y));
    beltLocked.current = true;
  }, [playfield]);

  // Measure & lock zone once (depends on beltY)
  useEffect(() => {
    if (!playfield || zoneLocked.current) return;
    const w = Math.max(140, Math.min(260, playfield.width * 0.5));
    const h = 120;
    const x = Math.round((playfield.width - w) / 2);
    const y = Math.round(beltY - h / 2);
    setZoneRect({ x, y, w: Math.round(w), h: Math.round(h) });
    zoneLocked.current = true;
  }, [playfield, beltY]);

  // Recompute overlay geometry only on orientation/size change
useEffect(() => {
  const sub = Dimensions.addEventListener('change', () => {
    beltLocked.current = false;
    zoneLocked.current = false;
    recomputeGeometry();   // ⬅️ do it immediately; no need to wait for layout
  });
  return () => sub.remove();
}, [recomputeGeometry]);


  // Level from score
  useEffect(() => {
    const newLevel = Math.floor(score / 25) + 1;
    if (newLevel !== level) setLevel(newLevel);
  }, [score, level]);

  // Difficulty
  const beltSpeed = useMemo(() => {
    const baseSpeed = 70;
    const growth = Math.pow(1.03, Math.max(0, level - 1));
    return baseSpeed * growth;
  }, [level]);

  const spawnIntervalMs = useMemo(() => {
    const base = 1100;
    const perLevel = 45;
    return Math.max(600, base - (level - 1) * perLevel);
  }, [level]);

  const maxConcurrent = useMemo(() => {
    const bonus = Math.floor((level - 1) / 5);
    return baseConcurrent + bonus;
  }, [level, baseConcurrent]);

  const clearSpawn = () => {
    if (spawnTimer.current) {
      clearInterval(spawnTimer.current);
      spawnTimer.current = null;
    }
  };

  const stopAllAnimations = () => {
    itemsRef.current.forEach(it => it.pos.stopAnimation());
  };

  const removeItemById = (id: string) => {
    setItems(cur => {
      const it = cur.find(i => i.id === id);
      if (it?.listenerId) {
        try { it.pos.removeListener(it.listenerId); } catch {}
      }
      return cur.filter(i => i.id !== id);
    });
  };


const resetGame = useCallback(() => {
  clearSpawn();
  cancelAnimationFrame(rafRef.current);
  stopAllAnimations();
  itemsRef.current.forEach(it => it.listenerId && it.pos.removeListener(it.listenerId));
  setItems([]);
  setActiveId(null);
  setScore(0);
  setLives(3);
  setStreak(0);
  setLevel(1);
  setTip(null);
  setPaused(false);
  setBaseConcurrent(Math.random() < 0.5 ? 3 : 4);

  // unlock, then recompute immediately from current playfield
  beltLocked.current = false;
  zoneLocked.current = false;
  recomputeGeometry();     // ⬅️ compute beltY + zoneRect right now
}, [recomputeGeometry]);

  // Spawn loop
  useEffect(() => {
    if (paused || lives <= 0 || !playfield || !zoneRect) return;
    clearSpawn();

    spawnTimer.current = setInterval(() => {
      setItems(prev => {
        if (prev.length >= maxConcurrent) return prev;

        const size = Math.round(52 + Math.random() * 16);
        const type = choose([...TYPES, ...TYPES] as BinType[]);
        const emoji = choose(EMOJI_VARIANTS[type]);

        const startX = -size - 8;
        const y = Math.round(beltY - size / 2 + (Math.random() * 14 - 7));
        const pos = new Animated.ValueXY({ x: startX, y });
        const id = uid();
        const item: Item = { id, type, emoji, size, pos, x: startX, y, flying: false };

        item.listenerId = pos.addListener(v => {
          item.x = v.x;
          item.y = v.y;
        });

        const distance = (playfield.width + size + 20) - startX;
        const duration = Math.max(250, Math.round((distance / beltSpeed) * 1000));

        Animated.timing(pos, {
          toValue: { x: playfield.width + size + 20, y },
          duration,
          easing: Easing.linear,
          useNativeDriver: false,
        }).start(({ finished }) => {
          if (!finished) return;
          removeItemById(id);
          setLives(hp => Math.max(0, hp - 1));
          setStreak(0);
        });

        return [...prev, item];
      });
    }, spawnIntervalMs);

    return clearSpawn;
  }, [paused, lives, spawnIntervalMs, playfield, beltY, beltSpeed, maxConcurrent, zoneRect]);

  // Active detection – nearest to zone center
  useEffect(() => {
    if (!zoneRect || lives <= 0) return;

    let raf = 0;
    const tol = 14;

    const tick = () => {
      const z = zoneRect;
      const list = itemsRef.current;

      let candidate: Item | null = null;
      let bestDist = Number.POSITIVE_INFINITY;

      const zx = z.x + z.w / 2;
      const zy = z.y + z.h / 2;

      for (let i = 0; i < list.length; i++) {
        const it = list[i];
        if (it.flying) continue;

        const cx = it.x + it.size / 2;
        const cy = it.y + it.size / 2;

        const inZone =
          cx >= z.x - tol &&
          cx <= z.x + z.w + tol &&
          cy >= z.y - tol &&
          cy <= z.y + z.h + tol;

        if (inZone) {
          const dx = cx - zx;
          const dy = cy - zy;
          const d2 = dx * dx + dy * dy;
          if (d2 < bestDist) {
            bestDist = d2;
            candidate = it;
          }
        }
      }

      const nextId = candidate ? candidate.id : null;
      setActiveId(prev => (prev === nextId ? prev : nextId));

      raf = requestAnimationFrame(tick);
    };

    raf = requestAnimationFrame(tick);
    rafRef.current = raf;

    return () => cancelAnimationFrame(raf);
  }, [zoneRect, lives]);

  const onLayoutPlayfield = (e: LayoutChangeEvent) => setPlayfield(e.nativeEvent.layout);
  const onLayoutBinsRow = (e: LayoutChangeEvent) => setBinsRowLayout(e.nativeEvent.layout);
  const onLayoutBin = useCallback(
    (type: BinType) => (e: LayoutChangeEvent) => {
      const layout = e.nativeEvent.layout;
      setBinsLayout(prev => ({ ...prev, [type]: layout }));
    },
    [],
  );

  const gameOver = lives <= 0;

  const classifyActive = (pressed: BinType) => {
    if (gameOver || paused) return;
    if (!activeId) return;

    const target = itemsRef.current.find(i => i.id === activeId && !i.flying);
    if (!target) return;

    if (!binsRowLayout) return;
    const destRect = binsLayout[pressed];
    if (!destRect) return;

    // Convert bin rect from bin-row space to playfield space
    const rx = destRect.x + binsRowLayout.x;
    const ry = destRect.y + binsRowLayout.y;
    const destX = rx + destRect.width / 2 - target.size / 2;
    const destY = ry + destRect.height / 2 - target.size / 2;

    if (pressed === target.type) {
      target.flying = true;
      Animated.timing(target.pos, {
        toValue: { x: destX, y: destY },
        duration: 320,
        easing: Easing.out(Easing.cubic),
        useNativeDriver: false,
      }).start(() => {
        removeItemById(target.id);
        const gained = 5 + Math.min(15, streak * 2);
        setScore(s => s + gained);
        setStreak(s => s + 1);
        if (Math.random() < 0.35) setTip(choose(TIPS));
      });
    } else {
      setLives(hp => Math.max(0, hp - 1));
      setStreak(0);
      setTip('Oops! Try a different bin.');

      const readX = () => (target.pos as any).__getValue().x as number;
      const y = target.y;

      Animated.sequence([
        Animated.timing(target.pos, { toValue: { x: readX() - 10, y }, duration: 60, useNativeDriver: false }),
        Animated.timing(target.pos, { toValue: { x: readX() + 10, y }, duration: 60, useNativeDriver: false }),
        Animated.timing(target.pos, { toValue: { x: readX(), y }, duration: 60, useNativeDriver: false }),
      ]).start();
    }
  };

  // Pause/Resume handling
  useEffect(() => {
    if (paused) {
      stopAllAnimations();
      return;
    }
    if (!playfield) return;

    setItems(prev => {
      prev.forEach(it => {
        if (it.flying) return;
        const distance = (playfield.width + it.size + 20) - it.x;
        const duration = Math.max(100, Math.round((distance / beltSpeed) * 1000));
        Animated.timing(it.pos, {
          toValue: { x: playfield.width + it.size + 20, y: it.y },
          duration,
          easing: Easing.linear,
          useNativeDriver: false,
        }).start(({ finished }) => {
          if (!finished) return;
          removeItemById(it.id);
          setLives(hp => Math.max(0, hp - 1));
          setStreak(0);
        });
      });
      return [...prev];
    });
  }, [paused, beltSpeed, playfield]);

  return (
    <View style={styles.container}>
      {/* Fixed-height HUD (prevents reflow on level up) */}
      <View style={styles.hud}>
        <Text style={styles.title} numberOfLines={1}>♻️ Conveyor Sorter</Text>
        <View style={styles.statsRow} >
          <Text style={styles.stat} numberOfLines={1}>Score: {score}</Text>
          <Text style={styles.stat} numberOfLines={1}>Streak: {streak}</Text>
          <Text style={styles.stat} numberOfLines={1}>Lives: {'❤️'.repeat(lives) + '🖤'.repeat(3 - lives)}</Text>
          {/* reserve width for 3 digits to avoid width growth */}
          <Text style={[styles.stat, styles.tabular]} numberOfLines={1}>
            Lvl {String(level).padStart(3, ' ')}
          </Text>
          <Text style={[styles.stat, styles.tabular]} numberOfLines={1}>
            Items: {String(maxConcurrent).padStart(2, ' ')}
          </Text>
        </View>
      </View>

      {/* Fixed-height tip spacer (doesn't push playfield) */}
      <View style={styles.tipHolder}>
        {tip ? (
          <View style={styles.tipBar}>
            <Text style={styles.tipText} numberOfLines={1}>{tip}</Text>
          </View>
        ) : null}
      </View>

      {/* Playfield never moves now */}
      <View style={styles.playfield} onLayout={onLayoutPlayfield} collapsable={false}>
        {/* Static overlay that contains belt + zone; rasterized to avoid jitter */}
        <View
          pointerEvents="none"
          style={StyleSheet.absoluteFill}
          renderToHardwareTextureAndroid
          // @ts-ignore RN iOS prop
          shouldRasterizeIOS
        >
          {/* Belt guide */}
          <View
            style={{
              position: 'absolute',
              left: 0,
              right: 0,
              top: beltY - 4,
              height: 8,
              backgroundColor: '#0B1220',
              borderTopWidth: 1,
              borderBottomWidth: 1,
              borderColor: '#1e293b',
            }}
          />
          {/* Scan Zone */}
          {zoneRect && (
            <View
              style={[
                styles.zone,
                {
                  left: zoneRect.x,
                  top: zoneRect.y,
                  width: zoneRect.w,
                  height: zoneRect.h,
                },
              ]}
            />
          )}
        </View>

        {/* Items */}
        {items.map(it => {
          const isActive = it.id === activeId;
          return (
            <Animated.View
              key={it.id}
              style={[
                styles.item,
                {
                  width: it.size,
                  height: it.size,
                  transform: [
                    { translateX: it.pos.x },
                    { translateY: it.pos.y },
                    { scale: isActive ? 1.06 : 1 },
                  ],
                  borderColor: isActive ? '#FACC15' : COLORS[it.type],
                  shadowOpacity: isActive ? 0.45 : 0.25,
                  zIndex: isActive ? 5 : 3,
                  opacity: isActive ? 1 : 0.95,
                },
              ]}
            >
              <Text style={styles.itemEmoji}>{it.emoji}</Text>
            </Animated.View>
          );
        })}

        {/* Bins Row (as buttons) */}
        <View style={styles.binsRow} onLayout={onLayoutBinsRow}>
          {binKeys.map((type) => (
            <Pressable
              key={type}
              onLayout={onLayoutBin(type)}
              onPress={() => classifyActive(type)}
              android_ripple={{ color: 'rgba(0,0,0,0.06)', borderless: false }}
              style={[styles.binBtn, { backgroundColor: COLORS[type] }]}
            >
              <Text style={styles.binEmoji}>{EMOJI_VARIANTS[type][0]}</Text>
              <Text style={styles.binLabel}>{BIN_LABEL[type]}</Text>
            </Pressable>
          ))}
        </View>
      </View>

      {/* Footer */}
      <View style={styles.footer}>
        <Pressable style={styles.button} onPress={() => setPaused(p => !p)}>
          <Text style={styles.buttonText}>{paused ? 'Resume' : 'Pause'}</Text>
        </Pressable>
        <Pressable style={[styles.button, styles.secondary]} onPress={resetGame}>
          <Text style={styles.buttonText}>Restart</Text>
        </Pressable>
      </View>

      {/* Game Over */}
      {gameOver && (
        <View style={styles.overlay}>
          <Text style={styles.overlayTitle}>Great Try! 🌟</Text>
          <Text style={styles.overlayText}>Final Score: {score}</Text>
          <Pressable style={[styles.button, styles.overlayBtn]} onPress={resetGame}>
            <Text style={styles.buttonText}>Play Again</Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#0F172A' },

  // Fixed-height HUD + no wrap
  hud: { height: HUD_HEIGHT, paddingTop: 32, paddingHorizontal: 16, paddingBottom: 8, justifyContent: 'center' },
  title: { color: 'white', fontSize: 22, fontWeight: '800', letterSpacing: 0.5 },
  statsRow: { flexDirection: 'row', gap: 12, marginTop: 6, flexWrap: 'nowrap', alignItems: 'center' },
  stat: { color: '#E2E8F0', fontSize: 14 },
  tabular: { fontVariant: ['tabular-nums'] as any }, // monospaced digits to avoid width jumps

  // Fixed-height tip holder
  tipHolder: { height: TIP_HEIGHT, alignItems: 'center', justifyContent: 'center' },
  tipBar: { backgroundColor: '#1E293B', paddingVertical: 6, paddingHorizontal: 12, borderRadius: 8, maxWidth: '90%' },
  tipText: { color: '#BAE6FD', fontSize: 14 },

  playfield: { flex: 1 },

  zone: {
    position: 'absolute',
    borderWidth: 2,
    borderStyle: 'dashed',
    borderRadius: 12,
    borderColor: '#94A3B8',
    backgroundColor: 'rgba(148,163,184,0.08)',
  },

  item: {
    position: 'absolute',
    borderWidth: 3,
    borderRadius: 12,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#0B1220',
    shadowColor: '#000',
    shadowOpacity: 0.25,
    shadowRadius: 8,
    elevation: 4,
  },
  itemEmoji: { fontSize: 28 },

  binsRow: {
    position: 'absolute',
    bottom: 18,
    left: 10,
    right: 10,
    flexDirection: 'row',
    gap: 10,
    justifyContent: 'space-between',
  },
  binBtn: {
    flex: 1,
    height: 110,
    borderRadius: 16,
    alignItems: 'center',
    justifyContent: 'center',
    shadowColor: '#000',
    shadowOpacity: 0.25,
    shadowRadius: 8,
    elevation: 4,
  },
  binEmoji: { fontSize: 24 },
  binLabel: { color: '#0B1220', fontSize: 14, fontWeight: '800', marginTop: 6 },

  footer: { flexDirection: 'row', justifyContent: 'center', gap: 12, paddingVertical: 14 },
  button: { backgroundColor: '#2563EB', paddingHorizontal: 16, paddingVertical: 10, borderRadius: 12 },
  secondary: { backgroundColor: '#0EA5E9' },
  buttonText: { color: 'white', fontWeight: '700' },

  overlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.6)',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
  },

  overlayTitle: { color: 'white', fontSize: 28, fontWeight: '900', marginBottom: 8 },
  overlayText: { color: 'white', fontSize: 18, marginBottom: 16 },
  overlayBtn: { backgroundColor: '#10B981' },
});
