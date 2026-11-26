<script>
  // 화면 맨 위 버터색 제목 영역. 화면마다 하나만 쓴다.
  // title: 문자열 또는 줄 단위 배열. 사용자가 쓴 글이 들어가므로 HTML로 넣지 않는다.
  // scatter: 첫 화면처럼 팝콘 몇 알을 흩어 놓을지 (다른 화면은 한 알만)
  import Popcorn from './Popcorn.svelte';
  let { label = '', title, sub = '', scatter = false, children } = $props();
  const kernels = scatter
    ? [{ size: 64, right: 18, top: 14, rot: -12 }, { size: 34, right: 2, top: 86, rot: 8 }, { size: 22, right: 78, top: 104, rot: 20 }]
    : [{ size: 36, right: 14, top: 12, rot: 10 }];
</script>

<section class="title-band" class:hero={scatter}>
  {#each kernels as k}
    <svg class="band-kernel" viewBox="0 0 46 46" aria-hidden="true"
      style="width:{k.size}px;height:{k.size}px;right:{k.right}px;top:{k.top}px;transform:rotate({k.rot}deg)"><Popcorn /></svg>
  {/each}
  {#if label}<p class="band-label">{label}</p>{/if}
  <h1>{#each [title].flat() as line, i}{#if i}<br>{/if}{line}{/each}</h1>
  {#if sub}<p class="band-sub">{sub}</p>{/if}
  {@render children?.()}
</section>
