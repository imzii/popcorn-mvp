<script>
  // 첫 화면: 새 프로젝트 만들기, 초대 코드로 참여, 시뮬레이션
  import { today, addDays, randomId, formatMinutes, toast } from '../lib/util.js';
  import { TYPES, POP_DURATIONS, DEFAULT_POP_MINUTES } from '../lib/content.js';
  import { app, backend, recentProjects, typeProfile } from '../lib/app.svelte.js';
  import { openProject } from '../lib/nav.js';
  import { startSimulation } from '../lib/simulation.js';

  const recent = recentProjects();

  let name = $state(''), topic = $state('');
  let types = $state(['plan']);
  let deadline = $state(addDays(today(), 14));
  let cardsPerPerson = $state(3);
  let popMinutes = $state(DEFAULT_POP_MINUTES);
  let joinCode = $state('');

  // 평가 기준: 리더가 직접 고치기 전까지는 고른 유형의 기준을 따른다.
  let criteriaEdited = $state(false);
  let customCriteria = $state([]);
  let newCriterion = $state('');
  const profile = $derived(typeProfile({ types }));
  const criteria = $derived(criteriaEdited ? customCriteria : profile.criteria.slice(0, 6));

  function toggleType(key) {
    if (types.includes(key)) {
      if (types.length === 1) return toast('유형을 1개 이상 골라 주세요');
      types = types.filter(t => t !== key);
    } else types = [...types, key];
  }

  function editCriteria(next) { customCriteria = next; criteriaEdited = true; }
  function removeCriterion(i) {
    if (criteria.length <= 1) return toast('평가 기준은 1개 이상 필요해요');
    editCriteria(criteria.filter((_, j) => j !== i));
  }
  function addCriterion() {
    const value = newCriterion.trim();
    if (!value) return;
    if (criteria.includes(value)) return toast('이미 있는 기준이에요');
    if (criteria.length >= 6) return toast('평가 기준은 6개까지 둘 수 있어요');
    editCriteria([...criteria, value]);
    newCriterion = '';
  }

  async function create(e) {
    e.preventDefault();
    if (!name.trim() || !topic.trim()) return toast('프로젝트 이름과 정할 것을 적어 주세요');
    const code = randomId();
    await backend.store.create(code, {
      v: 4, name: name.trim(), topic: topic.trim(), types: [...types], criteria: [...criteria],
      deadline, cardsPerPerson: +cardsPerPerson, popMinutes: +popMinutes,
      hostId: app.me, createdAt: Date.now(), phase: 'fit', members: {}, cards: {},
    });
    openProject(code);
  }

  function join(e) {
    e.preventDefault();
    const code = joinCode.trim().toUpperCase();
    if (code.length !== 6) return toast('6자리 코드를 입력해 주세요');
    openProject(code, { local: false });
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

<TitleBand scatter title={['생각은 톡톡', '자유롭게,', '협업은 유기적으로.']} />

{#if recent.length}
  <section class="card">
    <h3>이 기기에서 참여한 프로젝트</h3>
    {#each recent as p (p.code)}
      <div class="row between" style="padding:8px 0;border-top:1px solid var(--line)">
        <span class="grow"><b>{p.name}</b> <span class="sm sub">{p.code}</span></span>
        <a href="?r={p.code}"><button>열기</button></a>
      </div>
    {/each}
  </section>
{/if}

<form class="card" id="newProject" onsubmit={create} novalidate>
  <h2 style="margin-top:0">새 프로젝트 만들기</h2>
  <label for="name">프로젝트 이름</label>
  <input id="name" bind:value={name} placeholder="예: 창의적기획과문제해결 2조" autocomplete="off">
  <label for="topic">이번에 함께 정할 것</label>
  <input id="topic" bind:value={topic} placeholder="예: {profile.question}" autocomplete="off">

  <label id="typeLabel">프로젝트 유형 <span class="xs sub">(여러 개 선택)</span></label>
  <div class="choices" role="group" aria-labelledby="typeLabel">
    {#each Object.entries(TYPES) as [key, t]}
      <button type="button" data-type={key} aria-pressed={types.includes(key)} onclick={() => toggleType(key)}><b>{t.name}</b></button>
    {/each}
  </div>

  <label for="newCriterion">평가 기준</label>
  <div class="options">
    {#each criteria as c, i}
      <span class="chip on" style="font-size:13px;padding:5px 6px 5px 10px">{c}<button type="button" class="remove" aria-label="{c} 삭제" onclick={() => removeCriterion(i)}>×</button></span>
    {/each}
  </div>
  <div class="row" style="margin-top:8px">
    <input id="newCriterion" class="grow" maxlength="20" bind:value={newCriterion} placeholder="기준 추가 (예: 근거의 구체성)" autocomplete="off"
      onkeydown={e => { if (e.key === 'Enter') { e.preventDefault(); addCriterion(); } }}>
    <button type="button" id="addCriterion" onclick={addCriterion}>추가</button>
  </div>

  <div class="grid-2">
    <div><label for="deadline">최종 마감일</label><input id="deadline" type="date" bind:value={deadline}></div>
    <div><label for="cardsPerPerson">1인 의견 카드 수 <span class="xs sub">(최소)</span></label>
      <select id="cardsPerPerson" bind:value={cardsPerPerson}>{#each [1, 2, 3, 4, 5] as n}<option value={n}>{n}</option>{/each}</select></div>
    <div><label for="popMinutes">의견 내는 기간</label>
      <select id="popMinutes" bind:value={popMinutes}>{#each POP_DURATIONS as m}<option value={m}>{formatMinutes(m)}</option>{/each}</select></div>
  </div>
  <button class="btn-primary wide" style="margin-top:16px" type="submit">프로젝트 만들고 팀원 초대하기</button>
</form>

<form class="card" id="joinByCode" onsubmit={join} novalidate>
  <h3>초대 코드로 참여</h3>
  <div class="row">
    <label for="code" class="sr-only">초대 코드</label>
    <input id="code" class="grow" bind:value={joinCode} placeholder="6자리 코드" maxlength="6" autocapitalize="characters" autocomplete="off" style="text-transform:uppercase">
    <button type="submit">참여</button>
  </div>
</form>

<section class="card">
  <h3>시뮬레이션으로 둘러보기</h3>
  <div class="grid-2" style="margin-top:8px">
    <button id="simSame" onclick={() => startSimulation('same')}>동일 전공 팀 (경영 4명)</button>
    <button id="simMixed" onclick={() => startSimulation('mixed')}>융합 전공 팀 (4개 전공)</button>
  </div>
</section>
{#if !app.hasRemote}<p class="foot">서버 설정이 없어 이 브라우저 안에서만 저장돼요.</p>{/if}
