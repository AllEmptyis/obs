## git bash 사용
### 계정 입력
git config user.name "AllEmptyis"
git config user.email "parkda2000@gmail.com"

### repository
-  로컬 repository 지정, 해당 폴더로 이동 후

git init //git 초기화
gist remote add origin  https://github.com/AllEmptyis/obs.git // 원격 저장소 연결
git status // 상태 확인
git add . //파일 추가
git commit -m "command"
git push -u origin main
### origin 연동
git remote add origin https://github.com/AllEmptyis/obs.git
git branch -M main //현재 브랜치 이름을 main으로 강제 변경
git push -u origin main //현재 브랜치 main을 origin(원격저장소)의 main에 업로드

*-u: --set-upstream 축약형
자동으로 업로드 할 브랜치 이름 지정

### 기타 명령어
git pull         # 최신 상태 가져오기
git add .        # 변경된 파일 모두 선택
git commit -m "메시지"  # 버전 저장
git push         # GitHub에 파일 올리기
git remote -v # 연결된 원격 주소 보기
git branch -vv # 현재 브랜치가 어떤 원격 브랜치에 연결 되어 있는지
git remote remove origin


### 옵시디언 git 설정
- 플러그인 git 설치
- 원격 레포지토리 public 상태로 두기
- 만일 push가 안 될 시 환경 변수 문제일 가능성이 있으니 bash에서 git config 다시 해주기 (--global 사용x)