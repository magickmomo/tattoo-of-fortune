<script lang="ts">
  import { onMount } from 'svelte';

  // Adapted from the coordinate/viewport mechanics of You Won't Wear This T-Shirt's
  // DiscoveryGridClient. No commerce, authentication or generation services are imported.
  export let cellSize = 190;

  type Cell = { x: number; y: number; key: string };
  type Point = { x: number; y: number };
  const MIN_ZOOM = 0.5;
  const MAX_ZOOM = 2;
  const BUFFER = 2;
  const MAX_VISIBLE_CELLS = 350;
  let viewport: HTMLDivElement;
  let width = 900;
  let height = 600;
  let pan: Point = { x: 0, y: 0 };
  let zoom = 1;
  let cells: Cell[] = [];
  let active: Cell | null = null;
  let dragging = false;
  let drag: { pointer: number; x: number; y: number; startX: number; startY: number; moved: boolean } | null = null;
  let suppressClick = false;

  function clamp(value: number, min: number, max: number) {
    return Math.max(min, Math.min(max, value));
  }

  function refreshCells() {
    const size = cellSize * zoom;
    const minX = Math.floor((-pan.x - width / 2) / size) - BUFFER;
    const maxX = Math.ceil((width / 2 - pan.x) / size) + BUFFER;
    const minY = Math.floor((-pan.y - height / 2) / size) - BUFFER;
    const maxY = Math.ceil((height / 2 - pan.y) / size) + BUFFER;
    const next: Cell[] = [];
    for (let y = minY; y <= maxY; y++) {
      for (let x = minX; x <= maxX; x++) {
        if (next.length >= MAX_VISIBLE_CELLS) break;
        next.push({ x, y, key: x + ',' + y });
      }
      if (next.length >= MAX_VISIBLE_CELLS) break;
    }
    cells = next;
  }

  function movePan(x: number, y: number) {
    pan = { x, y };
    refreshCells();
  }

  function setZoom(next: number, screenX = width / 2, screenY = height / 2) {
    const value = clamp(next, MIN_ZOOM, MAX_ZOOM);
    // Hold the world coordinate under the pointer still while zooming.
    const ratio = value / zoom;
    pan = {
      x: (pan.x + width / 2 - screenX) * ratio - width / 2 + screenX,
      y: (pan.y + height / 2 - screenY) * ratio - height / 2 + screenY
    };
    zoom = value;
    refreshCells();
  }

  function onWheel(event: WheelEvent) {
    event.preventDefault();
    const rect = viewport.getBoundingClientRect();
    const factor = Math.exp(-event.deltaY * 0.0012);
    setZoom(zoom * factor, event.clientX - rect.left, event.clientY - rect.top);
  }

  function onPointerDown(event: PointerEvent) {
    if (event.button !== 0 || active) return;
    drag = { pointer: event.pointerId, x: event.clientX, y: event.clientY, startX: pan.x, startY: pan.y, moved: false };
    viewport.setPointerCapture(event.pointerId);
    dragging = true;
  }

  function onPointerMove(event: PointerEvent) {
    if (!drag || drag.pointer !== event.pointerId) return;
    const dx = event.clientX - drag.x;
    const dy = event.clientY - drag.y;
    if (Math.abs(dx) + Math.abs(dy) > 5) drag.moved = true;
    movePan(drag.startX + dx, drag.startY + dy);
  }

  function onPointerUp(event: PointerEvent) {
    if (!drag || drag.pointer !== event.pointerId) return;
    suppressClick = drag.moved;
    drag = null;
    dragging = false;
    if (viewport.hasPointerCapture(event.pointerId)) viewport.releasePointerCapture(event.pointerId);
  }

  function pick(cell: Cell) {
    if (suppressClick) { suppressClick = false; return; }
    active = cell;
  }

  function hash(x: number, y: number) {
    let n = Math.imul(x, 73856093) ^ Math.imul(y, 19349663) ^ 0x5f3759df;
    n = Math.imul(n ^ (n >>> 16), 2246822507);
    return (n ^ (n >>> 13)) >>> 0;
  }

  // Pure placeholder renderer: the same coordinates produce the same art.
  // Replace this with curated/generated tattoo artwork without touching grid mechanics.
  function motif(cell: Cell) {
    const seed = hash(cell.x, cell.y);
    const variant = seed % 6;
    const ink = '#171717';
    const start = '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 180 180" fill="none" stroke="' + ink + '" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">';
    const end = '</svg>';
    const shapes = [
      '<circle cx="90" cy="90" r="49"/><circle cx="90" cy="90" r="32"/><path d="M90 20V160M20 90H160M55 55L125 125M125 55L55 125"/><circle cx="90" cy="90" r="8"/>',
      '<path d="M90 20C62 55 50 80 90 148C130 80 118 55 90 20Z"/><path d="M90 35V148M90 78L66 60M90 106L112 85"/>',
      '<path d="M90 20L140 90L90 160L40 90Z"/><circle cx="90" cy="90" r="28"/><path d="M61 90Q90 53 119 90Q90 127 61 90Z"/><circle cx="90" cy="90" r="8"/>',
      '<path d="M90 22V155M58 51Q90 28 122 51M48 90Q90 62 132 90M56 127Q90 101 124 127"/><path d="M77 24L90 10L103 24M79 155L90 170L101 155"/>',
      '<circle cx="90" cy="90" r="54"/><path d="M90 35L103 77L146 90L103 103L90 145L77 103L34 90L77 77Z"/><circle cx="90" cy="90" r="14"/>',
      '<path d="M90 22C105 56 142 64 142 98C142 128 120 150 90 150C60 150 38 128 38 98C38 64 75 56 90 22Z"/><path d="M90 42C73 80 69 115 90 139C111 115 107 80 90 42Z"/><circle cx="90" cy="102" r="8"/>'
    ];
    return 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(start + shapes[variant] + end);
  }

  onMount(() => {
    const observer = new ResizeObserver((entries) => {
      const rect = entries[0]?.contentRect;
      if (!rect) return;
      width = rect.width;
      height = rect.height;
      refreshCells();
    });
    observer.observe(viewport);
    refreshCells();
    return () => observer.disconnect();
  });
</script>

<svelte:head>
  <title>Explore | Tattoo of Fortune</title>
  <meta name="description" content="Explore an endless grid of experimental tattoo motifs." />
</svelte:head>

<section class="explore">
  <header class="toolbar">
    <div><p class="eyebrow">Tattoo of Fortune / experimental</p><h1>Infinite flash</h1><p>Drag to explore · scroll to zoom · select a design</p></div>
    <div class="controls">
      <button type="button" on:click={() => setZoom(zoom / 1.2)} aria-label="Zoom out">−</button>
      <span>{Math.round(zoom * 100)}%</span>
      <button type="button" on:click={() => setZoom(zoom * 1.2)} aria-label="Zoom in">+</button>
      <button type="button" on:click={() => { pan = { x: 0, y: 0 }; zoom = 1; refreshCells(); }}>Reset</button>
    </div>
  </header>
  <div
    class:dragging
    class="viewport"
    bind:this={viewport}
    role="region"
    aria-label="Pannable tattoo exploration grid"
    on:wheel|nonpassive={onWheel}
    on:pointerdown={onPointerDown}
    on:pointermove={onPointerMove}
    on:pointerup={onPointerUp}
    on:pointercancel={onPointerUp}
  >
    <div class="plane" style:transform={'translate(' + (width / 2 + pan.x) + 'px,' + (height / 2 + pan.y) + 'px) scale(' + zoom + ')'}>
      {#each cells as cell (cell.key)}
        <button
          class="tile"
          type="button"
          style:left={(cell.x * cellSize) + 'px'}
          style:top={(cell.y * cellSize) + 'px'}
          style:width={cellSize + 'px'}
          style:height={cellSize + 'px'}
          aria-label={'View tattoo motif ' + cell.key}
          on:click={() => pick(cell)}
          on:dragstart={(event) => event.preventDefault()}
        >
          <img src={motif(cell)} alt="" draggable="false" />
          <span>{cell.key}</span>
        </button>
      {/each}
    </div>
  </div>
  <p class="footnote">Prototype motifs are generated locally and deterministically. No AI calls, accounts or payments.</p>
</section>

{#if active}
  <div class="backdrop" role="presentation" on:click={() => active = null}>
    <section class="dialog" role="dialog" aria-modal="true" aria-label="Tattoo design preview" on:click|stopPropagation>
      <button type="button" class="close" on:click={() => active = null} aria-label="Close preview">×</button>
      <img src={motif(active)} alt={'Experimental tattoo motif at ' + active.key} />
      <p>Coordinate {active.key}</p>
      <a href={motif(active)} download={'tattoo-' + active.x + '-' + active.y + '.svg'}>Download SVG</a>
    </section>
  </div>
{/if}

<style>
  .explore { margin: 0 auto; max-width: 1450px; padding: 0 1rem 1.5rem; color: #f3efe7; }
  .toolbar { display: flex; justify-content: space-between; align-items: center; gap: 1rem; padding: 1rem 0; flex-wrap: wrap; }
  .eyebrow { text-transform: uppercase; letter-spacing: .16em; font-size: .65rem; opacity: .7; }
  h1 { font-size: clamp(1.6rem, 4vw, 2.6rem); font-family: Georgia, serif; }
  .toolbar p { font-size: .82rem; opacity: .8; }
  .controls { display: flex; align-items: center; gap: .4rem; }
  .controls button { border: 1px solid #746f68; color: #fff; border-radius: .3rem; min-width: 2.3rem; padding: .5rem; }
  .controls span { min-width: 3.3rem; text-align: center; }
  .viewport { height: min(72vh, 830px); min-height: 400px; position: relative; overflow: hidden; background: #e5ded0; border: 1px solid #706658; border-radius: .5rem; cursor: grab; touch-action: none; user-select: none; }
  .viewport.dragging { cursor: grabbing; }
  .plane { position: absolute; inset: 0 auto auto 0; transform-origin: 0 0; }
  .tile { position: absolute; background: #fcfaf5; border: 1px solid #d5cbbf; display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 10px; color: #777; }
  .tile:hover, .tile:focus-visible { background: #f3eee4; outline: 2px solid #807566; outline-offset: -3px; }
  .tile img { width: 80%; height: 80%; object-fit: contain; pointer-events: none; }
  .tile span { font-size: 10px; letter-spacing: .05em; }
  .footnote { font-size: .75rem; color: #b7aea5; margin-top: .8rem; }
  .backdrop { position: fixed; inset: 0; z-index: 100; background: #000b; display: grid; place-items: center; padding: 1rem; }
  .dialog { background: #fcfaf5; color: #191919; width: min(420px, 100%); border-radius: .6rem; padding: 1.5rem; text-align: center; position: relative; }
  .dialog img { width: min(320px, 100%); margin: auto; }
  .dialog p { margin: 1rem 0; }
  .dialog a { display: inline-block; padding: .7rem 1rem; background: #151515; color: white; border-radius: .3rem; }
  .close { position: absolute; right: 1rem; top: .5rem; font-size: 1.8rem; }
</style>