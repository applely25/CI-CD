# Study CI-CD

### CI yaml File
```yaml
name: Deploy # action 이름

on:
  push: # main 브랜치에 push 될 때 실행
    branches: ["main"]

  workflow_dispatch: # 수동으로 실행 가능

jobs:
  deploy:
    runs-on: ubuntu-latest # 실행 환경
    permissions:
      contents: write # 쓰기 권한 (워크플로우가 콘텐츠를 수정하거나 업데이트를 할 수 있도록)
                      # 'gh-pages' 브랜티로의 배포 작업을 할 수 있음 
    concurrency: # 동시성 관리, 같은 작업이 중복 실행되는 것을 방지
      group: ${{ github.workflow }} # 실행 중인 워크플로우의 이름을 참조
      cancel-in-progress: true # 동일 그룹의 실행이 취소하도록 해줌

    steps:
      - name: Use repository source # 레포지토리 소스 체크아웃
        uses: actions/checkout@v3

      - name: Use node.js # Node.js 버전 18 설정
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Cache node modules # 캐시를 설정하여 빌드 속도 높임
        id: cache-yarn
        uses: actions/cache@v3
        env:
          cache-name: cache-node-modules
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-build-${{ env.cache-name }}\-${{ hashFiles('**/yarn.lock') }}
          # 'yarn.lock' 파일의 해시를 키로 사용하여 Node 모듈을 캐싱
          restore-keys: ${{ runner.os }}-build-${{ env.cache-name }}-
            ${{ runner.os }}-build-
            ${{ runner.os }}-

      - name: Install Dependencies # yarn을 전역으로 설치하고, 의존성 설치
        run: |
          npm install -g yarn
          npm install --legacy-peer-deps --save-dev --ignore-scripts install-peers
          yarn install

      - if: ${{ steps.cache-yarn.outputs.cache-hit != 'true' }} # 캐시가 없을 경우,
        name: List the state of node modules # 설치된 Node 모듈의 상태를 목록화
        continue-on-error: true
        run: yarn list

      - name: Set PUBLIC_URL # 환경변수 PUBLIC_URL을 설정하여 .env 파일에 저장
        run: |
          PUBLIC_URL=$(echo $GITHUB_REPOSITORY | sed -r 's/^.+\/(.+)$/\/\1\//')
          echo PUBLIC_URL=$PUBLIC_URL > .env

      - name: Build  # 애플리케이션을 빌드, 빌드된 inedx.html 파일을 404.html로 복사
        run: |
          npm run build
          cp ./build/index.html ./build/404.html

      - name: Deploy to gh-pages branch # 빌드된 파일을 'gh-pages' 브랜치에 배포
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build

```
