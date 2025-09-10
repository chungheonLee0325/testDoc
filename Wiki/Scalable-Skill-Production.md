# 확장 가능한 스킬 대량 구현

## 1. 문제 정의: 16개의 고유 스킬을 어떻게 효율적으로 생산할 것인가?

이 프로젝트는 4명의 요원이 각각 4개의 고유한 스킬(C, Q, E, X)을 가지며, 이는 총 16개의 스킬을 구현해야 함을 의미합니다. 각 스킬은 저마다 다른 애니메이션, 이펙트, 투사체, 그리고 고유한 동작 방식을 가집니다. 이러한 다양성을 수용하면서도 코드의 중복을 최소화하고, 앞으로 추가될 새로운 요원과 스킬에 대비할 수 있는 확장 가능한 구조를 만드는 것이 이 시스템의 핵심적인 도전 과제였습니다.

단순히 16개의 클래스를 각각 처음부터 만드는 방식은 비효율적이고 유지보수가 어렵습니다. 반대로, 모든 것을 데이터로만 처리하려 하면 피닉스의 '불길'처럼 복잡하고 고유한 로직을 구현하기 어렵습니다. 따라서 이 프로젝트는 두 방식의 장점을 모두 취하는 **하이브리드 아키텍처**를 채택했습니다.

## 2. 해결책: C++ 기반 클래스와 블루프린트 데이터의 하이브리드

이 프로젝트의 스킬 생산 파이프라인은 다음과 같은 명확한 원칙을 따릅니다.

> **"모든 스킬은 고유한 C++ 클래스를 가진다. 하지만 대부분의 로직은 부모 클래스(`UBaseGameplayAbility`)가 처리하고, 각 스킬의 개성은 블루프린트에서 정의된 데이터로 결정된다."**

### 공통점: 강력한 기반 클래스 (`UBaseGameplayAbility`)

모든 스킬의 C++ 클래스는 `UBaseGameplayAbility`를 상속받습니다. 이 덕분에 모든 스킬은 별도의 코드 없이도 다음과 같은 공통 기능을 즉시 상속받습니다.

*   3단계 상태 머신 (`Preparing` -> `Waiting` -> `Executing`)
*   스택(충전 횟수) 및 코스트 관리
*   쿨다운 처리 (`GameplayEffect`)
*   애니메이션 몽타주 재생
*   사운드 및 파티클 이펙트 재생
*   네트워크 리플리케이션

### 차이점: 데이터로 정의되는 스킬의 개성

각 스킬의 C++ 클래스를 상속받는 블루프린트 에셋에서는, 부모 클래스에 미리 정의된 변수들의 **데이터만 교체**하여 스킬의 정체성을 정의합니다. 프로그래머의 개입 없이 디자이너나 아티스트가 스킬의 대부분을 설정할 수 있습니다.

*   **`ActivationType`**: 스킬이 즉시 발동인지, 준비 단계가 필요한지 결정합니다.
*   **`*Montage`**: 각 상태(Prepare, Execute 등)에 맞는 1인칭/3인칭 애니메이션을 지정합니다.
*   **`*Effect`, `*Sound`**: 각 상태에 맞는 파티클과 사운드 에셋을 지정합니다.
*   **`ProjectileClass`**: 스킬이 발사할 투사체 액터를 지정합니다.
*   **`Cooldown`, `Cost`**: 스킬의 쿨다운 시간과 비용을 `GameplayEffect` 에셋으로 지정합니다.

### 예외: C++로 구현되는 최소한의 고유 로직

데이터만으로 표현할 수 없는 복잡하고 고유한 로직이 필요한 경우에만, 해당 스킬의 C++ 클래스에서 필요한 함수(`ExecuteAbility`, `OnLeftClickInput` 등)를 **오버라이드(override)**하여 최소한의 코드를 추가합니다. 예를 들어, 제트의 '상승기류'는 `ExecuteAbility`를 오버라이드하여 캐릭터에게 위쪽 방향으로 충격량(`LaunchCharacter`)을 가하는 단 몇 줄의 코드를 추가하는 것만으로 구현됩니다.

## 3. 구현 현황: 16개 스킬 분석 테이블

아래 표는 현재 구현된 16개 스킬이 위의 하이브리드 아키텍처를 통해 어떻게 효율적으로 구현되었는지 보여줍니다.

| Agent   | Slot | Skill Name      | Activation Type | Core Mechanic (C++ Override)                               | Note                                      |
| :------ | :--- | :-------------- | :-------------- | :--------------------------------------------------------- | :---------------------------------------- |
| **Phoenix** | C    | Blaze           | `WithPrepare`   | `ExecuteAbility`: `ABlazeWall` 액터 스폰 및 제어         | 복잡한 커스텀 액터 로직 필요              |
|         | Q    | Hot Hands       | `Instant`       | `ExecuteAbility`: `AFireOrb` 투사체 발사                   | 기본 투사체 발사 로직 재사용              |
|         | E    | Curveball       | `WithPrepare`   | `OnLeftClickInput`/`OnRightClickInput`: 방향 제어 및 투사체 발사 | 후속 입력 처리 및 커브 로직 필요          |
|         | X    | Run It Back     | `Instant`       | `ExecuteAbility`: 위치 저장 및 부활 `GameplayEffect` 적용 | 복잡한 상태 관리 및 부활 로직 필요        |
| **Jett**    | C    | Cloudburst      | `Instant`       | `ExecuteAbility`: 연막 투사체 발사                         | 기본 투사체 발사 로직 재사용              |
|         | Q    | Updraft         | `Instant`       | `ExecuteAbility`: `LaunchCharacter`로 수직 임펄스 적용     | 간단한 캐릭터 이동 로직                 |
|         | E    | Tailwind        | `Instant`       | `ExecuteAbility`: `LaunchCharacter`로 전방 임펄스 적용     | 간단한 캐릭터 이동 로직                 |
|         | X    | Blade Storm     | `Instant`       | `ExecuteAbility`: 무기 교체 및 특수 공격 어빌리티 부여   | 무기 시스템 및 GAS 연동 로직 필요         |
| **KAY/O**   | C    | FRAGMENT        | `Instant`       | `ExecuteAbility`: 수류탄 투사체 발사                       | 기본 투사체 발사 로직 재사용              |
|         | Q    | FLASH/DRIVE     | `WithPrepare`   | `OnLeftClickInput`/`OnRightClickInput`: 던지는 방식 변경   | 후속 입력에 따른 투사체 속성 변경         |
|         | E    | ZERO/POINT      | `Instant`       | `ExecuteAbility`: 특수 투사체 발사 및 적 탐지 로직       | 투사체에 커스텀 로직 필요                 |
|         | X    | NULL/CMD        | `Instant`       | `ExecuteAbility`: 자신에게 버프 및 주변에 디버프 GE 적용 | 간단한 GameplayEffect 적용                |
| **Sage**    | C    | Barrier Orb     | `WithPrepare`   | `ExecuteAbility`: `ABarrierWallActor` 스폰 및 위치 지정    | 복잡한 커스텀 액터 로직 필요              |
|         | Q    | Slow Orb        | `Instant`       | `ExecuteAbility`: 둔화 장판 투사체 발사                  | 기본 투사체 발사 로직 재사용              |
|         | E    | Healing Orb     | `WithPrepare`   | `ExecuteAbility`: 아군 대상 탐색 및 치유 GE 적용         | 대상 탐색 및 GameplayEffect 적용          |
|         | X    | Resurrection    | `WithPrepare`   | `ExecuteAbility`: 죽은 아군 탐색 및 부활 GE 적용         | 복잡한 대상 탐색 및 부활 로직 필요        |

## 4. 결론: 생산성과 확장성을 모두 잡다

이 하이브리드 아키텍처는 다음과 같은 명확한 이점을 제공하며 성공적으로 16개의 스킬을 구현하는 기반이 되었습니다.

*   **생산성**: 대부분의 스킬(표에서 '기본 투사체 발사' 등)은 새로운 C++ 코드 없이, 기존 `UBaseGameplayAbility`를 상속받은 블루프린트의 데이터만 수정하는 것만으로 빠르게 프로토타이핑하고 완성할 수 있었습니다.
*   **유지보수성**: 모든 스킬은 일관된 C++ 클래스 구조를 가지므로, 특정 스킬의 코드를 찾거나 수정하기가 매우 용이합니다. 공통 기능의 버그는 `UBaseGameplayAbility` 한 곳만 수정하면 모든 스킬에 반영됩니다.
*   **확장성**: 피닉스의 '불길'이나 세이지의 '부활'처럼 완전히 새로운 메커니즘이 필요한 스킬이 추가되더라도, 기존 시스템을 수정할 필요 없이 해당 스킬의 C++ 클래스 내에서만 고유한 로직을 자유롭게 구현할 수 있는 유연성을 제공합니다.

결론적으로, **"공통점은 부모 클래스로, 차이점은 데이터로, 예외는 자식 클래스의 오버라이드로"** 라는 원칙을 통해, 복잡하고 다양한 스킬들을 효율적으로 양산하고 안정적으로 관리하는 두 마리 토끼를 모두 잡을 수 있었습니다.

## 5. 관련 시스템 (Related Systems)

*   **[GAS 아키텍처](./Skill-System-Architecture.md)**: 이 문서에서 설명하는 생산 방식의 기반이 되는 전체적인 시스템 구조입니다.
*   **[어빌리티 기반 클래스와 활성화 흐름](./Skill-Implementation.md)**: 이 문서에서 언급된 `UBaseGameplayAbility`의 내부 동작에 대한 더 상세한 설명입니다.
