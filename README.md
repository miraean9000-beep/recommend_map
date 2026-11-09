# 국내 여행 리포트 CLI

여행일을 입력하면 Copa의 Claude API가 국내 도시 2~3곳과 계절별 날씨, 행사 후보, 추천 이유를 JSON으로 반환합니다. 프로그램은 추천 도시마다 Kakao Local에서 음식점을 검색하고 Markdown 리포트와 맛집 지도를 저장합니다.

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

```text
usage: travel.py [-h] -date YYYY-MM-DD [--no-cache]
                 [--restaurant-count 1~15] [--open-map | --no-map]
travel.py: error: argument -date: 실제 달력 날짜를 YYYY-MM-DD 형식으로 입력하세요. 예시: -date "2027-01-15"
```

`--restaurant-count`는 도시별 최대 음식점 수이며 기본값은 5입니다. `--no-map`을 지정하면 JSON과 Markdown만 저장합니다.

## 결과 확인

콘솔에 진행 상황과 저장 경로가 표시됩니다. 프로젝트의 `results/` 폴더에 `실행일_시각_travel_여행일` 이름으로 JSON과 Markdown, 기본 설정에서는 지도 HTML도 저장됩니다.

- **JSON**에는 여행일(`travel_date`), 검증된 1차 추천(`recommendation`), 음식점 목록(`restaurants`), 오류 목록(`errors`)이 들어 있습니다. 음식점의 `city`로 추천 도시를 구분하며 `x`는 경도, `y`는 위도입니다.
- **Markdown**에는 추천 지역과 이유, 날씨, 행사·축제 후보, 맛집, 오전·오후·저녁 일정, `errors`가 들어 있습니다.
- **HTML**에는 좌표가 있는 맛집을 번호로 표시하는 지도와 전체 맛집 목록이 들어 있습니다. `--open-map`을 붙이면 저장 후 기본 브라우저에서 지도가 열립니다. HTML 파일이나 리포트의 지도 링크를 열어 다시 볼 수도 있습니다. 배경 지도를 표시하려면 인터넷 연결이 필요합니다.

`results/`에 있는 `mock_` 파일은 모의 API 응답으로 생성한 실행 예시입니다. 예시의 음식점과 주소는 가상 데이터입니다.

`main()`은 키 설정을 확인한 뒤 API를 호출하기 전에 `find_latest_cache()`를 실행합니다. 여행일, 모델, 도시별 검색 수, 지도 저장 설정이 같고 오류 없이 저장된 최신 결과를 그대로 사용합니다. 만료 시간은 없으며, `--no-cache`로 새로 조회할 수 있습니다.

## API 요청과 오류 처리

Copa에는 Anthropic 형식의 `POST /v1/messages`로 모델과 프롬프트를 보내고 응답의 `content`에서 텍스트 블록을 읽습니다. Copa Virtual Key는 Bearer 인증 헤더로 보냅니다. Kakao Local에는 `GET /v2/local/search/keyword.json`으로 검색어와 음식점 카테고리, 결과 수를 보내고 응답의 `documents`에서 장소를 읽습니다.

[Anthropic Messages API 문서](https://platform.claude.com/docs/en/api/messages/create)

- **POST**는 모델에 새 결과 생성을 요청합니다. `POST /v1/messages`의 요청 본문 예시는 `{"model":"claude-haiku-4","max_tokens":2048,"messages":[{"role":"user","content":"여행지 추천"}]}`이고, 응답 예시는 `{"content":[{"type":"text","text":"생성 결과"}]}`입니다.
- **GET**은 장소 데이터를 조회합니다. `GET /v2/local/search/keyword.json?query=제주%20맛집&category_group_code=FD6&size=5`의 응답 예시는 `{"documents":[{"place_name":"예시 식당","x":"126.5","y":"33.5"}]}`입니다.

LLM의 1차 응답을 구조화된 JSON으로 출력하도록 요구하면 `json.loads()`로 쉽게 파싱하고 필수 키와 값의 타입을 검증할 수 있습니다. 검증한 `recommended_cities` 배열은 자유 형식의 문장에서 도시 이름을 다시 추출할 필요 없이 다음 단계인 카카오 맛집 검색의 입력으로 바로 재사용할 수 있습니다.

1차 응답은 다음 JSON 구조를 사용합니다.

```json
{
  "recommended_cities": ["제주특별자치도", "강릉"],
  "weather": "제주는 바람이 잦고 강릉은 쌀쌀한 시기입니다.",
  "events": ["겨울 문화행사 후보"],
  "reason": "제주는 해안 풍경을 즐기기 좋습니다. 강릉은 바다 산책과 실내 관람을 함께 계획할 수 있습니다."
}
```

`recommended_cities`는 도시 문자열 2~3개의 배열입니다. `weather`와 `reason`은 문자열이고 `events`는 후보 문자열 1~3개의 배열입니다. 검증 및 정규화 후 저장하는 `recommended_city`는 단수 키를 사용하는 기존 소비자를 위해 첫 번째 도시를 담습니다. 검색 단계는 `recommended_cities`를 순회하며 각 도시 이름을 바로 검색어로 사용합니다.

`travel.py`의 `normalize_city_name()`과 `CITY_NORMALIZATION_MAP`은 검색에 사용할 행정구역 이름을 줄입니다. 예를 들어 `서울특별시 → 서울`, `부산광역시 → 부산`, `제주특별자치도 제주시 → 제주 제주시`로 정리합니다.

| 상황 | 프로그램 동작 및 확인할 사항 |
| --- | --- |
| 키 미설정 | 설정 방법을 안내하고 API 호출 전에 종료합니다. |
| HTTP 401 또는 403 | 키 종류, 키 값, 앱 권한과 IP 제한 설정을 확인합니다. |
| HTTP 429 | 호출량 제한과 남은 쿼터를 확인합니다. |
| 연결 오류 또는 타임아웃 | 네트워크 상태를 확인합니다. |
| 1차 추천 JSON 또는 스키마 오류 | 필수 키와 타입을 강조한 프롬프트로 한 번 더 요청합니다. 추천에 최종 실패하면 오류 요약을 파일에 저장하고 종료 코드 1을 반환합니다. |
| 맛집 검색 0건 또는 Kakao API 오류 | 맛집을 `데이터 없음`으로 표시하고 리포트를 계속 생성합니다. API 오류는 `errors`에 기록합니다. |
| 최종 리포트 생성 실패 | 검증된 데이터로 기본 양식 리포트를 저장하고 종료 코드 1을 반환합니다. |
| 결과 파일 저장 실패 | `결과 파일 저장 실패. results 폴더의 쓰기 권한과 디스크 공간을 확인하세요.`를 출력하고 종료 코드 1을 반환합니다. |

추천 응답이 JSON으로 파싱되지 않거나 필수 키의 타입이 틀리면 `recommend()`가 프롬프트를 보정하고 자동으로 한 번 더 호출합니다. 두 번째 응답도 실패하면 `1차 추천 JSON을 두 번의 요청에서 읽거나 검증하지 못했습니다.`가 기록됩니다.

오류가 나면 콘솔에 단계와 제공자를 출력합니다. JSON의 `errors` 배열과 Markdown의 `## errors`에는 같은 오류를 남깁니다.

```json
{"errors":[{"stage":"restaurant_search","provider":"kakao","message":"제주 맛집 검색 실패. HTTP 403. 키 값과 API 사용 권한을 확인하세요."}]}
```

```markdown
## 맛집

### 제주

데이터 없음

## errors

- `stage=restaurant_search` `provider=kakao` message=제주 맛집 검색 실패. HTTP 403. 키 값과 API 사용 권한을 확인하세요.
```

## 주요 함수와 제공자 교체

| 함수 | 입력 | 출력 |
| --- | --- | --- |
| `recommend()` | 여행일, Copa 설정, 오류 목록 | 검증된 추천 JSON |
| `collect_restaurants()` | 도시 목록, 검색 제공자, 도시별 검색 수, 오류 목록 | 도시가 표시된 맛집 목록 |
| `generate_report()` | 추천·맛집 데이터, Copa 설정 | Markdown 본문 |
| `save_results()` | 실행 데이터, 리포트, 지도 저장 여부 | 결과 파일 경로 |

검색 제공자는 `PlaceSearchProvider`를 따릅니다. 다른 제공자를 붙이려면 `name`과 `search(city, size)`를 구현하고, 응답을 `name`, `address`, `category`, `url`, `x`, `y` 필드로 변환합니다. 호출 실패는 `ApiError`로 전달합니다. 예를 들어 `collect_restaurants(cities, KakaoPlaceSearchProvider(key), 5, errors)`의 두 번째 인자를 새 제공자 인스턴스로 바꾸면 됩니다.

기능을 파일 단위로 분리할 때는 공통 계약을 유지하며 LLM 호출, 장소 검색, 리포트 작성, 결과 저장을 각각 `llm.py`, `places.py`, `report.py`, `storage.py`로 옮길 수 있습니다. 현재 진입점과 함수는 `travel.py`에서 확인할 수 있습니다.

## 키 관리

실제 키는 로컬 `.env` 또는 환경변수에 보관하세요. 제출할 때는 빈 키 항목이 있는 `.env.example`을 사용하세요. `.gitignore`에 `.env`와 생성 결과 폴더가 포함되어 있습니다. 코드, README, 화면 캡처, 공유 로그에 실제 키를 넣지 마세요. 노출된 키는 해당 서비스에서 폐기하고 재발급하세요.

인증 헤더, 원본 오류 응답 본문, 쿼리를 포함한 요청 URL은 로그에 기록하지 않습니다. 예를 들어 카카오 인증 오류는 `[경고] stage=restaurant_search provider=kakao 제주 맛집 검색 실패. HTTP 403. 키 값과 API 사용 권한을 확인하세요.`로 표시됩니다. CI에서는 저장소의 비밀 값으로 `COPA_API_KEY`와 `KAKAO_REST_API_KEY`를 등록한 뒤 실행 단계의 환경변수로 주입합니다.
