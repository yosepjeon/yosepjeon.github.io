# Yosep's Blog

Hugo 기반 개인 블로그 (Terminal 테마 적용)

## 사이트 주소

https://yosepjeon.github.io

## 다른 PC에서 사용하기

### 1. 사전 요구사항

- [Hugo Extended](https://gohugo.io/installation/) v0.90.x 이상
- [Git](https://git-scm.com/)

**macOS (Homebrew):**
```bash
brew install hugo
```

**Windows (Chocolatey):**
```bash
choco install hugo-extended
```

**Windows (Scoop):**
```bash
scoop install hugo-extended
```

### 2. 저장소 클론

```bash
git clone https://github.com/yosepjeon/yosepjeon.github.io.git
cd yosepjeon.github.io
```

### 3. 로컬 서버 실행

```bash
hugo server
```

브라우저에서 http://localhost:1313 접속

### 4. 새 포스트 작성

```bash
hugo new posts/my-new-post.md
```

`content/posts/my-new-post.md` 파일이 생성됩니다. 파일을 편집하여 내용을 작성하세요.

### 5. 빌드 및 배포

```bash
# 변경사항 커밋
git add .
git commit -m "Add new post"
git push
```

GitHub Actions가 자동으로 빌드하고 배포합니다.

## 테마 정보

- **테마**: [Terminal](https://github.com/panr/hugo-theme-terminal)
- **색상 스킴**: Blue Screen of Death
- **폰트**: JetBrains Mono

## 디렉토리 구조

```
.
├── content/          # 블로그 콘텐츠
│   ├── posts/        # 블로그 포스트
│   └── about.md      # About 페이지
├── themes/terminal/  # Terminal 테마
├── hugo.toml         # Hugo 설정 파일
└── .github/workflows # GitHub Actions 워크플로우
```
