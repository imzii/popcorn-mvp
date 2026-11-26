<script>
  // 머리글: 로고, 프로젝트 이름, 단계 표시.
  // 단계는 알갱이로 보여 준다. 지난 단계는 터진 팝콘, 지금 단계는 빨간 표시, 남은 단계는 안 터진 알갱이.
  import { STAGES } from '../lib/content.js';
  import { app, currentPhase } from '../lib/app.svelte.js';
  import Popcorn from './Popcorn.svelte';
  const current = $derived(app.room ? STAGES.findIndex(s => s.phase === currentPhase()) : -1);
</script>

<header>
  <div class="header-row">
    <a class="logo" href="./"><img src="./favicon.svg" alt="" width="24" height="24">POPCORN</a>
    <span class="project-name">{app.room?.name ?? ''}</span>
  </div>
  {#if app.room}
    <nav class="steps" aria-label="진행 단계">
      {#each STAGES as stage, i}
        <div class="step" class:current={i === current} class:done={i < current} aria-current={i === current ? 'step' : undefined}>
          <svg viewBox="0 0 46 46" aria-hidden="true">
            {#if i < current}<Popcorn />
            {:else if i === current}<circle cx="23" cy="23" r="13" fill="#fff" stroke="var(--red)" stroke-width="4"/><circle cx="23" cy="23" r="5" fill="var(--red)"/>
            {:else}<circle cx="23" cy="23" r="7" fill="var(--line)"/>{/if}
          </svg>
          {stage.label}
        </div>
      {/each}
    </nav>
  {/if}
</header>
