# 국내 여행 리포트 CLI

여행일을 입력하면 Copa의 Claude API가 국내 도시 한 곳과 계절별 날씨, 행사 후보, 추천 이유를 JSON으로 반환합니다. 프로그램은 `recommended_city`를 Kakao Local 검색에 전달해 음식점 최대 5곳을 조회합니다. 검색 결과로 Markdown 리포트와 맛집 지도를 저장합니다.

날씨는 계절적 경향이며 행사는 검토 후보입니다. 방문 전에 실제 예보와 개최 일정을 확인하세요.

## 설치와 실행

Windows PowerShell에서 Python 3.10 이상을 사용합니다. 프로젝트 폴더에서 다음 명령을 실행합니다.

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

`.env`를 편집기에 열고 두 키를 입력합니다. 같은 이름의 환경변수를 이미 설정했다면 환경변수가 우선합니다.

| 이름 | 설정값 |
| --- | --- |
| `COPA_API_KEY` | Copa에서 발급받은 Virtual Key |
| `KAKAO_REST_API_KEY` | Kakao Developers 앱의 REST API 키. 앱에서 카카오맵 API 사용 권한을 확인하세요. |
| `COPA_MODEL` | 기본값은 `claude-haiku-4`(Claude Haiku 4.5)입니다. Claude Sonnet 4.6을 쓰려면 `claude-sonnet-4`를 지정합니다. |

[Kakao Local 키워드 검색 문서](https://developers.kakao.com/docs/ko/kakaomap/rest-api#search-by-keyword)

설정 후 여행일을 지정해 실행합니다.

```powershell
.\.venv\Scripts\python.exe travel.py -date "2027-01-15" --open-map
```

`-date`는 필수입니다. 날짜가 `YYYY-MM-DD` 형식과 다르거나 달력에 없는 날짜이면 사용법을 출력하고 종료합니다.

## 결과 확인

콘솔에 진행 상황과 저장 경로가 표시됩니다. 프로젝트의 `results/` 폴더에 실행 일시와 여행일을 포함한 파일 세 개가 저장됩니다.

- **JSON**에는 여행일(`travel_date`), 검증된 1차 추천(`recommendation`), 음식점 목록(`restaurants`), 오류 요약(`errors`)이 들어 있습니다. 음식점 좌표가 있으면 `x`는 경도, `y`는 위도입니다.
- **Markdown**에는 추천 지역과 이유, 날씨, 행사·축제 후보, 맛집, 오전·오후·저녁 일정, `errors`가 들어 있습니다.
- **HTML**에는 좌표가 있는 맛집을 번호로 표시하는 지도와 전체 맛집 목록이 들어 있습니다. `--open-map`을 붙이면 저장 후 기본 브라우저에서 지도가 열립니다. HTML 파일이나 리포트의 지도 링크를 열어 다시 볼 수도 있습니다. 배경 지도를 표시하려면 인터넷 연결이 필요합니다.

`results/`에 있는 `mock_` 파일은 모의 API 응답으로 생성한 실행 예시입니다. 예시의 음식점과 주소는 가상 데이터입니다.

## API 요청과 오류 처리

Copa에는 Anthropic 형식의 `POST /v1/messages`로 모델과 프롬프트를 보내고 응답의 `content`에서 텍스트 블록을 읽습니다. Copa Virtual Key는 Bearer 인증 헤더로 보냅니다. Kakao Local에는 `GET /v2/local/search/keyword.json`으로 검색어와 음식점 카테고리, 결과 수를 보내고 응답의 `documents`에서 장소를 읽습니다.

[Anthropic Messages API 문서](https://platform.claude.com/docs/en/api/messages/create)

| 상황 | 프로그램 동작 및 확인할 사항 |
| --- | --- |
| 키 미설정 | 설정 방법을 안내하고 API 호출 전에 종료합니다. |
| HTTP 401 또는 403 | 키 종류, 키 값, 앱 권한과 IP 제한 설정을 확인합니다. |
| HTTP 429 | 호출량 제한과 남은 쿼터를 확인합니다. |
| 연결 오류 또는 타임아웃 | 네트워크 상태를 확인합니다. |
| 1차 추천 JSON 또는 스키마 오류 | 필수 키와 타입을 강조한 프롬프트로 한 번 더 요청합니다. 추천에 최종 실패하면 오류 요약을 파일에 저장하고 종료 코드 1을 반환합니다. |
| 맛집 검색 0건 또는 Kakao API 오류 | 맛집을 `데이터 없음`으로 표시하고 리포트를 계속 생성합니다. API 오류는 `errors`에 기록합니다. |
| 최종 리포트 생성 실패 | 검증된 데이터로 기본 양식 리포트를 저장하고 종료 코드 1을 반환합니다. |

## 키 관리

실제 키는 로컬 `.env` 또는 환경변수에 보관하세요. 제출할 때는 빈 키 항목이 있는 `.env.example`을 사용하세요. `.gitignore`에 `.env`와 생성 결과 폴더가 포함되어 있습니다. 코드, README, 화면 캡처, 공유 로그에 실제 키를 넣지 마세요. 노출된 키는 해당 서비스에서 폐기하고 재발급하세요.
