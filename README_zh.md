<div align="center">

# bl-template-personal

**面向 [blcli](https://github.com/ggsrc/blcli) 的个人 GCP 平台模板** — 单项目、单集群、低配置，可按需开启完整可观测性栈。

*默认用 `kubectl` / `helm` 部署应用，不强制 ArgoCD。*

[![GitHub stars](https://img.shields.io/github/stars/ggsrc/bl-template-personal?style=flat-square)](https://github.com/ggsrc/bl-template-personal/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/ggsrc/bl-template-personal?style=flat-square)](https://github.com/ggsrc/bl-template-personal/network/members)
[![blcli](https://img.shields.io/badge/powered%20by-blcli-blue?style=flat-square)](https://github.com/ggsrc/blcli)
[![企业版模板](https://img.shields.io/badge/企业版-bl--template-orange?style=flat-square)](https://github.com/ggsrc/bl-template)

[快速开始](#推荐工作流) · [英文文档](README.md) · [PERSONAL_DEV.md](PERSONAL_DEV.md) · [Profiles](profiles/README.md)

</div>

<!-- ADOPTION:START -->
**采用情况：** 通过 GitHub Stars / Forks 观测 · 若本模板对你有帮助，欢迎 [Star](https://github.com/ggsrc/bl-template-personal)
<!-- ADOPTION:END -->

---

## 是什么？

[blcli](https://github.com/ggsrc/blcli) 的**个人 / 独立开发者**轻量模板。生成单个 GCP 项目 + GKE 集群基线，可选 DNS、证书、Istio 与监控栈，无 [bl-template](https://github.com/ggsrc/bl-template) 的多环境 Org 复杂度。

```
bl-template-personal  +  args.yaml  +  blcli  →  一个集群  →  kubectl / helm 部署应用
```

适合本地实验、AI 辅助改基础设施、控制 GCP 账单。

---

## 适合谁？

| 用户 | Profile | 场景 |
|------|---------|------|
| 个人开发者 | `minimal`（默认） | 生成与 check；最小账单 |
| 有真实域名 | `full` | DNS + 托管证书 + Istio + 监控 |
| 企业多环境 | — | 请用 [bl-template](https://github.com/ggsrc/bl-template) |

AI Agent 必改参数与常见误区见 [PERSONAL_DEV.md](./PERSONAL_DEV.md)。

---

## 为什么选个人版？

- **配置负担低** — `minimal` 仅核心 Terraform + sealed-secrets；需要时再 `--profile full`。
- **渐进复杂度** — 不换 blcli、不换协议，只换 profile。
- **默认无 ArgoCD** — 平台 `blcli apply kubernetes`；业务 `kubectl` / `helm`。
- **AI 友好** — 自描述 `args.yaml` + `blcli explain` + 文档化陷阱。

| bl-template（企业） | bl-template-personal（个人） |
|---|---|
| 多项目 corp / stg / prd | 单项目 + 单集群 |
| 含 ArgoCD GitOps | 默认不含 ArgoCD |
| Org 级 GCP | 个人账号（`OrganizationID: "0"`） |

---

## 谁在用？

欢迎补充案例。若你在实验或产品中使用本模板，请 [提 PR 编辑 README.md](https://github.com/ggsrc/bl-template-personal/edit/main/README.md)；列表会自动同步到本页。

<!-- ADOPTERS:START -->
<!-- Example:
- [Your Name](https://example.com) — solo GCP lab on blcli
-->
<!-- ADOPTERS:END -->

---

## Profile（推荐）

| Profile | 命令 | 说明 |
|---------|------|------|
| `minimal` | `blcli init-args -r github.com/ggsrc/bl-template-personal --org my-dev` | 默认；无 dns/cert；K8s 仅 sealed-secret |
| `full` | 加 `--profile full` | 增加 dns/cert、Istio、监控；**需真实域名** |

详见 [profiles/README.md](./profiles/README.md)。

---

## 推荐工作流

```bash
# 1. 生成 args
blcli init-args -r github.com/ggsrc/bl-template-personal --org my-dev -o workspace/config/args.yaml

# 2. 渲染配置（本地路径示例）
blcli init -r ./bl-template-personal -a workspace/config/args.yaml -o workspace/output/template-one -w

# 3. 检查
blcli check kubernetes -d workspace/output/template-one/kubernetes -r ./bl-template-personal

# 4. 部署（需真实 GCP 凭据与 Billing）
blcli apply terraform -d workspace/output/template-one/terraform --args workspace/config/args.yaml --project app
blcli apply kubernetes -d workspace/output/template-one/kubernetes
```

真实 `apply` 前请按 [PERSONAL_DEV.md](./PERSONAL_DEV.md) 修改 Billing ID、域名等。

---

## 文档导航

| 文档 | 说明 |
|------|------|
| [README.md](README.md) | 完整英文技术文档 |
| [PERSONAL_DEV.md](PERSONAL_DEV.md) | 个人开发者与 AI Agent 指南 |
| [profiles/README.md](profiles/README.md) | minimal / full 差异 |
| [blcli 文档](https://github.com/ggsrc/blcli) | CLI 安装与命令 |
