---
title: "Web Audio API로 브라우저에서 신디사이저 만들기"
description: "설치 없이 브라우저에서 도는 신스는 어떻게 만들까. AudioNode 그래프 설계, 엔벨로프 구현, 모바일 오디오 정책까지 SoundLab 제작기."
pubDate: 2026-07-07
---

SoundLab의 신디사이저는 플러그인도 서버도 없이 브라우저 안에서 소리를 만든다. Web Audio API 덕분인데, 처음 만들 때 알았으면 좋았을 것들을 정리한다.

## Web Audio의 기본 모델: 노드 그래프

Web Audio는 모듈러 신스와 똑같이 생겼다. 소리를 만드는 노드(OscillatorNode), 가공하는 노드(BiquadFilterNode, GainNode), 출력(destination)을 케이블처럼 연결한다.

```js
const ctx = new AudioContext();
const osc = ctx.createOscillator();   // 오실레이터: 파형 생성
const filter = ctx.createBiquadFilter(); // 필터: 배음 깎기
const amp = ctx.createGain();         // 앰프: 볼륨

osc.connect(filter).connect(amp).connect(ctx.destination);
osc.start();
```

이 구조가 좋은 점은 교육용 UI와 오디오 구현이 1:1로 대응된다는 것이다. 화면의 신호 흐름도가 곧 실제 코드 구조다.

## 엔벨로프(ADSR)는 스케줄링으로

건반을 누르면 소리가 서서히 커지고(Attack), 떼면 잦아드는(Release) 것이 엔벨로프다. Web Audio에는 ADSR 노드가 없어서 GainNode의 파라미터 스케줄링으로 직접 구현한다.

```js
function noteOn(amp, t, { attack, decay, sustain }) {
  const g = amp.gain;
  g.cancelScheduledValues(t);
  g.setValueAtTime(0, t);
  g.linearRampToValueAtTime(1, t + attack);
  g.linearRampToValueAtTime(sustain, t + attack + decay);
}
```

핵심은 `setTimeout`이 아니라 **오디오 클록 기준 스케줄링**을 쓰는 것. JS 타이머는 수십 ms씩 밀리지만 `AudioParam` 스케줄은 샘플 단위로 정확하다.

## 삽질 포인트들

**사용자 제스처 없이는 소리가 안 난다.** 모든 브라우저가 자동재생을 막는다. AudioContext는 `suspended` 상태로 시작하고, 첫 클릭/터치 핸들러 안에서 `ctx.resume()`을 호출해야 한다. SoundLab은 모든 데모에 "눌러서 소리 내보기" 단계를 공통으로 깔았다 — 정책 대응이면서 동시에 좋은 UX가 됐다.

**노드는 일회용이다.** OscillatorNode는 `stop()` 후 재사용할 수 없다. 건반을 누를 때마다 새로 만들어 연결하고, 끝나면 `disconnect()`로 정리해야 한다. 이걸 안 하면 페이지를 오래 쓸수록 소리가 겹치고 메모리가 샌다. 컴포넌트 언마운트와 오디오 그래프 해제를 묶는 헬퍼를 만들어 해결했다.

**파라미터를 즉시 바꾸면 '틱' 소리가 난다.** `gain.value = 0.5`처럼 값을 점프시키면 파형이 불연속이 되면서 클릭 노이즈가 난다. 항상 `setTargetAtTime`으로 수 ms에 걸쳐 부드럽게 이동시켜야 한다.

## 시각화는 AnalyserNode로

파형과 스펙트럼 표시는 AnalyserNode 하나면 된다. `getByteTimeDomainData`(파형)와 `getByteFrequencyData`(스펙트럼)를 requestAnimationFrame 루프에서 읽어 캔버스에 그린다. 그래프 어디에나 끼워 넣을 수 있어서, "필터를 돌리면 스펙트럼이 깎이는 모습"을 실시간으로 보여주는 교육 효과가 크다.

---

직접 만져보기: [SoundLab 신디사이저 샌드박스](https://soundlab.jjuapp.com/sandbox/synth/) · [지식 허브: 신디시스](https://soundlab.jjuapp.com/learn/synthesis/)
