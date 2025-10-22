팀 작업용 Git 매뉴얼
팀 공용 레포지토리(teamRepository)를 SSH 기반으로 일관되고 안전하게 운영하기 위한 표준 운영 문서입니다. 목적은 “작업은 브랜치에서, 병합은 PR로, main은 보호” 원칙을 지키며 충돌과 품질 리스크를 최소화하는 것입니다. GitHub Flow를 기본으로, 커밋 서명·브랜치 보호·CI 필수 체크를 권장합니다.

목차
원칙과 범위

역할과 책임

SSH 준비(키·에이전트·원격 URL)

공용 리포 최신화(공용 파일 포함)

개인 작업(브랜치 생성/작업/동기화/푸시)

PR 작성·리뷰·병합

공용 파일 동시 편집 충돌 해결

브랜치 이동/정리 및 되돌리기

커밋 서명(Verified) 설정

문제 해결(트러블슈팅)

정책 스냅샷(엔터프라이즈 권장)

빠른 수행 카드(치트시트)

부록: 표준 템플릿/설정 스니펫

0. 원칙과 범위
원칙: main은 직접 푸시 금지, PR·리뷰·CI 통과 후 병합. 변경은 작은 단위로, 서명 커밋 유지.

범위: 개인 개발환경, 팀 공용 레포(teamRepository), CI/배포, 문서·코드 전반.

1. 역할과 책임
오너/메인터이너: 브랜치 보호·CODEOWNERS·필수 체크 구성, 접근권한 점검, 템플릿 유지.

개발자: SSH 키 생성/보관, 브랜치 작업·PR 규칙 준수, 커밋 서명·테스트·문서 동반.

CI/배포: 최소권한 시크릿, 환경 분리, 배포 키 로테이션, 감사 로그 유지.

2. SSH 준비(키·에이전트·원격 URL)
기존 키 확인

text
ls -la ~/.ssh
cat ~/.ssh/id_ed25519.pub  # 없으면 생성
키 생성·등록(Ed25519 권장)

text
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub  # 출력값을 GitHub > Settings > SSH and GPG keys에 등록
연결 테스트

text
ssh -T git@github.com
SSH 원격 URL 사용

text
# 신규 클론
git clone git@github.com:ORG/teamRepository.git

# 기존 레포 전환
git remote set-url origin git@github.com:ORG/teamRepository.git
git remote -v
권장 SSH 설정(~/.ssh/config)

text
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes
  IdentitiesOnly yes
3. 공용 리포 최신화(공용 파일 포함)
목표: 모든 팀원이 main을 동일 상태로 유지. 공용 readme.md 변경 반영.

A) 최초 클론(모든 팀원)

text
git clone git@github.com:ORG/teamRepository.git
cd teamRepository
git remote -v
git fetch --all --prune
B) 공용 파일 업데이트(홍길동 예시) — main 보호 시 브랜치→PR

text
git switch -c docs/update-readme-hong
printf "\n홍길동 - add\n" >> readme.md
git add readme.md
git commit -m "docs(readme): update by Hong"
git push -u origin docs/update-readme-hong
# GitHub에서 PR 생성 → 리뷰/CI 통과 → Squash merge
C) 최신 버전 받기(홍길순/김철수 포함)

text
git switch main
git pull --ff-only origin main
# 필요 시: git stash && git pull --ff-only origin main && git stash pop
체크포인트

pull 전 로컬 변경은 커밋 또는 스태시.

--ff-only로 불필요한 merge 커밋 방지.

4. 개인 작업(브랜치 생성/작업/동기화/푸시)
A) 브랜치 생성/전환

text
git switch -c feature/123-login-button-style
# 또는 git checkout -b feature/123-login-button-style
git branch
B) 작업/커밋

text
echo "test!!!!" > testLDS.txt
git add .
git commit -m "2025-10-22: create testLDS.txt (LDS)"
C) 푸시

text
git push -u origin feature/123-login-button-style
D) 최신 main 정기 반영

text
git fetch origin
git rebase origin/main
# 충돌 해결 후
git add <files>
git rebase --continue
git push --force-with-lease
주의: 보호 브랜치(main)에는 force push 금지.

5. PR 작성·리뷰·병합
A) PR 전 점검

text
# 테스트/빌드/린트 (프로젝트에 맞춰 선택)
npm test || ./gradlew test || mvn -q -DskipTests=false test
npm run build || ./gradlew build || mvn -q -DskipTests=false package

git fetch origin
git rebase origin/main
B) PR 작성(GitHub UI)

base: main, compare: 작업 브랜치

포함: 이슈 링크, 변경 요약, 스크린샷/영향도, 테스트, 롤백 플랜

리뷰어 지정(CODEOWNERS), 라벨/프로젝트/마일스톤

C) 리뷰 대응

text
git add <files>
git commit -m "fix: address review feedback"
git push
D) 병합

기본: Squash merge, 병합 후 브랜치 삭제

보호 규칙: 필수 리뷰, CI 통과, up-to-date, 서명 커밋

6. 공용 파일 동시 편집 충돌 해결(readme.md 예시)
A) 충돌 발생

text
git pull --ff-only origin main  # 충돌 메시지
B) 해결

text
# 충돌 표식(<<<<<<< ======= >>>>>>>) 기준 수동 통합
git status
git add readme.md
# 리베이스 중이었다면
git rebase --continue
# 머지 상황이면
git commit -m "docs(readme): resolve conflicts"
git push --force-with-lease   # 리베이스로 히스토리 변경 시
C) 예방

공용 문서는 작은 PR로 빠르게 병합

병합 전 git fetch && git rebase origin/main

7. 브랜치 이동/정리 및 되돌리기
A) 이동

text
git switch feature/event     # 또는 git checkout feature/event
git switch main              # 또는 git checkout main
B) 정리

text
git branch -d feature/123-login-button-style
git push origin --delete feature/123-login-button-style
git fetch --prune
C) 되돌리기/수정

text
git commit --amend -m "feat: better message"     # 미푸시 시 안전
git checkout origin/main -- path/to/file          # 파일만 이전 상태로
8. 커밋 서명(Verified) 설정
SSH 서명(간편) 또는 GPG 키 사용 가능. 팀 정책으로 “서명 커밋 요구” 권장.

text
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# SSH 서명
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
9. 문제 해결(트러블슈팅)
A) Permission denied (publickey)

text
ssh-add -l || ssh-add ~/.ssh/id_ed25519
# GitHub 계정에 공개키 등록 여부 확인
# ~/.ssh/config의 IdentityFile 확인
B) 원격이 HTTPS

text
git remote -v
git remote set-url origin git@github.com:ORG/teamRepository.git
C) 호스트 키 경고

조직이 공유한 github.com 지문으로 검증 후 known_hosts 갱신

D) 리베이스 충돌 반복

text
git rebase --abort
git merge origin/main
10. 정책 스냅샷(엔터프라이즈 권장)
main 보호: PR 필수, 리뷰 ≥1, CI 통과, up-to-date, 서명 커밋, force push 금지

브랜치 규칙: feature/·bugfix/·hotfix/ 접두어, 이슈 번호 포함

작은 PR, Squash 병합, 병합 후 브랜치 삭제

정기 fetch + rebase로 드리프트 최소화

11. 빠른 수행 카드(치트시트)
클론:
git clone git@github.com:ORG/teamRepository.git

새 작업:
git switch -c feature/123-title → 작업 → git add . → git commit -m ... → git push -u origin <branch>

최신화:
git fetch origin && git rebase origin/main

PR:
GitHub에서 생성 → 리뷰/CI → Squash merge → 브랜치 삭제

충돌:
에디트 → git add → git rebase --continue 또는 머지커밋

12. 부록: 템플릿/설정 스니펫
A) CODEOWNERS(예시)

text
# .github/CODEOWNERS
/docs/   @org/docs-team
/src/    @org/fe-team @org/be-team
*        @org/maintainers
B) PR 템플릿(예시)

text
# .github/pull_request_template.md
## Summary
- 

## Issue
- closes #

## Changes
- 

## Screenshots/Impact
- 

## Test
- [ ] unit
- [ ] e2e

## Rollback Plan
- 
C) 브랜치 보호 권장 체크

Require pull request reviews: 1–2

Require status checks to pass

Require signed commits

Require branches to be up to date

Disallow force pushes/deletions for main

D) Git 메시지 컨벤션(선택)

type(scope): subject

예) feat(auth): add OAuth login