# 브라우저 WebRTC 렌더링 지연 최적화 가이드

## 적용된 최적화 항목

### 1. 하드웨어 디코더 최적화

**문제**: GPU/MediaCodec 디코더가 느리면 프레임이 큐에 쌓여 렌더 지연 발생

**적용 코드**:

- `video.latencyHint = 'interactive'`: 브라우저에 최저 지연 모드 요청
- `video.contentHint = 'motion'`: 동영상 콘텐츠임을 명시하여 디코더 최적화
- `playsinline`, `webkit-playsinline` 속성: 모바일에서 전체화면 전환 없이 인라인 재생 (iOS 최적화)

**기대 효과**: 디코딩 파이프라인 지연 10-30ms 감소

---

### 2. 렌더 스레드 병목 완화

**문제**: HTMLVideoElement 프레임 렌더링이 메인 쓰레드와 엮여 타이밍 밀림

**적용 코드**:

```javascript
el.style.willChange = 'transform'; // 브라우저에 변형 예고
el.style.transform = 'translateZ(0)'; // GPU 합성 레이어 분리
el.style.backfaceVisibility = 'hidden'; // 뒷면 렌더링 생략
el.style.contain = 'layout style paint'; // 레이아웃/페인트 범위 제한
```

**측정 지표**:

- `_mainThreadBlockMs`: requestAnimationFrame 간격 기반 메인쓰레드 블로킹 추정
- `_longTasksTotal`: PerformanceObserver로 50ms+ 작업 누적 시간 수집

**기대 효과**:

- 합성 레이어 분리로 레이아웃/페인트 비용 절감
- 메인쓰레드 블로킹 20-40% 감소

---

### 3. Browser Jitter Buffer 최소화

**문제**: WebRTC 내부 jitter buffer가 네트워크 불안정 시 팽창 (1~2프레임 → 더 많이)

**적용 코드**:

```javascript
r.playoutDelayHint = 0.04; // 기존 60ms → 40ms로 하향
```

**동적 조정**:

- 평균 렌더 지연 < 70ms && 현재 힌트 > 45ms → 5ms씩 감소 (최소 40ms)
- 95퍼센타일 > 140ms && 현재 힌트 < 120ms → 8ms씩 증가 (최대 120ms)

**측정 지표**:

- `jitterBufferDelay / jitterBufferEmittedCount`: 실제 플레이아웃 지연 평균
- `_renderLatencySamples`: 최근 120프레임 지연 샘플 (이동 평균/p95 계산)

**기대 효과**: 네트워크 안정 시 버퍼 지연 20ms 감소

---

### 4. 해상도/프레임레이트 부하 최적화

**문제**: 1080p 30fps는 디코딩 시 20-40ms/frame 부하, CPU/GPU 부족 시 드롭/지연

**적용 코드**:

```javascript
// 비디오 엘리먼트 크기를 실제 프레임 해상도에 맞춤
viewer.remoteView.width = videoWidth;
viewer.remoteView.height = videoHeight;
```

**동적 적응**:

- 연속 드롭률 > 15% 5회 → `pseudoJumpToLive()` (디코더 리셋)
- 연속 렌더 지연 > 140ms 6회 → DataChannel로 `ADAPT_LOWER_QUALITY` 메시지 전송

**기대 효과**:

- CSS scaling 제거로 브라우저 composite 비용 절감
- 과부하 감지 시 자동 품질 조정으로 지연 안정화

---

## 측정 방법

### 콘솔 출력 해석

3초마다 또는 지연 급변(>40ms) 시 출력되는 박스:

```
┌──────────────────────────────────────────────────────┐
│              실제 측정된 지연시간                      │
├──────────────────────────────────────────────────────┤
│ 🌐 실제 네트워크 지연:   50.5ms                       │  ← RTT/2
│ 🖼️ 실제 렌더링 지연:     85.2ms                       │  ← 디코딩+처리+버퍼
│ 📊 네트워크 지터:       12.3ms                       │  ← 패킷 도착 간격 편차
│ 🧵 메인쓰레드 블로킹:     8.5ms                       │  ← rAF 간격 초과분
│ ⌛ LongTask 비율:        2.8%                        │  ← 50ms+ 작업 비율
│ ⏱️ 총 지연시간:        148.0ms                       │  ← 전체 합계
│ 🖥️ 화면 렌더 지연:      15.2ms                       │  ← 실제 표시 지연
│ 🔁 디코더+버퍼 지연:     70.0ms                       │  ← 내부 파이프라인
└──────────────────────────────────────────────────────┘
```

### 목표 수치

- **총 지연**: < 150ms (양호), < 100ms (우수)
- **렌더링 지연**: < 80ms (디코더 병목 없음)
- **메인쓰레드 블로킹**: < 10ms (렌더 파이프라인 원활)
- **LongTask 비율**: < 5% (UI 반응성 양호)

---

## 실행 방법

1. **DQP 메트릭 활성화**:
    - 웹 페이지에서 "Enable DQP metrics" 체크박스 활성화
    - Viewer 시작

2. **브라우저 개발자 도구 콘솔 열기**:
    - Chrome/Edge: F12 → Console 탭
    - Safari: Cmd+Opt+C

3. **실시간 모니터링**:
    - 3초마다 지연 통계 박스 확인
    - 급격한 변화 시 즉시 출력

4. **최적화 효과 확인**:
    - 렌더링 지연 변화 추이 관찰
    - LongTask 비율 감소 확인
    - 프레임 드롭률 변화 모니터링

---

## 한계 및 주의사항

### 뷰어만으로 해결 불가능한 부분

1. **마스터 인코딩 설정**: 송신측 해상도/비트레이트가 과도하면 디코딩 부하 증가
2. **네트워크 품질**: RTT/패킷 손실은 뷰어에서 제어 불가
3. **디바이스 성능**: 저사양 디바이스의 하드웨어 디코더 한계

### 최적화 작동 조건

- `requestVideoFrameCallback` API 지원 브라우저 (Chrome 83+, Safari 15.4+)
- `PerformanceObserver` Long Task 지원 (Chrome 58+)
- WebRTC `playoutDelayHint` 지원 (Chrome 96+, Safari 미지원)

### 트레이드오프

- **낮은 playout delay**: 네트워크 지터 증가 시 프레임 드롭 가능성
- **GPU 합성 레이어**: 메모리 사용량 약간 증가 (보통 무시 가능)
- **빈번한 pseudoJump**: 일시적 화면 끊김 (0.1~0.2초)

---

## 추가 개선 제안

### Master 측 적응 코드 (별도 구현 필요)

```javascript
// DataChannel에서 ADAPT_LOWER_QUALITY 메시지 수신 시
dataChannel.onmessage = (msg) => {
    const data = JSON.parse(msg.data);
    if (data.type === 'ADAPT_LOWER_QUALITY') {
        // 송신 파라미터 조정
        const sender = peerConnection.getSenders().find((s) => s.track?.kind === 'video');
        const params = sender.getParameters();
        params.encodings[0].maxBitrate = 500000; // 500kbps로 제한
        params.encodings[0].scaleResolutionDownBy = 2; // 해상도 1/2
        sender.setParameters(params);
    }
};
```

### 고급 튜닝

1. **임계값 조정**: `RENDER_LATENCY_THRESHOLD_MS` (현재 140ms) 환경에 맞게 조정
2. **샘플 윈도우**: `_renderLatencySampleLimit` (현재 120) 변경으로 반응성 조절
3. **로그 빈도**: `nowPerf - lastLog > 3000` (3초) 간격 조정

---

## 문제 해결

### 지연이 여전히 높은 경우

1. 네트워크 RTT 확인 (>100ms면 네트워크 문제)
2. 디코딩 프레임 수 vs 드롭 수 비율 (>10%면 디바이스 성능 문제)
3. LongTask 비율 (>10%면 다른 스크립트 영향)

### 프레임 드롭이 많은 경우

1. `playoutDelayHint`를 60ms로 증가 (안정성 우선)
2. Master 측 해상도/비트레이트 하향
3. 하드웨어 가속 활성화 확인 (chrome://gpu)

### 콘솔 출력이 없는 경우

- `enableDQPmetrics` 옵션 활성화 확인
- `viewer.peerConnectionStatsInterval` 동작 확인
- 비디오 프레임 수신 여부 확인 (`framesDecoded > 0`)
