# 클래스 다이어그램

웹 데모의 구조를 클래스 다이어그램으로 정리했다(2026-11 기준, 커밋 `8225b9b`). 클래스는 6개(`Sim`, `Rng`, `SimView`, `App`, `History`, `Charts`)다. JS에서는 클래스가 아닌 일반 객체로 쓰는 것(분자, 소스, 작용기)과 데이터 모듈(`config.js`, `chemistry.js`)은 `<<object>>`, `<<data>>`, `<<module>>`로 표시했다. 밑줄(`_`)로 시작하는 내부 메서드는 대부분 뺐다.

- **원자는 객체가 아니다**: `Sim`이 원자마다 슬롯 번호 `i`를 주고, 위치·속도·원소·형식 전하 등을 배열(`px[i]`, `el[i]`, `fc[i]` …)에 나눠 담는다(SoA). 결합은 `nb[i]`(상대 슬롯)·`od[i]`(차수) 목록이다. Unity 이식 때 `NativeArray`로 옮기기 쉽게 하려는 구조다(`unity-port.md` 3절).
- **코어(`src/core`)는 화면과 무관하다**: `Sim`은 DOM·Three.js를 모르고, 화면 쪽(`SimView`, `App`, `History`, `Charts`)이 `Sim`을 읽거나 호출한다.

## 1. 전체 구조

화살표: `*--` 소유(만들고 들고 있음), `-->` 참조해서 호출, `..>` 사용(데이터를 읽거나 함수를 부름).

```mermaid
classDiagram
  direction TB

  class main {
    <<module>>
    프레임 루프
  }
  class App {
    패널 3개: 소스·제어·결과
    +state : idle | running | paused
    +start() reset() stimulate()
    +insertMolecules() refreshResults()
  }
  class SimView {
    Three.js 렌더링
    +render() pickSource(x, y)
  }
  class History {
    결과값 기록
    +sample(sim)
  }
  class Charts {
    시간 그래프 6개
    +draw()
  }
  class picking {
    <<module>>
    +pickWall()
  }
  class presets {
    <<module>>
    +applyExampleLayout()
    +fillMolecules()
  }
  class Sim {
    시뮬레이션 코어
    +step() getStats()
  }
  class Rng {
    시드 난수 mulberry32
    +next()
  }
  class CONFIG {
    <<data>>
    수치 상수
  }
  class chemistry {
    <<data>>
    원소·결합 쌍·템플릿·작용기 패턴·pKa
  }

  main ..> Sim : 생성
  main ..> SimView : 생성
  main ..> App : 생성
  App --> Sim : 조작
  App --> SimView : 선택 표시
  App *-- History
  App *-- Charts
  App ..> presets
  App ..> picking
  Charts --> History : 그리기
  History ..> Sim : getStats()
  SimView --> Sim : 읽기
  presets ..> Sim : 분자·소스 배치
  Sim *-- Rng
  Sim ..> CONFIG
  Sim ..> chemistry
```

## 2. Sim 상세 (코어)

```mermaid
classDiagram
  direction LR

  class Sim {
    +cfg : CONFIG
    +rng : Rng
    +time · stepCount · count · capacity
    원자별 배열: 슬롯 i
    +px py pz vx vy vz : Float64Array
    +el uid fc val used q groupMask
    +nb od bt : 결합 상대·차수·시각
    +comps : Map~id, Component~
    +groups : Map~key, Group~
    +sources : Source[]
    +counters · cooldown
    +reset(seed) · reseed(seed)
    +step() · run(steps)
    +addAtom() · placeMolecule() · removeComponent()
    +canBond() · bond() · dissociate()
    +canAccept() · canDonate() · coordinate() · protonate()
    +canTransfer() · transferProton()
    +acidPKa() · conjugatePKa()
    +matchPattern(center, pattern)
    +addSource() · updateSource() · removeSource() · clearSources()
    +recordInflow(tplKey)
    +stimulate(strength, duration) · stimActive
    +isRadical(i) · forEachAtom() · forEachBond()
    +getStats() Stats
    +formula(counts, charge)$
  }

  class Component {
    <<object>>
    분자 = 결합 그래프의 연결 요소
    +id
    +members : slot[]
    +counts : 원소별 개수
  }

  class Source {
    <<object>>
    분출원 또는 배출구
    +id · kind · face · pos · normal
    +molecule · rate · speed
    +radius · filter
    +emitted · removed · blockedSteps
  }

  class Group {
    <<object>>
    작용기
    +key : amino · carboxyl · ammonium · carboxylate
    +center : slot
    +members : slot[]
  }

  class Stats {
    <<object>>
    결과 패널·그래프·실험이 읽는 값
    +atoms · molecules · bonds · radicals
    +groups · groupsCumulative · aminoAcid
    +ions · cations · anions
    +temperature · pressure · pH
  }

  class Rng {
    +next() · range() · unitVector() · inDisc()
  }

  Sim *-- Rng
  Sim *-- "0..*" Component
  Sim *-- "0..*" Source
  Sim *-- "0..*" Group
  Sim ..> Stats : getStats()
```

## 3. 한 스텝의 흐름 (`Sim.step`)

```mermaid
flowchart LR
  A[분출] --> B[격자] --> C[원자 쌍 힘<br/>+ 결합·배위·이동 후보] --> D[결합 스프링<br/>+ 해리 후보] --> E[각도 힘] --> F[적분·벽 반사] --> G[진동 얼리기] --> H[사건 적용<br/>해리 → 배위 → 이동 → 차수↑ → 새 결합] --> I[배출구] --> J[작용기 증분 판정]
```

자세한 순서와 난수 소비는 [`unity-port.md`](unity-port.md) 2절.
