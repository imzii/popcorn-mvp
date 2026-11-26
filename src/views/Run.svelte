<script>
  // RUN: 협업 실행. 정한 것, 업무 표, 내 할 일, 팀원별 진행, 결과 내보내기, 새로 정할 것 추가
  import { toast, today, shortDate, daysBetween, newId, copyText, formatMinutes } from '../lib/util.js';
  import { TASK_STATUS, POP_DURATIONS } from '../lib/content.js';
  import { app, save, members, memberName, isLeader, typeProfile, criteriaOf, decisions, roundNumber } from '../lib/app.svelte.js';
  import { recommendRoles } from '../lib/ai.js';
  import Avatar from '../components/Avatar.svelte';
  import BottomBar from '../components/BottomBar.svelte';
  import TwoStepButton from '../components/TwoStepButton.svelte';

  const room = $derived(app.room);
  const tasks = $derived(Object.entries(room.tasks || {})
    .sort((a, b) => (a[1].due || '').localeCompare(b[1].due || '') || (a[1].order ?? a[1].i ?? 0) - (b[1].order ?? b[1].i ?? 0)));
  const suggested = $derived(recommendRoles(room));
  const done = $derived(tasks.filter(([, t]) => t.status === 2).length);
  const unassigned = $derived(tasks.filter(([, t]) => !t.who).length);
  const percent = $derived(tasks.length ? Math.round(done / tasks.length * 100) : 0);
  const myOpen = $derived(tasks.filter(([, t]) => t.who === app.me && t.status !== 2));
  const daysLeft = $derived(room.deadline ? daysBetween(today(), room.deadline) : null);

  let taskTitle = $state(''), taskDue = $state(room.deadline || today());
  let nextQuestion = $state(''), nextMinutes = $state(room.popMinutes || 1440);

  function addTask(e) {
    e.preventDefault();
    const title = taskTitle.trim();
    if (!title) return;
    save({ [`tasks.${newId('t', 6)}`]: { title, role: '', who: '', due: taskDue, status: 0, order: 99 } });
    taskTitle = '';
  }

  const cycleStatus = id => save({ [`tasks.${id}.status`]: ((room.tasks[id].status || 0) + 1) % TASK_STATUS.length });

  /** 지금 라운드를 보관하고, 같은 팀으로 새 질문의 POP을 연다. 업무와 팀원은 그대로다. */
  function startNextRound(e) {
    e.preventDefault();
    const question = nextQuestion.trim();
    if (!question) return toast('새로 정할 것을 적어 주세요');
    const now = Date.now(), n = roundNumber(), minutes = +nextMinutes;
    save({
      [`rounds.${n}`]: { topic: room.topic, cards: room.cards || {}, mix: room.mix || room.sync || null, popMinutes: room.popMinutes || null, closedAt: now },
      round: n + 1, topic: question, cards: {}, mix: null, sync: null,
      popMinutes: minutes, popEnd: now + minutes * 60000, phase: 'pop', phaseAt: now,
    });
  }

  /** 보고서에 붙일 수 있는 결과 정리 (마크다운: Notion은 표로, 한글에는 글로 붙는다) */
  function resultText() {
    return [
      `# ${room.name}`, '',
      `- 유형: ${typeProfile().names.join(', ')}`,
      `- 평가 기준: ${criteriaOf().join(', ')}`,
      `- 최종 마감일: ${room.deadline || '-'}`, '',
      '## 정한 것',
      ...decisions().map((d, i) => `${i + 1}. ${d.question}\n   - ${d.final || '-'}`), '',
      '## 팀원과 맡은 업무',
      ...members().map(([id, m]) => `- ${m.name} (${m.major}, ${m.trait}): ${tasks.filter(([, t]) => t.who === id).map(([, t]) => t.title).join(', ') || '-'}`), '',
      '## 업무',
      '| 업무 | 담당 | 마감 | 상태 |', '| --- | --- | --- | --- |',
      ...tasks.map(([, t]) => `| ${t.title} | ${t.who ? memberName(t.who) : '미배정'} | ${t.due || '-'} | ${TASK_STATUS[t.status]} |`),
    ].join('\n');
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

<TitleBand label="실행 중" title={room.name} />

<div class="card"><h3>정한 것</h3>
  {#each decisions() as d, i}
    <div class="idea"><div class="xs sub">{i + 1}. {d.question}</div><div class="idea-title" style="margin-top:2px">{d.final || '-'}</div></div>
  {/each}
  {#if isLeader()}
    <form id="newRound" class="no-print" novalidate style="margin-top:12px" onsubmit={startNextRound}>
      <label for="nextQuestion">새로 정할 것</label>
      <input id="nextQuestion" maxlength="80" bind:value={nextQuestion} placeholder="예: 발표는 누가, 어떤 순서로 할까?" autocomplete="off">
      <div class="row" style="margin-top:8px">
        <label for="nextMinutes" class="sr-only">의견 내는 기간</label>
        <select id="nextMinutes" class="grow" bind:value={nextMinutes}>
          {#each POP_DURATIONS as m}<option value={m}>{formatMinutes(m)} 동안 의견 받기</option>{/each}
        </select>
        <button type="submit">POP 시작</button>
      </div>
    </form>
  {/if}
</div>

<div class="card">
  <div class="row between"><b>전체 진행률 {percent}%</b>
    <span class="sm sub">{done}/{tasks.length}개 완료{#if daysLeft != null}, 마감까지 {daysLeft >= 0 ? daysLeft + '일' : '지남'}{/if}</span></div>
  <div class="gauge" style="margin-top:8px" role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow={percent}><i style:width="{percent}%"></i></div>
</div>

{#if myOpen.length}
  <div class="card emphasis no-print"><h3>내 할 일 {myOpen.length}개</h3>
    {#each myOpen as [id, t] (id)}<div class="sm" style="margin-top:6px">{t.title} <span class="sub">{shortDate(t.due)}까지</span></div>{/each}
  </div>
{/if}

<div class="card" style="padding:8px 12px">
  <table>
    <thead><tr><th>업무</th><th>담당</th><th>마감</th><th>상태</th></tr></thead>
    <tbody>
      {#each tasks as [id, t] (id)}
        <tr>
          <td><b class="sm">{t.title}</b>{#if t.role}<div class="xs sub">{t.role}</div>{/if}</td>
          <td>
            <label class="sr-only" for="owner-{id}">담당</label>
            <select id="owner-{id}" class="compact-select" value={t.who || ''} onchange={e => save({ [`tasks.${id}.who`]: e.target.value })}>
              <option value="">미배정</option>
              {#each members() as [mid, m] (mid)}<option value={mid}>{m.name}</option>{/each}
            </select>
            {#if !t.who}
              <div class="no-print" style="margin-top:4px">
                <button class="btn-link" data-take={id} onclick={() => save({ [`tasks.${id}.who`]: app.me })}>내가 맡기</button>
                {#if suggested[t.role]}<div class="xs sub">참고: {memberName(suggested[t.role])}</div>{/if}
              </div>
            {/if}
          </td>
          <td class="sm" style="white-space:nowrap">{t.due ? shortDate(t.due) : '-'}</td>
          <td><button class="status status-{t.status}" data-status={id} aria-label="{t.title} 상태: {TASK_STATUS[t.status]}. 눌러서 변경" onclick={() => cycleStatus(id)}>{TASK_STATUS[t.status]}</button></td>
        </tr>
      {/each}
    </tbody>
  </table>
  <form class="row no-print" style="margin:10px 0 4px" novalidate onsubmit={addTask}>
    <label for="taskTitle" class="sr-only">업무 추가</label><input id="taskTitle" class="grow" bind:value={taskTitle} placeholder="업무 추가" autocomplete="off">
    <input type="date" style="width:auto" bind:value={taskDue} aria-label="마감일"><button type="submit">추가</button>
  </form>
</div>

<div class="card"><h3>팀원별 진행</h3>
  {#each members() as [id, m] (id)}
    {@const own = tasks.filter(([, t]) => t.who === id)}
    {@const finished = own.filter(([, t]) => t.status === 2).length}
    <div class="row" style="margin:8px 0"><Avatar {id} /><span class="sm" style="width:64px">{m.name}</span>
      <div class="gauge grow"><i style:width="{own.length ? finished / own.length * 100 : 0}%"></i></div>
      <span class="sm sub" style="width:44px;text-align:right">{finished}/{own.length}</span></div>
  {/each}
</div>

<div class="card no-print"><h3>결과 내보내기</h3>
  <div class="row" style="margin-top:8px"><button id="copyResult" onclick={() => copyText(resultText())}>결과 복사 (Notion·한글)</button><button id="printResult" onclick={() => window.print()}>PDF로 저장</button></div>
</div>

<BottomBar message={unassigned ? `담당이 없는 업무가 ${unassigned}개 있어요` : percent === 100 ? '모든 업무를 마쳤어요' : ''}>
  {#snippet actions()}
    {#if isLeader()}<TwoStepButton id="backToMix" label="최종안 다시 보기" confirmLabel="한 번 더 누르면 이동" onconfirm={() => save({ phase: 'mix' })} />{/if}
  {/snippet}
</BottomBar>
