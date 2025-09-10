# 07. 회고(첫 GAS 적용)

## 배경
- 본 프로젝트는 첫 GAS(Gameplay Ability System) 적용 사례였습니다. C/Q/E 스킬 12개를 빠르게 확장 가능한 형태로 구현하는 것이 목표였습니다.

## 잘한 점
- 수명주기 표준화: Instant vs WithPrepare 두 트랙으로 모든 스킬 흐름을 단순화
- FollowUp 입력 통일: Waiting 단계에서 좌/우클릭만 구현하면 확정 로직 자동 연결(ASC GameplayEvent)
- 1P/3P 프리뷰 분리: 로컬 비복제(1P) + 서버 복제(3P)로 체감 지연·깜빡임 감소
- 모듈화: Projectile/GroundEffect, 공통 이펙트/사운드 유틸로 중복 제거

## 아쉬웠던 점(GameplayCue 미도입)
- FX/SFX·상태 알림에 Multicast RPC를 빈번히 사용했습니다. GAS의 GameplayCue를 적극 사용했다면 다음 이점이 있었습니다:
  - 선언적 트리거: `GameplayCue_X.*` 태그로 효과를 선언/구성 → 코드 의존성 감소
  - 네트워크 효율: Cue 시스템이 복제/예외를 관리해 수동 Multicast 호출을 줄임
  - 저작 친화: 디자이너가 Cue에 이펙트/사운드를 붙여도 동작 일관

예상 리팩토링 방향
- 실행 지점: `StartExecutePhase` 혹은 파생 Execute에서 `GameplayCue_Execute` 호출로 전환
- 공통 태그: `Cue.Ability.<Agent>.<Ability>.<Phase>` 표준화(Prepare/Wait/Execute/Finish)
- 점진 적용: Flash/Spike/Shop 피드백부터 Cue 전환 → 전역 공용 이펙트로 확장

## 개선 로드맵
- [ ] GameplayCue 도입 가이드 문서화 및 태그 표준 확정
- [ ] Flash/Spike/Shop의 Multicast → Cue 단계적 치환
- [ ] 자동화 테스트: Cue 트리거/종료 검증 스펙 추가
- [ ] 데이터테이블로 Cue 매핑 관리(사운드/FX 교체 용이)

## 관련 문서
- [01. Ability Framework](01_Ability_Framework.md)
- [02. Agent Abilities](02_Agent_Abilities.md)
- [03. Flash 시스템](03_Flash_System.md)
- [08. GameplayCue 리팩터링 예시](08_GameplayCue_Refactor_Example.md)
