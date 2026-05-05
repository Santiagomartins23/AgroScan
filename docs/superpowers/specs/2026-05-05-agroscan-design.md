# AgroScan — Design Spec
**Data:** 2026-05-05

---

## Visão Geral

Aplicação de diagnóstico de pragas e doenças em plantas via foto, com IA.
O agricultor envia uma imagem da planta e seleciona a cultura; o sistema retorna o diagnóstico
com nome da doença, severidade, nível de confiança e recomendações de tratamento.

Princípios que guiam as decisões do produto:

- **Precisão científica** — nível de confiança sempre visível; a IA instrída a nunca inflar certeza
- **Sustentabilidade** — defensivos só recomendados quando realmente necessário (severity `medium` ou `high`)
- **Acessibilidade** — linguagem simples, interface mobile-first, suporte às principais culturas brasileiras

---

## Stack

| Camada | Tecnologia |
|---|---|
| Backend | NestJS + TypeScript |
| Banco | PostgreSQL 16 + TypeORM |
| IA | Anthropic Claude API (Vision) |
| Upload | Multer — disco local (`./uploads/`) |
| Auth | JWT via passport-jwt |
| Frontend | React + Vite, CSS manual, mobile-first |

---

## Endpoints da API

### Auth (público)
```
POST /auth/register   body: { name, email, password }
POST /auth/login      body: { email, password } → { access_token }
```

### Crops (público)
```
GET /crops            → lista todas as culturas
GET /crops/:id
```

### Diagnoses (JWT obrigatório)
```
POST /diagnoses       multipart: { image: File, cropId: string }
GET  /diagnoses       ?page=1&limit=10
GET  /diagnoses/:id
GET  /diagnoses/stats → { total, needsTreatment, healthy, byCrop }
```

---

## Fluxo de Diagnóstico

```
1. Usuário envia imagem + cultura
2. Imagem salva em disco local
3. IA analisa e retorna JSON com doença, severidade, confiança e recomendações
4. Resultado salvo no banco vinculado ao usuário
5. Diagnóstico retornado completo
```

Se a IA retornar resposta inválida, o diagnóstico não é salvo e o sistema retorna erro claro.

---

## Frontend

**`/login`** — autenticação e registro

**`/upload`** — envio de imagem, seleção de cultura, exibição do resultado com badge de severidade, barra de confiança e recomendações (ou confirmação de planta saudável)

**`/history`** — histórico paginado com painel de stats: total analisado / precisou tratamento / saudável

Layout em coluna única, funcional no celular.

---

## Ordem de Implementação

1. Docker Compose + variáveis de ambiente
2. Backend scaffold NestJS
3. Autenticação (registro e login JWT)
4. Módulo de culturas + seed
5. Módulo de diagnóstico (storage → IA → banco)
6. Frontend (Login → Upload → History)
