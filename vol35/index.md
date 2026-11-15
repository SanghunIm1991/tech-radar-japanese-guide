# Technology Radar Vol.35（2026年11月）の非公式の解説・索引

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../README.md) を参照してください。

- 公式のページ: https://www.thoughtworks.com/radar
- 項目数: 117（Techniques 45、Platforms 22、Tools 36、Languages & Frameworks 14）
- 確認日: 2026-11-08

## テーマ

この版には4つのテーマが付いています（題名だけを載せています。要旨はのちほど追記します）。

- Harness engineering at scale
- Beyond frontier models
- Putting more agents to work
- Constructing the AI factory

## 象限×リングごとの技術の数

「—」の分類に置かれた技術はありません。

| 象限 ＼ リング | Adopt | Trial | Assess | Caution |
|---|---|---|---|---|
| Techniques | 6 | 17 | 12 | 10 |
| Tools | 3 | 16 | 17 | — |
| Platforms | 1 | 10 | 11 | — |
| Languages & Frameworks | 1 | 6 | 7 | — |

## 書いた技術

技術ごとに、図と動きで見せる**図解ページ**（HTML）と、出典つきの**詳しい解説**（Markdown）があります。まず図解ページを開き、詳しく知りたいときや出典を確かめたいときに詳しい解説を読んでください。図解ページはまだ一部の技術だけで、作ったものから「—」を置き換えます。図解ページは、このサイトで開くと表示されます（GitHub のリポジトリで開くと HTML のソースが表示されます）。

| 技術 | 図解ページ | 詳しい解説 | 象限 | リング | 状態 |
|---|---|---|---|---|---|
| Agent sandboxing | [図と動きで見る](techniques/agent-sandboxing-intro.html) | [Markdown](techniques/agent-sandboxing.md) | Techniques | Adopt | 下書き |
| Mutation testing | [図と動きで見る](techniques/mutation-testing-intro.html) | [Markdown](techniques/mutation-testing.md) | Techniques | Adopt | 下書き |
| Path-to-production mapping | [図と動きで見る](techniques/path-to-production-mapping-intro.html) | [Markdown](techniques/path-to-production-mapping.md) | Techniques | Adopt | 下書き |
| Post-quantum cryptography | [図と動きで見る](techniques/post-quantum-cryptography-intro.html) | [Markdown](techniques/post-quantum-cryptography.md) | Techniques | Adopt | 下書き |
| Progressive context disclosure | [図と動きで見る](techniques/progressive-context-disclosure-intro.html) | [Markdown](techniques/progressive-context-disclosure.md) | Techniques | Adopt | 下書き |
| Threat modeling | — | [Markdown](techniques/threat-modeling.md) | Techniques | Adopt | 下書き |
| Agent plugin marketplace | — | [Markdown](techniques/agent-plugin-marketplace.md) | Techniques | Trial | 下書き |
| Agent-maintained knowledge bases | — | [Markdown](techniques/agent-maintained-knowledge-bases.md) | Techniques | Trial | 下書き |
| Anchoring coding agents to a reference application | — | [Markdown](techniques/anchoring-coding-agents-to-a-reference-application.md) | Techniques | Trial | 下書き |
| Coding harness for general-purpose agents | — | [Markdown](techniques/coding-harness-for-general-purpose-agents.md) | Techniques | Trial | 下書き |
| Feedback flywheel | — | [Markdown](techniques/feedback-flywheel.md) | Techniques | Trial | 下書き |
| Graph-grounded codebase reasoning | — | [Markdown](techniques/graph-grounded-codebase-reasoning.md) | Techniques | Trial | 下書き |
| Grilling | — | [Markdown](techniques/grilling.md) | Techniques | Trial | 下書き |
| Interactive visual plans with agents | — | [Markdown](techniques/interactive-visual-plans-with-agents.md) | Techniques | Trial | 下書き |
| Managed agents runtime | — | [Markdown](techniques/managed-agents-runtime.md) | Techniques | Trial | 下書き |
| Open-weight models | — | [Markdown](techniques/open-weight-models.md) | Techniques | Trial | 下書き |
| Risk-tiered human oversight | — | [Markdown](techniques/risk-tiered-human-oversight.md) | Techniques | Trial | 下書き |
| Self-improving harness | — | [Markdown](techniques/self-improving-harness.md) | Techniques | Trial | 下書き |
| Small language models | — | [Markdown](techniques/small-language-models.md) | Techniques | Trial | 下書き |
| Stateful guardrails | — | [Markdown](techniques/stateful-guardrails.md) | Techniques | Trial | 下書き |
| Subagents for better context management | — | [Markdown](techniques/subagents-for-better-context-management.md) | Techniques | Trial | 下書き |
| Toxic flow analysis for AI | — | [Markdown](techniques/toxic-flow-analysis-for-ai.md) | Techniques | Trial | 下書き |
| Unattended overnight agentic builds | — | [Markdown](techniques/unattended-overnight-agentic-builds.md) | Techniques | Trial | 下書き |
| mise | — | [Markdown](tools/mise.md) | Tools | Adopt | 下書き |
| OpenAI Codex | — | [Markdown](tools/openai-codex.md) | Tools | Adopt | 下書き |
| OpenCode | — | [Markdown](tools/opencode.md) | Tools | Adopt | 下書き |
| Act | — | [Markdown](tools/act.md) | Tools | Trial | 下書き |
| Agent Package Manager | — | [Markdown](tools/agent-package-manager.md) | Tools | Trial | 下書き |
| ast-grep | — | [Markdown](tools/ast-grep.md) | Tools | Trial | 下書き |
| Beads | — | [Markdown](tools/beads.md) | Tools | Trial | 下書き |
| Chaos Mesh | — | [Markdown](tools/chaos-mesh.md) | Tools | Trial | 下書き |
| Cube Core | — | [Markdown](tools/cube-core.md) | Tools | Trial | 下書き |
| Figma Code Connect | — | [Markdown](tools/figma-code-connect.md) | Tools | Trial | 下書き |
| FluxMarkdown | — | [Markdown](tools/fluxmarkdown.md) | Tools | Trial | 下書き |
| Git AI | — | [Markdown](tools/git-ai.md) | Tools | Trial | 下書き |
| Linear | — | [Markdown](tools/linear.md) | Tools | Trial | 下書き |
| mattpocock/skills | — | [Markdown](tools/mattpocock-skills.md) | Tools | Trial | 下書き |
| OpenSpec | — | [Markdown](tools/openspec.md) | Tools | Trial | 下書き |
| OpenWiki | — | [Markdown](tools/openwiki.md) | Tools | Trial | 下書き |
| Pi | — | [Markdown](tools/pi.md) | Tools | Trial | 下書き |
| Sourcebot | — | [Markdown](tools/sourcebot.md) | Tools | Trial | 下書き |
| Timefold Solver | — | [Markdown](tools/timefold-solver.md) | Tools | Trial | 下書き |
| VictoriaMetrics | [図と動きで見る](platforms/victoriametrics-intro.html) | [Markdown](platforms/victoriametrics.md) | Platforms | Adopt | 下書き |
| Kafka share groups | — | [Markdown](platforms/kafka-share-groups.md) | Platforms | Trial | 下書き |
| KServe | — | [Markdown](platforms/kserve.md) | Platforms | Trial | 下書き |
| LibreChat | — | [Markdown](platforms/librechat.md) | Platforms | Trial | 下書き |
| LiteLLM | — | [Markdown](platforms/litellm.md) | Platforms | Trial | 下書き |
| Microsoft Copilot Retrieval API | — | [Markdown](platforms/microsoft-copilot-retrieval-api.md) | Platforms | Trial | 下書き |
| Restate | — | [Markdown](platforms/restate.md) | Platforms | Trial | 下書き |
| Talos Linux | — | [Markdown](platforms/talos-linux.md) | Platforms | Trial | 下書き |
| Temporal | — | [Markdown](platforms/temporal.md) | Platforms | Trial | 下書き |
| vCluster | — | [Markdown](platforms/vcluster.md) | Platforms | Trial | 下書き |
| VictoriaLogs | — | [Markdown](platforms/victorialogs.md) | Platforms | Trial | 下書き |
| Open Data Contract Standard (ODCS) | — | [Markdown](languages-and-frameworks/open-data-contract-standard-odcs.md) | Languages & Frameworks | Adopt | 下書き |
