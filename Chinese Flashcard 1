import React, { useState, useMemo, useCallback, useEffect, useRef } from 'react';

/* =========================
   Constants & Utilities
   ========================= */

const LEVELS = ['HSK1', 'HSK2', 'HSK3', 'HSK4', 'Combined', 'Review', 'Extra'];
const MODES = { ENGLISH: 'english', PINYIN: 'pinyin' };
const STORAGE_KEY = 'chinese_flashcards_state_mobile_extra_theme_kb_v1';

const normalize = (s = '') =>
  s
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/[\s\-_.]+/g, ' ')
    .trim();

const makeShuffledOrder = (n) => {
  const arr = Array.from({ length: n }, (_, i) => i);
  for (let i = n - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
};

const getCardKey = (card) => `${card.chinese}::${card.pinyin}`;

// Basic validator for Extra add form (CJK blocks)
const CHINESE_RE = /^[\u3400-\u9FFF\uF900-\uFAFF]+$/;

/* =========================
   Static vocab (trimmed samples)
   ========================= */

const VOCAB = (() => {
  const HSK1 = [
    { chinese: '爱', pinyin: 'ài', english: 'to love' },
    { chinese: '八', pinyin: 'bā', english: 'eight' },
    { chinese: '爸爸', pinyin: 'bàba', english: 'dad' },
    { chinese: '杯子', pinyin: 'bēizi', english: 'cup' },
    { chinese: '北京', pinyin: 'Běijīng', english: 'Beijing' }
    // … add more
  ];
  const HSK2 = [
    { chinese: '吧', pinyin: 'ba', english: 'particle' },
    { chinese: '白', pinyin: 'bái', english: 'white' },
    { chinese: '百', pinyin: 'bǎi', english: 'hundred' },
    { chinese: '帮助', pinyin: 'bāngzhù', english: 'to help' },
    { chinese: '报纸', pinyin: 'bàozhǐ', english: 'newspaper' }
    // … add more
  ];
  const HSK3 = [
    { chinese: '阿姨', pinyin: 'āyí', english: 'aunt' },
    { chinese: '安静', pinyin: 'ānjìng', english: 'quiet' },
    { chinese: '帮忙', pinyin: 'bāngmáng', english: 'to help' },
    { chinese: '打扫', pinyin: 'dǎsǎo', english: 'to clean' },
    { chinese: '打算', pinyin: 'dǎsuàn', english: 'to plan' },
    { chinese: '经理', pinyin: 'jīnglǐ', english: 'manager' },
    { chinese: '简单', pinyin: 'jiǎndān', english: 'simple' },
    { chinese: '旧', pinyin: 'jiù', english: 'old (used)' }
    // … add more
  ];
  const HSK4 = [
    { chinese: '安排', pinyin: 'ānpái', english: 'to arrange' },
    { chinese: '保护', pinyin: 'bǎohù', english: 'to protect' },
    { chinese: '表示', pinyin: 'biǎoshì', english: 'to express' },
    { chinese: '解决', pinyin: 'jiějué', english: 'to solve' },
    { chinese: '适合', pinyin: 'shìhé', english: 'to suit; fit' },
    { chinese: '友谊', pinyin: 'yǒuyì', english: 'friendship' },
    { chinese: '影响', pinyin: 'yǐngxiǎng', english: 'influence; affect' },
    { chinese: '方面', pinyin: 'fāngmiàn', english: 'aspect' }
    // … add more
  ];
  const Combined = [...HSK1, ...HSK2, ...HSK3, ...HSK4];
  return { HSK1, HSK2, HSK3, HSK4, Combined };
})();

/* =========================
   Hook: keyboard-safe viewport & insets
   ========================= */

function useKeyboardSafeViewport() {
  useEffect(() => {
    const root = document.documentElement;

    const compute = () => {
      const vv = window.visualViewport;
      const vh = (vv?.height ?? window.innerHeight) * 0.01;
      root.style.setProperty('--vh', `${vh}px`);

      // Estimated keyboard height = innerHeight - visualViewport.height (>= 0)
      const kb = Math.max(0, window.innerHeight - (vv?.height ?? window.innerHeight));
      root.style.setProperty('--kb', `${kb}px`);
    };

    compute();

    const vv = window.visualViewport;
    vv?.addEventListener('resize', compute);
    vv?.addEventListener('scroll', compute); // some keyboards move viewport origin
    window.addEventListener('resize', compute);

    return () => {
      vv?.removeEventListener('resize', compute);
      vv?.removeEventListener('scroll', compute);
      window.removeEventListener('resize', compute);
    };
  }, []);
}

/* =========================
   Persistence
   ========================= */

const EMPTY_SCORES = { [MODES.ENGLISH]: { correct: 0, total: 0 }, [MODES.PINYIN]: { correct: 0, total: 0 } };
const emptyPerLevel = LEVELS.reduce((acc, lvl) => ({ ...acc, [lvl]: null }), {});

/* =========================
   Component
   ========================= */

const ChineseFlashcards = () => {
  useKeyboardSafeViewport();

  // Theme CSS (autumn palette) + keyboard-safe tweaks
  const themedCSS = `
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&family=Noto+Serif+SC:wght@400;600&display=swap');

    :root{
      --paper: #FAF6F1;
      --ink: #2A211F;
      --muted: #8B8179;
      --border: #E7DDD2;

      --persimmon: #C85A2E;
      --maple:     #8C3B2A;
      --ginkgo:    #D9A441;
      --tea:       #6B8E62;

      --radius: 16px;
      --pad: 14px;
      --gap: 12px;
      --shadow: 0 10px 30px rgba(0,0,0,0.06);

      /* keyboard insets provided by JS (fallback 0) */
      --vh: 1vh;
      --kb: 0px;
    }

    *{box-sizing:border-box}
    body{
      margin:0;
      background:
        radial-gradient(1000px 600px at -10% -20%, #FFF8EE 0%, transparent 50%),
        radial-gradient(900px 500px at 110% 10%, #FFF2E8 0%, transparent 50%),
        var(--paper);
      color: var(--ink);
      font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, 'Noto Sans CJK SC', 'Noto Sans SC', Arial, sans-serif;
    }

    /* Use dynamic viewport so content height adjusts when keyboard opens */
    .wrap{
      min-height: calc(var(--vh) * 100);
      max-width: 720px;
      margin: 0 auto;
      padding: var(--pad);
      /* Make sure page can scroll above keyboard */
      padding-bottom: calc(env(safe-area-inset-bottom, 0px) + var(--pad));
      /* When browser doesn't resize layout for keyboard, JS --kb adds extra scroll space: */
      scroll-padding-bottom: calc(env(safe-area-inset-bottom, 0px) + var(--kb) + 80px);
    }

    .brand { display:flex; align-items:center; gap:10px; margin-bottom: 6px; }
    .brand .seal {
      width:36px;height:36px;border-radius:10px;
      background: linear-gradient(135deg, var(--persimmon), var(--maple));
      color: #fff; display:flex; align-items:center; justify-content:center;
      box-shadow: var(--shadow);
      font-family: 'Noto Serif SC', serif; font-weight:600; font-size: 20px; letter-spacing: 1px;
    }
    .brand h1 { font-size: 20px; margin: 0; letter-spacing: .2px; font-weight: 700; }
    .subtitle { margin: 4px 0 12px; color: var(--muted); font-size: 13px; }

    .grid{ display:grid; grid-template-columns: 1fr; gap: var(--gap); }

    .tile {
      width: 100%; text-align:left; border:1px solid var(--border); border-radius: var(--radius);
      background: #fff; padding: 14px; min-height: 68px; box-shadow: var(--shadow);
      position: relative; overflow: hidden;
    }
    .tile::after{
      content:''; position:absolute; right:-20px; top:-20px; width:120px; height:120px;
      background: radial-gradient(50% 50% at 50% 50%, rgba(200,90,46,0.20) 0%, rgba(200,90,46,0) 60%);
      transform: rotate(25deg); pointer-events:none;
    }
    .tile:disabled{ opacity:.55; cursor:not-allowed; }
    .tile .title{ font-weight:700; margin-bottom: 3px; }
    .tile .sub{ font-size: 12px; color: var(--muted); }

    .topbar { display:flex; align-items:center; justify-content:space-between; margin-bottom: var(--gap); }
    .btn, button{
      appearance:none; border:1px solid var(--border); background:#fff; color: var(--ink);
      padding: 12px 14px; border-radius: var(--radius); font-size:16px; min-height: 44px;
      transition: transform .06s ease, box-shadow .2s ease, background .2s ease;
      box-shadow: var(--shadow);
    }
    .btn:active { transform: translateY(1px) scale(.995); }
    .btn--ghost { background: transparent; border: none; box-shadow: none; color: var(--maple); padding: 8px 0; }
    .btn--full { width: 100%; }
    .btn--primary { background: linear-gradient(135deg, var(--persimmon), var(--maple)); color: #fff; border-color: transparent; }
    .btn--accent  { background: linear-gradient(135deg, var(--ginkgo), #E3BF5A); color: #3a2d13; border-color: transparent; }
    .btn:disabled { opacity: .6; }

    .card {
      border:1px solid var(--border); border-radius: calc(var(--radius) + 2px);
      padding: 18px; background: #fff; box-shadow: var(--shadow);
      position: relative; overflow:hidden;
    }
    .card::before{
      content:'秋'; position:absolute; right:-8px; bottom:-28px;
      font-family: 'Noto Serif SC', serif; font-size: 200px; line-height: 1; color: var(--maple);
      opacity: .06; letter-spacing: 2px; pointer-events: none; transform: rotate(-8deg);
    }
    .card .ribbon{ position:absolute; top:0; left:0; right:0; height:6px; background: linear-gradient(90deg, var(--persimmon), var(--ginkgo), var(--tea)); opacity: .9; }

    .character { font-family: 'Noto Serif SC', serif; font-size: 58px; line-height: 1; margin: 6px 0 8px; letter-spacing: .5px; }
    .hintline { font-size: 12px; color: var(--muted); }

    .inputrow { display:flex; gap: var(--gap); margin-top: var(--gap); }
    .inputrow input {
      flex: 1; min-height: 50px; padding: 12px 14px; font-size: 16px;
      border: 1px solid var(--border); border-radius: var(--radius); background: #fff;
    }

    .result { min-height: 28px; margin-top: 10px; font-weight: 600; }
    .answers { margin-top: 6px; font-size: 14px; }
    .answers .label { color: var(--muted); margin-right: 4px; }

    .underline{ text-decoration: underline; color: var(--maple); background: transparent; border: none; padding: 0; }
    .reviewRow{ margin-top: 12px; display:flex; gap: var(--gap); justify-content:center }

    .actions{
      position: sticky; bottom: 0;
      padding-top: var(--gap);
      background: linear-gradient(180deg, rgba(250,246,241,0), var(--paper) 35%);
      /* pad for safe area + keyboard (if browser overlays) */
      padding-bottom: calc(env(safe-area-inset-bottom, 0px) + var(--kb) + var(--pad));
      margin-top: var(--gap);
    }
    .actions .row{ display:grid; grid-template-columns: 1fr; gap: var(--gap); }
    .actions .row > * { width: 100%; }

    .progress{ height: 8px; background: #eee5dc; border-radius: 999px; overflow:hidden; margin-top: var(--gap); box-shadow: inset 0 1px 2px rgba(0,0,0,.05); }
    .progress > div { height: 100%; background: linear-gradient(90deg, var(--persimmon), var(--ginkgo)); width: 0; }

    .form{ display:flex; flex-direction: column; gap: 10px; }
    .err{ color:#8C3B2A; font-size: 12px; }
    .mono{ font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', monospace; font-size: 12px; color: var(--muted); }

    .list{ display:flex; flex-direction: column; gap: 8px; }
    .listItem{ display:flex; align-items:center; justify-content:space-between; border:1px solid var(--border); border-radius: var(--radius); padding: 12px; background:#fff; box-shadow: var(--shadow); }

    @media (min-width: 768px) {
      .grid{ grid-template-columns: repeat(2, 1fr); }
      .actions .row{ grid-template-columns: auto 1fr 1fr; }
      .inputrow{ justify-content: center; }
      .inputrow input{ max-width: 420px; }
    }

    @media (prefers-color-scheme: dark){
      :root{
        --paper:#0B0B0C; --ink:#F5F3F0; --border:#28221E; --muted:#AFA59D;
        --persimmon:#E3764E; --maple:#B0503A; --ginkgo:#E3BF5A; --tea:#7EA276;
      }
      body{ background:
        radial-gradient(900px 500px at -10% -20%, #1A1411 0%, transparent 50%),
        radial-gradient(800px 450px at 110% 10%, #1A1713 0%, transparent 50%),
        var(--paper); }
      .tile, .card, .btn, .inputrow input { background:#121212; }
      .progress{ background:#1E1A17; }
      .btn--ghost{ color: #FFDCCF; }
    }
  `;

  /* ------- Persistence ------- */

  const [perLevelState, setPerLevelState] = useState(emptyPerLevel);
  const [reviewKeys, setReviewKeys] = useState([]);
  const [customCards, setCustomCards] = useState([]); // Extra list

  useEffect(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return;
      const parsed = JSON.parse(raw);
      const restoredLevels = {};
      for (const lvl of LEVELS) {
        const s = parsed?.perLevel?.[lvl];
        if (s && Array.isArray(s.order) && typeof s.pos === 'number' && s.score) {
          restoredLevels[lvl] = {
            order: s.order.map(Number).filter(Number.isInteger),
            pos: Math.max(0, Number(s.pos) || 0),
            score: {
              [MODES.ENGLISH]: { correct: Math.max(0, Number(s.score?.[MODES.ENGLISH]?.correct) || 0), total: Math.max(0, Number(s.score?.[MODES.ENGLISH]?.total) || 0) },
              [MODES.PINYIN]:  { correct: Math.max(0, Number(s.score?.[MODES.PINYIN]?.correct)  || 0), total: Math.max(0, Number(s.score?.[MODES.PINYIN]?.total)  || 0) }
            }
          };
        } else {
          restoredLevels[lvl] = null;
        }
      }
      setPerLevelState(restoredLevels);
      setReviewKeys(Array.isArray(parsed?.reviewKeys) ? parsed.reviewKeys.filter(Boolean) : []);
      setCustomCards(Array.isArray(parsed?.customCards) ? parsed.customCards.filter(x => x && x.chinese && x.pinyin && x.english) : []);
    } catch (e) {
      console.warn('Restore failed:', e);
    }
  }, []);

  useEffect(() => {
    const payload = { perLevel: perLevelState, reviewKeys, customCards };
    localStorage.setItem(STORAGE_KEY, JSON.stringify(payload));
  }, [perLevelState, reviewKeys, customCards]);

  const reviewSet = useMemo(() => new Set(reviewKeys), [reviewKeys]);
  const allCards = VOCAB.Combined;
  const reviewCards = useMemo(() => (reviewSet.size ? allCards.filter((c) => reviewSet.has(getCardKey(c))) : []), [allCards, reviewSet]);

  /* ------- App state ------- */

  const [screen, setScreen] = useState('levelSelect'); // 'levelSelect' | 'modeSelect' | 'game' | 'extraMenu' | 'extraAdd' | 'extraList'
  const [level, setLevel] = useState(null);
  const [practiceMode, setPracticeMode] = useState(null);

  const [cards, setCards] = useState([]);
  const [order, setOrder] = useState([]);
  const [pos, setPos] = useState(0);

  const [answer, setAnswer] = useState('');
  const [showResult, setShowResult] = useState(false);
  const [isCorrect, setIsCorrect] = useState(false);
  const [hint, setHint] = useState(false);

  // Refs for keyboard-safe focusing
  const inputRef = useRef(null);

  /* ------- Deck helpers ------- */

  const getListForLevel = useCallback((lvl) => {
    if (lvl === 'Review') return reviewCards;
    if (lvl === 'Extra') return customCards;
    return VOCAB[lvl];
  }, [reviewCards, customCards]);

  const currentCard = useMemo(() => {
    if (!cards.length || !order.length) return null;
    const idx = order[pos] ?? 0;
    return cards[idx];
  }, [cards, order, pos]);

  const saveLevelState = useCallback((lvl, partial) => {
    setPerLevelState((prev) => {
      const current = prev[lvl] || { order: [], pos: 0, score: EMPTY_SCORES };
      const nextScore = partial.score
        ? {
            [MODES.ENGLISH]: { ...(current.score?.[MODES.ENGLISH] || EMPTY_SCORES[MODES.ENGLISH]), ...(partial.score?.[MODES.ENGLISH] || {}) },
            [MODES.PINYIN]:  { ...(current.score?.[MODES.PINYIN]  || EMPTY_SCORES[MODES.PINYIN]),  ...(partial.score?.[MODES.PINYIN]  || {}) }
          }
        : current.score || EMPTY_SCORES;
      return { ...prev, [lvl]: { ...current, ...partial, score: nextScore } };
    });
  }, []);

  const resetDeckForLevel = useCallback((lvl) => {
    const list = getListForLevel(lvl);
    const newOrder = makeShuffledOrder(list.length);
    setOrder(newOrder);
    setPos(0);
    saveLevelState(lvl, { order: newOrder, pos: 0, score: EMPTY_SCORES });
  }, [getListForLevel, saveLevelState]);

  const ensureDeckSynced = useCallback((lvl) => {
    const list = getListForLevel(lvl);
    const saved = perLevelState[lvl];
    if (saved && Array.isArray(saved.order) && saved.order.length === list.length) {
      setOrder(saved.order);
      setPos(Math.min(saved.pos ?? 0, saved.order.length - 1));
    } else {
      const fresh = makeShuffledOrder(list.length);
      setOrder(fresh);
      setPos(0);
      saveLevelState(lvl, { order: fresh, pos: 0, score: EMPTY_SCORES });
    }
  }, [getListForLevel, perLevelState, saveLevelState]);

  /* ------- Navigation ------- */

  const startLevel = useCallback((lvl) => {
    const list = getListForLevel(lvl);
    setLevel(lvl);
    setCards(list);

    if (lvl === 'Extra') { setScreen('extraMenu'); return; }
    if (lvl === 'Review' && list.length === 0) return;

    const saved = perLevelState[lvl];
    if (saved && Array.isArray(saved.order) && saved.order.length === list.length) {
      setOrder(saved.order);
      setPos(Math.min(saved.pos ?? 0, saved.order.length - 1));
    } else {
      const fresh = makeShuffledOrder(list.length);
      setOrder(fresh);
      setPos(0);
      saveLevelState(lvl, { order: fresh, pos: 0, score: EMPTY_SCORES });
    }

    setPracticeMode(null);
    setAnswer(''); setShowResult(false); setIsCorrect(false); setHint(false);
    setScreen('modeSelect');
  }, [getListForLevel, perLevelState, saveLevelState]);

  const beginPractice = (mode) => {
    if (level === 'Extra') {
      setCards(getListForLevel('Extra'));
      ensureDeckSynced('Extra');
    }
    setPracticeMode(mode);
    setAnswer(''); setShowResult(false); setIsCorrect(false); setHint(false);
    setScreen('game');

    // focus input on next paint and center it
    requestAnimationFrame(() => {
      inputRef.current?.focus();
      inputRef.current?.scrollIntoView({ block: 'center', behavior: 'smooth' });
    });
  };

  const goBackHome = () => {
    setScreen('levelSelect'); setPracticeMode(null); setLevel(null);
    setCards([]); setOrder([]); setPos(0);
    setAnswer(''); setShowResult(false); setIsCorrect(false); setHint(false);
  };

  /* ------- Game flow ------- */

  const nextCard = useCallback((reshuffleIfEnd = true) => {
    if (!level) return;
    const list = getListForLevel(level);
    const nextPos = pos + 1;
    if (nextPos < order.length) {
      setPos(nextPos);
      saveLevelState(level, { pos: nextPos });
    } else if (reshuffleIfEnd) {
      const fresh = makeShuffledOrder(list.length);
      setOrder(fresh); setPos(0);
      saveLevelState(level, { order: fresh, pos: 0 });
    }
    setAnswer(''); setShowResult(false); setIsCorrect(false); setHint(false);
    // refocus on next card
    requestAnimationFrame(() => {
      inputRef.current?.focus();
      inputRef.current?.scrollIntoView({ block: 'center', behavior: 'smooth' });
    });
  }, [level, getListForLevel, order.length, pos, saveLevelState]);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!currentCard || showResult) return;
    if (!answer.trim()) return;

    const a = normalize(answer);
    let ok = false;
    if (practiceMode === MODES.ENGLISH) ok = a === normalize(currentCard.english);
    else if (practiceMode === MODES.PINYIN) ok = a === normalize(currentCard.pinyin);
    else ok = a === normalize(currentCard.english) || a === normalize(currentCard.pinyin);

    setIsCorrect(ok);
    setShowResult(true);
  };

  /* ------- Review controls ------- */

  const inReview = currentCard ? reviewSet.has(getCardKey(currentCard)) : false;
  const addToReview = () => { if (currentCard) { const k = getCardKey(currentCard); setReviewKeys((prev) => (prev.includes(k) ? prev : [...prev, k])); } };
  const removeFromReview = () => {
    if (!currentCard) return;
    const key = getCardKey(currentCard);
    setReviewKeys((prev) => prev.filter((k) => k !== key));
    if (level === 'Review') {
      const newCards = reviewCards.filter((c) => getCardKey(c) !== key);
      setCards(newCards);
      const fresh = makeShuffledOrder(newCards.length);
      setOrder(fresh); setPos(0);
      saveLevelState('Review', { order: fresh, pos: 0 });
    }
  };

  /* ------- Extra: Add New / List ------- */

  const [newChinese, setNewChinese] = useState('');
  const [newPinyin, setNewPinyin] = useState('');
  const [newEnglish, setNewEnglish] = useState('');
  const [formError, setFormError] = useState('');

  const handleAddNew = (e) => {
    e.preventDefault();
    const c = newChinese.trim();
    const p = newPinyin.trim();
    const en = newEnglish.trim();
    if (!c || !p || !en) { setFormError('Please fill all fields.'); return; }
    if (!CHINESE_RE.test(c)) { setFormError('Character field must contain only Chinese characters.'); return; }
    const key = getCardKey({ chinese: c, pinyin: p, english: en });
    const exists = customCards.some((it) => getCardKey(it) === key);
    if (exists) { setFormError('This entry already exists in your Extra list.'); return; }

    setCustomCards((prev) => [...prev, { chinese: c, pinyin: p, english: en }]);
    setFormError(''); setNewChinese(''); setNewPinyin(''); setNewEnglish('');
  };

  const handleDeleteCustom = (index) => {
    const item = customCards[index];
    const key = getCardKey(item);
    setCustomCards((prev) => prev.filter((_, i) => i !== index));
    setReviewKeys((prev) => prev.filter((k) => k !== key));
  };

  /* ------- UI helpers ------- */

  const tileSubtitle = (lvl) => {
    const list = getListForLevel(lvl);
    if (lvl === 'Review' && list.length === 0) return '0 cards • add via ⭐ Review';
    if (lvl === 'Extra' && list.length === 0) return '0 custom • add some first';
    const saved = perLevelState[lvl];
    if (!saved) return `${list.length} cards • not started`;
    const at = Math.min((saved.pos ?? 0) + 1, saved.order?.length || list.length);
    const total = saved.order?.length || list.length;
    const eng = saved.score?.[MODES.ENGLISH] || { correct: 0, total: 0 };
    const pin = saved.score?.[MODES.PINYIN] || { correct: 0, total: 0 };
    return `${list.length} • Card ${at}/${total} • EN ${eng.correct}/${eng.total} • PY ${pin.correct}/${pin.total}`;
  };

  const inputPlaceholder =
    practiceMode === MODES.ENGLISH ? 'Type the English meaning' :
    practiceMode === MODES.PINYIN ? 'Type the pinyin (tones optional)' :
    'Type your answer';

  const instructionLine =
    practiceMode === MODES.ENGLISH ? 'Answer in English. Hint shows pinyin.' :
    practiceMode === MODES.PINYIN ? 'Answer in pinyin. Hint shows English.' :
    'Select a practice mode.';

  /* ------- Render ------- */

  return (
    <div className="wrap">
      <style>{themedCSS}</style>

      <div className="brand">
        <div className="seal">秋</div>
        <h1>Chinese Flashcards</h1>
      </div>
      <div className="subtitle">Minimal • Calm • Autumn</div>

      {/* Level Select */}
      {screen === 'levelSelect' && (
        <>
          <div className="grid">
            {LEVELS.map((lvl) => {
              const list = getListForLevel(lvl);
              const disabled = (lvl === 'Review' && list.length === 0);
              return (
                <button
                  key={lvl}
                  className="tile"
                  onClick={() => startLevel(lvl)}
                  disabled={disabled}
                  title={disabled ? 'Add cards to Review after checking an answer' : ''}
                >
                  <div className="title">{lvl}</div>
                  <div className="sub">{tileSubtitle(lvl)}</div>
                </button>
              );
            })}
          </div>

          <div style={{ marginTop: 12, display: 'flex', gap: 12, flexWrap: 'wrap', alignItems: 'center' }}>
            <button
              className="btn btn--ghost"
              onClick={() => {
                localStorage.removeItem(STORAGE_KEY);
                setPerLevelState(emptyPerLevel);
                setReviewKeys([]);
                // keep customCards for safety
              }}
              title="Clear progress & review (keeps Extra list)"
            >
              Reset progress
            </button>
            <span className="mono">Tip: Add tricky cards to <strong>⭐ Review</strong>.</span>
          </div>
        </>
      )}

      {/* Mode Select (HSK/Combined/Review) */}
      {screen === 'modeSelect' && level && level !== 'Extra' && (
        <>
          <div className="topbar">
            <button className="btn btn--ghost" onClick={() => setScreen('levelSelect')}>← Home</button>
            <div style={{ fontSize: 14, color: 'var(--muted)' }}>Level: {level}</div>
          </div>
          <h2 style={{ margin: '4px 0 8px', fontSize: 18 }}>Practice</h2>
          <div className="grid">
            <button className="btn btn--full btn--primary" onClick={() => beginPractice(MODES.ENGLISH)}>English</button>
            <button className="btn btn--full btn--accent" onClick={() => beginPractice(MODES.PINYIN)}>Pinyin</button>
          </div>
          <p className="subtitle" style={{ marginTop: 10 }}>
            English mode: answer in English, hint shows pinyin. Pinyin mode: answer in pinyin, hint shows English.
          </p>
        </>
      )}

      {/* Extra Menu */}
      {screen === 'extraMenu' && (
        <>
          <div className="topbar">
            <button className="btn btn--ghost" onClick={goBackHome}>← Home</button>
            <div style={{ fontSize: 14, color: 'var(--muted)' }}>Extra</div>
          </div>

          <h2 style={{ margin: '4px 0 8px', fontSize: 18 }}>Extra</h2>
          <div className="grid">
            <button className="btn btn--full btn--primary" onClick={() => beginPractice(MODES.ENGLISH)} disabled={customCards.length === 0}>English</button>
            <button className="btn btn--full btn--accent" onClick={() => beginPractice(MODES.PINYIN)} disabled={customCards.length === 0}>Pinyin</button>
            <button className="btn btn--full" onClick={() => setScreen('extraAdd')}>Add New</button>
            <button className="btn btn--full" onClick={() => setScreen('extraList')} disabled={customCards.length === 0}>List of New</button>
          </div>
          <p className="subtitle" style={{ marginTop: 10 }}>Add your own words, then practice them here.</p>
        </>
      )}

      {/* Extra -> Add New */}
      {screen === 'extraAdd' && (
        <>
          <div className="topbar">
            <button className="btn btn--ghost" onClick={() => setScreen('extraMenu')}>← Extra</button>
            <div style={{ fontSize: 14, color: 'var(--muted)' }}>Add New</div>
          </div>

          <div className="card" style={{ textAlign: 'left' }}>
            <div className="ribbon" />
            <form className="form" onSubmit={handleAddNew}>
              <div>
                <label className="mono">Chinese character(s)</label>
                <input
                  value={newChinese}
                  onChange={(e) => setNewChinese(e.target.value)}
                  placeholder="例如：爱 / 朋友 / 学习"
                  inputMode="text"
                  autoCapitalize="off"
                  autoCorrect="off"
                />
              </div>
              <div>
                <label className="mono">Pinyin</label>
                <input
                  value={newPinyin}
                  onChange={(e) => setNewPinyin(e.target.value)}
                  placeholder="ài / péngyou / xuéxí"
                  autoCapitalize="off"
                  autoCorrect="off"
                />
              </div>
              <div>
                <label className="mono">English</label>
                <input
                  value={newEnglish}
                  onChange={(e) => setNewEnglish(e.target.value)}
                  placeholder="to love / friend / to study"
                  autoCapitalize="off"
                  autoCorrect="off"
                />
              </div>
              {formError && <div className="err">{formError}</div>}
              <button className="btn btn--primary" type="submit">Add</button>
            </form>
          </div>
        </>
      )}

      {/* Extra -> List of New */}
      {screen === 'extraList' && (
        <>
          <div className="topbar">
            <button className="btn btn--ghost" onClick={() => setScreen('extraMenu')}>← Extra</button>
            <div style={{ fontSize: 14, color: 'var(--muted)' }}>List of New</div>
          </div>

          {customCards.length === 0 ? (
            <div className="card">No custom items yet. Add some in “Add New”.</div>
          ) : (
            <div className="list">
              {customCards.map((it, idx) => (
                <div key={getCardKey(it)} className="listItem">
                  <div>
                    <div style={{ fontSize: 18, fontFamily: "'Noto Serif SC', serif" }}>{it.chinese}</div>
                    <div className="mono">{it.pinyin} • {it.english}</div>
                  </div>
                  <button className="btn" onClick={() => handleDeleteCustom(idx)}>Delete</button>
                </div>
              ))}
            </div>
          )}
        </>
      )}

      {/* Game (HSK/Combined/Review/Extra) */}
      {screen === 'game' && currentCard && (
        <>
          <div className="topbar">
            <button className="btn btn--ghost" onClick={goBackHome}>← Home</button>
            <div style={{ fontSize: 14, color: 'var(--muted)' }}>
              {level === 'Extra' ? 'Extra' : `Level: ${level}`} • Mode: {practiceMode === MODES.ENGLISH ? 'English' : 'Pinyin'} • {pos + 1}/{order.length}
            </div>
          </div>

          <div className="card">
            <div className="ribbon" />
            <div className="character">{currentCard.chinese}</div>
            <div className="hintline">{instructionLine}</div>

            <form
              onSubmit={handleSubmit}
              className="inputrow"
              onFocus={(e) => {
                // Any input focus: center it and ensure keyboard padding is respected
                if (e.target.tagName === 'INPUT') {
                  e.target.scrollIntoView({ block: 'center', behavior: 'smooth' });
                }
              }}
            >
              <input
                ref={inputRef}
                value={answer}
                onChange={(e) => setAnswer(e.target.value)}
                placeholder={inputPlaceholder}
                disabled={showResult}
                autoCapitalize="off"
                autoCorrect="off"
                autoComplete="off"
                inputMode="text"
              />
              <button className="btn btn--primary" type="submit" disabled={showResult} title={showResult ? 'Already checked' : 'Check'}>
                {showResult ? '✓ Checked' : '▶ Check'}
              </button>
            </form>

            <div className="result">{showResult && (isCorrect ? '✅ Correct!' : '❌ Not quite')}</div>

            <div className="answers">
              {showResult ? (
                <>
                  <div><span className="label">Pinyin:</span> <strong>{currentCard.pinyin}</strong></div>
                  <div><span className="label">English:</span> <strong>{currentCard.english}</strong></div>
                </>
              ) : hint ? (
                practiceMode === MODES.ENGLISH ? (
                  <div><span className="label">Pinyin:</span> <strong>{currentCard.pinyin}</strong></div>
                ) : (
                  <div><span className="label">English:</span> <strong>{currentCard.english}</strong></div>
                )
              ) : (
                <button className="underline" onClick={() => setHint(true)}>Hint</button>
              )}
            </div>

            {showResult && (
              <div className="reviewRow">
                {!inReview ? (
                  <button className="btn" onClick={addToReview}>⭐ Add to Review</button>
                ) : (
                  <button className="btn" onClick={removeFromReview}>Remove from Review</button>
                )}
              </div>
            )}
          </div>

          <div className="actions">
            <div className="row">
              <button className="btn" onClick={() => { resetDeckForLevel(level); setAnswer(''); setShowResult(false); setIsCorrect(false); setHint(false); requestAnimationFrame(() => { inputRef.current?.focus(); inputRef.current?.scrollIntoView({ block: 'center' }); }); }}>⟲ Restart</button>
              <button className="btn" onClick={() => { if (!showResult) nextCard(); }} disabled={showResult}>⏭ Skip</button>
              <button className="btn btn--primary" onClick={() => { if (showResult) nextCard(); }} disabled={!showResult}>▶ Next</button>
            </div>
            <div className="progress"><div style={{ width: `${((pos + 1) / Math.max(order.length, 1)) * 100}%` }} /></div>
          </div>
        </>
      )}

      {screen === 'game' && !currentCard && (
        <div className="card">No card loaded. <button className="underline" onClick={() => setScreen(level === 'Extra' ? 'extraMenu' : 'modeSelect')}>Back</button></div>
      )}
    </div>
  );
};

export default ChineseFlashcards;
