# GSMSV-Contracts-V2

[ GSMSV-V2 | Contracts ] GSMSV V2 Repository 간 공통 계약

GSMSV v2 저장소들이 함께 쓰는 약속을 두는 곳이다.

## 이 저장소에 두는 것

- **웹 API 명세** (Client ↔ Server): OpenAPI 형식. API를 새로 만들거나 바꿀 때는 구현보다 이 저장소의 명세 PR을 먼저 올린다 (계약 먼저)
- **자원 이름 규칙 상수** (접두사·정규식·생성 길이): 원본을 여기 한 곳에 두고 Kotlin·Go·React 상수를 만든다

폴더 구조, 상수 파일 형식, 다른 저장소로 전달하는 방법은 아직 정하지 않았다.

이 저장소에 PR·이슈가 생기면 backend·frontend·cloud Discord 채널에 모두 알림이 간다.

## GSMSV v2 저장소

| 저장소 | 내용 |
|---|---|
| [GSMSV-Server-V2](https://github.com/GSMSV/GSMSV-Server-V2) | API 서버 (Kotlin) |
| [GSMSV-Client-V2](https://github.com/GSMSV/GSMSV-Client-V2) | 웹 콘솔 (React) |
| [GSMSV-Infra-V2](https://github.com/GSMSV/GSMSV-Infra-V2) | Proxmox·k3s를 다루는 실행 엔진 (Go) |
| [GSMSV-Contracts-V2](https://github.com/GSMSV/GSMSV-Contracts-V2) | 저장소끼리 함께 쓰는 약속 (API 명세, 자원 이름 규칙) |

## 참여하기

- 협업 규칙: [CONTRIBUTING](.github/CONTRIBUTING.md)
- 보안 문제 신고: [SECURITY](.github/SECURITY.md)
