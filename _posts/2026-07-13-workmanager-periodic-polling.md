---
title: "WorkManager 주기 폴링 실전: 명시적 초기화, 동적 주기, 그리고 네트워크가 꺼져 있을 때"
date: 2026-07-13 17:00:16 +0900
categories: [개발, 안드로이드]
tags: [android, workmanager, polling, background, hilt]
---

습관 인증 앱을 만들면서 기기에서 모은 인증 신호를 주기적으로 서버에 보내는 백그라운드 폴링을 WorkManager로 구현했다. 단순해 보이는 요구사항인데 실제로는 세 가지 벽에 부딪혔다. 초기화를 왜 명시적으로 해야 하는가, 폴링 주기를 왜 하나로 고정하면 안 되는가, 서버에 의존하는 폴링은 무엇이 위험한가. 이 글은 팀 전송 테크 스펙(§0.x로 인용)과 실제 코드로 그 셋을 어떻게 풀었는지 정리한다.

> 코드는 공개 저장소 [RuleUp-ASM/Android](https://github.com/RuleUp-ASM/Android)의 `verification` 모듈 (`VerificationSyncWorker`, `VerificationSyncSchedulerImpl`)에서 가져왔다. §0.x 표기는 팀 내부 전송 스펙 문서의 절 번호다.
{: .prompt-info }

## 1. 왜 명시적(on-demand) 초기화를 해야 하는가

WorkManager는 아무 설정도 하지 않으면 앱 프로세스가 뜨는 순간 `androidx.startup` 의 `InitializationProvider` 가 자동으로 초기화한다. 편해 보이지만 두 가지 문제가 있다.

첫째, Worker에 의존성을 주입할 수 없다. 자동 초기화는 기본 `WorkerFactory` 를 쓰는데, 기본 팩토리는 `(Context, WorkerParameters)` 두 개짜리 생성자만 호출할 줄 안다. 우리 Worker는 UseCase·저장소·로거 등 8개의 의존성을 Hilt로 주입받는 `@HiltWorker` 다. 자동 초기화가 먼저 일어나면 Hilt의 `HiltWorkerFactory` 가 등록될 기회 자체가 없고, 주기 작업이 발화하는 순간 Worker 인스턴스화에 실패한다.

둘째, 초기화 시점을 통제할 수 없다. ContentProvider는 `Application.onCreate()` 보다도 먼저 실행된다. DI 그래프가 만들어지기 전에 WorkManager가 세팅을 끝내는 순서 역전이 일어난다.

해결은 두 단계다. 먼저 매니페스트에서 자동 초기화 항목을 제거한다.

```xml
<!-- WorkManager 기본 이니셜라이저 제거 → App(Configuration.Provider)로 on-demand 초기화 -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

그다음 `Application` 이 `Configuration.Provider` 를 구현해, 첫 `getInstance()` 호출 시점에 우리가 만든 팩토리로 초기화되게 한다.

```kotlin
@HiltAndroidApp
class App : Application(), Configuration.Provider {
    // @HiltWorker 들을 인스턴스화하는 HiltWorkerFactory 를 등록한다.
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

이제 Worker가 생성자 주입을 받을 수 있다.

```kotlin
@HiltWorker
class VerificationSyncWorker @AssistedInject constructor(
    @Assisted appContext: Context,
    @Assisted params: WorkerParameters,
    private val runSyncUseCase: RunSyncUseCase,
    private val syncScheduler: SyncScheduler,
    // ... 진행률 캐시, 설정 저장소, 애널리틱스 로거 등
) : CoroutineWorker(appContext, params)
```

> 자동 초기화를 제거했는데 `Configuration.Provider` 구현을 빼먹으면, 첫 `getInstance()` 에서 `IllegalStateException: WorkManager is not initialized` 로 죽는다. 둘은 반드시 세트다.
{: .prompt-warning }

## 2. 폴링 주기를 왜 하나로 고정하면 안 되는가

처음엔 30분마다로 끝인 줄 알았다. 스펙을 쓰면서 보니 폴링 주기는 두 층이었고, 각 층 안에서도 상황마다 달라야 했다.

### 2-1. 전송 주기는 서버가 지시한다 (스펙 §0.3)

전송 주기를 클라이언트에 하드코딩하면 서버가 통제 수단을 잃는다. 진행 중인 챌린지가 없는 사용자에겐 30분 폴링이 낭비고, 트래픽이 몰릴 때 천천히 오라고 말할 방법도 없다. 그래서 스펙은 전송은 클라가 시작하되 주기는 서버가 관리하는 모델을 택했다.

주기의 최초 값부터 서버가 정한다. 로그인할 때 클라가 기기 정보(`sdkInt`, `model`, `lowRam`, 배터리 최적화 상태)를 올리면 서버가 기기에 맞는 주기를 산정해 내려준다. 취약한 기기에는 완화된 주기를, 좋은 기기에는 짧은 주기를 준다. 이 정책은 리프레시 토큰 갱신 주기인 1주마다 다시 받으므로, 앱 업데이트나 OS 업그레이드로 기기 특성이 바뀌어도 최대 1주 안에 따라온다. 이후에는 sync 응답마다 `flushIntervalSec` 가 전체값으로 내려온다.

```json
{
  "syncedAt": "2026-06-26T12:00:01Z",
  "flushIntervalSec": 1800,
  "updatedChallenges": [
    { "challengeId": "C1", "todayStatus": "SUCCESS", "progressRate": 25.0 }
  ],
  "ignoredSignalTypes": []
}
```

Worker는 성공할 때마다 이 값으로 다음 주기를 갈아끼운다.

```kotlin
override suspend fun doWork(): Result {
    val result = runSyncUseCase(scope, collectedAt)
    if (result != null) {
        progressCacheStore.upsert(result.updatedChallenges) // 로컬 캐시 갱신
        syncScheduler.reschedule(result.nextSyncAfterSec)   // 서버 flushIntervalSec 로 주기 재설정
    }
    return Result.success()
}
```

```kotlin
override fun reschedule(nextSyncAfterSec: Int) {
    // WorkManager 최소 주기 15분 floor 적용.
    val minutes = (nextSyncAfterSec / 60L).coerceAtLeast(15L)
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        WORK_NAME,
        ExistingPeriodicWorkPolicy.UPDATE, // 실행 중인 주기 작업의 간격만 교체
        buildRequest(minutes),
    )
}
```

여기서 두 가지 제약을 알아야 한다. WorkManager의 주기 작업 최소 간격은 15분이라, 서버가 그보다 짧게 지시해도 하한을 걸어야 한다. 걸지 않으면 WorkManager가 조용히 15분으로 올린다. 주기 변경은 `ExistingPeriodicWorkPolicy.UPDATE` 로 해야 대기열을 갈아엎지 않고 간격만 바뀐다. 최초 예약은 `KEEP` 이다. 앱을 켤 때마다 `REPLACE` 로 리셋하면 주기의 기준 시각이 계속 밀려서 영원히 실행되지 않는 폴링이 될 수 있다.

### 2-2. 수집 폴링은 신호마다 주기가 다르다 (스펙 §0.3)

전송과 별개로, 기기 안에서 신호를 수집하는 폴링이 있다. 스펙의 정책 초안이 신호마다 다른 주기를 잡아둔 데는 각각 이유가 있다.

```json
"collection": {
  "GEOFENCE":    { "enabled": true },                  // push 방식 — cadence 무관
  "SCREEN_TIME": { "enabled": true, "pollSec": 900 },  // OS purge 보다 짧게
  "WAKE":        { "enabled": true, "pollSec": 900 },  // 매일 아침 1회 보장
  "HEALTH":      { "enabled": true, "pollSec": 1800 }  // read quota 보수적
}
```

- 앱 사용 이벤트(SCREEN_TIME)는 `UsageStatsManager` 가 일정 기간 뒤 영구 삭제한다. 폴링 주기가 삭제 주기보다 길면 데이터가 증발한다. 그래서 더 짧게, 더 자주 읽는다.
- Health Connect(HEALTH)는 반대로 백그라운드 읽기에 호출 한도가 있다. 자주 부르면 `RateLimitException` 이 나면서 다음 전송까지 밀린다. 그래서 드물게, 챌린지 활성 시간대에만 읽는다.
- 기상 판정(WAKE)은 하루 중 의미 있는 순간이 아침뿐이다. 상시 폴링이 아니라 기상 시간대 직후 한 번을 보장하는 것이 요구사항이다.
- 지오펜스와 활동 인식(GEOFENCE, ACTIVITY)은 폴링이 아니라 수신기로 들어오므로 주기 자체가 없다.

폴링 주기를 상수 하나로 두는 순간, 너무 느려서 데이터를 잃거나 너무 빨라서 한도에 걸리거나 둘 중 하나를 겪는다. 주기는 신호가 언제 사라지는지와 API 제약이 정하는 값이지 앱이 고르는 취향이 아니었다.

## 3. 서버에 의존하는 폴링의 위험성, 그리고 대응

주기를 서버가 지시한다는 건 다음 실행 계획이 서버 응답에 묶여 있다는 뜻이다. 여기에 모바일 환경의 제약이 겹친다.

1. 네트워크가 꺼져 있을 수 있다. 비행기 모드, 지하철, 데이터 소진. 폴링이 발화해도 요청 자체가 불가능하다.
2. OS가 실행을 미룬다. Doze와 App Standby는 백그라운드 작업을 유예 구간으로 몰아넣는다.
3. 서버가 아플 수 있다. 실패했다고 다음 주기까지 아무것도 안 하면 데이터가 밀리고, 무작정 재시도하면 아픈 서버를 더 때린다.
4. 응답이 영영 안 올 수 있다. `flushIntervalSec` 를 못 받으면 재조정 루프가 끊긴다.
5. 클라가 아예 침묵할 수 있다. Doze나 전원 꺼짐, 장기 오프라인이면 폴링 자체가 기동되지 않는다. 클라 혼자서는 이 상태를 벗어날 방법이 없다.

<iframe src="/assets/diagrams/2026-07-13-workmanager-periodic-polling.html"
        title="수집과 전송을 분리한 폴링 구조 다이어그램"
        loading="lazy" width="100%" height="640"
        style="border:1px solid var(--main-border-color); border-radius:6px;"></iframe>

> 잘려 보이면 [전체 화면으로 열기](/assets/diagrams/2026-07-13-workmanager-periodic-polling.html).
{: .prompt-tip }

첫째, 네트워크 문제는 수집과 전송을 분리해서 푼다(스펙 §0.2). 이 스펙에서 가장 중요한 문장은 버퍼가 기준이고 전송은 단순 배치라는 것이다. 신호는 발생하거나 폴링한 시점에 Room 영속 버퍼에 먼저 쌓이고, 수집 워커는 네트워크 제약 없이 돈다. 전송 워커에만 `Constraints` 로 `NetworkType.CONNECTED` 를 건다. 오프라인이면 WorkManager가 전송을 보류했다가 연결이 복구되는 순간 실행하고, 그동안 버퍼는 계속 쌓인다. 네트워크가 꺼져 있어도 잃는 것은 즉시성뿐이고 데이터는 남는다.

```kotlin
private fun buildRequest(intervalMinutes: Long): PeriodicWorkRequest =
    PeriodicWorkRequest
        .Builder(VerificationSyncWorker::class.java, intervalMinutes, TimeUnit.MINUTES)
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED) // 전송 워커에만
                .build(),
        ).build()
```

밀린 버퍼를 몰아서 재전송하면 중복이 걱정된다. 스펙은 멱등성을 신호 단위로 설계했다. Health Connect는 `recordId` 로, 없는 신호는 `(userId, signalType, observedAt)` 조합으로 서버가 중복을 걸러낸다. 재전송이 중복 판정을 만들지 않는다는 보장이 있어야 클라가 겁 없이 다시 보낼 수 있다. 배치가 너무 커져 `413` 이 오면 나눠서 다시 보낸다.

둘째, Doze와는 싸우지 않는다(스펙 §0.6). 스펙 자체가 상시 Foreground Service를 금지하고, expedited WorkManager로 중요한 구간만 보호하는 실행 모델을 못 박았다. 부정확한 주기 작업을 받아들이는 대신, 적시성이 필요한 순간에는 expedited OneTimeWork로 따라잡기 작업을 쏜다. 쿼터가 소진되면 일반 작업으로 강등되도록 `RUN_AS_NON_EXPEDITED_WORK_REQUEST` 를 걸고, `ExistingWorkPolicy.KEEP` 으로 이벤트가 연달아 터질 때의 폭주를 막는다.

```kotlin
fun enqueueCatchUp(context: Context, policy: ExistingWorkPolicy = ExistingWorkPolicy.KEEP) {
    val request = OneTimeWorkRequest.Builder(VerificationSyncWorker::class.java)
        .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST) // 쿼터 소진 시 강등
        .setConstraints(/* CONNECTED 동일 */)
        .build()
    WorkManager.getInstance(context)
        .enqueueUniqueWork(CATCH_UP_WORK_NAME, policy, request)
}
```

셋째, 서버 오류는 세 갈래로 분류한다. 모든 예외를 `retry` 로 던지면 잘못된 페이로드를 영원히 재전송하는 좀비가 되고, 모든 예외를 `success` 로 삼키면 일시 장애에 데이터를 잃는다. 예외를 순수 함수로 판정해 폐기와 재시도를 구분했다. `400` 은 폐기하고 success로 끝내 무한 재전송을 막고, `429` 와 네트워크 오류는 `Result.retry()` 로 지수 백오프에 태운다.

```kotlin
internal fun syncOutcomeFor(error: Throwable?): SyncOutcome =
    when (error) {
        null -> SyncOutcome.SUCCESS
        is InvalidSignalPayloadException -> SyncOutcome.DISCARD // 400: 폐기, 무한 재전송 금지
        is SyncTooFrequentException -> SyncOutcome.RETRY        // 429: 백오프 재시도
        else -> SyncOutcome.RETRY
    }
```

넷째, 침묵을 신호로 만든다(스펙 §0.5, §0.7). 서버 의존 폴링에서 가장 고약한 실패는 조용한 실패다. 서버 입장에서는 신호가 안 오는 것이 인증을 안 한 것인지 폴링이 죽은 것인지 구분되지 않는다. 스펙은 이걸 두 겹으로 막는다. 하나는 보낼 신호가 없어도 sync를 치고 `gaps[]` 에 공백 사유를 담아 보내는 것이다. 권한 회수, OS의 이벤트 삭제, 15일 초과 같은 값으로 사유를 남기면, 서버가 신호 부재를 `NO_SIGNAL` 로 처리하기 전에 유예나 제외를 판단할 수 있다. 다른 하나는 envelope에 워커 heartbeat를 동봉하는 것이다. 마지막 성공 시각과 `standbyBucket`, 배터리 최적화 예외 여부를 함께 보내면 이 기기의 폴링이 언제부터 왜 밀렸는지 서버가 진단할 수 있다. 사용자 화면은 서버 응답이 아니라 로컬 진행률 캐시를 먼저 읽으므로, 서버가 잠시 죽어도 마지막 성공 상태를 보여준다.

다섯째, 그래도 클라가 침묵하면 서버가 깨운다. 앞의 네 가지는 모두 클라가 언젠가는 깨어난다는 전제 위에 있다. Doze가 깊거나 전원이 오래 꺼져 있으면 그 전제가 무너진다. 폴링 기반 구조의 마지막 구멍이다. 스펙은 이 경우 방향을 뒤집는다. 이 기기의 sync가 밀렸다는 사실을 heartbeat로 아는 쪽은 서버이므로, 서버가 해당 기기를 타게팅해 알림 UI 없는 FCM 데이터 메시지를 쏜다. 클라는 이를 받아 expedited WorkManager를 기동하고 버퍼에 쌓인 분량을 일괄 전송한다.

```text
서버 정책(기기별 주기)
  → 정상 경로: 클라가 주기마다 스스로 sync
  → 클라 미기동(Doze·오프라인): 서버가 FCM 데이터 메시지(타게팅) 발사
      → FCM wake → expedited WorkManager 기동 → 버퍼 누적분 일괄 전송 → ACK + 정책 델타
```

클라가 주도하는 폴링이 기본이고, 서버가 주도하는 푸시는 폴링이 죽었을 때의 복구 채널이다. 푸시를 기본으로 삼지 않는 이유는 두 가지다. FCM 데이터 메시지는 전달 보장이 없고 Doze에서 묶이거나 유예된다. 그리고 매 주기를 푸시로 돌리면 서버가 모든 기기의 스케줄러가 되어버린다. 반대로 폴링만 쓰면 죽은 클라를 살릴 수 없다. 둘을 기본과 폴백으로 겹쳐야 한쪽의 실패가 전체 실패가 되지 않는다. 앞서 본 따라잡기 워커가 지오펜스 이벤트뿐 아니라 이 FCM 트리거의 도착지로도 설계된 이유가 여기 있다. Hilt 그래프에 접근하지 못하는 수신기도 static 호출로 같은 sync 작업을 큐에 넣을 수 있다.

> 이 FCM 복구 경로는 스펙에 정의된 설계이고, 클라이언트의 수신부는 아직 후속 구현 항목이다. 폴링 기반 앱을 설계한다면 서버가 클라를 깨울 수단을 처음부터 구조 안에 넣어두길 권한다.
{: .prompt-info }

## 마무리

WorkManager 주기 폴링에서 챙긴 것들:

- 초기화는 명시적으로 한다. 자동 초기화를 제거하고 `Configuration.Provider` 와 `HiltWorkerFactory` 를 붙인다. Worker에 DI를 쓰려면 필수다.
- 주기는 두 층이고 전부 변한다. 전송 주기는 서버 정책이 정하고(`flushIntervalSec`, 15분 하한, `KEEP` 과 `UPDATE`), 수집 주기는 신호의 특성이 정한다.
- 폴링은 실패를 전제로 설계한다. 수집과 전송을 분리하고, 멱등 키로 재전송하고, 오류를 폐기와 재시도로 나누고, 공백은 `gaps[]` 로 생존은 heartbeat로 보고한다.
- 폴링이 죽으면 푸시가 살린다. 클라가 안 움직이면 서버가 FCM으로 깨워 따라잡기 작업을 돌린다.

주기적으로 서버에 물어보기는 한 줄짜리 요구사항이다. 하지만 기기의 전원과 네트워크, OS 정책, 서버 사정이 전부 변수인 환경에서는 폴링이 안 도는 경우를 기본값으로 놓고 설계해야 한다. 안 돌았다는 사실조차 서버가 알 수 있어야 한다는 것이 이번 구현에서 가장 크게 배운 점이다.

## 참고 자료

- [WorkManager 개요](https://developer.android.com/topic/libraries/architecture/workmanager)
- [커스텀 WorkManager 설정](https://developer.android.com/develop/background-work/background-tasks/persistent/configuration/custom-configuration) : 자동 이니셜라이저 제거와 `Configuration.Provider` 공식 가이드
- [WorkRequest 정의](https://developer.android.com/develop/background-work/background-tasks/persistent/getting-started/define-work) : 15분 최소 주기, Constraints, expedited work
- [Android에서 FCM 메시지 수신](https://firebase.google.com/docs/cloud-messaging/android/receive) : 데이터 메시지 수신 동작
