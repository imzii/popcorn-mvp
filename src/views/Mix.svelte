<script>
  // MIX: 의견 수렴. 소수 의견을 다수 의견보다 먼저 보여 주고, 평가 기준으로 후보를 평가한다.
  import { toast, today, addDays, newId } from '../lib/util.js';
  import { app, save, saveUi, isLeader, criteriaOf, members, roundNumber } from '../lib/app.svelte.js';
  import { splitOpinions, teamComposition, questionForMinority, questionForMajority, buildPlan } from '../lib/ai.js';
  import BottomBar from '../components/BottomBar.svelte';
  import Reaction from '../components/Reaction.svelte';
  import TwoStepButton from '../components/TwoStepButton.svelte';

  const room = $derived(app.room);

  // 이 기기에서 이번 라운드의 MIX를 처음 볼 때만 카드가 톡톡 나타난다.
  const firstLook = app.ui.mixSeen !== roundNumber();
  if (firstLook) { app.ui.mixSeen = roundNumber(); saveUi(); }
  const delay = i => `--i:${Math.min(i, 10)}`;
  const mix = $derived(room.mix || {});
  const criteria = $derived(criteriaOf());
  const opinions = $derived(splitOpinions(room));
  const team = $derived(teamComposition());
  const scores = $derived(mix.scores || {});
  const final = $derived(mix.final);

  // 앵커링 방지: 전원이 '평가 완료'를 누르기 전에는 다른 사람의 선택 수와 평가 결과를 보여 주지 않는다.
  const doneCount = $derived(members().filter(([id]) => mix.done?.[id]).length);
  const waiting = $derived(members().length - doneCount);
  const revealed = $derived(!!final || waiting === 0);
  const iAmDone = $derived(!!mix.done?.[app.me]);

  // 후보: 소수 의견 카드들, 그다음 다수 의견(대표 카드)
  const candidates = $derived([
    ...opinions.minority.map(c => ({ key: c.id, text: c.text, kind: '소수' })),
    ...(opinions.center ? [{ key: 'majority', text: opinions.center.text, kind: '다수' }] : []),
  ]);
  const totalOf = key => Object.values(scores[key] || {}).reduce((sum, checks) => sum + checks.filter(Boolean).length, 0);
  const checked = (key, i) => !!scores[key]?.[app.me]?.[i];

  function toggleCriterion(key, index) {
    const checks = criteria.map((_, i) => checked(key, i));
    checks[index] = !checks[index];
    save({ [`mix.scores.${key}.${app.me}`]: checks });
  }

  function finalize(c) {
    save({ 'mix.final': { key: c.key, text: c.text } });
    toast('최종안으로 확정했어요');
  }

  function startRun() {
    // 첫 라운드는 실행 계획을 만들고, 이후 라운드는 기존 업무에 '반영' 업무 하나만 더한다.
    if (!room.tasks) return save({ phase: 'run', phaseAt: Date.now(), tasks: buildPlan(room) });
    const due = [addDays(today(), 3), room.deadline].filter(Boolean).sort()[0];
    save({ phase: 'run', phaseAt: Date.now(),
      [`tasks.${newId('t', 6)}`]: { title: `반영: ${final.text}`, role: '', who: '', due, status: 0, order: 99 } });
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

{#snippet idea(card)}
  <div class="idea-title">{card.text}</div>{#if card.why}<div class="idea-why">{card.why}</div>{/if}
{/snippet}

<div class:reveal={firstLook}>
<TitleBand label="이번에 함께 정할 것" title={room.topic} sub="의견 {opinions.cards.length}장이 모였어요" />

<div class="card note"><div class="note-title">융합 협업 가이드</div>
  {#if team.kind === 'same'}
    의견 유사도가 {Math.round(opinions.similarityAvg * 100)}%예요. 같은 전공의 익숙한 틀에서 나왔을 수 있으니 소수 의견부터 이야기해 보세요.
  {:else}
    {team.majors.join(', ')} 전공이 모였어요. 낯선 용어가 있는 카드는 작성자가 한 줄로 풀어 설명하고, 전공 사이의 접점을 찾아보세요.
  {/if}
</div>

<section class="card">
  <div class="row between"><h3><span class="badge minority">B</span>소수 의견</h3><span class="sm sub">{opinions.minority.length}장</span></div>
  {#each opinions.minority as card, i (card.id)}
    <div class="idea" style={delay(i)}>{@render idea(card)}
      <div class="sm" style="margin-top:8px"><b>생각해 볼 질문</b> {questionForMinority(card)}</div>
      <Reaction key={card.id} {revealed} />
    </div>
  {:else}
    <p class="sub sm">모든 의견이 비슷한 방향이에요.</p>
  {/each}
</section>

<section class="card">
  <div class="row between"><h3><span class="badge">A</span>다수 의견</h3><span class="sm sub">{opinions.majority.length}장</span></div>
  {#each opinions.majority as card, i (card.id)}<div class="idea" style={delay(opinions.minority.length + i)}>{@render idea(card)}</div>{/each}
  {#if opinions.center}
    <div class="sm" style="margin-top:10px"><b>생각해 볼 질문</b> {questionForMajority(opinions.center)}</div>
    <Reaction key="majority" {revealed} />
  {/if}
</section>

<section class="card">
  <div class="row between"><h3>후보 평가</h3><span class="sm sub">평가 완료 {doneCount}/{members().length}명</span></div>
  {#if !revealed}<p class="sm sub" style="margin:4px 0 0">모두 평가를 마치면 결과가 공개돼요.</p>{/if}
  {#each candidates as c (c.key)}
    <div class="idea">
      <div class="row between">
        <span class="chip" class:on={c.kind === '소수'}>{c.kind} 의견</span>
        {#if revealed}<span class="xs sub">충족 {totalOf(c.key)}표</span>{/if}
      </div>
      <div class="idea-title" style="margin-top:6px">{c.text}</div>
      <div class="criteria">
        {#each criteria as name, i}
          <button type="button" data-score={c.key} aria-pressed={checked(c.key, i)} onclick={() => toggleCriterion(c.key, i)}>{name}</button>
        {/each}
      </div>
      {#if isLeader()}
        <TwoStepButton class="btn-link" data-final={c.key} label="이 안으로 최종 확정" confirmLabel="평가 중인 사람이 있어요. 한 번 더 누르면 확정"
          needConfirm={!revealed} onconfirm={() => finalize(c)} />
      {/if}
    </div>
  {/each}
</section>

{#if final}
  <div class="card emphasis"><h3>최종안</h3><p style="font-weight:700;font-size:17px;margin-top:4px">{final.text}</p></div>
{/if}

</div>

<BottomBar message={final ? '최종안이 확정됐어요' : waiting ? `${waiting}명이 평가 중이에요` : '모두 평가를 마쳤어요'}>
  {#snippet left()}
    {#if !iAmDone}<button id="finishScoring" onclick={() => save({ [`mix.done.${app.me}`]: true })}>평가 완료</button>{/if}
  {/snippet}
  {#snippet actions()}
    {#if isLeader()}<button class="btn-primary" id="startRun" disabled={!final} onclick={startRun}>RUN 시작</button>{/if}
  {/snippet}
</BottomBar>
