ENB 일일 업무 자동화 스크립트 (Daily Report Automation)이 프로젝트는 Slack에서 작성된 일일 업무 내용을 수집하여 AI로 파싱한 후, Google SpreadSheet에 기록하고 Slack으로 전체 브리핑을 전송하는 Google Apps Script 기반 자동화 툴입니다.
본 코드는 ENB(사업지원) 셀 전용 버전으로, 전사 표준 양식(전날 한 일 / 오늘 할 일 / 고민했던 부분)을 강제로 적용하여 데이터를 규격화합니다.

주요 기능
Slack 리마인더 자동화: 출/퇴근 시간에 맞춰 업무 공유 작성 요청 메시지 발송 (주말 제외).
미작성자 알림: 특정 시간까지 업무 보고를 작성하지 않은 대상자를 멘션하여 재알림 발송.
AI 기반 데이터 구조화 (OpenAI GPT-4o):
- 슬랙에 자유로운 형식으로 작성된 텍스트에서 '전날 한 일', '오늘 할 일', '고민했던 부분'을 정확하게 파싱합니다.
- 링크(Jira, Confluence 등)와 업무 내용을 분리하여 정제합니다.업무 보고 내용을 바탕으로 전체 업무의 핵심 3줄 요약을 자동 생성합니다.
- Jira 연동:담당자의 기한 임박(Due Soon) 및 기한 초과(Overdue) Jira 티켓을 자동으로 조회하여 '오늘 할 일' 목록에 추가합니다.
Google Sheets 자동 생성:수집된 데이터를 바탕으로 매일 새로운 구글 스프레드시트 파일을 생성하고 가독성 좋은 표 형태로 자동 포맷팅합니다.
Slack 일일 브리핑:취합된 전체 인원의 업무 내용과 AI 요약본을 정리하여 Slack 채널에 보기 좋게 브리핑(Block Kit 형태)합니다.


⚙️ 사전 준비 및 설정 (Configuration)
스크립트 상단의 설정 변수들을 환경에 맞게 기입해야 합니다.

1. API 키 및 토큰SLACK_TOKEN: Slack Bot OAuth Token (xoxb-...)
2. OPENAI_API_KEY: OpenAI API 키 (sk-...)
3. FOLDER_ID: 생성된 구글 시트를 저장할 Google Drive 폴더 ID2. 채널 및 대상자 설정
4. CHANNELS: 업무 보고를 수집할 채널 목록. (예: [{ id: 'C1234567', name: 'enb_platoon' }])TARGET_MEMBERS: 업무 보고 의무 대상자 목록. 미제출자 리마인더 발송 기준이 됩니다. (예: { 'U1234567': '홍길동' })BRIEFING_CHANNEL_ID: 전체 브리핑 메시지를 전송할 Slack 채널 ID.3. Jira 연동 설정 (선택사항)JIRA_BASE_URL, JIRA_ORG_ID: Jira 워크스페이스 정보JIRA_EMAIL, JIRA_API_TOKEN: Jira API 접근을 위한 이메일 및 토큰ID_MAP: Slack User ID와 Jira Account ID 간의 매핑 배열. 이 매핑을 기반으로 개인별 Jira 티켓을 매칭합니다.🚀 사용 방법 및 트리거(Trigger) 설정이 코드는 Google Apps Script 환경에서 실행되도록 작성되었습니다. 자동화를 위해 Apps Script의 [트리거 (Triggers)] 메뉴에서 다음 함수들을 원하는 시간에 동작하도록 설정하세요.함수명설명추천 트리거 시간morningDutyPrompt오전 일일 업무 공유 작성 요청 메시지 전송평일 오전 9:00 ~ 10:00eveningDutyPrompt오후 일일 업무 보고 작성 요청 메시지 전송평일 오후 5:30 ~ 6:00sendUnsubmittedReminder지정된 채널에서 업무 보고를 작성하지 않은 인원 멘션평일 오후 7:00runDailyReportAndBriefing(핵심) Slack 데이터 수집 → AI 파싱 → 구글 시트 생성 → 브리핑 발송평일 밤 11:30 또는 다음 날 새벽📋 데이터 처리 프로세스 (Workflow)데이터 수집 (collectAllChannelsData)기준 시간(오전 7시) 이후 Slack 채널에 올라온 메시지 이력을 가져옵니다.1차적으로 '전날 한 일' 또는 '오늘 할 일' 키워드가 포함된 메시지를 찾습니다.AI 파싱 (callOpenAIParser)OpenAI GPT 모델을 활용해 복잡한 원본 텍스트를 구조화된 JSON 데이터(yesterday, today, challenges, squad_...)로 변환합니다.Jira 병합 (attachJiraDueToAggregated_)사전에 매핑된 ID_MAP을 기반으로 해당 멤버의 급한 Jira 이슈를 조회하고, 파싱된 데이터의 '오늘 할 일' 최상단에 주입합니다.리포트 엑스포트 (SpreadsheetApp)수집된 정보를 바탕으로 yyyyMMdd_채널명 양식의 스프레드시트를 생성 및 저장하고, 배경색/테두리 등 엑셀 스타일을 일괄 적용합니다.요약 및 브리핑 (buildBriefingBlocks)AI를 통해 도출된 핵심 3줄 요약과 ENB 셀의 일일 업무 내역을 Slack Block Kit 양식으로 렌더링하여 브리핑 채널에 전송합니다.⚠️ 주의 사항본 스크립트의 AI 파서는 전사 프롬프트 규칙(전날 한 일 / 오늘 할 일 / 고민)을 엄격하게 준수하도록 짜여 있습니다. 팀원들이 해당 양식에 맞추어(혹은 유사한 키워드로) 슬랙에 작성해야 파싱 정확도가 높아집니다.PLATOON_CONFIG는 전사 호환을 위해 남겨져 있으며, AI가 텍스트 내에서 스쿼드 명칭을 자동 감지하는 데 사용됩니다.
