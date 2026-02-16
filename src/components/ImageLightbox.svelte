<script lang="ts">
  import { onMount } from "svelte";

  interface Props {
    images: Array<{ src: string; alt: string }>;
  }

  let { images }: Props = $props();

  let dialogRef: HTMLDialogElement | undefined = $state();
  let imageRef: HTMLImageElement | undefined = $state();
  let viewportRef: HTMLDivElement | undefined = $state();
  let currentIndex = $state(0);
  let scale = $state(1);
  let translateX = $state(0);
  let translateY = $state(0);
  let isDragging = $state(false);
  let isPortrait = $state(false);
  let viewportW = $state(0);
  let viewportH = $state(0);

  let isZoomed = $derived(scale > 1.05);

  let dragStart = { x: 0, y: 0 };
  let startTranslate = { x: 0, y: 0 };

  // Pinch state
  let initialPinchDistance = 0;
  let initialPinchScale = 1;

  function checkPortrait() {
    viewportW = window.innerWidth;
    viewportH = window.innerHeight;
    isPortrait = viewportW < 768 && viewportH > viewportW;
  }

  function openLightbox(index: number) {
    currentIndex = index;
    resetZoom();
    checkPortrait();
    dialogRef?.showModal();
    preloadAdjacent();
  }

  function closeLightbox() {
    dialogRef?.close();
    resetZoom();
  }

  function resetZoom() {
    scale = 1;
    translateX = 0;
    translateY = 0;
    isDragging = false;
  }

  function navigate(direction: 1 | -1) {
    currentIndex = (currentIndex + direction + images.length) % images.length;
    resetZoom();
    preloadAdjacent();
  }

  function preloadAdjacent() {
    const prev = (currentIndex - 1 + images.length) % images.length;
    const next = (currentIndex + 1) % images.length;
    for (const idx of [prev, next]) {
      const img = new Image();
      img.src = images[idx].src;
    }
  }

  function toggleZoom(e: MouseEvent) {
    if (isDragging) return;
    if (!isZoomed) {
      scale = 2.5;
      // Zoom toward click position
      if (viewportRef) {
        const rect = viewportRef.getBoundingClientRect();
        const relX = (e.clientX - rect.left) / rect.width - 0.5;
        const relY = (e.clientY - rect.top) / rect.height - 0.5;
        translateX = -relX * rect.width * 0.6;
        translateY = -relY * rect.height * 0.6;
      }
    } else {
      resetZoom();
    }
  }

  function handleWheel(e: WheelEvent) {
    e.preventDefault();
    const factor = e.deltaY > 0 ? 0.9 : 1.1;
    const newScale = Math.max(1, Math.min(5, scale * factor));
    if (newScale <= 1.05) {
      resetZoom();
    } else {
      scale = newScale;
    }
  }

  // Mouse drag
  let didDrag = false;

  function handleMouseDown(e: MouseEvent) {
    if (!isZoomed) return;
    e.preventDefault();
    isDragging = true;
    didDrag = false;
    dragStart = { x: e.clientX, y: e.clientY };
    startTranslate = { x: translateX, y: translateY };
  }

  function handleMouseMove(e: MouseEvent) {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    if (Math.abs(dx) > 3 || Math.abs(dy) > 3) didDrag = true;
    translateX = startTranslate.x + dx;
    translateY = startTranslate.y + dy;
  }

  function handleMouseUp() {
    isDragging = false;
  }

  function handleImageClick(e: MouseEvent) {
    if (didDrag) {
      didDrag = false;
      return;
    }
    toggleZoom(e);
  }

  // Touch support
  let touchDragStart = { x: 0, y: 0 };
  let touchStartTranslate = { x: 0, y: 0 };
  let isTouchDragging = false;

  function getPinchDistance(touches: TouchList): number {
    const dx = touches[0].clientX - touches[1].clientX;
    const dy = touches[0].clientY - touches[1].clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }

  function handleTouchStart(e: TouchEvent) {
    if (e.touches.length === 2) {
      e.preventDefault();
      initialPinchDistance = getPinchDistance(e.touches);
      initialPinchScale = scale;
    } else if (e.touches.length === 1 && isZoomed) {
      isTouchDragging = true;
      touchDragStart = { x: e.touches[0].clientX, y: e.touches[0].clientY };
      touchStartTranslate = { x: translateX, y: translateY };
    }
  }

  function handleTouchMove(e: TouchEvent) {
    if (e.touches.length === 2) {
      e.preventDefault();
      const dist = getPinchDistance(e.touches);
      const newScale = Math.max(1, Math.min(5, initialPinchScale * (dist / initialPinchDistance)));
      if (newScale <= 1.05) {
        resetZoom();
      } else {
        scale = newScale;
      }
    } else if (e.touches.length === 1 && isTouchDragging) {
      e.preventDefault();
      translateX = touchStartTranslate.x + (e.touches[0].clientX - touchDragStart.x);
      translateY = touchStartTranslate.y + (e.touches[0].clientY - touchDragStart.y);
    }
  }

  function handleTouchEnd() {
    isTouchDragging = false;
  }

  // Keyboard
  function handleKeydown(e: KeyboardEvent) {
    if (!dialogRef?.open) return;
    if (e.key === "ArrowLeft") { e.preventDefault(); navigate(-1); }
    else if (e.key === "ArrowRight") { e.preventDefault(); navigate(1); }
  }

  onMount(() => {
    const handler = ((e: CustomEvent) => {
      openLightbox(e.detail.index);
    }) as EventListener;

    window.addEventListener("open-lightbox", handler);
    window.addEventListener("keydown", handleKeydown);
    window.addEventListener("resize", checkPortrait);

    return () => {
      window.removeEventListener("open-lightbox", handler);
      window.removeEventListener("keydown", handleKeydown);
      window.removeEventListener("resize", checkPortrait);
    };
  });
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<dialog
  bind:this={dialogRef}
  class="modal"
  onclose={resetZoom}
>
  <div
    class="fixed inset-0 z-[60] flex items-center justify-center pointer-events-none backdrop-blur-md bg-black/70"
    style={isPortrait ? `transform: rotate(90deg); transform-origin: center center; width: ${viewportH}px; height: ${viewportW}px; top: 50%; left: 50%; margin-top: -${viewportW / 2}px; margin-left: -${viewportH / 2}px;` : ''}
  >
    <!-- Top bar: counter + close -->
    <div class="absolute top-4 left-0 right-0 flex items-center justify-between px-4 md:px-8 pointer-events-auto z-20">
      <div class="bg-black/70 text-white text-sm px-3 py-1 rounded-full shadow-lg ring-1 ring-white/20">
        {currentIndex + 1} / {images.length}
      </div>
      <div class="flex items-center gap-2">
        <button
          onclick={() => isPortrait = !isPortrait}
          class="w-10 h-10 flex items-center justify-center rounded-full bg-black/70 text-white hover:bg-black/90 transition-colors cursor-pointer shadow-lg ring-1 ring-white/20 md:hidden"
          aria-label="Toggle rotation"
        >
          <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 12c0-1.232-.046-2.453-.138-3.662a4.006 4.006 0 00-3.7-3.7 48.678 48.678 0 00-7.324 0 4.006 4.006 0 00-3.7 3.7c-.017.22-.032.441-.046.662M19.5 12l3-3m-3 3l-3-3m-12 3c0 1.232.046 2.453.138 3.662a4.006 4.006 0 003.7 3.7 48.656 48.656 0 007.324 0 4.006 4.006 0 003.7-3.7c.017-.22.032-.441.046-.662M4.5 12l3 3m-3-3l-3 3" />
          </svg>
        </button>
        <button
          onclick={closeLightbox}
          class="w-10 h-10 flex items-center justify-center rounded-full bg-black/70 text-white hover:bg-black/90 transition-colors cursor-pointer shadow-lg ring-1 ring-white/20"
          aria-label="Close"
        >
          <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
          </svg>
        </button>
      </div>
    </div>

    <!-- Nav: Previous -->
    {#if images.length > 1}
      <button
        onclick={() => navigate(-1)}
        class="absolute left-4 md:left-8 top-1/2 -translate-y-1/2 z-20 w-10 h-10 flex items-center justify-center rounded-full bg-black/70 text-white hover:bg-black/90 transition-colors cursor-pointer shadow-lg ring-1 ring-white/20 pointer-events-auto"
        aria-label="Previous image"
      >
        <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />
        </svg>
      </button>

      <!-- Nav: Next -->
      <button
        onclick={() => navigate(1)}
        class="absolute right-4 md:right-8 top-1/2 -translate-y-1/2 z-20 w-10 h-10 flex items-center justify-center rounded-full bg-black/70 text-white hover:bg-black/90 transition-colors cursor-pointer shadow-lg ring-1 ring-white/20 pointer-events-auto"
        aria-label="Next image"
      >
        <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" d="M8.25 4.5l7.5 7.5-7.5 7.5" />
        </svg>
      </button>
    {/if}

    <!-- Image viewport -->
    <div
      bind:this={viewportRef}
      class="flex items-center justify-center select-none overflow-hidden pointer-events-auto mx-4 md:mx-20 my-16"
      class:cursor-zoom-in={!isZoomed}
      class:cursor-grab={isZoomed && !isDragging}
      class:cursor-grabbing={isDragging}
      role="img"
      aria-label={images[currentIndex]?.alt}
      onwheel={handleWheel}
      onmousedown={handleMouseDown}
      onmousemove={handleMouseMove}
      onmouseup={handleMouseUp}
      onmouseleave={handleMouseUp}
      ontouchstart={handleTouchStart}
      ontouchmove={handleTouchMove}
      ontouchend={handleTouchEnd}
    >
      <img
        bind:this={imageRef}
        src={images[currentIndex]?.src}
        alt={images[currentIndex]?.alt}
        draggable="false"
        class="max-w-full object-contain rounded-lg"
        style="max-height: {isPortrait ? `${viewportW - 128}px` : '85vh'}; transform: scale({scale}) translate({translateX / scale}px, {translateY / scale}px); transition: {isDragging || isTouchDragging ? 'none' : 'transform 0.3s cubic-bezier(0.4, 0, 0.2, 1)'};"
        onclick={handleImageClick}
      />
    </div>
  </div>

  <!-- Backdrop -->
  <form method="dialog" class="modal-backdrop bg-transparent">
    <button>close</button>
  </form>
</dialog>
