# Junseo’s Security Notes

김준서의 보안·개발 학습 블로그입니다. Knights Frontier 3기 첫 미션의 여섯 카테고리로 구성했습니다.

사이트: **https://dongiveupj.github.io/security-notes/**

## 새 글 추가하기

GitHub 저장소에서 **Add file → Create new file**을 선택합니다. 파일 이름은 `_posts/2026-11-01-my-first-note.md`처럼 날짜와 영문 제목으로 만듭니다.

```yaml
---
layout: post
title: 글 제목
date: 2026-11-01 12:00:00 +0900
category: development
description: 목록에 표시할 짧은 소개
---

## 배경
무엇을 공부했는지 적습니다.

## 확인 과정
실제로 확인한 방법과 결과를 적습니다.

## 배운 점과 남은 질문
배운 점, 한계, 다음에 할 일을 적습니다.
```

`category`에는 아래 값 중 하나를 사용합니다.

| 메뉴 | category 값 |
|---|---|
| 개발 | development |
| CTF/Wargame | ctf-wargame |
| Bug Bounty | bug-bounty |
| 블로그/기술문서 | technical-writing |
| 논문/컨퍼런스 | papers-conferences |
| 공모전/자격증 | competitions-certifications |

미래 날짜의 글은 해당 시간이 되기 전까지 표시되지 않습니다. `Commit changes` 후 GitHub Actions의 Pages 배포가 끝나면 새 글을 확인할 수 있습니다. 글의 카테고리와 글 수는 자동으로 반영됩니다.

## 수정 위치

- 자기소개: `about.md`
- 제목·설명·카테고리 소개: `_config.yml`
- 홈 화면: `index.html`
- 공통 디자인·카테고리 필터: `_layouts/default.html`
- 글 레이아웃: `_layouts/post.html`

## Pages 설정

Settings → Pages → Build and deployment에서 **Deploy from a branch**, **main**, **/(root)**를 선택합니다. 기본 Jekyll 빌드를 사용하므로 `.nojekyll` 파일을 추가하지 않습니다.

이 저장소에 비밀번호, API 키, 비공개 보고서, 공개가 제한된 문제의 답안 등을 올리지 마세요.
