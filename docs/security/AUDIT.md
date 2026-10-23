# 보안약점 진단 결과 (1차)

- 진단일: 2026-10-23
- 대상 커밋: `2f5dc5b` (dev-jiwon)
- 기준: KISA 소프트웨어 보안약점 진단가이드 49개 항목, Python 시큐어코딩 가이드(2023)
- 도구: bandit 1.x (정적분석) + 수동 코드 리뷰 + Streamlit AppTest 기반 동작 확인

## 평가 기준 요약 (2라운드 보안 부문 40%)

| 순위 | 기준 |
|---|---|
| 1순위 | 빌드 라인당 소스코드 보안약점 수 |
| 2순위 | 보안약점 조치율 |
| 3순위 | 보안약점을 감소하기 위한 구조적 설계 |

→ 약점 "밀도"가 1순위이므로 미사용 코드 제거도 점수에 직접 영향을 준다.
→ 개발기획서 "2. 주요 기능 및 개발환경"에 구조적 설계를 기술해야 한다.

## 진단 결과

### 🔴 치명 — 동적분석(모의해킹)에서 바로 악용 가능

| ID | 내용 | KISA 항목 | 위치 | 상태 |
|---|---|---|---|---|
| C-1 | 수동 입력 코드가 builtins 제한 없이 실행됨. `import os` 로 `.env`(API 키)·`dp_demo.db`(비밀번호 해시) 읽기, OS 명령 실행 가능 | 입력 2 코드삽입, 입력 5 OS 명령어 삽입 | `ui/manual_executor.py` | 미조치 |
| C-2 | 사용자 코드가 쓸 수 있는 `result.pkl` 을 메인 서버가 `pickle.load` → 서버 프로세스에서 임의 코드 실행 | 코드오류 5 신뢰할 수 없는 데이터의 역직렬화 | `ui/manual_executor.py` | 미조치 |
| C-3 | LLM 생성 코드를 서버 프로세스 안에서 `exec`. 프롬프트 인젝션으로 C-1 과 동일 공격 가능. 타임아웃 시 스레드를 종료할 수 없어 무한루프가 계속 실행됨 | 입력 2 코드삽입, 시간및상태 2 종료되지 않는 반복문 | `dp_agent_wrap.py` `InlineStatementExecutor`, `core/executors.py` | 미조치 |
| C-4 | 수동 모드 결과가 노이즈 없는 원본값. `df.head()` 로 개인 레코드가 그대로 노출되어 DP 보호가 우회됨 | 보안기능 2 부적절한 인가 / 중요정보 노출 | `ui/pages/agent_qa.py` 결과 블록 | 미조치 (정책 결정 필요) |

### 🟠 높음 — 정적분석에서 검출

| ID | 내용 | KISA 항목 | 위치 |
|---|---|---|---|
| H-1 | 로그인 실패 횟수 제한 없음 | 보안기능 16 반복된 인증시도 제한 기능 부재 | `ui/auth.py`, `ui/db.py verify` |
| H-2 | 비밀번호 정책이 길이(8자)만 검사 | 보안기능 9 취약한 비밀번호 허용 | `ui/db.py create_user` |
| H-3 | 데모 계정 비밀번호 하드코딩 (`demo1234`, `admin1234`) | 보안기능 6 하드코드된 중요정보 | `ui/db.py init` |
| H-4 | 예외 메시지(`{e}`)를 화면에 그대로 노출, `traceback.print_exc()` 사용 | 에러처리 1 오류메시지 정보노출 | `ui/pages/agent_qa.py`, `ui/boot.py`, `ui/agent_bridge.py` |
| H-5 | 광범위한 `except Exception` 30곳, `except: pass` 2곳 | 에러처리 3 부적절한 예외처리, 2 오류상황 대응 부재 | 전반 |
| H-6 | 디버그 `print`, `assert` 잔존 | 캡슐화 2 제거되지 않고 남은 디버그 코드 | `dp_agent_wrap.py`, `ui/requests.py` |
| H-7 | 식별자를 f-string 으로 조립한 SQL (`PRAGMA`/`ALTER TABLE`) | 입력 1 SQL 삽입 | `ui/db.py _ensure_column` |
| H-8 | Hugging Face 데이터셋 revision 미고정 | 보안기능 15 무결성 검사 없는 코드 다운로드 / 공급망 | `core/utils.py` |
| H-9 | 의존성 버전 미고정 (`requirements.txt`) | SW 공급망 보안 | `requirements.txt` |

### 🟡 미사용 코드 (라인당 약점 밀도 개선)

| 경로 | 비고 |
|---|---|
| `ui_design/` | `ui/` 의 이전 복사본, 앱에서 import 하지 않음 |
| `main.py`, `config/`, `annotation/`, `new_agent.ipynb` | SemEval 원본 실험 코드. `yaml.load`, `eval` 등 약점 포함 |

## 이미 적용된 보안 설계 (유지·강조)

- 비밀번호: PBKDF2-HMAC-SHA256 + 랜덤 솔트(`secrets.token_hex`), 200,000회 반복
- 로그인 비교: `hmac.compare_digest` 상수 시간 비교, 없는 계정도 동일 연산으로 응답 시간 편향 방지
- SQL: 사용자 입력은 모두 파라미터 바인딩
- XSS: `unsafe_allow_html` 사용 시 사용자 값은 `html.escape`
- 비밀정보: `.env`, `dp_demo.db` 는 `.gitignore` 로 저장소에서 제외
- 수동 코드 실행: 별도 프로세스 + 타임아웃 (C-1/C-2 조치로 강화 예정)

## bandit 원본 결과 (조치 전)

High 0 · Medium 8 · Low 10

| 심각도 | ID | 위치 | 내용 |
|---|---|---|---|
| MEDIUM | B102 | `ui/manual_executor.py`, `dp_agent_wrap.py`, `core/executors.py` ×2 | exec 사용 |
| MEDIUM | B301 | `ui/manual_executor.py` | pickle 역직렬화 |
| MEDIUM | B506 | `main.py` | yaml.load |
| MEDIUM | B307 | `main.py` | eval |
| MEDIUM | B615 | `core/utils.py` | HF 다운로드 revision 미고정 |
| LOW | B603/B404/B403 | `ui/manual_executor.py` | subprocess, pickle import |
| LOW | B110 | `ui/pages/agent_qa.py`, `ui/boot.py` | try/except/pass |
| LOW | B101 | `ui/requests.py`, `ui_design/requests.py`, `dp_agent_wrap.py` ×2 | assert |
| LOW | B311 | `ui_design/pages/privacy.py` | 비암호용 난수 |

## 조치 계획

| 일정 | 작업 |
|---|---|
| 10/23–24 | 🟡 미사용 코드 정리 → C-1~C-3 실행기 재설계 (AST 화이트리스트, JSON 결과 전달, LLM 코드도 격리 프로세스에서 실행) |
| 10/25–26 | C-4, H-1~H-9 |
| 10/27 | 재진단 + 모의해킹 시나리오 테스트 + 조치 전후 비교표 |
| 10/28–30 | 개발기획서 보안 구조설계 반영, 발표자료·시연영상 |
