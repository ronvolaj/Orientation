<script>
  // ---- Settings ----
  const SHAKE_THRESHOLD = 15;   // m/s² of movement (without gravity) that counts as a shake
  const QUIET_MS = 400;         // how long it must be calm after shaking before settling
  const SETTLE_MS = 500;        // how long the phone must stay on one face to lock the number
  let tolerance = $state(35);   // degrees you can tilt and still count as the same face

  // ---- Raw sensor values ----
  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);

  // ---- Dice state ----
  let started = $state(false);
  let status = $state('idle');  // 'idle' | 'rolling' | 'locked'
  let shown = $state(null);     // number currently on screen

  let lastShake = 0;
  let candidate = null;
  let candidateSince = 0;
  let rollTimer = null;

  const PIPS = {
    1: [4],
    2: [0, 8],
    3: [0, 4, 8],
    4: [0, 2, 6, 8],
    5: [0, 2, 4, 6, 8],
    6: [0, 2, 3, 5, 6, 8]
  };

  // Which face is "on top", from beta/gamma.
  // "up" = direction of the sky in the phone's own axes:
  //   z = out of the screen, y = towards the top edge, x = towards the right edge.
  // Opposite faces add up to 7, like a real die.
  function faceFromOrientation(b, g) {
    if (b == null || g == null) return null;
    const rad = Math.PI / 180;
    const ux = -Math.sin(g * rad) * Math.cos(b * rad);
    const uy = Math.sin(b * rad);
    const uz = Math.cos(g * rad) * Math.cos(b * rad);
    const t = Math.cos(tolerance * rad); // inside this cone = still the same face

    if (uz > t) return 6;    // flat, screen facing up
    if (uz < -t) return 1;   // flat, screen facing down
    if (uy > t) return 2;    // standing upright (portrait)
    if (uy < -t) return 5;   // upside-down portrait
    if (ux < -t) return 3;   // on its right side (landscape)
    if (ux > t) return 4;    // on its left side (landscape)
    return null;             // in between two faces -> no decision
  }

  function startRolling() {
    status = 'rolling';
    candidate = null;
    if (!rollTimer) rollTimer = setInterval(tick, 80);
  }

  function lock(face) {
    clearInterval(rollTimer);
    rollTimer = null;
    shown = face;
    status = 'locked';
    navigator.vibrate?.(60);
  }

  function tick() {
    const now = Date.now();

    // Still shaking (or just stopped): keep flashing random numbers
    if (now - lastShake < QUIET_MS) {
      shown = 1 + Math.floor(Math.random() * 6);
      return;
    }

    // Calm now: wait until the phone rests on one face long enough
    const face = faceFromOrientation(beta, gamma);
    if (face === null) {
      candidate = null;
      shown = 1 + Math.floor(Math.random() * 6);
      return;
    }
    if (face !== candidate) {
      candidate = face;
      candidateSince = now;
    }
    shown = face;
    if (now - candidateSince >= SETTLE_MS) lock(face);
  }

  function onOrientation(event) {
    alpha = event.alpha;
    beta = event.beta;
    gamma = event.gamma;
  }

  function onMotion(event) {
    let force;
    const a = event.acceleration;
    if (a && a.x != null) {
      force = Math.hypot(a.x, a.y, a.z);
    } else {
      const g = event.accelerationIncludingGravity;
      if (!g || g.x == null) return;
      force = Math.abs(Math.hypot(g.x, g.y, g.z) - 9.81);
    }
    if (force > SHAKE_THRESHOLD) {
      lastShake = Date.now();
      if (status !== 'rolling') startRolling();
    }
  }

  async function start() {
    // iOS needs explicit permission for both sensors (must come from a tap)
    if (typeof DeviceOrientationEvent !== 'undefined' &&
        typeof DeviceOrientationEvent.requestPermission === 'function') {
      if ((await DeviceOrientationEvent.requestPermission()) !== 'granted') return;
    }
    if (typeof DeviceMotionEvent !== 'undefined' &&
        typeof DeviceMotionEvent.requestPermission === 'function') {
      if ((await DeviceMotionEvent.requestPermission()) !== 'granted') return;
    }

    window.addEventListener('deviceorientation', onOrientation);
    window.addEventListener('devicemotion', onMotion);
    started = true;
  }

  $effect(() => {
    return () => {
      window.removeEventListener('deviceorientation', onOrientation);
      window.removeEventListener('devicemotion', onMotion);
      clearInterval(rollTimer);
    };
  });
</script>

<main>
  <h1>Shake Dice</h1>

  {#if !started}
    <button onclick={start}>Start</button>
  {:else}
    <div class="die" class:rolling={status === 'rolling'}>
      {#if shown}
        {#each Array(9) as _, i}
          <span class="cell">{#if PIPS[shown].includes(i)}<span class="pip"></span>{/if}</span>
        {/each}
      {/if}
    </div>

    <p class="number">{shown ?? '–'}</p>

    <p class="status">
      {#if status === 'idle'}Shake the phone to roll
      {:else if status === 'rolling'}Rolling… stop shaking and hold the phone still
      {:else}Locked. Shake again for a new number{/if}
    </p>

    <label>
      Tolerance: {tolerance}°
      <input type="range" min="10" max="44" bind:value={tolerance} />
    </label>

    <details>
      <summary>How to hold it</summary>
      <ul>
        <li>Flat, screen up → 6</li>
        <li>Flat, screen down → 1</li>
        <li>Upright (portrait) → 2</li>
        <li>Upside down (portrait) → 5</li>
        <li>On its right side → 3</li>
        <li>On its left side → 4</li>
      </ul>
    </details>

    <p class="raw">Alpha: {alpha?.toFixed(0)} · Beta: {beta?.toFixed(0)} · Gamma: {gamma?.toFixed(0)}</p>
  {/if}
</main>

<style>
  main { font-family: system-ui, sans-serif; text-align: center; padding: 24px 16px; }
  button { font-size: 1.2rem; padding: 12px 32px; border-radius: 12px; border: none; background: #222; color: white; }
  .die {
    width: 180px; height: 180px; margin: 24px auto 8px; padding: 20px; box-sizing: border-box;
    background: white; border: 3px solid #222; border-radius: 28px;
    display: grid; grid-template: repeat(3, 1fr) / repeat(3, 1fr);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  }
  .die.rolling { animation: wobble 0.25s infinite; }
  .cell { display: flex; align-items: center; justify-content: center; }
  .pip { width: 30px; height: 30px; border-radius: 50%; background: #222; }
  .number { font-size: 2.5rem; font-weight: 700; margin: 0; }
  .status { color: #555; min-height: 1.5em; }
  label { display: block; margin: 16px 0; }
  details { max-width: 280px; margin: 0 auto; text-align: left; }
  .raw { font-size: 0.8rem; color: #888; }
  @keyframes wobble {
    0% { transform: rotate(-6deg); }
    50% { transform: rotate(6deg); }
    100% { transform: rotate(-6deg); }
  }
</style>