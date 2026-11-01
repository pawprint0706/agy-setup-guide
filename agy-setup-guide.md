# Antigravity CLI (`agy`) 상태표시줄(`statusLine`) 커스텀 구축 매뉴얼 (AI 에이전트용)

이 문서는 AI 에이전트가 **Google Antigravity CLI (`agy`)** 의 하단 상태표시줄(`statusLine`) 우측 끝에 **컨텍스트 사용량(`Ctx`)** 과 **현재 선택된 모델의 플랜 사용량(`Plan`)** 을 표시하도록 동일하게 재현하기 위한 단계별 가이드입니다.

---

## 1. 작업 원칙 및 주의사항

1. **작업 범위 제한**: 오직 Antigravity CLI (`agy`, `~/.gemini/antigravity-cli/`)만 수정합니다. 다른 하네스(`.claude`, `.codex` 등)는 절대 탐색하거나 수정하지 않습니다.
2. **바이너리 직접 패치 금지**: `agy.exe`(`%LOCALAPPDATA%\agy\bin\agy.exe`)는 Authenticode 서명이 적용되어 있으며 자동 업데이트 시 덮어써집니다. 반드시 `settings.json`의 공식 `statusLine` 커맨드 훅(`StatusLineConfig`)을 사용합니다.
3. **레이아웃 규칙**:
   - **좌측**: Vim 모드 표시기(`INSERT`, `VISUAL`, `V-LINE` 활성 시) + `? for shortcuts`
   - **우측**: 기존 우측 표시 항목들 뒤에 ` · ` 구분자로 `Ctx <N>% · Plan <M>%`를 **가장 오른쪽 끝(최우측)** 에 덧붙입니다.
     - 순서: `[실행 모드] · [모델 이름 · Effort] · [활성 subagent 수] · [백그라운드 task 수 · /tasks] · Ctx <N>% · Plan <M>%`
   - 좌측과 우측 사이는 `terminal_width - 2` 너비를 기준으로 공백 패딩하여 양쪽 정렬합니다.

---

## 2. `agy`의 `StatusLineData` 동작 원리 및 스키마

`agy`는 `~/.gemini/antigravity-cli/settings.json`에 `"statusLine"` 설정이 활성화되어 있으면, 상태표시줄을 갱신할 때마다 지정된 `command` 프로세스를 실행하고 `stdin`으로 아래 구조의 `StatusLineData` JSON을 전달합니다. 프로세스가 `stdout`으로 출력한 문자열(개행 제외)이 상태표시줄에 렌더링됩니다.

### 2.1 `settings.json` 내 `statusLine` 구조 (`CliSetting`)
```json
"statusLine": {
  "type": "command",
  "command": "C:/Users/<사용자명>/.gemini/antigravity-cli/bin/agy-statusline.exe",
  "enabled": true,
  "stack_with_default": false
}
```
- macOS의 경우 `"command"` 경로에 `/Users/<사용자명>/.gemini/antigravity-cli/bin/agy-statusline`을 지정합니다.
- `"stack_with_default": false`로 설정하면 기본 내장 상태표시줄 대신 커스텀 출력만 단일 줄로 표시됩니다.

### 2.2 `stdin`으로 전달되는 `StatusLineData` JSON 스키마
```json
{
  "agent": {
    "name": "string"
  },
  "context_window": {
    "total_input_tokens": 0,
    "total_output_tokens": 0,
    "context_window_size": 1048576,
    "current_usage": {
      "input_tokens": 0,
      "output_tokens": 0,
      "cache_creation_input_tokens": 0,
      "cache_read_input_tokens": 0
    },
    "used_percentage": 14.2,
    "remaining_percentage": 85.8
  },
  "cycle_mode": "auto",
  "model": {
    "id": "gemini-4-argon-medium",
    "display_name": "Gemini 4 Argon (Medium)",
    "effort": "medium"
  },
  "quota": {
    "gemini-argon-wtu-weekly": {
      "remaining_fraction": 0.752994,
      "remaining_amount": null,
      "reset_time": "2026-11-05T00:00:00Z",
      "reset_in_seconds": 555210,
      "disabled": false
    },
    "gemini-5h": { "remaining_fraction": 1.0 },
    "gemini-weekly": { "remaining_fraction": 0.990159 },
    "3p-5h": { "remaining_fraction": 0.995714 },
    "3p-weekly": { "remaining_fraction": 0.999286 }
  },
  "subagents": [
    { "name": "string", "role": "string", "status": "running" }
  ],
  "task_count": 0,
  "terminal_width": 120,
  "vim": {
    "mode": "NORMAL"
  }
}
```

### 2.3 모델별 플랜 쿼터 버킷(`quota`) 매핑 규칙
각 버킷의 사용률(%)은 `round((1.0 - remaining_fraction) * 100)` 으로 계산합니다.

| 현재 선택된 모델 (`model.id` / `model.display_name`) | 매핑할 `quota` 버킷 키 |
| :--- | :--- |
| **Gemini 4 Argon** (`argon` 포함) | `"gemini-argon-wtu-weekly"` (또는 `"argon"`이 포함된 활성 버킷) |
| **Claude / GPT 계열** (`claude`, `gpt`, `sonnet`, `opus`, `haiku` 포함) | `"3p-5h"`, `"3p-weekly"` 중 더 높은 사용률(%) |
| **기타 Gemini 계열** (`gemini-3.8-flash` 등 `gemini` 포함) | `"gemini-5h"`, `"gemini-weekly"` 중 더 높은 사용률(%) |

---

## 3. 단계별 구축 절차

### Step 1. Go 소스코드 작성 (`~/.gemini/antigravity-cli/bin/agy-statusline.go`)

빠른 실행 속도와 서브프로세스 오버헤드 최소화를 위해 네이티브 실행 파일로 빌드합니다. 아래 코드를 `~/.gemini/antigravity-cli/bin/agy-statusline.go`에 저장합니다.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"math"
	"os"
	"regexp"
	"strings"
	"unicode/utf8"
)

type StatusLineData struct {
	Agent *struct {
		Name string `json:"name"`
	} `json:"agent"`
	ContextWindow struct {
		TotalInputTokens  int `json:"total_input_tokens"`
		TotalOutputTokens int `json:"total_output_tokens"`
		ContextWindowSize int `json:"context_window_size"`
		CurrentUsage      *struct {
			InputTokens              int `json:"input_tokens"`
			OutputTokens             int `json:"output_tokens"`
			CacheCreationInputTokens int `json:"cache_creation_input_tokens"`
			CacheReadInputTokens     int `json:"cache_read_input_tokens"`
		} `json:"current_usage"`
		UsedPercentage      *float64 `json:"used_percentage"`
		RemainingPercentage *float64 `json:"remaining_percentage"`
	} `json:"context_window"`
	CycleMode string `json:"cycle_mode"`
	Model     struct {
		ID          string `json:"id"`
		DisplayName string `json:"display_name"`
		Effort      string `json:"effort"`
	} `json:"model"`
	Quota     map[string]QuotaBucket `json:"quota"`
	Subagents []struct {
		Name   string `json:"name"`
		Role   string `json:"role"`
		Status string `json:"status"`
	} `json:"subagents"`
	TaskCount     int `json:"task_count"`
	TerminalWidth int `json:"terminal_width"`
	Vim           *struct {
		Mode string `json:"mode"`
	} `json:"vim"`
}

type QuotaBucket struct {
	RemainingFraction *float64 `json:"remaining_fraction"`
	RemainingAmount   *int64   `json:"remaining_amount"`
	ResetTime         *string  `json:"reset_time"`
	ResetInSeconds    *int64   `json:"reset_in_seconds"`
	Disabled          bool     `json:"disabled"`
}

var (
	effortSuffixRe = regexp.MustCompile(`(?i)\s*\((low|medium|high|xhigh|max)\)\s*$`)
	ansiRe         = regexp.MustCompile(`\x1b\[[0-9;]*m`)
)

const (
	dimStart = "\x1b[2m"
	reset    = "\x1b[0m"
)

func dim(s string) string {
	return dimStart + s + reset
}

func formatEffort(e string) string {
	switch strings.ToLower(strings.TrimSpace(e)) {
	case "low":
		return "Low"
	case "medium":
		return "Medium"
	case "high":
		return "High"
	case "xhigh":
		return "XHigh"
	case "max":
		return "Max"
	default:
		if len(e) == 0 {
			return ""
		}
		return strings.ToUpper(e[:1]) + strings.ToLower(e[1:])
	}
}

func renderModeLabel(mode string) string {
	switch mode {
	case "plan":
		return "\x1b[36mplan\x1b[0m"
	case "auto":
		return "\x1b[33mauto\x1b[0m"
	case "accept-edits":
		return "\x1b[32maccept-edits\x1b[0m"
	case "best-of-n":
		return "\x1b[35mbest-of-n\x1b[0m"
	default:
		return dim(mode)
	}
}

func renderModelWithBadge(data *StatusLineData) string {
	name := strings.TrimSpace(data.Model.DisplayName)
	if name == "" {
		name = strings.TrimSpace(data.Model.ID)
	}
	if name == "" {
		return ""
	}
	baseName := effortSuffixRe.ReplaceAllString(name, "")
	effort := formatEffort(data.Model.Effort)
	if effort == "" {
		if m := effortSuffixRe.FindStringSubmatch(name); len(m) > 1 {
			effort = formatEffort(m[1])
		}
	}

	var modelStr string
	if effort != "" {
		modelStr = dim(baseName + " · " + effort)
	} else {
		modelStr = dim(baseName)
	}

	if data.Agent != nil && strings.TrimSpace(data.Agent.Name) != "" {
		return dim(fmt.Sprintf("%s (%s)  /agents", strings.TrimSpace(data.Agent.Name), stripANSI(modelStr)))
	}
	return modelStr
}

func computeContextPercent(data *StatusLineData) int {
	if data.ContextWindow.UsedPercentage != nil {
		pct := int(math.Round(*data.ContextWindow.UsedPercentage))
		if pct < 0 {
			return 0
		}
		if pct > 100 {
			return 100
		}
		return pct
	}
	if data.ContextWindow.RemainingPercentage != nil {
		pct := int(math.Round(100.0 - *data.ContextWindow.RemainingPercentage))
		if pct < 0 {
			return 0
		}
		if pct > 100 {
			return 100
		}
		return pct
	}
	if data.ContextWindow.ContextWindowSize > 0 {
		used := 0
		if data.ContextWindow.CurrentUsage != nil {
			u := data.ContextWindow.CurrentUsage
			used = u.InputTokens + u.OutputTokens + u.CacheCreationInputTokens + u.CacheReadInputTokens
		}
		if used == 0 {
			used = data.ContextWindow.TotalInputTokens + data.ContextWindow.TotalOutputTokens
		}
		pct := int(math.Round(float64(used) * 100.0 / float64(data.ContextWindow.ContextWindowSize)))
		if pct < 0 {
			return 0
		}
		if pct > 100 {
			return 100
		}
		return pct
	}
	return 0
}

func bucketUsedPercent(b QuotaBucket) (int, bool) {
	if b.Disabled || b.RemainingFraction == nil {
		return 0, false
	}
	used := (1.0 - *b.RemainingFraction) * 100.0
	pct := int(math.Round(used))
	if pct < 0 {
		pct = 0
	}
	if pct > 100 {
		pct = 100
	}
	return pct, true
}

func computePlanPercent(data *StatusLineData) (int, bool) {
	if len(data.Quota) == 0 {
		return 0, false
	}

	modelKey := strings.ToLower(data.Model.ID + " " + data.Model.DisplayName)

	// 1. Gemini Argon 모델 -> gemini-argon-wtu-weekly (또는 argon 포함 버킷)
	if strings.Contains(modelKey, "argon") {
		if b, ok := data.Quota["gemini-argon-wtu-weekly"]; ok {
			if pct, valid := bucketUsedPercent(b); valid {
				return pct, true
			}
		}
		for k, b := range data.Quota {
			if strings.Contains(strings.ToLower(k), "argon") {
				if pct, valid := bucketUsedPercent(b); valid {
					return pct, true
				}
			}
		}
	}

	// 2. Claude / GPT (3p) 모델 -> 3p-5h / 3p-weekly
	if strings.Contains(modelKey, "claude") || strings.Contains(modelKey, "gpt") || strings.Contains(modelKey, "sonnet") || strings.Contains(modelKey, "opus") || strings.Contains(modelKey, "haiku") {
		return pickGroupPercent(data.Quota, "3p-5h", "3p-weekly")
	}

	// 3. 기타 Gemini 모델 -> gemini-5h / gemini-weekly
	if strings.Contains(modelKey, "gemini") {
		if pct, ok := pickGroupPercent(data.Quota, "gemini-5h", "gemini-weekly"); ok {
			return pct, true
		}
	}

	// Fallback
	for _, key := range []string{"gemini-argon-wtu-weekly", "gemini-5h", "gemini-weekly", "3p-5h", "3p-weekly"} {
		if b, ok := data.Quota[key]; ok {
			if pct, valid := bucketUsedPercent(b); valid {
				return pct, true
			}
		}
	}
	return 0, false
}

func pickGroupPercent(quota map[string]QuotaBucket, fiveHourKey, weeklyKey string) (int, bool) {
	b5, ok5 := quota[fiveHourKey]
	p5, v5 := 0, false
	if ok5 {
		p5, v5 = bucketUsedPercent(b5)
	}

	bw, okw := quota[weeklyKey]
	pw, vw := 0, false
	if okw {
		pw, vw = bucketUsedPercent(bw)
	}

	if v5 && vw {
		if pw > p5 {
			return pw, true
		}
		return p5, true
	}
	if v5 {
		return p5, true
	}
	if vw {
		return pw, true
	}
	return 0, false
}

func stripANSI(s string) string {
	return ansiRe.ReplaceAllString(s, "")
}

func visibleWidth(s string) int {
	return utf8.RuneCountInString(stripANSI(s))
}

func renderLeft(data *StatusLineData) string {
	var parts []string
	if data.Vim != nil {
		switch data.Vim.Mode {
		case "INSERT":
			parts = append(parts, "\x1b[32m-- INSERT --\x1b[0m")
		case "VISUAL":
			parts = append(parts, "\x1b[35m-- VISUAL --\x1b[0m")
		case "VISUAL LINE":
			parts = append(parts, "\x1b[35m-- V-LINE --\x1b[0m")
		}
	}
	parts = append(parts, dim("? for shortcuts"))
	return strings.Join(parts, "  ")
}

func renderRight(data *StatusLineData) string {
	var items []string

	// 1. 실행 모드 표시기
	if m := strings.TrimSpace(data.CycleMode); m != "" && m != "default" && m != "normal" {
		items = append(items, renderModeLabel(m))
	}

	// 2. 모델 이름 + Effort 표시기
	if modelBadge := renderModelWithBadge(data); modelBadge != "" {
		items = append(items, modelBadge)
	}

	// 3. 실행 중인 서브에이전트 수
	runningSubagents := 0
	for _, sa := range data.Subagents {
		st := strings.ToLower(strings.TrimSpace(sa.Status))
		if st == "running" || st == "working" || st == "initializing" {
			runningSubagents++
		}
	}
	if runningSubagents > 0 {
		items = append(items, dim(fmt.Sprintf("%d subagent(s)", runningSubagents)))
	}

	// 4. 백그라운드 태스크 수
	if data.TaskCount > 0 {
		items = append(items, dim(fmt.Sprintf("%d task(s) · /tasks", data.TaskCount)))
	}

	// 5. 최우측: 컨텍스트 사용량(Ctx) 및 현재 모델 플랜 사용량(Plan)
	ctxPct := computeContextPercent(data)
	items = append(items, dim(fmt.Sprintf("Ctx %d%%", ctxPct)))

	if planPct, ok := computePlanPercent(data); ok {
		items = append(items, dim(fmt.Sprintf("Plan %d%%", planPct)))
	} else {
		items = append(items, dim("Plan --%"))
	}

	sep := dim(" · ")
	return strings.Join(items, sep)
}

func main() {
	raw, err := io.ReadAll(os.Stdin)
	if err != nil || len(strings.TrimSpace(string(raw))) == 0 {
		return
	}

	var data StatusLineData
	if err := json.Unmarshal(raw, &data); err != nil {
		return
	}

	left := renderLeft(&data)
	right := renderRight(&data)

	width := data.TerminalWidth - 2
	leftW := visibleWidth(left)
	rightW := visibleWidth(right)

	if width > leftW+rightW+2 {
		spaces := width - leftW - rightW
		fmt.Print(left + strings.Repeat(" ", spaces) + right)
	} else {
		fmt.Print(left + "  " + right)
	}
}
```

---

### Step 2. 바이너리 컴파일

#### Windows (PowerShell)

PowerShell에서 아래 명령으로 실행 파일을 빌드합니다(Go가 없을 경우 Python/Node 스크립트로 대체 가능하지만 Go 단일 실행 파일을 권장합니다).

```powershell
$binDir = "$HOME\.gemini\antigravity-cli\bin"
New-Item -ItemType Directory -Force -Path $binDir | Out-Null
Set-Location $binDir
if (-not (Test-Path go.mod)) { go mod init agy-statusline }
go build -ldflags="-s -w" -o "$binDir\agy-statusline.exe" "$binDir\agy-statusline.go"
```

#### macOS (`zsh` / `bash`)

macOS에서는 `.exe` 확장자 없이 네이티브 실행 파일(`darwin/arm64` 또는 `darwin/amd64`)로 빌드합니다. `agy` 내부 샌드박스에서 빌드할 경우 `GOCACHE=/tmp/go-cache`를 지정합니다.

```zsh
BIN_DIR="$HOME/.gemini/antigravity-cli/bin"
mkdir -p "$BIN_DIR"
GOCACHE=/tmp/go-cache go build -ldflags="-s -w" -o "$BIN_DIR/agy-statusline" "$BIN_DIR/agy-statusline.go"
chmod +x "$BIN_DIR/agy-statusline"
```

---

### Step 3. `~/.gemini/antigravity-cli/settings.json` 업데이트

기존 설정(`model`, `trustedWorkspaces` 등)을 유지한 채 `"statusLine"` 필드를 추가합니다. 경로 구분자는 슬래시(`/`)를 사용합니다.

#### Windows 예시
```json
{
  "model": "Gemini 4 Argon (Medium)",
  "statusLine": {
    "type": "command",
    "command": "C:/Users/suhoimage/.gemini/antigravity-cli/bin/agy-statusline.exe",
    "enabled": true,
    "stack_with_default": false
  },
  "trustedWorkspaces": [
    "C:\\Users\\suhoimage",
    "D:\\DEV\\PP\\rdp-key-swap-for-mac"
  ]
}
```

#### macOS 예시
```json
{
  "model": "Gemini 4 Argon (Medium)",
  "statusLine": {
    "type": "command",
    "command": "/Users/<사용자명>/.gemini/antigravity-cli/bin/agy-statusline",
    "enabled": true,
    "stack_with_default": false
  }
}
```

> **macOS / AI 에이전트 작업 시 주의사항 (`System Protection Boundary`)**
> `agy` 내부에서 동작하는 AI 에이전트는 시스템 보호 경계 규칙(`Matches hardcoded system protection boundary rule`)에 의해 `~/.gemini/antigravity-cli/settings.json`을 직접 읽거나 수정할 수 없습니다. 따라서 에이전트는 워크스페이스 내에 `agy-statusline.go`, 빌드된 `agy-statusline` 바이너리, 그리고 설치 스크립트(`install-macos.sh`)를 생성·검증한 뒤 사용자가 터미널에서 `install-macos.sh`를 1회 실행하도록 안내합니다.

#### macOS 원클릭 설치 스크립트 (`install-macos.sh`)
```zsh
#!/usr/bin/env zsh
set -euo pipefail

SCRIPT_DIR="${0:A:h}"
BIN_DIR="$HOME/.gemini/antigravity-cli/bin"
SETTINGS_FILE="$HOME/.gemini/antigravity-cli/settings.json"
TARGET_BIN="$BIN_DIR/agy-statusline"

mkdir -p "$BIN_DIR"
cp "$SCRIPT_DIR/agy-statusline.go" "$BIN_DIR/agy-statusline.go"
cp "$SCRIPT_DIR/agy-statusline" "$TARGET_BIN"
chmod +x "$TARGET_BIN"

python3 - "$SETTINGS_FILE" "$TARGET_BIN" <<'PY'
import json
import os
import sys

settings_path, target_bin = sys.argv[1], sys.argv[2]
data = {}
if os.path.exists(settings_path):
    with open(settings_path, "r", encoding="utf-8") as f:
        content = f.read().strip()
        if content:
            data = json.loads(content)

data["statusLine"] = {
    "type": "command",
    "command": target_bin,
    "enabled": True,
    "stack_with_default": False,
}

with open(settings_path, "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
    f.write("\n")

print(f"Updated {settings_path} successfully.")
PY
```

---

### Step 4. 동작 검증

샘플 페이로드를 파이프로 전달해 정상 출력 여부를 검증합니다.

#### Windows (PowerShell)
```powershell
$sample = '{"model":{"id":"gemini-4-argon-medium","display_name":"Gemini 4 Argon (Medium)","effort":"medium"},"context_window":{"used_percentage":14.2,"context_window_size":1048576},"quota":{"gemini-argon-wtu-weekly":{"remaining_fraction":0.752994}},"cycle_mode":"auto","task_count":1,"terminal_width":110}'
$sample | & "$HOME\.gemini\antigravity-cli\bin\agy-statusline.exe"
```

#### macOS (`zsh` / `bash`)
```zsh
printf '%s' '{"model":{"id":"gemini-4-argon-medium","display_name":"Gemini 4 Argon (Medium)","effort":"medium"},"context_window":{"used_percentage":14.2,"context_window_size":1048576},"quota":{"gemini-argon-wtu-weekly":{"remaining_fraction":0.752994}},"cycle_mode":"auto","task_count":1,"terminal_width":110}' \
  | "$HOME/.gemini/antigravity-cli/bin/agy-statusline" && echo ""
```

**기대 출력:**
```text
? for shortcuts                     auto · Gemini 4 Argon · Medium · 1 task(s) · /tasks · Ctx 14% · Plan 25%
```

실제 `agy` 세션에서는 `/statusline on`을 입력하거나 `agy`를 재시작하면 즉시 반영됩니다.
