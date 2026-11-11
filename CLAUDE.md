# CLAUDE.md

Chat de suporte SaaS multiempresa (triagem configurável, filas, distribuição). **Fase de especificação**: ainda não há código, só os documentos de arquitetura abaixo — eles são a fonte da verdade até existir implementação (e, depois, até existir OpenAPI gerado).

- `chat-suporte-arquitetura.md` — visão geral, decisões, multi-tenancy, módulos do backend, modelo de dados, motor de triagem, filas/distribuição, regras de atendimento, WebSocket, widget, auth, roadmap, riscos.
- `chat-suporte-frontend-plano.md` — stack do front, estrutura de pastas, rotas/permissões, contrato (tipos, endpoints REST, eventos WebSocket), diretrizes visuais, protótipos de tela, ordem de implementação, ligações (fase 2).

## Comunicação
- Português. Mínimo de palavras. Objetivo.
- Sem introdução, resumo repetido ou elogio. Resposta direta, detalhe só se necessário.
- Dúvida real → perguntar. Decisão óbvia → decidir e informar em uma linha.

## O que é / o que vai ser
Backend Java 21+/Spring Boot (monólito modular) + frontend Vite/React/TypeScript com duas entradas de build (`app` = console do atendente + admin, `widget` = iframe do cliente final). Multi-tenancy por **schema PostgreSQL por empresa** (`tenant_<slug>`). Quando o código começar, backend e frontend ganham repositórios, pipeline e deploy próprios (`chat-suporte-backend`, `chat-suporte-frontend`); este repositório (`apolo`) continua com a especificação e a automação de PR compartilhada.

## Decisões já tomadas
| Tema | Decisão |
|---|---|
| Backend | Java 21+ / Spring Boot, monólito modular (ver módulos em `chat-suporte-arquitetura.md` §5) |
| Banco | PostgreSQL, um schema por tenant + `public` para dados da plataforma |
| Front | Vite + React + TypeScript, um projeto com dois entry points (`app`, `widget`) |
| Tempo real | Spring WebSocket + STOMP; `@stomp/stompjs` no front |
| Editor de fluxo | React Flow (`@xyflow/react`), formato `nodes`/`edges` salvo direto como JSONB |
| Canais MVP | Widget web + integração com sistemas da empresa (token assinado); WhatsApp e ligações ficam para a fase 2 |
| Infra | Redis (presença/locks/cache), RabbitMQ opcional no MVP (relay STOMP ao escalar), S3/MinIO para anexos |

## Regras críticas (valem a partir do início da implementação)
- **Nome de schema por regex antes de qualquer SQL.** Nunca concatenar dado vindo do usuário em `SET search_path`.
- **`search_path` sempre resetado no `releaseConnection`.** É o maior risco do modelo de multi-tenancy — uma conexão devolvida ao pool com schema errado vaza dados entre empresas.
- **`TenantContext` limpo no `finally`** de toda requisição/mensagem STOMP/job. Código assíncrono (`@Async`, listeners) não herda o ThreadLocal — propagar explicitamente.
- **Migrações Flyway** idempotentes e compatíveis com a versão anterior (expand/contract). Falha num tenant não para os demais: marca `MIGRACAO_PENDENTE` e segue.
- **Distribuição de conversas sem duplicar nem estourar capacidade**: `FOR UPDATE SKIP LOCKED` na seleção da conversa, update condicional ou lock Redis na reserva do atendente.
- **WebSocket**: `ChannelInterceptor` valida tenant + permissão no `SUBSCRIBE`. Nunca confiar no destino que o cliente manda. Mensagem é persistida antes de publicada.
- **Segredo de integração nunca chega ao navegador.** Chamada à API da empresa (dados ao vivo do contato, integrações) sempre passa pelo backend.
- **Nunca commitar segredo** (`.env`, certificados, chaves, connection string com senha literal).
- **Widget é um bundle separado e pequeno** (< 80 KB gzip a página, < 5 KB o loader). `widget/` só importa de `shared/chat-ui`, `shared/api`, `shared/realtime` — nunca de `features/` (regra de lint de fronteiras).
- **Triagem**: só a versão `PUBLICADA` de um fluxo entra em conversas novas; conversa em andamento continua na versão em que começou. Validação de publicação é só no backend.

## Onde mexer (quando a implementação começar)
| Tarefa | Local (ver doc) |
|---|---|
| Tenancy, onboarding, resolução de tenant, migrações | módulo `tenancy` — arquitetura §4 |
| Usuários, papéis, JWT | módulo `identity` — arquitetura §12 |
| Widget, embed, WhatsApp (gateway de entrada/saída) | módulo `channel` — arquitetura §11 |
| Editor e motor de triagem | módulo `triage` — arquitetura §7 |
| Filas, estratégias de distribuição | módulo `routing` — arquitetura §8 |
| Conversas, mensagens, transferência, encerramento | módulo `conversation` — arquitetura §6, §9 |
| Presença, pausas, capacidade | módulo `agent` |
| Horários, SLA | módulos `schedule`, `sla` — arquitetura §9 |
| Relatórios | módulo `reporting` |
| Tipos, cliente da API, hooks de query/mutation | `shared/api` — frontend-plano §3, §6 |
| Cliente STOMP e sincronização de cache | `shared/realtime` — frontend-plano §6.3 |
| Console do atendente / admin / widget | `features/*` e `widget/` — frontend-plano §4 |
| Editor de fluxo (front) | `features/admin/flows` — frontend-plano §4, §8.2 |
| Protótipo de tela / fluxo de produto | frontend-plano §8 (console, admin) e §11 (ligações) |

## Código
- Sem código ainda. Ao implementar, seguir rigorosamente a arquitetura, o modelo de dados e o contrato (tipos/endpoints/eventos) já especificados — qualquer divergência é decisão nova, não um detalhe de implementação.
- Seguir o padrão do arquivo vizinho. KISS, YAGNI.
- Comentar o porquê, não o quê.
- Build verde (backend e/ou frontend, conforme o que a mudança tocar) antes de commit.

## Git (Claude decide e executa)
- Claude cria branch, commita, faz push e abre PR por conta própria. **Nunca perguntar se pode commitar** — commitar ao concluir cada mudança lógica e informar o hash em uma linha.
- Nunca commitar direto na `main`. Branch: `feat/`, `fix/`, `refactor/`, `chore/`, `docs/`, `ci/` + kebab-case curto.
- Conventional Commits em português (`feat(triagem): ...`, `docs(arquitetura): ...`). Commits pequenos, um por mudança lógica.
- PR via `gh pr create` contra `main`: título = conventional commit; corpo = o quê, por quê, como testar.
- Branch curta (1–3 dias), sempre a partir de `main` atualizada. Um assunto por PR.
- PR abre com a label `claude-merge` → fluxo automático: `Code Review` + `Build` → `claude-fix` aplica ajustes (máx. 2 rodadas) → `Claude Merge` faz squash e apaga a branch.
- `Claude Merge` exige: Build e Code Review verdes no head, nenhuma thread aberta, sem conflito. Tocou caminho de `.github/sensiveis.txt` (segredo, automação) → exige aprovação humana.
- Sem branch protection (plano free do GitHub): a regra é o fluxo acima. Nunca push direto na `main`.
- Merge manual, force-push e exclusão de branch remota fora do workflow: só com pedido explícito.
