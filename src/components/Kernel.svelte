<script>
  // 팀원 한 명의 제출 현황. 다 쓰기 전에는 진행 링(장수), 최소 장수를 채우면 팝콘으로 터진다.
  // 터지는 움직임은 이 화면에서 상태가 바뀌는 순간에만 한 번 나온다(처음부터 다 쓴 사람은 그냥 팝콘).
  let { count, min, name, me = false } = $props();

  const R = 15, C = 2 * Math.PI * R;
  const done = $derived(count >= min);
  const filled = $derived(C * Math.min(count, min) / min);

  let wasDone = count >= min;
  let justPopped = $state(false);
  $effect(() => {
    if (done && !wasDone) justPopped = true;
    wasDone = done;
  });
</script>

<div class="kernel">
  <svg viewBox="0 0 46 46" class:pop={justPopped} aria-hidden="true">
    {#if done}
      <g fill="#F5B800"><circle cx="16" cy="25" r="10"/><circle cx="30" cy="25" r="10"/><circle cx="23" cy="15" r="10.5"/><circle cx="23" cy="31" r="9"/></g>
      <circle cx="19" cy="14" r="3" fill="#FFE38A"/>
    {:else}
      <circle cx="23" cy="23" r={R} fill="none" stroke="var(--line)" stroke-width="4"/>
      {#if count}<circle cx="23" cy="23" r={R} fill="none" stroke="var(--ink)" stroke-width="4" stroke-linecap="round" stroke-dasharray="{filled} {C}" transform="rotate(-90 23 23)"/>{/if}
      <text x="23" y="27.5" text-anchor="middle" font-size="12" font-weight="700" fill="var(--ink)">{count}</text>
    {/if}
  </svg>
  <b>{name}{me ? ' (나)' : ''}</b>
  <span>{done ? '다 썼어요' : `${count}/${min}장`}</span>
</div>
