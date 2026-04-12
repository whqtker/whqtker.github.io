---
share: true
categories:
  - AI
title: Serena MCP란?
date: 2026-04-12 19:46:21 +0900
slug: 2026-04-12-serena-mcp란?
tags:
  - Serena
  - MCP
  - Claude_Code
  - Copilot
  - Antigravity
description: Serena MCP란?
---

## Serena MCP

`Serena MCP`는 Oraios AI에서 개발한 오픈소스 코딩 에이전트 툴킷이다. Serena의 핵심은 `LSP(Language Server Protocol)`를 기반으로 한 **시맨틱 코드 분석**이다.

> 시맨틱 코드 분석은 코드를 단순한 문자열이 아니라 의미 단위(함수, 클래스, 변수, 모듈 등)와 그 관계를 이해하는 방식으로 분석하는 것이다. 즉, **특정 심볼이 어디에서 정의되고 어디에서 참조되는지, 어떤 호출 관계를 가지는지** 등을 파악함으로써 코드베이스 전반의 구조를 이해할 수 있게 한다.

AI 코딩 어시스턴트를 사용하다 보면 파일 전체를 붙여넣어야 하거나, 코드에서 원하는 함수를 찾지 못하는 상황이 종종 발생한다. Serena는 코드를 단순한 텍스트가 아닌, 실제 개발자처럼 구조적으로 이해하고 조작할 수 있도록 도와준다.

LSP는 원래 VSCode 같은 IDE가 `pylsp`, `tsserver`와 같은 언어별 분석 엔진과 소통하기 위해 만든 표준 규격이다. Serena는 이 프로토콜을 LLM 환경으로 가져와 AI 에이전트가 IDE 기능을 직접 활용할 수 있게 한다.

> 언어별 분석 엔진은 특정 프로그래밍 언어의 문법과 구조를 깊이 이해하고 코드 자동 완성, 오류 감지, 참조 찾기 등의 기능을 제공하는 별도의 백그라운드 프로세스이다.

쉽게 말해, 기존 AI 코딩 도구가 코드를 `grep`으로 검색한다면 Serena는 코드를 `AST` 수준에서 이해한다고 볼 수 있다.

> AST(Abstract Syntax Tree, 추상 구문 트리)는 소스 코드를 컴퓨터가 이해할 수 있도록 트리 형태의 자료구조로 표현한 것이다.

---

## 기존 방식의 한계

일반적인 LLM 코딩 워크플로우는 다음과 같은 문제를 가진다.

- 전체 파일을 컨텍스트로 전달해야 하므로 토큰 소모가 크다.
- 코드를 텍스트로만 인식하므로 심볼 간 관계를 파악하지 못해 정확도가 낮아진다.
- 대규모 코드베이스에서는 단순 텍스트 검색의 한계로 인해 원하는 심볼을 정확히 찾기 어렵다.

기존 에이전트가 텍스트를 분석하고 추측 기반으로 수정해 오류 가능성이 높았다면, Serena는 LSP를 통해 심볼 수준에서 코드를 분석하고 수정하므로 보다 정확하고 안전한 결과를 기대할 수 있다.

토큰 효율 측면에서도 차이가 크다. 전체 파일을 통째로 넘기는 대신 관련 심볼만 추출해 LLM에 전달하므로, 특히 대규모 코드베이스에서 불필요한 토큰 소모를 크게 줄일 수 있다.

## 핵심 기능

1. 함수, 클래스, 변수를 심볼 단위로 검색하고 전체 프로젝트 맥락에서 관계를 파악한다.
2. 코드를 의미 단위로 인식해 `replace_symbol_body`, `insert_after_symbol` 등으로 안전하게 수정한다.
3. `find_referencing_symbols`로 특정 함수나 클래스를 참조하는 모든 위치를 즉시 찾는다.
4. 프로젝트별 아키텍처 맥락, 규칙 등을 `.serena/memories`에 저장하고 세션 간 유지한다.

지원 언어는 Python, TypeScript/JavaScript, PHP, Go, Rust, C#, Java 등 주요 언어를 모두 포함한다.

---

## 설치 및 연결

먼저 `uv`를 설치한다.

```bash
brew install uv
```

Serena 레포지토리를 클론한다.

```bash
cd ~/work
git clone https://github.com/oraios/serena
cd serena

mkdir -p ~/.serena
cp src/serena/resources/serena_config.template.yml ~/.serena/serena_config.yml
```

### Claude Code

프로젝트 디렉토리로 이동 후 아래 명령어를 실행한다. `.claude.json`에 자동으로 MCP 설정이 추가된다.

```bash
claude mcp add serena -- uvx --from git+https://github.com/oraios/serena \ serena start-mcp-server --context ide-assistant --project $(pwd)
```

### Copilot

프로젝트 루트의 `.vscode` 에 아래 `mcp.json`을 추가한다.

```json
{
  "servers": {
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from",
        "git+https://github.com/oraios/serena",
        "serena",
        "start-mcp-server",
        "--context",
        "claude-code",
        "--project",
        "${workspaceFolder}"
      ]
    }
  }
}
```

### Antigravity

1. Antigravity 에이전트 패널 우측 상단 `...` 메뉴 클릭

2. `MCP Servers` → `Manage MCP Servers` → `View raw config` 클릭

3. `mcp_config.json` 파일이 열리면 아래 내용을 추가한다.

```json
{
  "mcpServers": {
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from",
        "git+https://github.com/oraios/serena",
        "serena",
        "start-mcp-server",
        "--context",
        "claude-code",
        "--project",
        "/프로젝트/절대경로"
      ]
    }
  }
}
```

## 프로젝트 온보딩

Claude Code 실행 후 `/mcp__serena__initial_instructions` 명령어를 입력하면 Serena가 프로젝트를 자동으로 분석한다.

이 명령은 새 세션 시작 시, 메모리 compacting 이후, 프로젝트 전환 시마다 실행해야 Serena를 온전히 활용할 수 있다.

## 생성되는 프로젝트 파일

Serena를 활성화하면 프로젝트 루트에 `.serena/` 폴더가 생성된다.

| 경로                    | 역할                                   |
| --------------------- | ------------------------------------ |
| `.serena/cache/`      | LSP 기반 심볼 인덱싱 정보를 캐싱하여 분석 속도를 높인다.   |
| `.serena/memories/`   | 프로젝트 아키텍처, 규칙 등 AI가 학습한 장기 지식을 보관한다. |
| `.serena/project.yml` | 프로젝트 경로, 상태, 메타데이터를 기록한 통합 명세 파일이다.  |
