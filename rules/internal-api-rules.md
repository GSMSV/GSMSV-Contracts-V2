# 내부 API(Server ↔ Infra) 공통 규약

> 상태: 이 문서의 규약은 **확정**이다 (2026-11-12). 12장에 적은 것만 아직 정하지 않았다.
>
> 읽는 사람: API 서버(Kotlin, `GSMSV-Server-V2`)에서 실행 엔진을 부르는 코드를 짜는 사람, 실행 엔진(Go, `GSMSV-Infra-V2`)을 짜는 사람.
>
> 함께 읽을 문서: [Kotlin·Go 경계](../policies/kotlin-go-boundary.md)(누가 무엇을 하는가) · [내부 API 명세](../openapi/internal.yaml)(엔드포인트) · [23번(자원 이름 규칙)](resource-naming.md) · [상수 값](../constants/resource-id.md)

## 1. 이 문서가 정하는 것

Kotlin·Go 경계 문서는 **누가 무엇을 하는가**를 정한다. 이 문서는 둘이 **어떤 모양으로 주고받는가**를 정한다. 엔드포인트 하나하나의 요청·응답은 [내부 API 명세](../openapi/internal.yaml)에 있고, 모든 엔드포인트가 이 문서를 따른다.

- 호출 방향은 **Server → Infra 하나뿐**이다. Infra는 Server를 부르지 않는다 (콜백·웹훅 없음). 오래 걸리는 일의 결과는 Server가 물어서 받는다 ([경계 6.1](../policies/kotlin-go-boundary.md#61-오래-걸리는-작업))
- Infra는 내부에서만 닿는다. 브라우저·CLI는 Infra에 붙지 않는다
- Infra는 권한을 판단하지 않고, GSMSV DB에 접근하지 않고, 사용자에게 보일 문장을 만들지 않는다 ([경계 4장 레드라인](../policies/kotlin-go-boundary.md#레드라인--go에-넣지-않는-것)). 사용자에게 가는 문구와 웹 API 오류(`{code, message, request_id}`)는 Server가 만든다
- **계약 먼저**: Infra에 엔드포인트를 새로 만들거나 바꿀 때는 구현보다 [내부 API 명세](../openapi/internal.yaml) PR을 먼저 올린다

## 2. 전송과 형식

| 항목 | 규약 |
|---|---|
| 방식 | HTTP/1.1 + JSON (REST). gRPC는 쓰지 않는다 |
| 본문 | `Content-Type: application/json; charset=utf-8`. GET 요청에는 본문을 싣지 않는다 |
| 주소 | `{Infra 기본 주소}{경로}`. 경로 앞에 `/api`나 버전을 붙이지 않는다 (예: `/network/vms/{vm_id}/ports`). 기본 주소는 운영 정보라 이 저장소에 쓰지 않는다 |
| 칸 이름 | snake_case (`internal_ip`, `allowed_source`) |
| enum 값 | 소문자 snake_case (`tcp`, `running`, `vm.create`). 오류 `code`만 대문자 snake_case |
| 모르는 칸 | 받는 쪽은 **무시한다**. Infra는 요청의 모르는 칸을, Server는 응답의 모르는 칸을 무시한다. 칸을 더하는 변경이 상대를 깨지 않게 하려는 것이다 |
| 빠진 값 | 선택 칸은 `null`과 칸이 없는 것을 같은 뜻으로 받는다. 필수 칸이 없으면 400 `INVALID_REQUEST`다 |
| 시각 | RFC 3339, UTC, `Z`로 끝난다 (예: `2026-11-12T09:00:00Z`) |
| 단위 | 칸 이름 끝에 단위를 붙인다. `_mb`는 MiB, `_gb`는 GiB(1024 단위), `_sec`는 초. 값은 정수다 |

- HTTP/1.1로 정한 이유: 내부 호출은 평문 HTTP로 한 번씩 주고받으면 충분하다. 버전을 정하지 않으면 클라이언트가 HTTP/2 업그레이드를 시도할 수 있어 두 언어의 기본 동작이 엇갈릴 수 있다
- Server는 웹 API 요청의 모르는 칸을 거절하도록 설정돼 있다. 그래서 **Infra 응답을 받는 DTO마다 모르는 칸을 무시하는 설정을 붙인다** (지금 있는 네트워크 응답 DTO처럼)
- REST로 정한 이유: Server가 이미 이 모양으로 Infra 클라이언트를 만들었다(Server-V2 `infra` 패키지). 내부 호출량이 적고 팀이 REST에 익숙하다 ([경계 6.6](../policies/kotlin-go-boundary.md#66-계약-정의와-개발-편의)). 칸 이름·enum 규칙을 웹 API와 같게 두면 Server가 값을 옮겨 담을 때 바꿀 것이 없다

### 2.1 자원을 가리키는 값

- 자원은 **자원 id**(`vm-…`, `port-…`, `domain-…`, `snap-…`)로 가리킨다. GRN은 쓰지 않는다 ([23번](resource-naming.md): 시스템 안에서는 자원 id, 사람이 보는 곳에서는 GRN)
- Infra는 받은 id를 [상수 값](../constants/resource-id.md)의 정규식으로 검사하고, 틀리면 400 `INVALID_RESOURCE_ID`로 거절한다. GRN 파서는 두지 않는다. id가 게이트웨이 규칙·Proxmox·Caddy 명령의 인자로 넘어가기 때문이다
- Infra가 만드는 인프라 객체(Proxmox VM 이름, 스냅샷 이름, 게이트웨이 규칙 주석, Caddy 라우트 id, Kubernetes 이름)에는 이 id를 적는다 ([23번](resource-naming.md))
- **노드와 VM 번호**는 VM을 만들 때 Server가 본문으로 한 번 넘긴다. 그 뒤 호출은 `vm_id`만 보내고, Infra가 Proxmox VM 이름(= `vm_id`)으로 노드와 VM 번호를 찾는다. 노드끼리 VM 번호가 겹칠 수 있어서 VM 번호만으로 찾지 않는다
- 작업 id(`job_id`)와 `boot_id`는 Infra가 정하는 값이다. 자원 id가 아니라서 [상수 값](../constants/resource-id.md)의 접두사 목록에 없다. Server는 해석하지 않고 저장·비교·전달만 한다. Infra는 `job-`·`boot-` 뒤에 상수 값 1장의 글자로 12자를 뽑아 만든다. 사용자에게 보이는 작업 id는 Server의 operation id뿐이다

## 3. 헤더

| 헤더 | 언제 | 규약 |
|---|---|---|
| `X-Request-Id` | 모든 요청·응답 | Server의 요청 id(`req-…`)를 그대로 싣는다. Infra는 로그에 `request_id`로 남기고, 응답 헤더와 오류 본문에 같은 값을 넣는다. 없거나 형식(`^req-[A-Za-z0-9_-]{1,64}$`)이 틀리면 Infra가 같은 형식으로 새로 만든다 (거절하지 않는다) |
| `Idempotency-Key` | 명령형 작업 요청 (4.2) | Server의 operation id를 싣는다. 형식은 `^[A-Za-z0-9_-]{1,64}$`. 4.3을 따른다 |
| 서비스 사이 인증 | 모든 요청 (헬스 체크 제외) | 방식과 자격증명은 이 저장소에 쓰지 않는다 ([경계 6.5](../policies/kotlin-go-boundary.md#65-인증배치자격증명)). 실패하면 401 `UNAUTHENTICATED` |

- `X-Request-Id`를 싣는 이유: Server와 Infra의 로그를 같은 값으로 이어 본다 ([22번](authorization.md)의 감사 로그 원안, [경계 6.8](../policies/kotlin-go-boundary.md#68-경계를-검증하는-방법)). Server는 요청 id의 형식을 이미 이렇게 쓴다
- `Idempotency-Key`에 operation id를 쓰는 이유: 사용자가 보낸 멱등 키는 Server가 operation 하나로 묶는다. 같은 사용자 요청이 다시 와도 operation id가 같으므로, Infra는 사용자 키를 몰라도 된다

## 4. 동작의 세 가지 모양

| 모양 | HTTP | 쓰는 곳 | 성공 응답 |
|---|---|---|---|
| 조회 | `GET` | 헬스 체크, 작업 상태, 노드 여유, 지표 | `200` + 본문 |
| 선언형 맞추기 | `PUT` (동기) | VM 포트 집합, VM 방화벽, HTTPS 라우트, Kubernetes 테넌트 | `200` + `{ "applied": true, ... }` |
| 명령형 작업 | `POST`·`PATCH`·`DELETE` (비동기) | VM 만들기·지우기·전원·사양, 스냅샷 만들기·지우기·복원 | `202` + `{ job_id, status, queue_position }` |

### 4.1 선언형 맞추기

- 본문은 대상의 원하는 상태 **전체**다. 목록에 없는 것은 지운다. 빈 목록은 "전부 회수"다
- Infra는 현재 상태와 비교해 **차이만** 적용한다. 같은 본문을 여러 번 보내도 결과가 같으므로 `Idempotency-Key`를 쓰지 않는다
- 대상은 포트 집합·방화벽이면 VM 하나, HTTPS 라우트면 목록 전체 하나다. 같은 VM이라도 포트 집합과 방화벽은 서로 다른 대상이다
- **Server는 같은 대상에 맞추기 요청을 한 번에 하나만 보낸다.** 앞 요청의 응답을 받은 뒤 다음 요청을 보내고, 보낼 본문은 차례를 잡은 뒤 DB에서 읽는다. Server가 여러 대여도 지켜지게 DB 잠금으로 차례를 잡는다. 이유: 두 요청이 거꾸로 도착하면 옛 상태가 마지막이 된다. 예를 들어 지운 포트 규칙이 다시 열린다
- Infra도 같은 대상에 대한 요청을 하나씩 차례로 처리한다. 나중에 처리한 요청이 마지막 상태가 된다
- 성공하면 `200`이고 `applied`는 항상 `true`다. 반영에 실패하면 2xx가 아닌 상태와 오류 본문(5장)으로 답한다. `applied: false`는 보내지 않는다 (Server는 받으면 실패로 처리한다)
- 응답의 개수 칸(`added`, `removed`)은 정수 개수다
- Infra는 원하는 상태를 **받기 전에는 아무것도 지우지 않는다.** 다시 떠서 기억이 비어 있을 때 "빈 목록"으로 착각해 규칙을 지우는 일을 막는다

### 4.2 명령형 작업

- Infra는 요청을 받자마자 작업 id를 정해 `202`로 답하고, 일은 뒤에서 한다. 형식·id·본문 검사는 접수 전에 동기로 하고, 틀리면 4xx로 답한다
- 작업 상태는 `GET /jobs/{job_id}`로 묻는다. 본문은 [명세](../openapi/internal.yaml)의 `Job`이다

| 칸 | 값 |
|---|---|
| `status` | `queued` → `running` → `succeeded` / `failed` / `canceled` |
| `stage` | VM 만들기: `queued` → `cloning` → `configuring` → `booting` → `networking`. 그 밖의 작업: `queued` → `running`. 끝나도 마지막 단계를 그대로 둔다 |
| `queue_position` | `queued`일 때 앞에 기다리는 작업 수. 그 밖에는 0 |
| `type` | 작업 종류 (`vm.create`, `vm.delete`, `vm.power`, `vm.resize`, `snap.create`, `snap.delete`, `snap.restore`). 앞부분은 자원 접두사와 같다 |
| `resource_id` | 작업 대상 자원 id |
| `error` | `failed`일 때만. `{ code, retryable, details }` (5장과 같은 code) |

- VM 만들기의 단계는 사용자에게 보여 주는 단계(대기 중(앞에 N명) → 복제 중 → 설정 중 → 부팅 중 → 네트워크 등록 중 → 완료, [기능별 정책](../policies/feature-policies.md))와 하나씩 맞는다. 화면 문구는 Server가 만든다
- 폴링 간격은 Server가 정한다. 작업 상태 조회는 Infra 메모리만 읽으므로 1~2초 간격이어도 Proxmox에 부담을 주지 않는다
- 취소: `POST /jobs/{job_id}/cancel`. `queued`면 바로 `canceled`, `running`이면 다음 단계로 넘어가기 전에 멈추고 만들다 만 것을 정리한다. 이미 끝난 작업이면 현재 상태만 준다
- **진실은 Server DB에 있다.** Infra의 작업 기록은 메모리에만 있다. 끝난 작업은 끝난 뒤 1시간 이상 남기고, Infra가 다시 뜨면 사라진다
- `GET /jobs/{job_id}`가 404 `JOB_NOT_FOUND`이면, Server는 대상 자원의 실제 상태를 Infra에 물어 operation을 마무리한다 ([경계 6.1](../policies/kotlin-go-boundary.md#61-오래-걸리는-작업)). 실제 상태를 묻는 엔드포인트는 VM 엔드포인트와 함께 명세에 더한다
- Infra는 작업을 실패로 끝낼 때 만들다 만 것을 먼저 정리한다. 정리까지 실패하면 `error.details.cleanup`에 `failed`를 적는다

### 4.3 멱등 키

- 명령형 작업 요청에는 `Idempotency-Key`가 **필수**다. 없으면 400 `INVALID_REQUEST`다
- 같은 키·같은 본문이 다시 오면 Infra는 새로 실행하지 않고 처음 작업을 `202`로 다시 준다 (진행 중이든 끝났든)
- 같은 키에 본문이 다르면 422 `IDEMPOTENCY_KEY_REUSED`다
- Infra가 다시 떠서 키 기억을 잃었을 때를 위해, 대상 자원의 id로 한 번 더 막는다. 예: VM 만들기에서 같은 `vm_id` 이름의 VM이 Proxmox에 이미 있으면 새로 만들지 않고 409 `ALREADY_EXISTS`로 답한다. Server는 이 응답을 받으면 실제 상태를 물어 확인한다

## 5. 오류

### 5.1 본문

```json
{ "code": "VMID_TAKEN", "retryable": false, "details": { "vmid": 101 }, "request_id": "req-3f2a9c1e7b6d4a58b0e2c4d6f8a1b3c5" }
```

- `code`: 오류 종류. 대문자와 밑줄. 웹 API와 뜻이 같으면 같은 이름을 쓴다 (`INVALID_REQUEST`, `INVALID_RESOURCE_ID`, `UNAUTHENTICATED`, `RESOURCE_NOT_FOUND`, `ALREADY_EXISTS`, `INTERNAL`)
- `retryable`: 같은 요청을 **그대로** 다시 보내도 되는지. `false`인 실패는 요청을 바꾸지 않으면 다시 해도 같다
- `details`: 원인 데이터(기계가 읽는 값). 칸 이름은 웹 API 오류의 `details`와 같다. 비밀값·자격증명은 넣지 않는다. 노드 이름이 들어갈 수 있으므로 Server는 `details`를 사용자 응답에 옮기지 않는다. 없으면 빠진다
- 사람이 읽을 문장(`message`)은 넣지 않는다. 사용자 문구는 Server가 `code`를 보고 만든다

### 5.2 HTTP 상태

| 상태 | 뜻 | `retryable` |
|---|---|---|
| 400 | 요청 형식이 틀렸다 | false |
| 401 | 서비스 사이 인증에 실패했다 | false |
| 404 | Infra가 대상(VM·작업 등)을 찾지 못했다 | false |
| 409 | 다른 것과 부딪혀 적용할 수 없다 | false |
| 422 | 형식은 맞지만 실행할 수 없는 요청이다 | false |
| 500 | Infra 안의 예상하지 못한 오류 | false |
| 503 | Proxmox·게이트웨이·Caddy·Kubernetes가 응답하지 않는다 | true |
| 504 | Infra 안에서 정한 시간 안에 끝내지 못했다 | true |

다시 시도해도 되는 실패는 **503과 504뿐이다.** Server는 지금도 503·504·연결 실패를 사용자에게 503 `UPSTREAM_UNAVAILABLE`로, 그 밖의 실패를 502 `GATEWAY_SYNC_FAILED`로 바꾼다. 위 표는 이 변환과 그대로 맞는다. 4xx를 code별로 다르게 처리해야 하는 기능(예: VM 번호 충돌)은 Server가 오류 본문의 `code`를 읽는다.

### 5.3 code 목록

| 상태 | `code` | 뜻 |
|---|---|---|
| 400 | `INVALID_REQUEST` | JSON이 아니거나, 필수 칸이 없거나, 값이 범위 밖이거나, 필수 헤더가 없다 |
| 400 | `INVALID_RESOURCE_ID` | id 형식이 틀렸다 |
| 401 | `UNAUTHENTICATED` | 서비스 사이 인증 실패 |
| 404 | `RESOURCE_NOT_FOUND` | 대상 자원이 Infra 쪽(Proxmox 등)에 없다 |
| 404 | `JOB_NOT_FOUND` | 작업 기록이 없다 (끝난 지 오래됐거나 Infra가 다시 떴다) |
| 409 | `ALREADY_EXISTS` | 같은 id의 인프라 객체가 이미 있다 |
| 409 | `VMID_TAKEN` | 받은 VM 번호를 그 노드에서 이미 쓰고 있다 |
| 409 | `PORT_TAKEN` | 받은 외부 포트를 다른 VM의 규칙이 쓰고 있다 |
| 409 | `INVALID_STATE` | 지금 상태에서 할 수 없는 동작이다 (예: 만드는 중인 VM의 전원) |
| 422 | `IDEMPOTENCY_KEY_REUSED` | 같은 멱등 키에 다른 본문이 왔다 |
| 422 | `NODE_CAPACITY` | 그 노드에 자리가 없다 |
| 422 | `TEMPLATE_NOT_FOUND` | 그 노드에 템플릿이 없다 |
| 422 | `DISK_SHRINK_NOT_ALLOWED` | 디스크를 줄이려 했다 |
| 500 | `INTERNAL` | Infra 안의 예상하지 못한 오류 |
| 503 | `UPSTREAM_UNAVAILABLE` | 외부 시스템이 응답하지 않는다 |
| 504 | `UPSTREAM_TIMEOUT` | 정한 시간 안에 끝내지 못했다 |

- code를 더할 때는 이 표에 먼저 더한다. Server는 모르는 code를 받으면 상태와 `retryable`만 보고 처리한다
- `VMID_TAKEN`은 `retryable: false`다. 같은 요청을 다시 보내면 또 실패하기 때문이다. Server는 **다음 VM 번호 후보로 새 요청**을 만든다 ([경계 4장](../policies/kotlin-go-boundary.md#4-경계표)). 본문이 바뀌므로 새 operation id(멱등 키)로 보낸다

## 6. 시간 제한과 재시도

| 항목 | 규약 |
|---|---|
| Server의 시간 제한 | 연결 2초, 응답 10초 (Server 설정 기본값) |
| Infra의 동기 응답 | 조회·선언형 맞추기·작업 접수는 **8초 안에** 답한다. 넘으면 하던 일을 멈추고 504 `UPSTREAM_TIMEOUT`으로 답한다 |
| 오래 걸리는 일 | 8초를 넘을 수 있는 일은 명령형 작업(4.2)으로 만든다 |
| Server의 재시도 | 503·504·연결 실패·응답 시간 초과이고, 멱등한 호출(GET, 선언형 PUT, `Idempotency-Key`가 있는 작업 요청)일 때만 최대 2번 더 보낸다 (0.5초, 1초 뒤) |
| 그 밖의 실패 | 다시 보내지 않는다 |

- 8초로 잡은 이유: Server가 10초에서 끊기 전에 Infra가 먼저 504로 답해야, Server가 "응답 없음"과 "Infra가 시간을 넘김"을 구분하고 두 쪽 로그에 같은 `request_id`가 남는다
- 응답 시간 초과 뒤에도 다시 보내도 되는 이유: Infra가 이미 반영했더라도 선언형 PUT은 같은 본문이면 결과가 같고, 작업 요청은 같은 멱등 키면 같은 작업을 돌려준다

## 7. 원하는 상태를 다시 맞추기 (조정)

선언형 자원(포트 집합, VM 방화벽, HTTPS 라우트, Kubernetes 테넌트)이 Server DB와 어긋나지 않게 두 겹으로 맞춘다.

| 누가 | 언제 | 무엇을 |
|---|---|---|
| Infra | 정해진 간격마다 | 마지막으로 받은 원하는 상태와 실제 상태(게이트웨이 규칙, VM 방화벽, Caddy 설정)를 비교해 되돌린다 ([경계 4장](../policies/kotlin-go-boundary.md#4-경계표)의 조정 루프) |
| Server | 자원이 바뀔 때 | 그 대상의 원하는 상태 전체를 보낸다 (4.1의 차례를 지킨다) |
| Server | Infra가 다시 떴을 때 | `GET /healthz`의 `boot_id`가 바뀌었으면 모든 대상의 원하는 상태를 다시 보낸다 |
| Server | 정해진 간격마다 | 모든 대상의 원하는 상태를 다시 보낸다 (놓친 변경을 막는 안전망) |

- 원천은 Server DB다. Infra는 받은 상태를 메모리에만 두고 디스크에 남기지 않는다
- Infra가 다시 뜬 뒤 원하는 상태를 받기 전에는 아무것도 지우지 않는다 (4.1)
- Server는 다시 보낼 때도 DB 트랜잭션 안에서 Infra를 부르지 않는다. DB에서 읽고 트랜잭션을 끝낸 뒤 보낸다. Infra 응답을 기다리는 동안(최대 10초) DB 잠금을 잡고 있지 않게 하려는 것이다. 차례는 4.1의 잠금으로 잡는다
- 간격 값(Infra의 비교 간격, Server의 재전송 간격, `boot_id` 확인 간격)은 운영하며 정하는 설정값이다

## 8. 헬스 체크

| 경로 | 뜻 | 응답 | 인증 |
|---|---|---|---|
| `GET /healthz` | 프로세스가 요청을 받을 수 있다. 외부 시스템은 보지 않는다 | `200 { status: "up", boot_id }` | 없음 |
| `GET /readyz` | Proxmox·게이트웨이·Caddy·Kubernetes에 닿는다 | 모두 닿으면 `200`, 하나라도 닿지 않으면 `503`. `{ status: "up" \| "down", checks }` | 없음 |

- 본문에 주소·노드 이름·장비 정보·버전을 넣지 않는다
- 인증을 빼는 이유: 배포 도구와 감시 도구가 자격증명 없이 상태를 봐야 한다. 내부 정보가 없으므로 인증 없이 열어도 된다

## 9. 가짜 모드와 테스트

- Infra에는 **가짜 모드**가 있다. Proxmox·게이트웨이·Caddy·Kubernetes 없이 뜨고, 모든 엔드포인트가 이 규약과 명세 그대로 답한다 ([기능별 정책](../policies/feature-policies.md) 33번). 켜는 방법은 Infra 저장소에 적는다
- 가짜 모드의 작업은 실제와 같은 단계를 짧은 간격으로 거쳐 끝난다
- 가짜 모드에서만, 요청 헤더 `X-Fake-Fail: {code}`를 보내면 그 code의 오류로 답한다. Server가 오류 처리를 시험할 때 쓴다. 가짜 모드가 아니면 이 헤더를 무시한다
- Server의 `local`·`dev` 프로필은 지금처럼 Server 안의 모의 클라이언트를 쓴다. Infra 가짜 모드는 두 저장소를 함께 띄우는 시험에 쓴다
- 양쪽 테스트는 [명세](../openapi/internal.yaml)의 예시 본문으로 요청·응답 모양을 검사한다. 테스트에 쓰는 id도 [상수 값](../constants/resource-id.md)의 형식을 지킨다 (Infra가 형식을 검사하기 때문이다)

## 10. 규약과 명세를 바꾸는 방법

- 경로에 버전을 두지 않는다. 호출하는 쪽이 Server 하나뿐이고 두 저장소를 같은 팀이 함께 배포하기 때문이다
- **깨지지 않는 변경**(선택 칸 추가, 엔드포인트 추가, code 추가)은 명세 PR이 합쳐진 뒤 바로 구현한다. 받는 쪽은 모르는 칸을 무시한다 (2장)
- **깨지는 변경**(칸 이름·뜻 바꾸기, 필수 칸 추가, 칸·엔드포인트 지우기)은 두 번에 나눈다. 새 모양을 더해 양쪽이 쓰게 한 뒤 옛 모양을 지운다
- 이 문서를 바꾸는 PR은 Server·Infra 담당이 함께 확인한다

## 11. 하지 않는 것

- Infra가 Server를 부르기 (콜백·웹훅)
- Infra가 GSMSV DB를 읽거나 쓰기, 권한 판단, 사용자 문장 만들기
- 내부 API에서 GRN 쓰기 (자원 id만 쓴다)
- 오류 본문·응답에 비밀값·자격증명 넣기
- `applied: false`를 200으로 보내기
- 원하는 상태를 받기 전에 규칙 지우기
- 같은 대상에 맞추기 요청을 동시에 여러 개 보내기
- DB 트랜잭션 안에서 Infra 부르기

## 12. 아직 정하지 않은 것

| 항목 | 정하는 곳 |
|---|---|
| 서비스 사이 인증의 방식과 자격증명 | 비공개 문서 (백엔드·Go 담당) |
| VM 엔드포인트 (만들기·지우기·전원·사양·스냅샷·노드 여유·지표·실제 상태 조회) | Go VM 엔진 구현 전 명세 PR |
| 노드별 VM 목록, 노드 실사용량 | 해당 기능 구현 전 명세 PR |
| VM 방화벽의 기본 정책(어느 규칙에도 걸리지 않은 트래픽을 받을지·막을지)과 `direction: out` 규칙에서 `internal_ip`를 쓰는 자리 | Go 방화벽 구현 전 Server·Go 담당 (명세 `PUT /network/vms/{vm_id}/firewall`) |
| Kubernetes 테넌트, 원터치 배포·서버리스의 Server → Infra 호출 | 서비스 실행 환경을 정한 뒤 명세 PR |
| 조정 간격 값 (7장) | 운영하며 정하는 설정값 |
