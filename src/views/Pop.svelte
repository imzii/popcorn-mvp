<script>
  // POP: 각자 의견 내기. 다른 팀원의 카드는 보이지 않고, 제출 현황만 실시간으로 보인다.
  // 전원이 최소 장수를 내거나 기간이 끝나면 MIX로 자동으로 넘어간다.
  import { toast, newId, formatRemaining, formatMinutes } from '../lib/util.js';
  import { app, saveUi, save, members, cardsOf, isLeader, typeProfile, roundNumber } from '../lib/app.svelte.js';
  import { seasoningQuestion, nextWhy } from '../lib/ai.js';
  import BottomBar from '../components/BottomBar.svelte';
  import TwoStepButton from '../components/TwoStepButton.svelte';
  import Kernel from '../components/Kernel.svelte';

  const ui = app.ui;
  // 정할 것이 바뀌면(다음 라운드) 이 기기의 작성 중 상태를 비운다.
  if (ui.round !== roundNumber()) { Object.assign(ui, { round: roundNumber(), draft: '', why: '', hintIndex: 0, whyChain: [] }); saveUi(); }

  // 남은 시간 표시용 시계
  let now = $state(Date.now());
  $effect(() => { const t = setInterval(() => (now = Date.now()), 1000); return () => clearInterval(t); });

  const room = $derived(app.room);
  const minCards = $derived(room.cardsPerPerson || room.per || 1);
  const cards = $derived(Object.values(room.cards || {}));
  const countOf = id => cards.filter(c => c.by === id).length;
  const doneCount = $derived(members().filter(([id]) => countOf(id) >= minCards).length);
  const waiting = $derived(members().length - doneCount);
  const left = $derived(room.popEnd ? room.popEnd - now : Infinity);
  const over = $derived(left <= 0);
  const mine = $derived(cardsOf(app.me));
  const hint = $derived(seasoningQuestion(room, ui.hintIndex));
  const extendBy = $derived((room.popMinutes || 1440) >= 720 ? 1440 : 10);

  // 모든 기기가 같은 조건을 확인하므로, 누가 먼저 넘기든 결과는 같다.
  let advanced = false;
  $effect(() => {
    if (!advanced && cards.length && (waiting === 0 || over)) { advanced = true; save({ phase: 'mix', phaseAt: Date.now() }); }
  });

  let hintKey = $state(0);   // 새 질문이 나올 때 튀는 효과를 다시 재생하기 위한 키
  let whyAnswer = $state('');

  async function addCard(e) {
    e.preventDefault();
    if (over) return toast('발산 기간이 끝났어요');
    const text = ui.draft.trim();
    if (!text) return toast('의견을 한 줄 적어 주세요');
    const why = ui.why.trim();
    ui.draft = ''; ui.why = ''; saveUi();
    await save({ [`cards.${newId('c')}`]: { text, why, by: app.me, at: Date.now() } });
    document.getElementById('ideaText')?.focus();
  }

  function nextHint() { ui.hintIndex++; saveUi(); hintKey++; }

  function dig(e) {
    e.preventDefault();
    const answer = whyAnswer.trim();
    if (!answer) return;
    ui.whyChain.push({ q: nextWhy(ui.whyChain), a: answer }); saveUi();
    whyAnswer = '';
  }

  function extend() {
    save({ popEnd: Math.max(Date.now(), room.popEnd || 0) + extendBy * 60000 });
    toast(`${formatMinutes(extendBy)} 연장했어요`);
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

<TitleBand label="이번에 함께 정할 것" title={room.topic} />

<div class="card" id="popStatus">
  <div class="row between"><b>제출 현황</b><span class="sm sub">카드 {cards.length}장, 다 쓴 사람 {doneCount}/{members().length}명</span></div>
  <div class="row between" style="margin:10px 0 6px">
    <span class="sm">{over ? '발산 기간이 끝났어요' : '남은 시간'}</span>
    <span class="timer">{over ? '0:00' : Number.isFinite(left) ? formatRemaining(left) : '-'}</span>
  </div>
  <div class="kernels">
    {#each members() as [id, m] (id)}<Kernel count={countOf(id)} min={minCards} name={m.name} me={id === app.me} />{/each}
  </div>
</div>

<form class="card" id="cardForm" onsubmit={addCard} novalidate>
  <h3>내 의견 카드 <span class="sub">{mine.length}장 (최소 {minCards}장)</span></h3>
  <fieldset disabled={over}>
    <label for="ideaText">의견 한 줄</label>
    <input id="ideaText" maxlength="120" bind:value={ui.draft} oninput={saveUi} placeholder="예: {typeProfile().opinion}" autocomplete="off">
    <label for="ideaWhy">이유 한 줄 <span class="xs sub">(선택)</span></label>
    <input id="ideaWhy" maxlength="120" bind:value={ui.why} oninput={saveUi} placeholder="왜 그렇게 생각했나요?" autocomplete="off">
    <button class="btn-primary wide" style="margin-top:12px" type="submit">카드 추가</button>
  </fieldset>
  {#each mine as [id, c] (id)}
    <div class="idea row between">
      <div class="grow"><div class="idea-title">{c.text}</div>{#if c.why}<div class="idea-why">{c.why}</div>{/if}</div>
      {#if !over}<button type="button" class="btn-link" aria-label="카드 삭제" onclick={() => save({ [`cards.${id}`]: null })}>삭제</button>{/if}
    </div>
  {/each}
</form>

{#if !over}
  <section class="card seasoning" aria-labelledby="seasoningTitle">
    <div class="seasoning-head">
      <svg class="shaker" viewBox="0 0 24 24" width="26" height="26" aria-hidden="true">
        <circle cx="7" cy="4" r="1.2"/><circle cx="12" cy="2.6" r="1.2"/><circle cx="17" cy="4" r="1.2"/>
        <path d="M8 8.5h8l1.5 3V21a1 1 0 0 1-1 1h-9a1 1 0 0 1-1-1v-9.5z" fill="#fff" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
        <path d="M8 8.5c0-1.4 1.8-2.5 4-2.5s4 1.1 4 2.5" fill="none" stroke="currentColor" stroke-width="1.6"/>
        <circle cx="10.5" cy="15" r=".9"/><circle cx="13.5" cy="15" r=".9"/><circle cx="12" cy="17.6" r=".9"/>
      </svg>
      <h3 id="seasoningTitle">시즈닝</h3><span class="seasoning-tag">{hint.tag}</span>
    </div>
    {#key hintKey}<p class="seasoning-question" class:popped={hintKey > 0}>{hint.q}</p>{/key}
    <button type="button" id="nextHint" onclick={nextHint}>한 번 더 뿌리기</button>
  </section>

  <section class="card dig" aria-labelledby="digTitle">
    <div class="row between">
      <h3 id="digTitle">생각 파고들기 <span class="sub" style="font-weight:500">5 Whys</span></h3>
      <span class="dig-steps" aria-label="{ui.whyChain.length}/5단계">{#each [0, 1, 2, 3, 4] as i}<i class:on={i < ui.whyChain.length}></i>{/each}</span>
    </div>
    {#if ui.whyChain.length}
      <ol class="dig-chain">{#each ui.whyChain as step}<li><span class="sub">{step.q}</span><br>{step.a}</li>{/each}</ol>
    {/if}
    {#if ui.whyChain.length < 5}
      <form id="whyForm" novalidate onsubmit={dig}>
        <label for="whyAnswer" class="dig-question">{nextWhy(ui.whyChain)}</label>
        <div class="row"><input id="whyAnswer" class="grow" bind:value={whyAnswer} autocomplete="off" placeholder="떠오르는 대로 답해 보세요"><button type="submit">답하기</button></div>
      </form>
    {:else}
      <p class="sm" style="margin-top:8px">다섯 번 파고들었어요. 마지막 답을 의견 카드로 옮겨 보세요.</p>
    {/if}
    {#if ui.whyChain.length}<button type="button" class="btn-link" onclick={() => { ui.whyChain = []; saveUi(); }}>처음부터 다시</button>{/if}
  </section>
{/if}

{#if isLeader()}
  <BottomBar message={waiting ? `${waiting}명이 아직 작성 중이에요` : ''}>
    {#snippet left()}<button id="extend" onclick={extend}>{formatMinutes(extendBy)} 연장</button>{/snippet}
    {#snippet actions()}
      <TwoStepButton class="btn-primary" id="revealNow" label="지금 공개" confirmLabel="한 번 더 누르면 공개" needConfirm={waiting > 0}
        disabled={!cards.length} onconfirm={() => save({ phase: 'mix', phaseAt: Date.now() })} />
    {/snippet}
  </BottomBar>
{:else}
  <BottomBar message={over ? '곧 다음 단계로 넘어가요' : minCards - mine.length > 0 ? `의견을 ${minCards - mine.length}장 더 써 주세요` : '다 썼어요'} />
{/if}
