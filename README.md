# OWKR 내전 워크샵

OWKR 내전에서 사용할 OverPy 워크샵 소스를 준비하는 저장소입니다.

> **준비 중** · 현재 `gamemode.opy`에는 OverPy starter pack 예제가 들어 있습니다.
> 운영용 내전 모드나 배포된 워크샵 공유 코드는 아직 이 저장소에 안내되어 있지 않습니다.

## 현재 파일

- `gamemode.opy` · 설정과 상호작용 키 이동, 위치 표시를 포함한 시작 예제
- `AGENTS.md` · OverPy 작업과 검증 지침

## 개발·검증

OverPy 도구를 준비한 뒤 저장소 루트에서 소스를 컴파일합니다.

```sh
overpy compile -i gamemode.opy -o /tmp/owkr-workshop.ws.txt
```

생성한 Workshop 텍스트를 게임의 워크샵 편집기에 붙여넣어 확인합니다.
컴파일 성공과 실제 게임에서의 동작 검증은 구분하며, 내전 운영에 적용하기 전에는
별도 사용자 지정 게임에서 참가·퇴장, 재시작과 여러 플레이어의 동시 동작을 확인해야 합니다.

게임 규칙이나 기능을 변경하기 전 `AGENTS.md`를 확인하세요.

## 관련 프로젝트

- [OWKR 커뮤니티](https://github.com/Overwatch-KR) · [Discord](https://discord.gg/overwatchkr)
- [OWKR Match](https://github.com/Overwatch-KR/owkr-balance) · 내전 참가 명단과 팀 편성
- [Gather Bot](https://github.com/Overwatch-KR/owkr-gather-bot) · Discord 내전 모집과 티어 수집
- [내전 규정](https://github.com/Overwatch-KR/owkr-docs) · 규정, 참가 안내와 FAQ
- [관리자 시험](https://github.com/Overwatch-KR/owkr-exam) · 내전 관리자 시험 웹
- [내전 워크샵](https://github.com/Overwatch-KR/owkr-scrim-workshop) · 준비 중인 OverPy 소스
