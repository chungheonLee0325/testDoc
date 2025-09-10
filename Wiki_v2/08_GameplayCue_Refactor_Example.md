# 08. GameplayCue 리팩터링 예시

Multicast RPC 중심의 이펙트/사운드 동기화를 GAS GameplayCue로 전환하는 미니 사례입니다.

## Before: Multicast RPC 기반 FX/SFX
```cpp
// Ability 실행부
void UMy_Ability::StartExecutePhase(...){
  // 투사체/이펙트 수동 재생
  SpawnProjectile();
  PlayCommonEffects(ExecuteEffect, ExecuteSound, Location);
  // 이외 멀티캐스트로 알림
  MulticastRPC_OnAbilityExecuted();
}
```

## After: GameplayCue 기반 선언적 트리거
```cpp
// Cue Tag 예시: Cue.Ability.Phoenix.Blaze.Execute
void UMy_Ability::StartExecutePhase(...){
  SpawnProjectile();
  UAbilitySystemBlueprintLibrary::SendGameplayCue(
    GetAvatarActorFromActorInfo(),
    FGameplayTag::RequestGameplayTag(TEXT("Cue.Ability.Phoenix.Blaze.Execute")),
    FGameplayCueParameters());
}
```

## 코드 Diff (Before → After)

아래는 실제 어빌리티(예: `KAYO_Q_FLASHDRIVE`)에서 Multicast → GameplayCue로 치환하는 최소 변경 예시입니다.

```diff
diff --git a/Source/Valorant/AbilitySystem/Abilities/KAYO/KAYO_Q_FLASHDRIVE.cpp b/Source/Valorant/AbilitySystem/Abilities/KAYO/KAYO_Q_FLASHDRIVE.cpp
index abcdef1..1234567 100644
--- a/Source/Valorant/AbilitySystem/Abilities/KAYO/KAYO_Q_FLASHDRIVE.cpp
+++ b/Source/Valorant/AbilitySystem/Abilities/KAYO/KAYO_Q_FLASHDRIVE.cpp
@@
-#include "KAYO_Q_FLASHDRIVE.h"
+#include "KAYO_Q_FLASHDRIVE.h"
+#include "AbilitySystemBlueprintLibrary.h"            // (+) GameplayCue 전송
@@
 bool UKAYO_Q_FLASHDRIVE::ThrowFlashbang(bool bAltFire)
 {
   // ... 생략 ...
   if (Flashbang)
   {
-    // 플래시뱅의 ActiveProjectileMovement 함수 호출 (bAltFire 파라미터로 던지기 방식 결정)
-    Flashbang->ActiveProjectileMovement(bAltFire);
-    SpawnedProjectile = Flashbang;
-
-    // 던지기 사운드 재생
-    if (ThrowSound)
-    {
-      UGameplayStatics::PlaySoundAtLocation(GetWorld(), ThrowSound, Character->GetActorLocation());
-    }
-
-    // 발사 효과 재생
-    PlayCommonEffects(ProjectileLaunchEffect, ProjectileLaunchSound, SpawnLocation);
-
-    // (기존) 실행 알림 멀티캐스트
-    // MulticastRPC_OnAbilityExecuted();
-
-    return true;
+    // 투사체 활성화
+    Flashbang->ActiveProjectileMovement(bAltFire);
+    SpawnedProjectile = Flashbang;
+
+    // (변경) GameplayCue로 FX/SFX 트리거
+    FGameplayCueParameters CueParams;
+    CueParams.Location = SpawnLocation;
+    CueParams.Instigator = Character;
+    UAbilitySystemBlueprintLibrary::SendGameplayCue(
+      GetAvatarActorFromActorInfo(),
+      FGameplayTag::RequestGameplayTag(TEXT("Cue.Ability.KAYO.Flash.Execute")),
+      CueParams);
+
+    return true;
   }
   else
   {
     UE_LOG(LogTemp, Error, TEXT("KAYO Q - 플래시뱅 생성 실패"));
     return false;
   }
 }
```

Cue 측(에디터)은 `Cue.Ability.KAYO.Flash.Execute`에 VFX/SFX/CameraShake를 구성하면, 코드 변경 없이도 동작합니다.

```diff
diff --git a/Source/Valorant/AbilitySystem/Abilities/Phoenix/Phoenix_C_BlazeSplineWall.cpp b/Source/Valorant/AbilitySystem/Abilities/Phoenix/Phoenix_C_BlazeSplineWall.cpp
index ffffff0..aaaaaaa 100644
--- a/Source/Valorant/AbilitySystem/Abilities/Phoenix/Phoenix_C_BlazeSplineWall.cpp
+++ b/Source/Valorant/AbilitySystem/Abilities/Phoenix/Phoenix_C_BlazeSplineWall.cpp
@@
 void UPhoenix_C_BlazeSplineWall::OnLeftClickInput()
 {
-  // 기존: 확정 후 멀티캐스트로 FX 알림
-  // MulticastRPC_OnAbilityExecuted();
-  PlayCommonEffects(ExecuteEffect, ExecuteSound, FinalLocation);
+  // 변경: Cue로 확정 이펙트 트리거
+  FGameplayCueParameters Params; Params.Location = FinalLocation;
+  UAbilitySystemBlueprintLibrary::SendGameplayCue(
+    GetAvatarActorFromActorInfo(),
+    FGameplayTag::RequestGameplayTag(TEXT("Cue.Ability.Phoenix.Blaze.Execute")),
+    Params);
 }
```

## 통합 체크리스트
- [ ] Cue 태그 등록: `Cue.Ability.<Agent>.<Ability>.<Phase>` (Config 또는 DataAsset)
- [ ] 어빌리티에서 Multicast 제거, `UAbilitySystemBlueprintLibrary::SendGameplayCue` 호출로 대체
- [ ] Cue Notify(Static/Actor)에서 VFX/SFX/카메라 구성
- [ ] 전파 범위/조건(Only Locally Controlled 등) Cue Notify에서 제어
- [ ] (선택) 기존 `PlayCommonEffects`는 로컬 디버그/프리뷰용으로 제한

## 테스트 팁
- 에디터: Ability 실행 시 Cue가 호출되는지 로그/시각 효과 확인
- 커맨드라인: `UnrealEditor-Cmd.exe <uproject> -ExecCmds="Automation RunTests Ability; Quit"`
- 네트워크: Listen 서버 + 클라이언트 2인 구성에서 FX/SFX 동작 동일성 확인

## 태그/자원 구성
- Cue 태그 표준: `Cue.Ability.<Agent>.<Ability>.<Phase>` (Prepare/Wait/Execute/Finish)
- Cue Notify(Static/Actor): VFX/SFX/카메라쉐이크를 에디터에서 구성 → 코드 변경 최소화

## 이점
- 네트워크: Cue가 복제/예외 처리 → 수동 Multicast 호출 감소
- 유지보수: 태그 단일화로 탐색성/교체 용이
- 협업: 디자이너가 Cue에 자산을 붙여도 동작 일관

## 단계적 적용 제안
1) Flash: Execute 시 Cue.Ability.KAYO.Flash.Execute → 후처리/위젯 구독
2) Spike: Plant/Defuse 전이마다 Cue.Ability.Spike.<State>
3) Shop: 구매 성공/실패 Cue로 피드백 통일

관련: [07. 회고(첫 GAS 적용)](07_Retrospective.md)
