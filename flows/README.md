# NiFi Flow 관리 가이드

이 폴더는 NiFi 플로우 JSON을 안전하게 관리하기 위한 구조입니다.

## 권장 구조

- `flows/NiFi_flow.template.json`: Git 추적 대상 (팀 공유용 템플릿)
- `flows/private/NiFi_flow.raw.json`: Git 제외 대상 (개인 백업 원본)

## 운영 원칙

1. NiFi에서 export한 원본은 `flows/private/`에 보관합니다.
2. 공유가 필요한 경우 민감 정보/환경 의존 값을 제거해 `NiFi_flow.template.json`으로 저장합니다.
3. 커밋에는 템플릿만 포함하고, raw 백업은 포함하지 않습니다.
