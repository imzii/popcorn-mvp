<script>
  // 참여: 자기소개(1/2)와 협업 성향 진단(2/2). 다른 기기에서 이어하기도 여기서 한다.
  import { toast } from '../lib/util.js';
  import { QUIZ, TRAITS } from '../lib/content.js';
  import { app, saveUi, save, members, memberName, typeProfile, claimIdentity } from '../lib/app.svelte.js';
  import BottomBar from '../components/BottomBar.svelte';

  const profile = app.ui.profile;
  const answers = app.ui.quiz;
  let step = $state(profile.step || 1);
  let claiming = $state(null);   // 이어하기로 고른 팀원 id (한 번 더 누르면 전환)

  const roles = $derived(typeProfile().roles);
  const allAnswered = $derived(QUIZ.every((_, i) => answers[i] != null));

  function toggleWant(role) {
    profile.wants = profile.wants.includes(role) ? profile.wants.filter(r => r !== role) : [...profile.wants, role];
    saveUi();
  }

  function claim(id) {
    if (claiming !== id) { claiming = id; return; }
    claimIdentity(id);
    location.reload();
  }

  function next() {
    if (!profile.name?.trim() || !profile.major?.trim()) {
      toast('이름과 전공을 적어 주세요');
      document.getElementById(profile.name?.trim() ? 'major' : 'memberName')?.focus();
      return;
    }
    step = profile.step = 2; saveUi();
  }

  function back() { step = profile.step = 1; saveUi(); }

  function answer(i, v) { answers[i] = v; saveUi(); }

  async function join() {
    const scores = {};
    QUIZ.forEach((q, i) => { const trait = q[2 + answers[i]]; scores[trait] = (scores[trait] || 0) + 1; });
    const trait = Object.keys(TRAITS).sort((x, y) => (scores[y] || 0) - (scores[x] || 0))[0];
    await save({ [`members.${app.me}`]: {
      name: profile.name.trim(), major: profile.major.trim(), skills: (profile.skills || '').trim(),
      wants: [...profile.wants], trait, scores, joinedAt: Date.now(),
    } });
    toast(`${trait}: ${TRAITS[trait]}`);
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

{#if step === 1}
  <TitleBand label="이번에 함께 정할 것" title={app.room.topic} sub={app.room.name} />
  <form class="card" novalidate onsubmit={e => e.preventDefault()}>
    <h2 style="margin-top:0">나를 소개해 주세요 <span class="xs sub">1/2</span></h2>
    {#if members().length}
      <details class="sm" style="margin:-4px 0 6px">
        <summary style="cursor:pointer;font-weight:600;padding:6px 0">이미 참여했어요 (다른 기기에서 이어하기)</summary>
        <div class="options" style="margin-top:6px">
          {#each members() as [id, m] (id)}
            <button type="button" data-claim={id} aria-pressed={claiming === id} onclick={() => claim(id)}>
              {claiming === id ? `${memberName(id)}(으)로 이어하기` : m.name}
            </button>
          {/each}
        </div>
      </details>
    {/if}
    <label for="memberName">이름</label><input id="memberName" bind:value={profile.name} oninput={saveUi} autocomplete="name">
    <label for="major">전공</label><input id="major" bind:value={profile.major} oninput={saveUi} placeholder="예: 경영학과">
    <label for="skills">자신 있는 분야나 경험</label><input id="skills" bind:value={profile.skills} oninput={saveUi} placeholder="예: PPT, 자료조사, 영상편집">
    <label>희망 역할 <span class="xs sub">(여러 개 선택)</span></label>
    <div class="options">
      {#each roles as [role]}
        <button type="button" data-want={role} aria-pressed={profile.wants.includes(role)} onclick={() => toggleWant(role)}>{role}</button>
      {/each}
    </div>
  </form>
  <BottomBar>{#snippet actions()}<button class="btn-primary" id="next" onclick={next}>다음</button>{/snippet}</BottomBar>
{:else}
  <h1>협업 성향 진단 <span class="xs sub">2/2</span></h1>
  {#each QUIZ as [a, b], i}
    <div class="card"><h3>Q{i + 1}. 둘 중 더 편한 쪽은?</h3>
      <div class="choices" role="radiogroup" aria-label="질문 {i + 1}">
        {#each [a, b] as text, v}
          <button type="button" role="radio" data-q={i} data-v={v} aria-pressed={answers[i] === v} aria-checked={answers[i] === v} onclick={() => answer(i, v)}>{text}</button>
        {/each}
      </div>
    </div>
  {/each}
  <BottomBar>
    {#snippet left()}<button id="back" onclick={back}>이전</button>{/snippet}
    {#snippet actions()}<button class="btn-primary" id="join" disabled={!allAnswered} onclick={join}>참여하기</button>{/snippet}
  </BottomBar>
{/if}
