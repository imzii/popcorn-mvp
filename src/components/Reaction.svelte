<script>
  // '반영해 보자' O / X 선택. key는 소수 의견 카드 id 또는 다수 의견 묶음 'majority'.
  // 저장 값은 예전과 같이 yes(O) / hold(X)를 쓴다.
  // 전원이 평가를 마치기 전에는 다른 사람의 선택 수를 숨긴다(앵커링 방지).
  import { app, save } from '../lib/app.svelte.js';
  let { key, revealed } = $props();

  const votes = $derived(app.room.mix?.reactions?.[key] || {});
  const mine = $derived(votes[app.me]);
  const count = v => Object.values(votes).filter(x => x === v).length;
  const pick = v => save({ [`mix.reactions.${key}.${app.me}`]: mine === v ? null : v });
</script>

<div class="reaction">
  <span class="reaction-label" id="reaction-{key}">반영해 보자</span>
  <div class="toggle" role="group" aria-labelledby="reaction-{key}">
    <button type="button" data-react={key} data-value="yes" aria-pressed={mine === 'yes'} aria-label="반영해 보자 O" onclick={() => pick('yes')}>O{#if revealed}<span class="count">{count('yes')}</span>{/if}</button>
    <button type="button" data-react={key} data-value="hold" aria-pressed={mine === 'hold'} aria-label="반영해 보자 X" onclick={() => pick('hold')}>X{#if revealed}<span class="count">{count('hold')}</span>{/if}</button>
  </div>
</div>
