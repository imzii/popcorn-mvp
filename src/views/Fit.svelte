<script>
  // FIT: 팀원 초대, 팀 구성, 역할 조율
  // 역할 조율표는 각자 낸 희망 역할을 먼저 보여 주고, 희망이 겹치거나 아무도 원하지 않는 역할에만
  // 성향 기반 추천을 '참고'로 붙인다. 추천이 먼저 보이면 그대로 따르게 되기 쉬워서다(앵커링).
  import { copyText } from '../lib/util.js';
  import { app, save, members, memberName, isLeader, typeProfile } from '../lib/app.svelte.js';
  import { teamComposition, recommendRoles, recommendReason } from '../lib/ai.js';
  import Avatar from '../components/Avatar.svelte';
  import BottomBar from '../components/BottomBar.svelte';

  const room = $derived(app.room);
  const team = $derived(teamComposition());
  const inviteLink = `${location.origin}${location.pathname}?r=${app.code}${app.isLocal && app.hasRemote ? '&local=1' : ''}`;

  const roles = $derived(typeProfile().roles.map(role => {
    const wanters = members().filter(([, m]) => (m.wants || []).includes(role[0])).map(([id]) => id);
    const owner = recommendRoles(room)[role[0]];
    return { role, wanters, hint: wanters.length === 1 || !owner ? null : { owner, reason: recommendReason(room.members[owner], role) } };
  }));
  const toDiscuss = $derived(roles.filter(r => r.wanters.length !== 1).length);
  const enough = $derived(members().length >= 2);

  function share() {
    navigator.share({ title: 'POPCORN 초대', text: `${room.name} 팀 프로젝트에 참여해 주세요`, url: inviteLink }).catch(() => {});
  }

  function startPop() {
    const now = Date.now();
    save({ phase: 'pop', phaseAt: now, popEnd: now + (room.popMinutes || room.popMin || 1440) * 60000 });
  }
  import TitleBand from '../components/TitleBand.svelte';
</script>

<TitleBand label="이번에 함께 정할 것" title={room.topic} sub={room.name} />

<div class="card"><h3>팀원 초대</h3>
  <div class="row between" style="margin:10px 0">
    <span class="invite-code" aria-label="초대 코드">{app.code}</span>
    <button id="copyLink" onclick={() => copyText(inviteLink)}>초대 링크 복사</button>
  </div>
  {#if navigator.share}<button class="btn-link" onclick={share}>공유하기…</button>{/if}
</div>

<div class="card">
  <div class="row between"><h3>팀원 {members().length}명</h3><span class="chip on">{team.kind === 'same' ? '동일 전공 팀' : '융합 전공 팀'}</span></div>
  <div class="people" style="margin-top:8px">
    {#each members() as [id, m] (id)}
      <div class="person"><Avatar {id} /><div>
        <b>{m.name}</b>{#if id === room.hostId}<span class="xs sub"> (리더)</span>{/if}
        <div class="xs sub">{m.major}, {m.trait}</div>
      </div></div>
    {/each}
  </div>
</div>

{#if members().length}
  <div class="card">
    <div class="row between"><h3>역할 조율</h3><span class="sm sub">{toDiscuss ? `이야기할 역할 ${toDiscuss}개` : '모두 희망대로 나뉘었어요'}</span></div>
    <table style="margin-top:6px">
      <thead><tr><th>역할</th><th>희망한 사람</th><th>상태</th></tr></thead>
      <tbody>
        {#each roles as { role, wanters, hint }}
          <tr>
            <td><b>{role[0]}</b><div class="xs sub">{role[1]}시간</div></td>
            <td>
              <div class="people">
                {#each wanters as id}<span class="row" style="gap:6px"><Avatar {id} />{memberName(id)}</span>{:else}-{/each}
              </div>
              {#if hint}<div class="xs sub" style="margin-top:4px">참고: {memberName(hint.owner)} ({hint.reason})</div>{/if}
            </td>
            <td>
              {#if wanters.length === 1}<span class="xs sub">희망 1명</span>
              {:else if wanters.length}<span class="chip on">조율 필요</span>
              {:else}<span class="xs sub">희망자 없음</span>{/if}
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  </div>
{/if}

{#if isLeader()}
  <BottomBar message={enough ? '' : '팀원이 2명 이상이면 시작할 수 있어요'}>
    {#snippet actions()}<button class="btn-primary" id="startPop" disabled={!enough} onclick={startPop}>POP 시작</button>{/snippet}
  </BottomBar>
{/if}
