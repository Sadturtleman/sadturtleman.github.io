---
title: 온보딩에서 유저 정보와 버전 체크가 꼭 필요한 이유
date: 2026-06-26 20:00:00 +0900
categories: [개발, 설계]
tags: [온보딩, 앱설계, 버전관리, ux]
---

앱을 켜면 가장 먼저 보이는 화면, 온보딩 단계에서는 보통 두 가지를 확인합니다. 유저 정보(로그인 상태, 약관 동의, 프로필 완성 여부)와 앱 버전입니다. 바로 메인 화면을 띄우면 안 되나 싶지만, 이 두 체크를 건너뛰면 나중에 더 비싼 비용을 치릅니다. 이 글은 둘을 왜 온보딩에서 가장 먼저 처리해야 하는지 정리합니다.

## 왜 유저 정보를 먼저 확인하는가

유저 상태를 확인하지 않으면 앱은 이 사람이 누구인지 모르는 채로 화면을 그립니다. 여기서 문제가 생깁니다.

- 잘못된 화면 분기: 비로그인 유저에게 마이페이지를 띄우거나, 이미 가입한 유저에게 회원가입 화면을 다시 보여주는 사고가 납니다.
- 약관 동의 누락: 법적으로 필요한 동의를 받지 않은 채 서비스를 제공하면 컴플라이언스 문제가 됩니다.
- 개인화 실패: 유저의 등급, 권한, 설정을 모르면 첫 화면부터 어긋난 데이터를 보여줍니다.

유저 정보 체크는 이후 모든 화면 분기의 전제 조건입니다. 토큰이 유효한지, 프로필이 완성됐는지, 동의가 끝났는지를 먼저 정리해야 그다음 흐름이 안전합니다.

## 왜 버전 체크가 중요한가

앱은 배포하고 나면 유저마다 제각기 다른 버전을 설치한 채 돌아갑니다. 버전 체크 없이 운영하면 이런 일이 벌어집니다.

- 강제 업데이트가 필요한 상황: 보안 취약점이나 결제 로직 버그가 있는 구버전을 그대로 쓰게 두면 위험합니다.
- 서버 API 호환성: 서버는 신버전 스펙으로 바뀌었는데 구버전 앱이 옛 API를 호출하면 크래시나 데이터 오류가 납니다.
- 점진적 권고 업데이트: 당장 막을 정도는 아니지만 업데이트하면 더 좋다는 안내를 띄울 수 있습니다.

그래서 보통 버전을 세 가지로 나눠 다룹니다.

| 구분 | 의미 | 동작 |
|------|------|------|
| `force` | 최소 지원 버전 미만 | 업데이트 전까지 진입 차단 |
| `optional` | 권장 버전 미만 | 안내 후 계속 사용 가능 |
| `ok` | 최신/허용 버전 | 그대로 진입 |

## 유저 정보와 함께 기기 정보까지 받는 이유

온보딩에서 유저를 식별할 때 아이디와 토큰만 받지 않고 맥 주소(MAC), 유심(USIM) 정보, 국가 코드 같은 기기 정보도 함께 수집하는 경우가 많습니다. 왜 이런 것까지 받나 싶은데, 각각 목적이 있습니다.

- 국가 코드와 통신사 정보(MCC·MNC, USIM): 어느 나라, 어느 통신사에서 접속하는지에 따라 서비스 가능 지역, 언어와 통화, 법적 규제(예: 특정 국가에서 막아야 하는 기능)를 분기합니다. 유심에서 읽은 국가·통신사 코드는 로케일 설정보다 더 믿을 만한 지역 판별 근거입니다.
- 기기 식별자(MAC, 기기 고유 ID): 한 계정에 묶인 기기를 구분하고, 다중 기기 로그인 관리와 기기 변경 감지, 이상 접속 탐지에 씁니다. 평소와 다른 기기에서 로그인하면 추가 인증을 요구할 수 있습니다.
- 어뷰징과 중복 가입 방지: 한 사람이 여러 계정을 만들어 이벤트 보상을 가로채는 행위를 막을 때 기기 정보가 핵심 단서가 됩니다.
- 푸시와 환경 최적화: OS 버전, 기기 모델, 화면 해상도를 알면 그 기기에 맞는 리소스와 기능을 내려줄 수 있습니다.

다만 이 정보들은 대부분 개인정보이거나 식별 가능한 데이터입니다. 수집 항목과 목적을 약관에 명시하고 동의를 받아야 하며, 최신 OS는 MAC 주소를 무작위화하거나 접근을 제한하므로 플랫폼이 허용하는 식별자 정책을 따라야 합니다.

> 기기 정보는 목적이 분명한 항목만 최소로 받아야 합니다. 받을 수 있다는 이유로 다 받으면 개인정보 규제(GDPR, 국내 개인정보보호법) 위반으로 이어질 수 있습니다.
{: .prompt-danger }

## 온보딩 흐름을 그려보면

두 체크는 보통 버전을 먼저 보고 그다음 유저 정보를 봅니다. 차단해야 할 구버전이라면 유저 정보를 확인할 필요가 없기 때문입니다.

<iframe src="/assets/diagrams/2026-06-26-onboarding-user-version-check.html"
        title="온보딩 진입 게이트 흐름 다이어그램"
        loading="lazy" width="100%" height="600"
        style="border:1px solid var(--main-border-color); border-radius:6px;"></iframe>

> 잘려 보이면 [전체 화면으로 열기](/assets/diagrams/2026-06-26-onboarding-user-version-check.html).
{: .prompt-tip }

## 코드로 보는 체크 로직

아래는 온보딩 진입 시 버전과 유저 정보를 순서대로 검사해 다음 행동을 결정하는 의사코드입니다. 실제 구현에서는 서버가 내려준 `minVersion`, `latestVersion` 과 토큰 상태를 함께 봅니다.

```typescript
type GateResult =
  | { type: "force_update" }
  | { type: "optional_update" }
  | { type: "need_login" }
  | { type: "need_onboarding" }
  | { type: "enter_home" };

async function resolveOnboarding(): Promise<GateResult> {
  // 1) 버전 체크가 가장 먼저
  const { minVersion, latestVersion } = await fetchVersionPolicy();
  const current = getAppVersion();

  if (isLower(current, minVersion)) {
    return { type: "force_update" }; // 진입 차단
  }

  // 2) 유저 정보 체크
  const token = await loadAuthToken();
  if (!token || !(await isTokenValid(token))) {
    return { type: "need_login" };
  }

  const user = await fetchUserProfile(token);
  if (!user.agreedTerms || !user.profileCompleted) {
    return { type: "need_onboarding" };
  }

  // 2-1) 기기 정보 수집 후 서버에 동기화 (어뷰징·지역 분기·이상 접속 탐지용)
  const device = {
    deviceId: getDeviceId(),       // 플랫폼 허용 식별자
    countryCode: getSimCountry(),  // 유심 기반 국가 코드 (MCC)
    carrier: getCarrierName(),     // 통신사 (MNC)
    osVersion: getOsVersion(),
    model: getDeviceModel(),
  };
  await syncDeviceInfo(token, device);

  // 3) optional 업데이트는 막지 않고 안내만
  if (isLower(current, latestVersion)) {
    return { type: "optional_update" };
  }

  return { type: "enter_home" };
}
```

> 버전 체크와 유저 체크는 클라이언트 값만 믿지 말고 서버 응답을 근거로 판단해야 합니다. 클라이언트에 박아둔 상수는 배포 후 바꿀 수 없고, 위·변조도 가능하기 때문입니다.
{: .prompt-warning }

## 정리

- 유저 정보 체크는 이후 화면 분기와 개인화, 컴플라이언스의 전제 조건이다.
- 기기 정보(맥 주소, 유심, 국가 코드 등)는 지역 분기와 이상 접속 탐지, 어뷰징 방지를 위해 받되 목적이 분명한 항목만 최소로 동의 하에 수집한다.
- 버전 체크는 보안과 API 호환성, 운영 유연성을 위해 필요하며 `force`/`optional`/`ok` 로 나눠 다룬다.
- 순서는 버전을 먼저, 유저 정보를 나중에 보는 쪽이 자연스럽다. 막아야 할 버전이면 유저 정보까지 갈 필요가 없다.
- 두 판단 모두 서버 응답을 기준으로 해야 배포 후에도 정책을 바꿀 수 있다.

온보딩에서 한 번 걸러주면 뒤따르는 화면들은 상태를 다시 의심하지 않아도 됩니다. 분기가 줄고 예외 처리도 같이 줄어듭니다.

## 참고 자료

- [Version your app](https://developer.android.com/studio/publish/versioning) : 버전 코드와 버전 이름을 나누는 기준
- [In-app updates](https://developer.android.com/guide/playcore/in-app-updates) : Google Play가 제공하는 즉시·유연 업데이트 흐름
- [Best practices for unique identifiers](https://developer.android.com/training/articles/user-data-ids) : 용도별로 어떤 기기 식별자를 써야 하는지
