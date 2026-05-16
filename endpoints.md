# Kube News — Documentação Técnica

## Visão Geral

Portal de notícias em Node.js projetado como aplicação de demonstração para containers e Kubernetes. Funcionalidade simples, mas com gaps relevantes para qualquer ambiente que se aproxime de produção.

---

## Stack

| Camada | Tecnologia | Versão |
|--------|-----------|--------|
| Runtime | Node.js | — |
| Framework | Express | 4.18.1 |
| Templates | EJS | 3.1.7 |
| ORM | Sequelize | 6.19.0 |
| Banco | PostgreSQL | — |
| Métricas | prom-client + express-prom-bundle | 14 / 6.4 |

---

## Arquitetura

```
                        ┌─────────────────────────────┐
                        │         Express App          │
                        │           :8080              │
                        │                              │
          ┌─────────────┤  middleware.js               │
          │             │  (contador de requisições)   │
          │             │                              │
     Usuário            │  server.js (rotas web + API) │
          │             │  system-life.js (health)     │
          │             └──────────────┬───────────────┘
          │                            │
          │                    Sequelize ORM
          │                    alter: true no boot ⚠️
          │                            │
          │                   ┌────────▼────────┐
          └──────────────────▶│   PostgreSQL     │
                              │   porta 5432     │
                              └─────────────────┘
```

---

## Modelo de Dados

Entidade única: **Post**

| Campo | Tipo | Limite | Observação |
|-------|------|--------|-----------|
| `title` | String | 30 chars | validado só na rota web |
| `summary` | String | 50 chars | validado só na rota web |
| `content` | String | 2000 chars | validado só na rota web |
| `publishDate` | DateOnly | — | sem timezone |

A rota `POST /api/post` **não valida nenhum campo** — insere direto no banco.

---

## Configuração via Variáveis de Ambiente

| Variável | Padrão | Risco |
|----------|--------|-------|
| `DB_HOST` | `localhost` | — |
| `DB_PORT` | `5432` | — |
| `DB_DATABASE` | `kubedevnews` | — |
| `DB_USERNAME` | `kubedevnews` | — |
| `DB_PASSWORD` | `Pg#123` | **senha exposta no código** |
| `DB_SSL_REQUIRE` | `false` | SSL desligado por padrão |

---

## Mapa de Riscos (visão SRE)

| Severidade | Risco |
|-----------|-------|
| CRÍTICO | Senha hardcoded no repositório |
| CRÍTICO | `/unhealth` e `/unreadyfor` sem autenticação |
| CRÍTICO | `sync({ alter: true })` no boot em produção |
| ALTO | Sem tratamento de erros nas rotas (crash total) |
| ALTO | Sem graceful shutdown / SIGTERM handler |
| ALTO | Sem rate limiting na API |
| ALTO | Logs sem estrutura (`console.log` puro) |
| MÉDIO | Pool de conexões não configurado |
| MÉDIO | Métrica `http_requests_total` duplicada |
| MÉDIO | Sem HTTPS nativo |
| MÉDIO | Sem tracing distribuído |
| BAIXO | Sem testes automatizados |
| BAIXO | `publishDate` sem timezone |

---

## O Que Está Bem

- Probes de liveness e readiness implementados — boa base para Kubernetes.
- Exposição de métricas Prometheus já integrada.
- Configuração via variáveis de ambiente — compatível com 12-factor app.
- Suporte a SSL no banco quando habilitado.

---

## Endpoints

### GET /
**Descricao** Lista todas as postagens e renderiza a página inicial
**Parametros** nenhum
**Retorno** página HTML com todas as notícias (`index.ejs`)
**Codigos HTTP:** 200 (sucesso), 500 (erro interno)

---

### GET /post
**Descricao** Exibe o formulário para criação de uma nova notícia
**Parametros** nenhum
**Retorno** página HTML com formulário vazio (`edit-news.ejs`)
**Codigos HTTP:** 200 (sucesso)

---

### POST /post
**Descricao** Salva uma nova notícia com validação dos campos
**Parametros** body (form-urlencoded): `title` (max 30 chars), `resumo` (max 50 chars), `description` (max 2000 chars)
**Retorno** redireciona para `/` em caso de sucesso, ou reexibe o formulário com erro se inválido
**Codigos HTTP:** 302 (sucesso, redirect), 200 (dados inválidos, reexibe formulário), 500 (erro interno)

---

### GET /post/:id
**Descricao** Exibe uma notícia específica pelo seu ID
**Parametros** path: `id` (identificador da postagem)
**Retorno** página HTML com o conteúdo da notícia (`view-news.ejs`)
**Codigos HTTP:** 200 (sucesso), 500 (erro interno ou ID não encontrado)

---

### POST /api/post
**Descricao** Cria múltiplas postagens em lote via API
**Parametros** body (JSON): `{ "artigos": [{ "title", "resumo", "description" }] }`
**Retorno** array JSON com os artigos criados
**Codigos HTTP:** 200 (sucesso), 500 (erro interno)

---

### GET /health
**Descricao** Verifica se a aplicação está em execução (liveness probe)
**Parametros** nenhum
**Retorno** JSON `{ "state": "up", "machine": "<hostname>" }`
**Codigos HTTP:** 200 (aplicação saudável), 500 (aplicação marcada como unhealthy)

---

### GET /ready
**Descricao** Verifica se a aplicação está pronta para receber tráfego (readiness probe)
**Parametros** nenhum
**Retorno** texto `Ok`
**Codigos HTTP:** 200 (pronta), 500 (não pronta)

---

### PUT /unhealth
**Descricao** Força a aplicação a retornar 500 em todas as requisições subsequentes
**Parametros** nenhum
**Retorno** texto `OK`
**Codigos HTTP:** 200 (comando aplicado)

---

### PUT /unreadyfor/:seconds
**Descricao** Marca a aplicação como "not ready" por um período determinado
**Parametros** path: `seconds` (número de segundos para ficar indisponível)
**Retorno** texto `OK`
**Codigos HTTP:** 200 (comando aplicado)

---

### GET /metrics
**Descricao** Expõe métricas da aplicação no formato Prometheus (gerado automaticamente pelo `express-prom-bundle`)
**Parametros** nenhum
**Retorno** texto no formato Prometheus com métricas de requisições, método, path, status e métricas padrão do Node.js
**Codigos HTTP:** 200 (sucesso)
