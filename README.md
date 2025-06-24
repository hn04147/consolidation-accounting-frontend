# React + Vite

This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Babel](https://babeljs.io/) for Fast Refresh
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/) for Fast Refresh

## Expanding the ESLint configuration

If you are developing a production application, we recommend using TypeScript with type-aware lint rules enabled. Check out the [TS template](https://github.com/vitejs/vite/tree/main/packages/create-vite/template-react-ts) for information on how to integrate TypeScript and [`typescript-eslint`](https://typescript-eslint.io) in your project.

----

## React 프런트엔드 GitHub Actions CI/CD를 활용한 AWS S3 배포 가이드 (Yarn 사용)

이 가이드는 React 프런트엔드 프로젝트를 GitHub에 저장하고, **GitHub Actions**를 사용하여 CI/CD 파이프라인을 구축하며, 최종적으로 AWS S3에 배포하는 방법을 상세하게 설명합니다. **Yarn**을 패키지 매니저로 사용하여 처음 개발하는 사람도 쉽게 따라 할 수 있도록 모든 단계를 예시 코드와 함께 안내합니다.

-----

### 1단계: 프로젝트 초기 설정 및 기본 환경 구축

#### 1.1 AWS 계정 생성 및 IAM 사용자 설정

아직 AWS 계정이 없다면 [AWS 웹사이트](https://aws.amazon.com/)에서 계정을 생성하세요. 보안을 위해 루트 계정 대신 **IAM (Identity and Access Management) 사용자**를 생성하고 해당 사용자로 작업하는 것을 권장합니다.

1.  **IAM 대시보드**로 이동합니다.
2.  **사용자** 메뉴에서 **사용자 추가**를 클릭합니다.
3.  사용자 이름을 입력하고, **AWS 자격 증명 유형 선택**에서 "액세스 키 - 프로그래밍 방식 액세스"를 선택합니다.
4.  권한 설정에서 필요한 권한(`AmazonS3FullAccess` 등)을 부여합니다. 처음에는 광범위한 권한을 부여하고 점차적으로 필요한 최소한의 권한으로 줄여나가는 것이 좋습니다. (S3에만 배포할 것이므로 `AmazonS3FullAccess`만으로 충분합니다.)
5.  **액세스 키** 및 **비밀 액세스 키**를 기록해 두세요. 이 키는 다시 볼 수 없으므로 안전하게 보관해야 합니다. 이 키는 GitHub Actions에서 AWS 인증에 사용됩니다.

#### 1.2 AWS CLI 설치 및 구성 (선택 사항 - 로컬 테스트용)

로컬에서 AWS S3와 상호작용할 일이 있을 경우 AWS CLI를 설치하고 구성할 수 있습니다. CI/CD는 GitHub Actions에서 직접 AWS CLI를 사용하므로 필수는 아닙니다.

```bash
pip install awscli
```

설치 후, 이전에 발급받은 IAM 사용자 자격 증명으로 AWS CLI를 구성합니다.

```bashbash
aws configure
AWS Access Key ID [None]: YOUR_AWS_ACCESS_KEY_ID
AWS Secret Access Key [None]: YOUR_AWS_SECRET_ACCESS_KEY
Default region name [None]: ap-northeast-2 # 서울 리전 (원하는 리전으로 변경 가능)
Default output format [None]: json
```

#### 1.3 Yarn 설치

npm이 설치되어 있다면 다음 명령어로 Yarn을 설치할 수 있습니다.

```bash
npm install -g yarn
```

설치 확인:

```bash
yarn --version
```

-----

### 2단계: React 프로젝트 생성 및 GitHub 저장소 초기화

#### 2.1 React 프로젝트 폴더 생성 및 초기화

새로운 React 프로젝트를 생성합니다. 여기서는 Vite를 사용하겠습니다.

```bash
mkdir my-react-frontend
cd my-react-frontend
yarn create vite . --template react
```

프롬프트에 따라 프로젝트 이름을 입력하고, React 템플릿을 선택합니다. 프로젝트가 생성되면 다음과 같이 Yarn 의존성을 설치합니다.

```bash
yarn install
```

#### 2.2 GitHub 저장소 생성 및 연결

GitHub에서 새 저장소(`my-react-frontend`와 같은 이름)를 생성합니다. 저장소 생성 시 `.gitignore`나 `LICENSE` 파일은 추가하지 마세요. Vite가 이미 생성해 주었기 때문입니다.

로컬 프로젝트에서 GitHub 저장소에 연결합니다.

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/your-username/my-react-frontend.git # YOUR_USERNAME과 YOUR_REPO_NAME을 실제 값으로 변경
git push -u origin main
```

-----

### 3단계: AWS S3 버킷 설정

React 빌드 파일을 호스팅할 S3 버킷을 생성하고 정적 웹사이트 호스팅을 활성화합니다.

#### 3.1 S3 버킷 생성

1.  **AWS 콘솔**에 로그인하여 **S3** 서비스로 이동합니다.
2.  왼쪽 메뉴에서 **버킷**을 클릭한 다음, **버킷 생성**을 클릭합니다.
3.  **AWS 리전**을 프로젝트 배포를 원하는 리전(예: `ap-northeast-2` 서울)으로 선택합니다. **버킷 이름**은 전역적으로 고유해야 하며, 웹사이트 URL의 일부가 됩니다 (예: `my-react-frontend-bucket-2024`).
4.  **객체 소유권**은 "ACL 활성화됨"을 선택하고 "버킷 소유자 선호"로 설정합니다.
5.  **모든 퍼블릭 액세스 차단** 섹션에서 **"모든 퍼블릭 액세스 차단" 체크박스를 해제**합니다. (정적 웹사이트 호스팅을 위해 퍼블릭 접근이 필요합니다. 프로덕션 환경에서는 CloudFront와 같은 CDN을 사용하여 보안을 강화하는 것이 일반적입니다.)
6.  나머지 설정은 기본값으로 두고 **버킷 생성**을 클릭합니다.

#### 3.2 S3 버킷 정책 설정

버킷에 퍼블릭 읽기 권한을 부여하여 사용자가 웹사이트 파일에 접근할 수 있도록 합니다.

1.  생성된 버킷(예: `my-react-frontend-bucket-2024`)을 선택하고 **권한** 탭으로 이동합니다.

2.  **버킷 정책** 섹션에서 **편집**을 클릭합니다.

3.  다음 정책을 입력합니다. \*\*`YOUR_BUCKET_NAME`\*\*을 실제 버킷 이름으로 변경하세요.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadGetObject",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
            }
        ]
    }
    ```

4.  **변경 사항 저장**을 클릭합니다.

#### 3.3 정적 웹 사이트 호스팅 활성화

1.  버킷의 **속성** 탭으로 이동합니다.
2.  가장 아래로 스크롤하여 **정적 웹 사이트 호스팅** 섹션을 찾습니다. **편집**을 클릭합니다.
3.  **정적 웹 사이트 호스팅 활성화**를 선택합니다.
4.  **인덱스 문서**에 `index.html`을 입력합니다.
5.  **오류 문서**에도 `index.html`을 입력합니다 (React 라우팅을 위해).
6.  **변경 사항 저장**을 클릭합니다.
7.  저장 후, 다시 **정적 웹 사이트 호스팅** 섹션을 보면 **버킷 웹 사이트 엔드포인트** URL이 표시됩니다. 이 URL이 여러분의 웹사이트 주소입니다. 이 URL을 복사해 두세요.

-----

### 4단계: GitHub Actions CI/CD 워크플로우 설정

이제 GitHub Actions 워크플로우를 생성하여 코드가 `main` 브랜치에 푸시될 때마다 자동으로 빌드하고 S3에 배포하도록 설정합니다.

#### 4.1 GitHub 저장소 Secrets 설정

AWS 인증 정보를 GitHub Actions 워크플로우에서 안전하게 사용하기 위해 GitHub Secrets에 저장합니다.

1.  GitHub에서 여러분의 프런트엔드 저장소(예: `my-react-frontend`)로 이동합니다.

2.  **Settings** 탭을 클릭합니다.

3.  왼쪽 사이드바에서 **Security** \> **Secrets and variables** \> **Actions**를 클릭합니다.

4.  **New repository secret** 버튼을 클릭하여 다음 두 개의 시크릿을 추가합니다.

      * **Name**: `AWS_ACCESS_KEY_ID`
          * **Value**: IAM 사용자를 생성할 때 발급받은 AWS 액세스 키 ID
      * **Name**: `AWS_SECRET_ACCESS_KEY`
          * **Value**: IAM 사용자를 생성할 때 발급받은 AWS 비밀 액세스 키

#### 4.2 GitHub Actions 워크플로우 파일 생성

프로젝트 루트 디렉토리(`my-react-frontend`)에 `.github/workflows` 디렉토리를 생성하고 `deploy.yml` 파일을 만듭니다.

```bash
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
```

**`.github/workflows/deploy.yml` 파일 내용:**

```yaml
name: Deploy React App to S3

on:
  push:
    branches:
      - main # main 브랜치에 푸시될 때마다 워크플로우 실행

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest # 최신 Ubuntu 환경에서 실행

    steps:
      - name: Checkout repository # GitHub 저장소 코드 체크아웃
        uses: actions/checkout@v4

      - name: Set up Node.js # Node.js 환경 설정
        uses: actions/setup-node@v4
        with:
          node-version: '20' # 프로젝트에 맞는 Node.js 버전 (예: 18, 20)

      - name: Install dependencies with Yarn # Yarn으로 의존성 설치
        run: yarn install --frozen-lockfile # lockfile 기반으로 설치하여 일관성 유지

      - name: Build React app # React 앱 빌드
        run: yarn build

      - name: Configure AWS credentials # AWS 인증 정보 설정 (GitHub Secrets 사용)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2 # AWS S3 버킷이 위치한 리전으로 변경

      - name: Deploy to S3 # 빌드된 파일을 S3 버킷에 동기화
        run: aws s3 sync ./dist s3://YOUR_BUCKET_NAME --delete # YOUR_BUCKET_NAME을 실제 S3 버킷 이름으로 변경
        env:
          AWS_REGION: ap-northeast-2 # AWS S3 버킷이 위치한 리전으로 변경
```

**코드 설명:**

  * **`name: Deploy React App to S3`**: 워크플로우의 이름입니다.
  * **`on: push: branches: - main`**: `main` 브랜치에 코드가 푸시될 때 이 워크플로우를 실행하도록 지정합니다.
  * **`jobs: build-and-deploy`**: 단일 작업(`build-and-deploy`)을 정의합니다.
  * **`runs-on: ubuntu-latest`**: 작업이 실행될 환경을 최신 Ubuntu로 설정합니다.
  * **`steps`**: 작업을 구성하는 일련의 단계입니다.
      * **`Checkout repository`**: GitHub 저장소의 코드를 워크플로우 환경으로 가져옵니다.
      * **`Set up Node.js`**: Node.js 환경을 설정합니다. `node-version`을 프로젝트에 사용하는 버전으로 설정하세요.
      * **`Install dependencies with Yarn`**: `yarn install`을 실행하여 프로젝트 의존성을 설치합니다. `--frozen-lockfile`은 `yarn.lock` 파일을 기반으로 정확히 동일한 버전을 설치하도록 강제하여 빌드 일관성을 보장합니다.
      * **`Build React app`**: `yarn build` 명령을 실행하여 React 프로젝트를 빌드합니다. Vite를 사용하는 경우 기본적으로 `dist` 디렉토리에 빌드 결과물이 생성됩니다.
      * **`Configure AWS credentials`**: `aws-actions/configure-aws-credentials` 액션을 사용하여 AWS 자격 증명을 설정합니다. 이 단계에서 GitHub Secrets에 저장된 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 사용합니다.
      * **`Deploy to S3`**: `aws s3 sync` 명령을 사용하여 빌드된 `dist` 디렉토리의 내용을 S3 버킷에 동기화합니다. `--delete` 옵션은 S3 버킷에 없는 로컬 파일을 삭제하여 이전 빌드 아티팩트를 정리합니다. `YOUR_BUCKET_NAME`을 실제 S3 버킷 이름으로 **반드시 변경**해야 합니다.

-----

### 5단계: 배포 테스트

이제 모든 설정이 완료되었습니다. 변경 사항을 GitHub에 푸시하여 CI/CD 파이프라인을 테스트합니다.

#### 5.1 변경 사항 커밋 및 푸시

`my-react-frontend` 프로젝트 디렉토리에서 다음 명령어를 실행합니다.

```bash
git add .
git commit -m "Add GitHub Actions CI/CD for S3 deployment"
git push origin main
```

#### 5.2 GitHub Actions 워크플로우 확인

1.  GitHub 저장소에서 **Actions** 탭을 클릭합니다.
2.  "Deploy React App to S3"라는 워크플로우가 실행 중인 것을 확인할 수 있습니다.
3.  워크플로우를 클릭하여 각 단계가 성공적으로 실행되는지 확인합니다. "Deploy to S3" 단계가 성공적으로 완료되면 배포가 성공한 것입니다.

#### 5.3 배포된 웹사이트 확인

S3 버킷 속성에서 확인했던 **버킷 웹 사이트 엔드포인트** URL (예: `http://my-react-frontend-bucket-2024.s3-website.ap-northeast-2.amazonaws.com`)로 접속하여 React 앱이 정상적으로 보이는지 확인합니다.

-----

### 6단계: 후속 조치 및 고급 설정 (선택 사항)

  * **환경 변수 관리:** 프로덕션 API URL 등 민감하지 않은 환경 변수는 `my-react-frontend/.env.production` 파일에 정의하고, 빌드 시 Vite가 자동으로 주입하도록 설정할 수 있습니다. 예를 들어, `VITE_API_URL=https://your-api.com`과 같이 정의하면 React 코드에서 `import.meta.env.VITE_API_URL`로 접근할 수 있습니다.
  * **캐시 무효화 (CloudFront 사용 시):** S3에 직접 배포하는 경우 브라우저 캐시 문제로 최신 버전이 즉시 반영되지 않을 수 있습니다. CloudFront와 같은 CDN을 S3 앞에 두면 캐시 무효화 기능을 사용하여 최신 버전이 빠르게 전파되도록 할 수 있습니다.
  * **커스텀 도메인:** Route 53을 사용하여 커스텀 도메인(예: `www.your-domain.com`)을 S3 웹사이트 엔드포인트에 연결할 수 있습니다. CloudFront를 사용하면 HTTPS를 쉽게 적용할 수 있습니다.
  * **Pull Request 기반 워크플로우:** `main` 브랜치에 직접 푸시하는 대신, Pull Request가 머지될 때만 배포가 실행되도록 워크플로우를 수정할 수 있습니다.
    ```yaml
    on:
      pull_request:
        branches:
          - main
        types: [closed] # PR이 머지되어 닫힐 때
    ```
    또는
    ```yaml
    on:
      push:
        branches:
          - main
    ```
    과 같이 `main` 브랜치에 푸시될 때만 실행하도록 유지하되, PR 리뷰를 통해 코드를 검증하는 프로세스를 추가하는 것이 일반적입니다.