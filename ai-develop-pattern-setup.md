# Claude Code — Pattern de Setup para Sistemas Críticos

> Blueprint operacional para uso do Claude Code em sistemas de alta volumetria, performance e resiliência, com SDD, Skills e Multi-agents.

---

## O problema que este pattern resolve

Claude Code sem estrutura vira "vibe coding" em escala: o agente produz código coerente localmente mas deriva do intent conforme o projeto cresce. Em sistemas críticos — liquidação, antecipação de recebíveis, reconciliação — esse drift é inaceitável. O pattern abaixo trata o Claude Code como um **time de engenheiros coordenados por specs**, não como um autocomplete glorificado.

---

## Os três pilares do setup

### 1. SDD — A spec é o artefato primário

SDD (Spec-Driven Development) emergiu como resposta direta ao failure mode do "vibe coding" com LLMs — agentes que produzem código plausível que deriva do intent, alucinam APIs e decaem conforme o projeto escala. A ideia central: **a spec compila para código, não o contrário**.

O ciclo é: `Discuss → Plan → Execute → Verify → Ship`. Cada fase tem sua própria pesquisa, plano e execução.

#### Estrutura de diretórios do projeto

```
project-root/
├── CLAUDE.md                        # Constituição do projeto (hub)
├── STATUS.md                        # Estado atual de execução (escrito pelo agente)
├── docs/
│   └── specs/
│       ├── 00-overview.md           # Visão, metas, contexto de domínio
│       ├── 01-requirements.md       # Requisitos funcionais e não-funcionais
│       ├── 02-architecture.md       # Decisões arquiteturais, ADRs
│       ├── 03-domain-model.md       # Entidades, aggregates, linguagem ubíqua
│       └── 04-nfr-slos.md           # Latência, throughput, resiliência, compliance
├── changes/
│   └── YYYY-MM-DD-feature-name/
│       ├── SPEC.md                  # O quê (critérios de aceitação em Given/When/Then)
│       └── PLAN.md                  # Como (fases de implementação, waves, dependências)
├── .claude/
│   ├── settings.local.json          # Permissões explícitas de escrita
│   ├── skills/                      # Conhecimento especializado por domínio
│   │   ├── go-hexagonal.md
│   │   ├── payments-domain.md
│   │   ├── event-sourcing.md
│   │   ├── observability.md
│   │   ├── security-compliance.md
│   │   └── testing-strategy.md
│   ├── agents/                      # Configuração de subagents em YAML
│   │   ├── domain-implementer.yaml
│   │   ├── test-verifier.yaml
│   │   └── security-reviewer.yaml
│   └── rules/                       # Regras escopadas por glob (ex: *.go, *_test.go)
```

---

#### O CLAUDE.md — A constituição do projeto

O CLAUDE.md é carregado no system prompt no início de cada sessão e recarregado após compactação de contexto. O agente pode editar o arquivo conforme trabalha, tornando-o um documento vivo. Use estrutura **hub-and-spoke**: o CLAUDE.md principal indexa para arquivos mais profundos — não despeje tudo no topo.

**Template de CLAUDE.md para sistemas fintech/adquirência:**

```markdown
# [Nome do Sistema]

## Domínio
Adquirência / Antecipação de Recebíveis. Termos de domínio: EC, MDR, agenda de
recebíveis, liquidação D+1, chargeback, BACEN, sub-adquirente. Toda implementação
respeita a semântica de domínio definida em docs/specs/03-domain-model.md.

## Constraints inegociáveis
- NUNCA force push ou delete de branch sem confirmação explícita
- NUNCA alterar migrations já aplicadas — criar nova migration
- NUNCA remover idempotency key de operações financeiras
- NUNCA expor PII sem mascaramento (LGPD art. 46) — CPF, conta, PAN
- Toda operação de escrita em ledger é append-only, sem UPDATE ou DELETE
- NUNCA commitar segredos, tokens ou credenciais

## Stack
Go 1.23+, PostgreSQL 16, Kafka (MSK), Redis 7, OpenTelemetry, gRPC

## Padrões arquiteturais
- Hexagonal Architecture: domain puro em /internal/domain (zero deps de infra)
- CQRS: comandos em /internal/command, queries em /internal/query
- Event Sourcing para ledger: ver docs/specs/02-architecture.md#event-store
- Cell-based isolation para tenants de alta volumetria
- Idempotency keys obrigatórias em toda operação de escrita exposta via API

## Observabilidade (obrigatória em todo novo código)
- Span OpenTelemetry em toda operação cross-boundary
- Canonical log line em handlers: request_id, tenant_id, latency_ms, status, error_code
- Métricas de negócio: volume_financeiro_processado, taxa_erro_por_ec, p99_latencia

## Workflow de estado
- Ao iniciar uma sessão: ler STATUS.md antes de qualquer ação
- Ao finalizar uma wave: atualizar STATUS.md com progresso e próximos passos
- Commits atômicos por task — nunca agrupar tasks em um único commit

## Referências
- ADRs: docs/specs/02-architecture.md
- Domain model: docs/specs/03-domain-model.md
- SLOs e NFRs: docs/specs/04-nfr-slos.md
- Skills disponíveis: .claude/skills/
```

---

### 2. Skills — Conhecimento especializado por domínio

Skills são arquivos de instrução reutilizáveis que o agente carrega sob demanda. Não as coloque no CLAUDE.md principal — isso consumiria contexto em toda sessão, mesmo quando irrelevantes. Referencie-as explicitamente no PLAN.md de cada mudança.

#### Estrutura de uma skill

```markdown
# Skill: Event Sourcing em Go

## Quando usar esta skill
Ao implementar aggregates que precisam de auditoria completa ou replayability.
Requerida para qualquer toque no ledger de recebíveis.

## Padrão de aggregate

type ReceivableAggregate struct {
    id       ReceivableID
    version  int64
    changes  []DomainEvent
    // estado derivado dos eventos
    status   ReceivableStatus
    amount   Money
}

func (r *ReceivableAggregate) Anticipate(cmd AnticipateCommand) error {
    // validar invariants
    if r.status != StatusEligible {
        return ErrNotEligibleForAnticipation
    }
    // aplicar evento (nunca estado direto)
    r.apply(ReceivableAnticipated{
        ID:        r.id,
        Amount:    cmd.Amount,
        RequestedAt: cmd.RequestedAt,
    })
    return nil
}

## Invariants obrigatórios
- Evento deve ser imutável após persistência
- Sequence number é monotonicamente crescente por aggregate ID
- Snapshot a cada 100 eventos para performance de replay
- Nenhum command handler lê estado de outro aggregate

## Anti-padrões neste projeto
- NUNCA atualizar evento existente — append only
- NUNCA ler estado de outro aggregate dentro de um command handler
- NUNCA misturar lógica de query com command handler
```

#### Skills recomendadas para sistemas de pagamentos

| Skill | Conteúdo |
|---|---|
| `go-hexagonal.md` | Estrutura de pastas, regras de dependência, ports & adapters |
| `payments-domain.md` | Glossário, invariants financeiros, regras de negócio BR |
| `event-sourcing.md` | Aggregates, projections, snapshots, replay |
| `observability.md` | Template de canonical log line, naming de métricas, span attributes |
| `security-compliance.md` | LGPD, PCI-DSS, mascaramento de PAN/CPF, auditoria |
| `testing-strategy.md` | Pirâmide de testes, contract tests, chaos patterns, testcontainers |

---

### 3. Multi-Agents — Paralelismo com contexto isolado

Este é o multiplicador real de produtividade. O orchestrator despacha subagents em paralelo, cada um com contexto isolado (fresh context window de até 200K tokens). A sessão principal permanece lean enquanto os subagents fazem o trabalho pesado.

#### Execução em waves

Planos são agrupados por dependência em waves. Planos independentes rodam simultaneamente; planos dependentes esperam. Cada task recebe seu próprio commit atômico.

```
Wave 1 (paralelo):  domain-model | event-store-schema | api-contracts
Wave 2 (paralelo):  command-handlers | query-handlers
Wave 3 (sequencial): integration-tests → performance-benchmark
Wave 4 (gate):      security-review (humano)
```

#### Configuração de subagents em YAML

```yaml
# .claude/agents/domain-implementer.yaml
name: domain-implementer
model: claude-sonnet-4-6        # Sonnet para implementação de domínio
system_prompt: |
  Você é um engenheiro sênior especializado em domínio de adquirência.
  Implementa aggregates, command handlers e domain services seguindo
  hexagonal architecture e event sourcing.
  SEMPRE carregar skill @.claude/skills/event-sourcing.md antes de criar aggregates.
  NUNCA introduzir dependências de infraestrutura em /internal/domain.
  Ao finalizar: commit atômico com mensagem no formato feat(domain): <descrição>.
tools: [read_file, write_file, bash]
max_tokens: 8000
```

```yaml
# .claude/agents/test-verifier.yaml
name: test-verifier
model: claude-haiku-4-5         # Haiku para tarefas de verificação
system_prompt: |
  Verifica cobertura de testes e conformidade com a spec fornecida.
  Roda a suite de testes e reporta desvios como structured output em STATUS.md.
  Foco em: contract tests, casos de borda financeiros, invariants de domínio.
tools: [read_file, bash]
max_tokens: 4000
```

```yaml
# .claude/agents/security-reviewer.yaml
name: security-reviewer
model: claude-sonnet-4-6
system_prompt: |
  Revisa código para conformidade com LGPD, PCI-DSS e políticas internas.
  Verifica: exposição de PII, ausência de idempotency keys, SQL injection,
  secrets hardcoded, ausência de rate limiting em endpoints públicos.
  Reporta findings com severidade (CRITICAL | HIGH | MEDIUM | LOW).
tools: [read_file]
max_tokens: 6000
```

**Regra de custo:** codifique o modelo nos YAMLs e commite as configs no repositório. Isso evita que qualquer dev use Opus para tarefas que o Haiku resolve — enforced por configuração, não por força de vontade.

---

## O fluxo completo por feature

```
┌─────────────────────────────────────────────────────────────────┐
│  1. DISCUSS                                                      │
│     Você + orchestrator refinam a spec                           │
│     • Critérios de aceitação em Given/When/Then                  │
│     • Invariants de domínio explicitados                         │
│     • SLOs definidos (ex: P99 < 200ms, throughput: 5k TPS)      │
│     Output: SPEC.md em changes/YYYY-MM-DD-feature/              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. PLAN                                                         │
│     Orchestrator gera PLAN.md com tasks independentes            │
│     • Agrupa tasks em waves por dependência                      │
│     • Cada task referencia skills necessárias                    │
│     • Gate de revisão humana antes de executar                   │
│     Output: PLAN.md em changes/YYYY-MM-DD-feature/              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. EXECUTE                                                      │
│     Orchestrator despacha subagents em paralelo por wave         │
│     • Cada subagent: contexto isolado + commit atômico           │
│     • STATUS.md atualizado após cada wave                        │
│     • /clear entre waves para manter contexto do orchestrator    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. VERIFY                                                       │
│     test-verifier subagent roda suite completa                   │
│     • Checa conformidade spec vs. implementação                  │
│     • security-reviewer valida compliance                        │
│     • Desvios reportados para revisão humana                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. SHIP                                                         │
│     Revisão humana nos gate points críticos                      │
│     • Merge após aprovação no gate de segurança/compliance       │
│     • Deploy via pipeline com feature flag                       │
└─────────────────────────────────────────────────────────────────┘
```

#### Prompt de orquestração (exemplo real)

```
implement @changes/2026-05-30-receivable-anticipation/PLAN.md

use the Task tool for each implementation task
each task must be executed by a dedicated subagent with isolated context
load skill @.claude/skills/event-sourcing.md for all domain tasks
load skill @.claude/skills/observability.md for all tasks
load skill @.claude/skills/security-compliance.md for any handler touching PII
commit atomically after each task before proceeding to the next
you are the orchestrator — subagents are your engineers
update STATUS.md at the end of each wave
```

---

## Gates obrigatórios para sistemas críticos

Nenhuma ferramenta de SDD (Claude Code, Kiro, Cursor) oferece drift detection automático confiável. Os gates são humanos nos pontos certos.

| Gate | Trigger | Responsável | Critério de aprovação |
|---|---|---|---|
| **Pre-flight** | Antes de iniciar execução | Você revisa PLAN.md | Tasks claras, waves coerentes, skills referenciadas |
| **Domain invariants** | Após implementação de aggregates | Review do domain model | Invariants respeitados, sem deps de infra no domain |
| **Security/Compliance** | Qualquer toque em PII, auth, ledger | Obrigatório humano | Zero findings CRITICAL ou HIGH |
| **Performance** | Antes de merge em paths críticos | Benchmark vs. SLO | P99 dentro do budget definido em 04-nfr-slos.md |
| **Escalation** | Subagent reporta ambiguidade | Orchestrator → você | Spec atualizada para eliminar ambiguidade |

---

## Gestão de contexto

O contexto é o recurso mais escasso. A maioria das falhas em workflows longos é esgotamento ou poluição de contexto, não qualidade do modelo.

**Regras práticas:**

- **`/clear` entre tasks distintas.** Long-running agents acumulam histórico stale que é reenviado em cada turno. `/clear` entre tasks reduz custo por mensagem em 30–50%.
- **STATUS.md como memória persistente.** O orchestrator escreve estado ao final de cada wave. Na retomada de sessão, lê o STATUS.md antes de qualquer ação — nunca confia no histórico de conversa.
- **Skills sob demanda, não no CLAUDE.md.** Skills referenciadas no PLAN.md são carregadas pelo subagent apenas quando necessário.
- **Subagents não herdam contexto do orchestrator.** Isso é feature, não bug. Cada subagent começa limpo — passe só o necessário via referência ao arquivo de spec/skill.
- **Tamanho do CLAUDE.md.** Mantenha abaixo de 500 linhas. Acima disso, extraia em subdocs indexados.

---

## Anti-padrões que destroem sistemas críticos

**CLAUDE.md genérico.** Um CLAUDE.md com "use clean code" produz código genérico. Para adquirência, o CLAUDE.md precisa saber o que é uma agenda de recebíveis, o que é idempotency key num estorno, o que LGPD implica para CPF em logs.

**Um agente único para tudo.** Um agente com 50 arquivos de contexto produz código mediano. Dois subagents com 10 arquivos cada produzem código melhor, mais rápido, com contexto limpo.

**Sem gates de segurança/compliance.** Deixar o agente tocar diretamente em caminhos de autenticação, ledger ou dados sensíveis sem gate humano é risco inaceitável em fintech regulado.

**STATUS.md opcional.** Em workflows multi-sessão, o STATUS.md é a única memória persistente confiável. Sem ele, cada sessão começa do zero e o agente vai reinventar o que já foi feito.

**Spec escrita depois do código.** O agente vai tratar o código existente como a spec real e a spec escrita como comentário. A spec precisa existir antes de qualquer linha de implementação.

**Orquestração sem commit por task.** Sem commits atômicos por task, uma falha no meio da wave força retrabalho sem checkpoint. Cada task commitada é um ponto de retomada seguro.

---

## Configuração de permissões

```json
// .claude/settings.local.json
{
  "permissions": {
    "allow": [
      "Write(changes/**)",
      "Write(docs/specs/**)",
      "Write(STATUS.md)",
      "Write(.claude/skills/**)",
      "Edit(changes/**)",
      "Edit(docs/specs/**)",
      "Edit(STATUS.md)",
      "Bash(go build ./...)",
      "Bash(go test ./...)",
      "Bash(go vet ./...)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "deny": [
      "Write(.env*)",
      "Edit(.env*)",
      "Bash(git push *)",
      "Bash(git force*)",
      "Bash(git reset --hard *)",
      "Bash(rm -rf *)"
    ]
  }
}
```

---

## Setup mínimo para começar agora

```bash
# Criar estrutura inicial em qualquer projeto Go/fintech
mkdir -p docs/specs changes .claude/{skills,agents,rules}

# Arquivo de permissões explícitas
cat > .claude/settings.local.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Write(changes/**)", "Write(docs/specs/**)",
      "Write(STATUS.md)", "Edit(changes/**)",
      "Bash(go *)", "Bash(git add *)", "Bash(git commit *)"
    ],
    "deny": [
      "Write(.env*)", "Bash(git push *)", "Bash(git force*)"
    ]
  }
}
EOF

# STATUS.md inicial
cat > STATUS.md << 'EOF'
# Status do Projeto

## Fase atual
Configuração inicial — nenhuma feature em andamento.

## Próximos passos
1. Preencher docs/specs/00-overview.md com visão e contexto de domínio
2. Definir primeira feature em changes/

## Histórico de waves
(vazio — projeto iniciado)
EOF

# Primeira instrução ao Claude Code (copiar e colar no terminal)
# "Leia @CLAUDE.md e @docs/specs/00-overview.md,
#  depois gere o SPEC.md para [feature] em @changes/YYYY-MM-DD-[feature]/
#  usando critérios Given/When/Then com invariants de domínio explícitos.
#  Não escreva código ainda — apenas a spec para revisão."
```

---

## Resumo executivo

| Pilar | O que faz | Arquivo central |
|---|---|---|
| **SDD** | Garante que o código seja derivado da spec, não o contrário | `CLAUDE.md` + `changes/*/SPEC.md` |
| **Skills** | Injeta conhecimento especializado sem poluir o contexto base | `.claude/skills/*.md` |
| **Multi-agents** | Paraleliza execução com contexto isolado por task | `.claude/agents/*.yaml` + `Task tool` |
| **Gates** | Mantém controle humano nos pontos de maior risco | Revisão de PLAN.md + security gate |
| **STATUS.md** | Memória persistente entre sessões | `STATUS.md` (escrito pelo orchestrator) |

O pattern completo — CLAUDE.md robusto com domínio real + specs estruturadas + skills especializadas + multi-agent com waves + gates humanos nos pontos críticos — é o que separa um projeto que escala de um que vira dívida técnica após a terceira feature.
