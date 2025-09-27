import React, { useEffect, useRef, useState } from "react";

// SpaceDodger - Jogo simples em uma única componente React
// Instruções rápidas:
// - Coloque este arquivo em um projeto React (Vite/CRA). Ex: src/App.jsx
// - Tenha Tailwind instalado (opcional) para o estilo; o jogo também funciona sem Tailwind.
// - Use setas/esquerda-direita ou toque/arraste no celular.

export default function SpaceDodger() {
  // Configurações do jogo
  const WIDTH = 800;
  const HEIGHT = 600;
  const ASTEROID_SPAWN_INTERVAL = 1000; // ms
  const ASTEROID_SPEED_MIN = 2;
  const ASTEROID_SPEED_MAX = 5;

  const [running, setRunning] = useState(false);
  const [score, setScore] = useState(0);
  const [best, setBest] = useState(() => {
    try {
      return parseInt(localStorage.getItem("space_dodger_best") || "0", 10);
    } catch {
      return 0;
    }
  });
  const [message, setMessage] = useState("Clique em 'Iniciar' para jogar");

  // Refs para estado mutável no loop
  const playerRef = useRef({ x: WIDTH / 2 - 20, y: HEIGHT - 80, w: 40, h: 60 });
  const asteroidsRef = useRef([]);
  const lastSpawnRef = useRef(0);
  const keysRef = useRef({ left: false, right: false });
  const rafRef = useRef(null);
  const lastTimeRef = useRef(0);
  const containerRef = useRef(null);

  // Controle de teclado
  useEffect(() => {
    function down(e) {
      if (e.key === "ArrowLeft" || e.key === "a") keysRef.current.left = true;
      if (e.key === "ArrowRight" || e.key === "d") keysRef.current.right = true;
      if (e.key === " " && !running) startGame();
    }
    function up(e) {
      if (e.key === "ArrowLeft" || e.key === "a") keysRef.current.left = false;
      if (e.key === "ArrowRight" || e.key === "d") keysRef.current.right = false;
    }
    window.addEventListener("keydown", down);
    window.addEventListener("keyup", up);
    return () => {
      window.removeEventListener("keydown", down);
      window.removeEventListener("keyup", up);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [running]);

  // Funções utilitárias
  function rand(min, max) {
    return Math.random() * (max - min) + min;
  }

  function spawnAsteroid() {
    const size = Math.floor(rand(24, 80));
    const x = rand(0, WIDTH - size);
    const speed = rand(ASTEROID_SPEED_MIN, ASTEROID_SPEED_MAX);
    asteroidsRef.current.push({ x, y: -size, w: size, h: size, speed });
  }

  function resetGame() {
    asteroidsRef.current = [];
    playerRef.current = { x: WIDTH / 2 - 20, y: HEIGHT - 80, w: 40, h: 60 };
    lastSpawnRef.current = 0;
    lastTimeRef.current = 0;
    setScore(0);
    setMessage("Bora! Evite os asteroides e marque pontos.");
  }

  function startGame() {
    resetGame();
    setRunning(true);
    setMessage("");
    // start loop
    if (!rafRef.current) rafRef.current = requestAnimationFrame(loop);
  }

  function endGame() {
    setRunning(false);
    setMessage("Game Over — clique em 'Iniciar' para tentar novamente");
    if (score > best) {
      setBest(score);
      try {
        localStorage.setItem("space_dodger_best", String(score));
      } catch {}
    }
    if (rafRef.current) {
      cancelAnimationFrame(rafRef.current);
      rafRef.current = null;
    }
  }

  function rectsCollide(a, b) {
    return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h);
  }

  // Loop principal (render/physics)
  function loop(ts) {
    if (!lastTimeRef.current) lastTimeRef.current = ts;
    const dt = ts - lastTimeRef.current;
    lastTimeRef.current = ts;

    // Spawn asteroids
    lastSpawnRef.current += dt;
    if (lastSpawnRef.current >= ASTEROID_SPAWN_INTERVAL) {
      spawnAsteroid();
      lastSpawnRef.current = 0;
      // aumenta dificuldade levemente ao longo do tempo
      if (ASTEROID_SPAWN_INTERVAL > 250) {
        // eslint-disable-next-line no-unused-expressions
        // (nota: constante declarada acima permanece; isso é só uma simulação de dificuldade)
      }
    }

    // Atualiza jogador com base nas teclas
    const speedX = 6;
    if (keysRef.current.left) playerRef.current.x -= speedX + 1;
    if (keysRef.current.right) playerRef.current.x += speedX + 1;
    // keep inside
    if (playerRef.current.x < 0) playerRef.current.x = 0;
    if (playerRef.current.x + playerRef.current.w > WIDTH) playerRef.current.x = WIDTH - playerRef.current.w;

    // Move asteroides
    for (let i = asteroidsRef.current.length - 1; i >= 0; i--) {
      const a = asteroidsRef.current[i];
      a.y += a.speed + dt * 0.002; // acelera levemente com o tempo
      if (a.y > HEIGHT) {
        asteroidsRef.current.splice(i, 1);
        setScore((s) => s + 1);
      }
    }

    // Colisão
    for (const a of asteroidsRef.current) {
      if (rectsCollide(a, playerRef.current)) {
        endGame();
        return;
      }
    }

    // Renderiza via DOM (atualiza transform)
    const stage = containerRef.current;
    if (stage) {
      // atualiza posição do jogador
      const playerEl = stage.querySelector(".sd-player");
      if (playerEl) playerEl.style.transform = `translate(${playerRef.current.x}px, ${playerRef.current.y}px)`;

      // atualiza asteroides
      const asteroidsEl = stage.querySelector('.sd-asteroids');
      if (asteroidsEl) {
        // sincroniza número de elementos DOM com asteroidsRef
        const need = asteroidsRef.current.length;
        while (asteroidsEl.children.length < need) {
          const div = document.createElement('div');
          div.className = 'sd-asteroid';
          asteroidsEl.appendChild(div);
        }
        while (asteroidsEl.children.length > need) {
          asteroidsEl.removeChild(asteroidsEl.lastChild);
        }
        // atualiza cada um
        for (let i = 0; i < need; i++) {
          const a = asteroidsRef.current[i];
          const el = asteroidsEl.children[i];
          el.style.width = a.w + 'px';
          el.style.height = a.h + 'px';
          el.style.transform = `translate(${a.x}px, ${a.y}px)`;
          el.style.borderRadius = Math.min(a.w, a.h) / 2 + 'px';
        }
      }

      // atualiza HUD
      const scoreEl = stage.querySelector('.sd-score');
      if (scoreEl) scoreEl.textContent = `Score: ${score}`;
      const bestEl = stage.querySelector('.sd-best');
      if (bestEl) bestEl.textContent = `Best: ${best}`;
    }

    if (running) rafRef.current = requestAnimationFrame(loop);
    else rafRef.current = null;
  }

  // touch / drag support para mobile
  useEffect(() => {
    const el = containerRef.current;
    if (!el) return;
    let dragging = false;
    let startX = 0;

    function down(e) {
      dragging = true;
      startX = (e.touches ? e.touches[0].clientX : e.clientX) - el.getBoundingClientRect().left;
    }
    function move(e) {
      if (!dragging) return;
      const x = (e.touches ? e.touches[0].clientX : e.clientX) - el.getBoundingClientRect().left;
      playerRef.current.x = Math.max(0, Math.min(WIDTH - playerRef.current.w, x - playerRef.current.w / 2));
      e.preventDefault();
    }
    function up() {
      dragging = false;
    }

    el.addEventListener('mousedown', down);
    window.addEventListener('mousemove', move);
    window.addEventListener('mouseup', up);

    el.addEventListener('touchstart', down, { passive: false });
    window.addEventListener('touchmove', move, { passive: false });
    window.addEventListener('touchend', up);

    return () => {
      el.removeEventListener('mousedown', down);
      window.removeEventListener('mousemove', move);
      window.removeEventListener('mouseup', up);

      el.removeEventListener('touchstart', down);
      window.removeEventListener('touchmove', move);
      window.removeEventListener('touchend', up);
    };
  }, []);

  // limpa RAF quando componente desmonta
  useEffect(() => () => {
    if (rafRef.current) cancelAnimationFrame(rafRef.current);
  }, []);

  // JSX
  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-b from-black via-slate-900 to-black p-6">
      <div className="w-full max-w-4xl">
        <div className="flex items-center justify-between mb-4 text-white">
          <h1 className="text-2xl font-bold">SpaceDodger</h1>
          <div className="space-x-3">
            <button
              onClick={() => (running ? endGame() : startGame())}
              className="px-4 py-2 rounded bg-violet-600 hover:bg-violet-500"
            >
              {running ? 'Parar' : 'Iniciar'}
            </button>
            <button
              onClick={() => { resetGame(); setMessage('Pronto! Aperte Iniciar.'); }}
              className="px-4 py-2 rounded bg-slate-600 hover:bg-slate-500"
            >
              Reiniciar
            </button>
          </div>
        </div>

        <div className="relative bg-gradient-to-b from-slate-800 to-slate-900 rounded-lg shadow-lg overflow-hidden" style={{ width: WIDTH, height: HEIGHT }} ref={containerRef}>
          {/* HUD */}
          <div className="absolute top-3 left-3 text-white font-semibold sd-score">Score: {score}</div>
          <div className="absolute top-3 right-3 text-white font-semibold sd-best">Best: {best}</div>

          {/* palco do jogo */}
          <div className="absolute left-0 top-0 w-full h-full pointer-events-auto">
            {/* asteroides (injetados dinamicamente) */}
            <div className="sd-asteroids" style={{ position: 'absolute', left: 0, top: 0, width: '100%', height: '100%' }} />

            {/* jogador */}
            <div
              className="sd-player absolute"
              style={{ width: playerRef.current.w + 'px', height: playerRef.current.h + 'px', transform: `translate(${playerRef.current.x}px, ${playerRef.current.y}px)`, transition: 'transform 0.03s linear' }}
            >
              <div style={{ width: '100%', height: '100%', background: 'linear-gradient(180deg,#60a5fa,#1e3a8a)', borderRadius: '8px', boxShadow: '0 8px 20px rgba(0,0,0,0.6)', display: 'flex', alignItems: 'center', justifyContent: 'center', color: 'white', fontWeight: 700 }}>
                🚀
              </div>
            </div>

            {/* efeito de estrela de fundo */}
            <Starfield width={WIDTH} height={HEIGHT} />

            {/* message overlay */}
            {!running && (
              <div className="absolute inset-0 flex flex-col items-center justify-center text-center px-6">
                <div className="bg-black/60 rounded-lg p-6 text-white max-w-lg">
                  <h2 className="text-xl font-bold mb-2">{message}</h2>
                  <p className="text-sm mb-4">Use ← → ou toque/arraste para mover. Evite os asteroides — cada um que passa aumenta seu score.</p>
                  <div className="space-x-3">
                    <button onClick={startGame} className="px-4 py-2 rounded bg-green-600">Iniciar</button>
                    <button onClick={() => { setScore(0); setBest(0); localStorage.removeItem('space_dodger_best'); }} className="px-4 py-2 rounded bg-red-600">Zerar Best</button>
                  </div>
                </div>
              </div>
            )}
          </div>

          {/* estilos inline para elementos dinâmicos do jogo */}
          <style>{`
            .sd-asteroid { position: absolute; background: radial-gradient(circle at 30% 20%, #b58858, #7a4327 60%); box-shadow: inset -6px -6px 10px rgba(0,0,0,0.4); }
            .sd-player { z-index: 30; }
          `}</style>
        </div>

        <div className="mt-4 text-white text-sm">
          <strong>Dicas:</strong> personalize cores, velocidade e spawn para ajustar a dificuldade. Para transformar em PWA, adicione manifest.json e service worker.
        </div>
      </div>
    </div>
  );
}

// Starfield: gera pontos de estrela em segundo plano usando DOM (leve)
function Starfield({ width, height }) {
  const ref = useRef(null);
  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    // cria algumas estrelas
    const count = Math.floor((width * height) / 50000);
    for (let i = 0; i < count; i++) {
      const s = document.createElement('div');
      const x = Math.random() * width;
      const y = Math.random() * height;
      const size = Math.random() * 2 + 1;
      s.style.position = 'absolute';
      s.style.left = x + 'px';
      s.style.top = y + 'px';
      s.style.width = size + 'px';
      s.style.height = size + 'px';
      s.style.background = 'white';
      s.style.opacity = String(0.3 + Math.random() * 0.7);
      s.style.borderRadius = '50%';
      s.style.filter = 'blur(0.4px)';
      el.appendChild(s);
    }
    return () => { if (el) el.innerHTML = ''; };
  }, [width, height]);

  return <div ref={ref} style={{ position: 'absolute', left: 0, top: 0, width: width + 'px', height: height + 'px', zIndex: 5, pointerEvents: 'none' }} />;
}
