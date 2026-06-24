# 🛠️ Godot 기술 가이드 — 광물 키우기

> 이 게임을 Godot로 구현할 때의 **기술 규칙과 함정.** Layer 5(구현)의 기준 문서.
> 코워크는 코드를 짜기 전 이 문서를 읽는다.

---

## 0. 엔진 / 렌더러
- **Godot 4.x** (안정판), 언어 **GDScript** (C# 배제).
- **렌더러: Compatibility (OpenGL ES 3.0) 고정.** 저사양 기기 호환 및 장기 구동 시 발열 최적화에 가장 유리하다.

---

## 1. ★ 큰 숫자 문제 (가장 중요한 함정)
방치형 게임은 수치가 1e15, 1e100 이상으로 무한히 커지므로 일반 `int`, `float`는 오버플로한다.
- **(확정) 검증된 Godot용 BigNumber 애드온 사용:** `addons/BigNumber`를 프로젝트 최우선 탑재한다.
- **체이닝 규칙:** 모든 경제/밸런스 공식 스크립트는 라이브러리 메서드(`gold.add(income)`)를 사용한다. 일반 연산자 사용 시 데이터 오염으로 간주한다.
- **포매터 분리:** 내부 값과 UI 표시 값(K, M, B, T 등 알파벳 표기)을 변환하는 `BigNumberFormatter` 전역 싱글톤을 독립 운영한다.

---

## 2. 씬 구조 및 화면 전환 (MVC 패턴 & 백그라운드)
```text
Main (Node2D)              ── 화면 전환·전체 관리
├── MineScreen             ── 광물 클릭·채굴
├── ForgeScreen            ── 검 강화·인챈트
├── DefenseScreen          ── 디펜스 모드
├── DungeonScreen          ── 던전/레이드
└── UI (CanvasLayer)       ── 재화 표시·탭 바·팝업 (항상 유지)
```
- 화면(View)의 존재 여부와 상관없이, 핵심 생산/전투 루프는 오토로드 싱글톤 내부에서 수식과 타이머(틱) 단위로 중단 없이 돌아간다.
- **비주얼 스냅샷 리셋 규칙:** 유저가 디펜스 화면으로 복귀할 때, 복귀 시점의 웨이브 데이터에 맞춰 몬스터를 새롭게 스폰하고 연출을 0부터 시작하도록 그래픽 상태를 리셋(Catch-up)하여 연출 병목을 차단한다.

---

## 3. 오토로드 싱글톤 (전역 상태)
Project Settings → Autoload 에 등록하여 사용:
- **GameState** — 현재 보유 재화, 검 상태, 환생 횟수, 오프라인 최대 누적 시간 저장 변수 보유.
- **EconomyManager** — 채굴 산출·강화 비용/확률·오프라인 보상 공식 구현체.
- **SaveManager** — 저장/로드, 오프라인 시간 및 마이그레이션 제어.
- **Events** — 전역 시스템 간 결합도를 낮추는 시그널 버스.

---

## 4. 세이브 시스템 + 보안 및 오프라인 보상 (★기획 수정 반영)
- **디버그/릴리즈 분기 암호화:** `OS.is_debug_build()` 상태일 때는 순수 JSON 텍스트로, 배포 빌드에서는 `FileAccess.open_encrypted_with_pass()`를 사용해 암호화한다.
- **타임 트래블(시간 조작) 방어벽:** 저장 시 기기 시스템 시간을 함께 암호화하며, 로드 시 현재 시간이 마지막 저장 시간보다 과거라면 치팅으로 판정해 보상을 거부한다.
- **단계적 오프라인 누적 시간 업그레이드 규칙:**
  - 기본 오프라인 최대 누적 시간은 **4시간**으로 제한한다.
  - 유저는 게임 내 재화(골드/환생포인트 등) 업그레이드를 통해 이 최대치를 **6시간, 8시간, 12시간 등 단계적으로 확장**할 수 있다.
- **온/오프라인 보상 분리 및 3-Way 수령 로직:**
  - **오프라인 보상 계산 시 대박(Jackpot) 확률은 무조건 0% 고정 처리한다.** 오직 일반 광물과 기본 골드/강화석만 시간 비례로 산출한다. (온라인 플레이 유치 목적)
  - 유저가 오프라인 후 접속 시 UI에서 **3가지 선택지**를 제공한다:
    1. **기본 수령:** 오프라인 기본 보상만 100% 수령.
    2. **광고 보고 @배 수령:** 30초 광고 시청 후 기본 보상의 $X$배(예: 2~3배) 수령. (광고 제거 패스 소지자는 광고 없이 즉시 $X$배 수령 활성화)
    3. **젬 쓰고 @배 수령:** 소량의 젬을 소모하여 광고 없이 즉시 $X$배 수령.

### 💻 오프라인 보상 및 3-Way 수령 구조 예시 (GDScript)
```gdscript
# EconomyManager.gd 내부 로직 예시
const BASE_OFFLINE_MAX_TIME = 14400.0 # 기본 4시간 (초 단위)

func calculate_offline_reward(elapsed_time: float) -> Dictionary:
    # GameState에 저장된 유저의 현재 최대 시간 스펙 가져오기
    var max_allowed_time = GameState.get_max_offline_time() 
    var final_time = min(elapsed_time, max_allowed_time)
    
    # 대박 확률 0% 고정, 오직 기본 파밍 수치만 계산
    var base_gold = EconomyManager.get_base_gold_per_sec().multiply(final_time)
    var base_stone = EconomyManager.get_base_stone_per_sec() * int(final_time)
    
    return {"gold": base_gold, "stone": base_stone}

func apply_multiplier(reward: Dictionary, multiplier: float) -> void:
    var final_gold = reward["gold"].multiply(multiplier)
    GameState.add_gold(final_gold)
    GameState.add_enhance_stone(int(reward["stone"] * multiplier))
```

---

## 5. 성능 및 UI 갱신 규칙 (장기 구동 최적화)
- **데이터 UI 스로틀링:** `GameState` 데이터 변경 시 UI 레이어에 이벤트를 전송하는 시그널 버스는 **최대 초당 10회(0.1초 간격)**로 제한하여 레이아웃 재계산 부하를 최소화한다. 타격감용 플로팅 텍스트 Fx는 딜레이 없이 즉시 독립 생성한다.
- **단일 노드 커스텀 드로우:** 수백 개의 대미지 라벨 노드를 생성하지 않고, 단 하나의 매니저 노드(`DamageRenderManager`) 내부에서 구조체 배열로 위치/알파값을 관리하며 `_draw()` 함수를 통해 단 한 번의 드로우 콜로 화면에 일괄 렌더링한다.
- **런타임 경로 린터:** 디버그 빌드 실행 시 리소스 폴더(`res://`)의 주요 경로를 순회 검증하여 코드 내 하드코딩된 대소문자와 실제 파일 시스템이 불일치할 경우 경고를 발생시켜 크로스 플랫폼 크래시를 원천 방지한다.
```