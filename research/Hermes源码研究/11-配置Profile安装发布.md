---
created: '2026-04-30'
tags:
- hermes-agent
- config
- profiles
status: draft
---

# 配置、Profile、安装与发布源码解析

## 阅读目标

理解 Hermes 如何管理 `config.yaml`、`.env`、profile-aware home、日志、setup wizard、测试脚本和安装发布流程。

## 关键文件

- `hermes_cli/config.py`
- `hermes_constants.py`
- `hermes_logging.py`
- `hermes_cli/main.py`
- `hermes_cli/setup.py`
- `hermes_cli/profiles.py`
- `scripts/run_tests.sh`
- `scripts/release.py`
- `scripts/install.sh`
- `packaging/`
- `Dockerfile`
- `flake.nix`

## 关键原则

- 非 secret 配置进 `config.yaml`。
- API keys / tokens / passwords 才进 `.env`。
- 所有 Hermes home 路径用 `get_hermes_home()`。
- 用户可见路径用 `display_hermes_home()`。
- profile override 必须在其他模块 import 前完成。
- 测试必须通过 `scripts/run_tests.sh`，不要直接跑 `pytest`。

## 重点问题

- `load_cli_config()`、`load_config()`、Gateway raw YAML load 三条路径差异。
- `_config_version` 何时需要 bump。
- profile root 为什么 HOME-anchored 而不是 HERMES_HOME-anchored。
- setup wizard 如何写 config 和 env。
- release/install 脚本如何处理不同平台。

## 待补源码细读

- [ ] DEFAULT_CONFIG 结构
- [ ] config deep merge
- [ ] profile command 生命周期
- [ ] logging setup
- [ ] run_tests.sh CI parity 细节

