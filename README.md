# Chirpy Starter

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

A minimal, ready-to-use template for creating a blog with the [**Chirpy**][chirpy] Jekyll theme. Get up and running in minutes with all critical files pre-configured.

## Why This Starter Exists

When installing Chirpy through [RubyGems.org][gem], Jekyll can only read a subset of theme files (`_data`, `_layouts`, `_includes`, `_sass`, `assets`) and limited `_config.yml` options from the gem. As a result, users cannot enjoy the full out-of-the-box experience that Chirpy offers.

To unlock all features, the following files must be present in your Jekyll site:

```shell
.
├── _config.yml
├── _plugins
├── _tabs
└── index.html
```

This starter bundles those files from the latest **Chirpy** release along with a [CD][CD] workflow, so you can start writing immediately.

## Usage

Check out the [theme's docs](https://github.com/cotes2020/jekyll-theme-chirpy/wiki).

## 로컬 개발 환경 설정

### 1. Ruby 준비

이 저장소는 [`mise`](https://mise.jdx.dev/)로 Ruby 버전을 고정한다(`mise.toml` 참고, 현재 `3.3`). macOS 시스템 기본 Ruby는 버전이 낮아 `html-proofer` 등 최신 gem을 설치할 수 없으므로, `mise`로 별도 Ruby를 설치해서 사용한다.

```shell
# mise 설치 (최초 1회, https://mise.jdx.dev/getting-started.html 참고)
brew install mise

# 이 프로젝트 디렉토리에서 mise.toml에 명시된 Ruby 설치
mise install
```

셸 프로필(`~/.zshrc` 등)에 `eval "$(mise activate zsh)"`가 설정되어 있으면 이 디렉토리에서 `ruby`, `bundle`, `jekyll` 명령이 자동으로 `mise`가 관리하는 Ruby를 사용한다. 설정되어 있지 않다면 매 명령 앞에 `mise exec --`를 붙인다 (예: `mise exec -- bundle install`).

### 2. 의존성 설치

```shell
bundle install
```

### 3. 로컬 서버 실행

```shell
bundle exec jekyll serve
```

기본적으로 `http://127.0.0.1:4000`에서 접속할 수 있다. 초안(draft) 글까지 보고 싶다면:

```shell
bundle exec jekyll serve --livereload --drafts
```

### 4. (선택) VS Code Dev Container

`.devcontainer`가 구성되어 있어, VS Code에서 "Reopen in Container"를 사용하면 Ruby 설치 없이 컨테이너 안에서 바로 개발할 수 있다.

## Contributing

This repository is automatically updated with new releases from the theme repository. If you encounter any issues or want to contribute to its improvement, please visit the [theme repository][chirpy] to provide feedback.

## License

This work is published under [MIT][mit] License.

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[CD]: https://en.wikipedia.org/wiki/Continuous_deployment
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
