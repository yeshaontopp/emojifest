import { useState, useEffect, useCallback, useRef } from 'react';
import removeBackground from '@imgly/background-removal';

type Screen = 'camera' | 'preview' | 'library';
type StyleMode = 'glossy' | 'y2k' | 'minimal';

// State shared across the app
interface AppState {
  stickerBlob: Blob | null;       // transparent-bg PNG blob
  stickerUrl: string | null;      // object URL for display
  vault: VaultItem[];
}

interface VaultItem {
  id: string;
  url: string;
  blob: Blob;
}

const INITIAL_VAULT: VaultItem[] = [];

// ─── Helpers ──────────────────────────────────────────────────────────────────

function uid() {
  return Math.random().toString(36).slice(2);
}

// ─── Icons ───────────────────────────────────────────────────────────────────

function SparkleIcon({ size = 14, className = '', style }: { size?: number; className?: string; style?: React.CSSProperties }) {
  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill="currentColor" className={className} style={style}>
      <path d="M12 1l2.5 8.5L23 12l-8.5 2.5L12 23l-2.5-8.5L1 12l8.5-2.5z" />
    </svg>
  );
}

// ─── Toast ────────────────────────────────────────────────────────────────────

function Toast({ message, onDone }: { message: string; onDone: () => void }) {
  useEffect(() => {
    const t = setTimeout(onDone, 2600);
    return () => clearTimeout(t);
  }, [onDone]);

  return (
    <div className="fixed bottom-32 left-1/2 z-50 pointer-events-none"
      style={{ transform: 'translateX(-50%)', animation: 'toast-in .25s ease forwards', whiteSpace: 'nowrap' }}>
      <div className="glass rounded-2xl px-5 py-3 flex items-center gap-2 shadow-xl"
        style={{ border: '1.5px solid rgba(255,105,180,.4)' }}>
        <span className="text-base">📋</span>
        <span className="font-bold text-pink-600 text-sm">{message}</span>
        <SparkleIcon size={12} className="text-pink-400 sparkle" />
      </div>
    </div>
  );
}

// ─── Loading overlay ─────────────────────────────────────────────────────────

function LoadingOverlay({ label, sub }: { label: string; sub?: string }) {
  return (
    <div className="absolute inset-0 z-30 flex flex-col items-center justify-center gap-4 rounded-3xl"
      style={{ background: 'rgba(20,10,30,.8)', backdropFilter: 'blur(6px)' }}>
      <div className="relative w-16 h-16 flex items-center justify-center">
        <div className="absolute inset-0 rounded-full"
          style={{ border: '3px solid transparent', borderTopColor: '#FF69B4', borderRightColor: '#FF1493', animation: 'spin .8s linear infinite' }} />
        <SparkleIcon size={22} className="text-pink-400 sparkle" />
      </div>
      <div className="flex flex-col items-center gap-1">
        <p className="font-bubble text-white text-base tracking-wide drop-shadow">{label}</p>
        {sub && <p className="text-pink-300 text-xs opacity-80">{sub}</p>}
      </div>
    </div>
  );
}

// ─── Shared chrome ────────────────────────────────────────────────────────────

function TopBar({ title, onBack, rightSlot }: { title: string; onBack?: () => void; rightSlot?: React.ReactNode }) {
  return (
    <div className="glass-nav flex-shrink-0 flex items-center justify-between px-5 pt-10 pb-4">
      <div className="w-9 h-9 flex items-center justify-center rounded-full flex-shrink-0"
        style={{ background: 'rgba(255,255,255,.25)' }}>
        {onBack
          ? <button onClick={onBack} className="text-white text-base font-bold leading-none w-full h-full flex items-center justify-center">←</button>
          : <span className="text-white text-sm">⚙️</span>}
      </div>
      <span className="font-bubble text-white text-xl tracking-wide drop-shadow-sm truncate mx-2">{title}</span>
      <div className="w-9 h-9 flex items-center justify-center rounded-full flex-shrink-0"
        style={{ background: 'rgba(255,255,255,.25)' }}>
        {rightSlot ?? <span className="opacity-0">·</span>}
      </div>
    </div>
  );
}

function BottomNav({ active, onNavigate }: { active: Screen; onNavigate: (s: Screen) => void }) {
  const items: { id: Screen; icon: string; label: string }[] = [
    { id: 'camera', icon: '📸', label: 'Studio' },
    { id: 'preview', icon: '✨', label: 'Create' },
    { id: 'library', icon: '🗃️', label: 'Vault' },
  ];
  return (
    <div className="flex-shrink-0 px-4 pb-5 pt-2">
      <div className="glass rounded-2xl px-2 py-2 flex">
        {items.map(item => (
          <button key={item.id} onClick={() => onNavigate(item.id)}
            className={`flex-1 flex flex-col items-center gap-1 py-1.5 rounded-xl transition-all duration-200 ${active === item.id ? 'bg-gradient-to-b from-pink-400/30 to-pink-500/20' : ''}`}>
            <span className="text-xl leading-none">{item.icon}</span>
            <span className={`text-[10px] font-bold leading-none ${active === item.id ? 'text-pink-600' : 'text-pink-400/60'}`}>{item.label}</span>
          </button>
        ))}
      </div>
    </div>
  );
}

// ─── Screen 1: Camera Studio ──────────────────────────────────────────────────

type LoadPhase = 'idle' | 'reading' | 'removing-bg' | 'done';

function CameraScreen({
  onNavigate,
  onProcessed,
}: {
  onNavigate: (s: Screen) => void;
  onProcessed: (blob: Blob, url: string) => void;
}) {
  const [style, setStyle] = useState<StyleMode>('glossy');
  const [phase, setPhase] = useState<LoadPhase>('idle');
  const [previewUrl, setPreviewUrl] = useState<string | null>(null);

  // Two separate refs so 'capture' opens camera, 'upload' opens gallery
  const uploadRef = useRef<HTMLInputElement>(null);
  const captureRef = useRef<HTMLInputElement>(null);

  const processFile = useCallback(async (file: File) => {
    // 1. Show the raw image immediately
    const rawUrl = URL.createObjectURL(file);
    setPreviewUrl(rawUrl);
    setPhase('reading');

    await new Promise(r => setTimeout(r, 600)); // brief pause so user sees their photo

    // 2. Remove background
    setPhase('removing-bg');
    try {
      const resultBlob = await removeBackground(file, {
        publicPath: 'https://cdn.jsdelivr.net/npm/@imgly/background-removal@1.4.5/dist/',
        model: 'small',
        output: { format: 'image/png', quality: 0.9 },
      });
      const resultUrl = URL.createObjectURL(resultBlob);
      URL.revokeObjectURL(rawUrl);
      setPreviewUrl(resultUrl);
      setPhase('done');
      onProcessed(resultBlob, resultUrl);

      // Auto-navigate after brief preview
      setTimeout(() => onNavigate('preview'), 700);
    } catch (err) {
      console.error('Background removal failed:', err);
      // Fallback: use raw image without bg removal
      const fallbackBlob = new Blob([await file.arrayBuffer()], { type: 'image/png' });
      onProcessed(fallbackBlob, rawUrl);
      setPhase('done');
      setTimeout(() => onNavigate('preview'), 700);
    }
  }, [onProcessed, onNavigate]);

  const handleFileChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) processFile(file);
    // Reset input so same file can be re-selected
    e.target.value = '';
  }, [processFile]);

  const overlayLabel =
    phase === 'reading' ? 'Loading photo…' :
    phase === 'removing-bg' ? 'Cutting out background…' :
    phase === 'done' ? 'Done! ✦' : '';

  const overlaySub =
    phase === 'removing-bg' ? 'AI magic in progress ✨' : undefined;

  const styles: { id: StyleMode; label: string }[] = [
    { id: 'glossy', label: '✦ Glossy 3D' },
    { id: 'y2k', label: '◈ Y2K Pixel' },
    { id: 'minimal', label: '◻ Minimal Line' },
  ];

  return (
    <div className="flex flex-col h-full"
      style={{ background: 'linear-gradient(160deg,#fff 0%,#FFF0F8 40%,#FFE4F4 100%)' }}>
      <div className="absolute -top-16 -right-16 w-64 h-64 rounded-full opacity-30 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF69B4 0%,transparent 70%)' }} />
      <div className="absolute bottom-32 -left-20 w-56 h-56 rounded-full opacity-20 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF1493 0%,transparent 70%)' }} />

      {/* Hidden inputs */}
      <input ref={uploadRef} type="file" accept="image/*" className="hidden" onChange={handleFileChange} />
      <input ref={captureRef} type="file" accept="image/*" capture="user" className="hidden" onChange={handleFileChange} />

      <TopBar title="Emoji Studio ✦" />

      <div className="flex-1 flex flex-col overflow-y-auto px-5 py-4 gap-4 relative z-10">

        {/* Viewfinder */}
        <div className="viewfinder viewfinder-corners rounded-3xl overflow-hidden relative w-full flex-shrink-0"
          style={{ height: '280px' }}>

          {/* Background / idle state */}
          {!previewUrl && (
            <div className="absolute inset-0 flex flex-col items-center justify-center gap-3"
              style={{ background: 'linear-gradient(180deg,rgba(20,10,30,.85) 0%,rgba(40,20,50,.9) 100%)' }}>
              <div className="w-28 h-28 rounded-full flex items-center justify-center"
                style={{ background: 'rgba(255,105,180,.15)', border: '1.5px solid rgba(255,105,180,.4)' }}>
                <span className="text-5xl opacity-60">🫥</span>
              </div>
              <p className="text-pink-300 text-xs font-semibold opacity-70">tap Upload or Capture to start</p>
              {[...Array(6)].map((_, i) => (
                <div key={i} className="absolute w-1.5 h-1.5 rounded-full"
                  style={{
                    background: 'rgba(255,105,180,.6)',
                    top: `${[10,18,12,72,8,60][i]}%`,
                    left: `${[15,80,48,22,65,55][i]}%`,
                    animation: `sparkle-pulse ${1.5 + i * .3}s ease-in-out infinite`,
                  }} />
              ))}
            </div>
          )}

          {/* User's photo */}
          {previewUrl && (
            <img src={previewUrl} alt="Your photo"
              className="absolute inset-0 w-full h-full object-cover"
              style={{
                objectPosition: 'center top',
                filter: phase === 'done' ? 'drop-shadow(0 0 20px rgba(255,105,180,.6))' : 'none',
                background: phase === 'done' ? 'transparent' : undefined,
              }} />
          )}

          {/* Loading overlay */}
          {(phase === 'reading' || phase === 'removing-bg') && (
            <LoadingOverlay label={overlayLabel} sub={overlaySub} />
          )}

          {/* Done flash */}
          {phase === 'done' && (
            <div className="absolute inset-0 flex items-end justify-center pb-4 pointer-events-none">
              <div className="glass rounded-full px-4 py-1.5 flex items-center gap-1.5">
                <span className="text-xs font-bold text-pink-600">Sticker ready!</span>
                <SparkleIcon size={11} className="text-pink-400 sparkle" />
              </div>
            </div>
          )}

          {/* Corner brackets */}
          <div className="absolute top-3 right-3 w-7 h-7 pointer-events-none"
            style={{ borderTop: '2.5px solid rgba(255,105,180,.8)', borderRight: '2.5px solid rgba(255,105,180,.8)', borderRadius: '0 4px 0 0' }} />
          <div className="absolute bottom-3 left-3 w-7 h-7 pointer-events-none"
            style={{ borderBottom: '2.5px solid rgba(255,105,180,.8)', borderLeft: '2.5px solid rgba(255,105,180,.8)', borderRadius: '0 0 0 4px' }} />

          {/* Face-detect badge (hide while processing) */}
          {phase === 'idle' && (
            <div className="absolute top-4 left-1/2 -translate-x-1/2 glass rounded-full px-3 py-1 flex items-center gap-1.5">
              <span className="w-1.5 h-1.5 rounded-full bg-pink-400 animate-pulse" />
              <span className="text-xs font-semibold text-pink-600">FACE DETECT</span>
            </div>
          )}

          {/* Upload / Capture buttons */}
          {phase === 'idle' && (
            <div className="absolute bottom-4 left-1/2 -translate-x-1/2 flex gap-3">
              <button onClick={() => uploadRef.current?.click()}
                className="glass rounded-full px-4 py-2 flex items-center gap-1.5 active:scale-95 transition-transform">
                <span className="text-sm">📁</span>
                <span className="text-xs font-bold text-pink-600">Upload</span>
              </button>
              <button onClick={() => captureRef.current?.click()}
                className="glass rounded-full px-4 py-2 flex items-center gap-1.5 active:scale-95 transition-transform">
                <span className="text-sm">📷</span>
                <span className="text-xs font-bold text-pink-600">Capture</span>
              </button>
            </div>
          )}
        </div>

        {/* Style toggle */}
        <div className="flex flex-col gap-2">
          <p className="text-center text-xs font-bold text-pink-400 uppercase tracking-widest">Style Mode</p>
          <div className="toggle-pill rounded-2xl p-1 flex gap-1">
            {styles.map(s => (
              <button key={s.id} onClick={() => setStyle(s.id)}
                className={`flex-1 py-2.5 rounded-xl text-xs font-bold transition-all duration-200 ${style === s.id ? 'toggle-pill-active' : 'text-pink-500'}`}>
                {s.label}
              </button>
            ))}
          </div>
        </div>

        {/* Generate — re-opens gallery */}
        <button
          onClick={() => uploadRef.current?.click()}
          disabled={phase !== 'idle'}
          className="btn-generate glow-pink w-full py-4 rounded-2xl text-white text-lg shadow-lg active:scale-[.98] transition-all disabled:opacity-60">
          {phase === 'idle' ? '✦ Choose Photo ✦' : '⏳ Processing…'}
        </button>
      </div>

      <BottomNav active="camera" onNavigate={onNavigate} />
    </div>
  );
}

// ─── Screen 2: Emoji Preview & Customizer ────────────────────────────────────

function PreviewScreen({
  onNavigate,
  stickerUrl,
  stickerBlob,
  onSaveToVault,
  onToast,
}: {
  onNavigate: (s: Screen) => void;
  stickerUrl: string | null;
  stickerBlob: Blob | null;
  onSaveToVault: (blob: Blob, url: string) => void;
  onToast: (msg: string) => void;
}) {
  const [acc, setAcc] = useState({ stars: true, hairclip: false, sticker: true });
  const [saved, setSaved] = useState(false);
  const [copying, setCopying] = useState(false);
  const toggle = (k: keyof typeof acc) => setAcc(a => ({ ...a, [k]: !a[k] }));

  const accList = [
    { key: 'stars' as const, icon: '✦', label: 'Glitter Stars' },
    { key: 'hairclip' as const, icon: '🎀', label: 'Hair Clip' },
    { key: 'sticker' as const, icon: '⬡', label: 'Die-Cut Border' },
  ];

  const handleSave = () => {
    if (!stickerBlob || !stickerUrl) return;
    onSaveToVault(stickerBlob, stickerUrl);
    setSaved(true);
    setTimeout(() => onNavigate('library'), 700);
  };

  const handleCopy = async () => {
    if (!stickerBlob) {
      onToast('No sticker to copy yet!');
      return;
    }
    setCopying(true);
    try {
      // Ensure the blob is PNG for ClipboardItem
      const pngBlob = stickerBlob.type === 'image/png'
        ? stickerBlob
        : await (async () => {
            const img = new Image();
            img.src = URL.createObjectURL(stickerBlob);
            await new Promise(r => { img.onload = r; });
            const canvas = document.createElement('canvas');
            canvas.width = img.naturalWidth;
            canvas.height = img.naturalHeight;
            canvas.getContext('2d')!.drawImage(img, 0, 0);
            return new Promise<Blob>((res, rej) => canvas.toBlob(b => b ? res(b) : rej(), 'image/png'));
          })();

      await navigator.clipboard.write([
        new ClipboardItem({ 'image/png': pngBlob }),
      ]);
      onToast('Copied PNG to clipboard! 🎉');
    } catch (err) {
      console.error(err);
      onToast('Copy not supported on this browser');
    } finally {
      setCopying(false);
    }
  };

  const hasSticker = !!stickerUrl;

  return (
    <div className="flex flex-col h-full"
      style={{ background: 'linear-gradient(160deg,#fff 0%,#FFF0F8 40%,#FFE4F4 100%)' }}>
      <div className="absolute -top-10 -left-10 w-52 h-52 rounded-full opacity-25 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF1493 0%,transparent 70%)' }} />
      <div className="absolute bottom-40 -right-16 w-60 h-60 rounded-full opacity-20 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF69B4 0%,transparent 70%)' }} />

      <TopBar title="Customizer ✦" onBack={() => onNavigate('camera')} />

      <div className="flex-1 flex flex-col overflow-y-auto px-5 py-4 gap-5 relative z-10">

        {/* Sticker canvas */}
        <div className="flex items-center justify-center flex-shrink-0" style={{ height: '220px' }}>
          <div className="relative w-48 h-48 flex items-center justify-center">

            {/* Die-cut border */}
            {acc.sticker && hasSticker && (
              <div className="absolute inset-0 rounded-full"
                style={{ border: '5px solid white', boxShadow: '0 0 0 2px rgba(255,105,180,.5),0 8px 32px rgba(255,105,180,.35)' }} />
            )}

            {/* Glow */}
            {hasSticker && (
              <div className="absolute inset-4 rounded-full opacity-35"
                style={{ background: 'radial-gradient(circle,#FF69B4,transparent 70%)' }} />
            )}

            {/* User sticker or placeholder */}
            {hasSticker ? (
              <img src={stickerUrl!} alt="Your sticker"
                className="relative z-10 w-36 h-36 object-contain float-emoji"
                style={{
                  borderRadius: acc.sticker ? '50%' : '0',
                  filter: 'drop-shadow(0 4px 16px rgba(255,105,180,.5))',
                }} />
            ) : (
              <div className="w-36 h-36 rounded-full flex flex-col items-center justify-center gap-2"
                style={{ background: 'rgba(255,182,218,.3)', border: '2px dashed rgba(255,105,180,.4)' }}>
                <span className="text-3xl opacity-40">🫥</span>
                <p className="text-pink-400 text-[10px] font-bold text-center">Go to Studio<br />to add a photo</p>
              </div>
            )}

            {/* Glitter stars */}
            {acc.stars && hasSticker && <>
              <SparkleIcon size={18} className="absolute top-3 right-5 text-yellow-300 sparkle z-20" />
              <SparkleIcon size={12} className="absolute top-9 left-3 text-pink-400 sparkle z-20" style={{ animationDelay: '.4s' }} />
              <SparkleIcon size={20} className="absolute bottom-3 right-1 text-yellow-200 sparkle z-20" style={{ animationDelay: '.8s' }} />
              <SparkleIcon size={10} className="absolute bottom-9 left-5 text-pink-300 sparkle z-20" style={{ animationDelay: '1.2s' }} />
            </>}

            {/* Hair clip */}
            {acc.hairclip && hasSticker && (
              <span className="absolute -top-3 left-11 text-2xl sparkle z-20">🎀</span>
            )}
          </div>
        </div>

        {/* Accessories */}
        <div className="flex flex-col gap-2">
          <p className="text-center text-xs font-bold text-pink-400 uppercase tracking-widest">Y2K Accessories</p>
          <div className="grid grid-cols-3 gap-2">
            {accList.map(a => (
              <button key={a.key} onClick={() => toggle(a.key)}
                className={`accessory-toggle rounded-2xl py-3 flex flex-col items-center gap-1.5 ${acc[a.key] ? 'active' : ''}`}>
                <span className="text-xl leading-none">{a.icon}</span>
                <span className="text-[9px] font-bold text-pink-500 leading-tight text-center">{a.label}</span>
              </button>
            ))}
          </div>
        </div>

        {/* Action buttons */}
        <div className="flex gap-3">
          <button onClick={handleCopy} disabled={copying || !hasSticker}
            className="action-btn flex-1 py-3.5 rounded-2xl text-pink-600 text-sm flex items-center justify-center gap-2 shadow-sm disabled:opacity-50 transition-all active:scale-[.98]">
            {copying ? <><span className="animate-spin">⟳</span> Copying…</> : <><span>📋</span> Copy PNG</>}
          </button>
          <button onClick={handleSave} disabled={saved || !hasSticker}
            className="flex-1 py-3.5 rounded-2xl text-white text-sm flex items-center justify-center gap-2 font-bold transition-all active:scale-[.98] disabled:opacity-60"
            style={{ background: saved ? 'linear-gradient(135deg,#aaa,#888)' : 'linear-gradient(135deg,#FF69B4,#FF1493)', boxShadow: saved ? 'none' : '0 4px 16px rgba(255,20,147,.35)' }}>
            <span>{saved ? '✓' : '🗃️'}</span> {saved ? 'Saved!' : 'Save to Vault'}
          </button>
        </div>
      </div>

      <BottomNav active="preview" onNavigate={onNavigate} />
    </div>
  );
}

// ─── Screen 3: Library / Sticker Vault ───────────────────────────────────────

function FolderCard({ folder }: { folder: { name: string; icon: string; count: number; emojis: string[] } }) {
  return (
    <div className="folder-card rounded-3xl overflow-hidden flex flex-col cursor-pointer hover:scale-[1.02] transition-all duration-200 active:scale-[.98]">
      <div className="flex justify-center pt-3 pb-1 flex-shrink-0">
        <div className="keychain-ring w-6 h-6 rounded-full border-2"
          style={{ borderColor: '#c0c0c0', boxShadow: '0 2px 4px rgba(0,0,0,.15),inset 0 1px 2px rgba(255,255,255,.6)' }} />
      </div>
      <div className="px-3 pb-1 flex-shrink-0">
        <div className="inline-block rounded-t-xl px-3 py-1"
          style={{ background: 'rgba(255,20,147,.65)', backdropFilter: 'blur(4px)' }}>
          <span className="text-white font-bold text-[10px] drop-shadow-sm">{folder.name}</span>
        </div>
      </div>
      <div className="folder-pocket mx-2 mb-2 rounded-2xl p-3 flex flex-col gap-2">
        <div className="grid grid-cols-3 gap-1">
          {folder.emojis.slice(0, 6).map((e, i) => (
            <div key={i} className="aspect-square rounded-lg flex items-center justify-center text-base"
              style={{ background: 'rgba(255,255,255,.5)' }}>
              {e}
            </div>
          ))}
        </div>
        <div className="flex items-center justify-between">
          <span className="text-[10px] font-bold text-pink-500">{folder.icon} {folder.name}</span>
          <span className="text-[9px] font-semibold text-pink-400 opacity-75">{folder.count} items</span>
        </div>
      </div>
    </div>
  );
}

function LibraryScreen({
  onNavigate,
  vault,
  onEmojiClick,
}: {
  onNavigate: (s: Screen) => void;
  vault: VaultItem[];
  onEmojiClick: (item: VaultItem) => void;
}) {
  const folders = [
    { name: 'Reactions', icon: '💬', count: 12, emojis: ['🥹','😍','🤩','😎','🥰','😜'] },
    { name: 'Pets', icon: '🐾', count: 8,  emojis: ['🐱','🐶','🐰','🐹','🦊','🐼'] },
    { name: 'Y2K Vibe', icon: '✦', count: 16, emojis: ['✨','💅','🫧','🤌','💖','🦋'] },
  ];

  return (
    <div className="flex flex-col h-full"
      style={{ background: 'linear-gradient(160deg,#fff 0%,#FFF0F8 40%,#FFE4F4 100%)' }}>
      <div className="absolute top-20 -right-12 w-48 h-48 rounded-full opacity-25 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF69B4 0%,transparent 70%)' }} />
      <div className="absolute bottom-48 -left-16 w-56 h-56 rounded-full opacity-20 pointer-events-none"
        style={{ background: 'radial-gradient(circle,#FF1493 0%,transparent 70%)' }} />

      <div className="glass-nav flex-shrink-0 flex items-center justify-between px-5 pt-10 pb-4">
        <button className="w-9 h-9 flex items-center justify-center rounded-full flex-shrink-0"
          style={{ background: 'rgba(255,255,255,.25)' }}>
          <span className="text-white font-bold text-lg leading-none">☰</span>
        </button>
        <h1 className="font-pacifico text-white text-2xl drop-shadow-sm tracking-wide">Library</h1>
        <button onClick={() => onNavigate('camera')} className="w-9 h-9 flex items-center justify-center rounded-full flex-shrink-0"
          style={{ background: 'rgba(255,255,255,.25)' }}>
          <span className="text-white font-bold text-xl leading-none">+</span>
        </button>
      </div>

      <div className="flex-1 overflow-y-auto px-5 py-4 flex flex-col gap-4 relative z-10">

        <div className="flex items-center justify-between flex-shrink-0">
          <span className="font-bubble text-pink-500 text-base leading-none">My Stickers</span>
          <span className="text-xs font-bold text-pink-400 opacity-70">{vault.length} saved</span>
        </div>

        {vault.length === 0 ? (
          <div className="flex flex-col items-center justify-center gap-3 py-6 rounded-3xl"
            style={{ background: 'rgba(255,105,180,.06)', border: '1.5px dashed rgba(255,105,180,.3)' }}>
            <span className="text-4xl opacity-40">🫧</span>
            <p className="text-pink-400 text-xs font-bold text-center">Your vault is empty!<br />Generate a sticker in Studio.</p>
            <button onClick={() => onNavigate('camera')}
              className="btn-generate px-5 py-2 rounded-xl text-white text-xs">
              Go to Studio →
            </button>
          </div>
        ) : (
          <div className="flex gap-3 overflow-x-auto pb-1 flex-shrink-0">
            {vault.map(item => (
              <button key={item.id} onClick={() => onEmojiClick(item)}
                title="Tap to copy"
                className="flex-shrink-0 sticker-card w-14 h-14 rounded-2xl overflow-hidden flex items-center justify-center cursor-pointer hover:scale-105 transition-transform active:scale-90">
                <img src={item.url} alt="Sticker" className="w-12 h-12 object-contain" />
              </button>
            ))}
            <button onClick={() => onNavigate('camera')}
              className="flex-shrink-0 dot-card w-14 h-14 rounded-2xl flex items-center justify-center text-pink-400 text-2xl font-bold cursor-pointer hover:scale-105 transition-transform">
              +
            </button>
          </div>
        )}

        <div className="flex items-center justify-between flex-shrink-0">
          <span className="font-bubble text-pink-500 text-base leading-none">Sticker Packs</span>
          <button className="text-xs font-bold text-pink-400 opacity-70">Manage →</button>
        </div>

        <div className="grid grid-cols-2 gap-4 pb-2">
          {folders.map((folder, i) => <FolderCard key={i} folder={folder} />)}
          <div className="dot-card rounded-3xl p-4 flex flex-col items-center justify-center gap-3 cursor-pointer hover:scale-[1.02] transition-transform active:scale-[.98]"
            style={{ minHeight: '150px' }}>
            <div className="w-10 h-10 rounded-full flex items-center justify-center flex-shrink-0"
              style={{ background: 'rgba(255,105,180,.15)', border: '1.5px dashed rgba(255,105,180,.5)' }}>
              <span className="text-pink-400 text-2xl font-bold leading-none">+</span>
            </div>
            <p className="text-pink-400 font-bold text-xs text-center leading-snug">Create New<br />Pack</p>
          </div>
        </div>
      </div>

      <BottomNav active="library" onNavigate={onNavigate} />
    </div>
  );
}

// ─── Root ─────────────────────────────────────────────────────────────────────

export default function App() {
  const [screen, setScreen] = useState<Screen>('camera');
  const [stickerBlob, setStickerBlob] = useState<Blob | null>(null);
  const [stickerUrl, setStickerUrl] = useState<string | null>(null);
  const [vault, setVault] = useState<VaultItem[]>(INITIAL_VAULT);
  const [toast, setToast] = useState<string | null>(null);

  const handleProcessed = useCallback((blob: Blob, url: string) => {
    setStickerBlob(blob);
    setStickerUrl(url);
  }, []);

  const handleSaveToVault = useCallback((blob: Blob, url: string) => {
    setVault(v => [{ id: uid(), blob, url }, ...v]);
  }, []);

  const handleVaultItemClick = useCallback(async (item: VaultItem) => {
    try {
      const pngBlob = item.blob.type === 'image/png' ? item.blob : item.blob;
      await navigator.clipboard.write([new ClipboardItem({ 'image/png': pngBlob })]);
      setToast('Copied PNG to clipboard! 🎉');
    } catch {
      setToast('Copied PNG to clipboard!');
    }
  }, []);

  const clearToast = useCallback(() => setToast(null), []);

  return (
    <div className="size-full flex items-center justify-center"
      style={{ background: 'linear-gradient(135deg,#fce4f0 0%,#f8e8f8 50%,#ffe4f4 100%)' }}>
      <div className="relative w-full h-full max-w-sm mx-auto overflow-hidden"
        style={{ maxHeight: '844px', boxShadow: '0 32px 80px rgba(255,20,147,.2),0 8px 32px rgba(0,0,0,.12)' }}>

        {screen === 'camera' && (
          <CameraScreen onNavigate={setScreen} onProcessed={handleProcessed} />
        )}
        {screen === 'preview' && (
          <PreviewScreen
            onNavigate={setScreen}
            stickerUrl={stickerUrl}
            stickerBlob={stickerBlob}
            onSaveToVault={handleSaveToVault}
            onToast={setToast}
          />
        )}
        {screen === 'library' && (
          <LibraryScreen
            onNavigate={setScreen}
            vault={vault}
            onEmojiClick={handleVaultItemClick}
          />
        )}

        {toast && <Toast message={toast} onDone={clearToast} />}
      </div>
    </div>
  );
}
