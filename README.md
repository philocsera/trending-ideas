# Trending Ideas

매일 아침, **GitHub Trending(daily)에 어제는 없던 신규 레포**를 추리고, 그 레포에서 영감을 받아 만들 수 있는 **프로젝트·서비스 아이디어**를 한국어로 제안해 GitHub Pages에 올린다.

- **외부 LLM API 미사용.** 아이디어는 정해진 시각에 자동 실행되는 Claude Code 스킬이 **inline으로 직접** 만든다 → 구독료 외 추가 비용 0.
- **서버 0.** 스크랩·판별·생성·발행이 전부 로컬 launchd 잡 한 번으로 끝난다. GitHub Pages는 정적 파일만 서빙.
- 기존 `job_feed` / `good_morning` 파이프라인과 동일한 컨벤션(스킬=JSON 작성, 래퍼=git push, 정적 뷰어).

## 동작 흐름 (매일 07:00 KST)

```
launchd(07:00 KST) → run_trending_ideas.sh
   → claude -p "/trending_ideas <오늘>" --permission-mode bypassPermissions
       1. WebFetch github.com/trending?since=daily   (인증 불필요)
       2. state/last.json(어제 목록)과 차집합 → 신규 레포
       3. 신규 레포별 README 보강(WebFetch, 병렬) → 한국어 아이디어 2~3개 생성
       4. data/<오늘>.json · latest.json · index.json 작성, state/last.json 갱신
   → 래퍼가 health-check 후 git add/commit/push → GitHub Pages 갱신
```

> 07:00 KST는 트렌딩 daily가 리셋되는 09:00 KST(00:00 UTC) **이전**이라, 직전 UTC일의 트렌딩을 온전히 담는다.

## 파일 구조

```
trending/
├── index.html                  # 정적 뷰어(SPA). 스킬이 절대 수정하지 않음.
├── data/
│   ├── index.json              # { "dates": [...] }  (최신순)
│   ├── latest.json             # 가장 최근 날짜 파일의 사본
│   └── <YYYY-MM-DD>.json        # 그날의 신규 레포 + 아이디어
├── state/
│   └── last.json               # 직전 실행의 전체 트렌딩 slug 목록(차집합 기준선)
├── run_trending_ideas.sh        # launchd 래퍼(헬스체크 + git push)
└── com.yeoukkori.trending.plist # launchd 스케줄(07:00 KST)
```

스킬 본문: `~/.claude/commands/trending_ideas.md` (슬래시 명령 `/trending_ideas`)

## 데이터 스키마 (`data/<date>.json`)

```jsonc
{
  "date": "2026-06-26",
  "is_baseline": false,        // 첫 실행(시드)일 때만 true
  "trending_count": 25,        // 그날 트렌딩 전체 수
  "new_count": 6,              // 신규 레포 수
  "new_repos": [{
    "rank": 1, "name": "owner/repo", "url": "...",
    "description": "원문", "description_ko": "한국어 한 줄",
    "language": "Python", "stars": 12345, "stars_today": 312, "topics": [],
    "ideas": [{
      "title": "...", "target_user": "...", "what": "...",
      "why_now": "...", "mvp": "...", "difficulty": "쉬움|보통|어려움",
      "monetization": "..."
    }]
  }]
}
```

## 셋업 (최초 1회 — 계정/권한이 필요한 단계)

1) **GitHub 저장소 생성 + 푸시** (public이어야 Pages 무료)
```bash
cd /Users/johyeonseong/Downloads/playground/trending
gh repo create philocsera/trending-ideas --public --source . --remote origin --push
# gh가 없으면: GitHub에서 빈 repo 만들고
#   git remote add origin https://github.com/philocsera/trending-ideas.git
#   git push -u origin main
```

2) **GitHub Pages 켜기**: 저장소 Settings → Pages → Build and deployment → Source = **Deploy from a branch** → Branch = `main` / `(root)` → Save.
   - 게시 URL: `https://philocsera.github.io/trending-ideas/`

3) **매일 자동 실행 등록** (launchd)
```bash
cp /Users/johyeonseong/Downloads/playground/trending/com.yeoukkori.trending.plist \
   ~/Library/LaunchAgents/com.yeoukkori.trending.plist
launchctl unload ~/Library/LaunchAgents/com.yeoukkori.trending.plist 2>/dev/null
launchctl load   ~/Library/LaunchAgents/com.yeoukkori.trending.plist
launchctl list | grep trending     # 등록 확인
```

4) **수동 테스트(선택)** — 내일까지 안 기다리고 바로 한 번 돌려보기
```bash
/Users/johyeonseong/Downloads/playground/trending/run_trending_ideas.sh
tail -n 40 /tmp/yeoukkori-trending-ideas.log
```

## 로컬 미리보기

`file://`로 열면 fetch가 막히므로 간단한 서버로 연다:
```bash
cd /Users/johyeonseong/Downloads/playground/trending && python3 -m http.server 8765
# → http://localhost:8765/
```

## 메모

- **첫날(시드)**: 기준선이 없으니 신규 판별 대신 현재 트렌딩 상위를 시드로 보여주고 `state/last.json`을 만든다. 진짜 신규 판별은 **다음 실행부터**.
- **신규의 정의**: 스타 임계값이 아니라 “오늘 트렌딩에 있고 직전 스냅샷엔 없던 레포”(차집합).
- **가드레일**: 트렌딩 파싱이 10개 미만이면 OSS Insight API로 폴백, 그래도 실패하면 기준선을 덮어쓰지 않고 종료(상태 보존).
