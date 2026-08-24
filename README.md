# Prisma ORM + PostgreSQL 🐘


> Guia completo sobre o Prisma ORM, escrito com base na versão **7 (linha estável desde novembro de 2025, atualmente na 7.8.x+)**, com o **Client Rust-free** (100% TypeScript, gerado como ESM), o novo `prisma.config.ts`, driver adapters e ligação direta a uma base de dados **PostgreSQL** dentro de uma app **Next.js 16**. O Prisma 8 já existe em early access como reescrita completa em TypeScript — este guia foca-se no que está estável e pronto para produção hoje.


---


## Índice


1. [O que é o Prisma e porque usar um ORM](#1-o-que-é-o-prisma-e-porque-usar-um-orm)

2. [Pré-requisitos e instalação](#2-pré-requisitos-e-instalação)

3. [Inicializar o Prisma num projeto Next.js](#3-inicializar-o-prisma-num-projeto-nextjs)

4. [O Prisma Schema: datasource, generator e models](#4-o-prisma-schema-datasource-generator-e-models)

5. [Modelagem de dados: tipos, relações, enums e índices](#5-modelagem-de-dados-tipos-relações-enums-e-índices)

6. [Migrations: criar, aplicar e evoluir a base de dados](#6-migrations-criar-aplicar-e-evoluir-a-base-de-dados)

7. [Gerar o Prisma Client](#7-gerar-o-prisma-client)

8. [Conectar ao PostgreSQL: connection string, singleton e driver adapters](#8-conectar-ao-postgresql-connection-string-singleton-e-driver-adapters)

9. [Queries CRUD básicas](#9-queries-crud-básicas)

10. [Relações: include, select e nested writes](#10-relações-include-select-e-nested-writes)

11. [Filtros, paginação e ordenação](#11-filtros-paginação-e-ordenação)

12. [Transactions](#12-transactions)

13. [Prisma no Next.js: Server Components, Server Actions e Route Handlers](#13-prisma-no-nextjs-server-components-server-actions-e-route-handlers)

14. [Connection pooling em produção e serverless](#14-connection-pooling-em-produção-e-serverless)

15. [Seed de dados](#15-seed-de-dados)

16. [Prisma Studio](#16-prisma-studio)

17. [Erros comuns e como tratá-los](#17-erros-comuns-e-como-tratá-los)

18. [Arquitetura e boas práticas](#18-arquitetura-e-boas-práticas)

19. [Roadmap: como te tornares expert](#19-roadmap-como-te-tornares-expert)


---


## 1. O que é o Prisma e porque usar um ORM


O Prisma é um ORM (*Object-Relational Mapper*) para Node.js e TypeScript. Em vez de escreveres SQL cru espalhado pelo código, defines o teu modelo de dados num único ficheiro (`schema.prisma`) e o Prisma gera, a partir dele, um cliente 100% tipado para consultares a base de dados.


| Problema sem ORM | Solução do Prisma |
|---|---|
| SQL cru espalhado pelo código, difícil de manter | Schema único, declarativo, como fonte da verdade |
| Sem autocomplete nem checagem de tipos nas queries | Client gerado, totalmente tipado a partir do teu schema |
| Migrations feitas à mão, arriscadas | `prisma migrate` versiona e aplica alterações de forma segura |
| Erros de runtime por typos em nomes de colunas | Erros apanhados em tempo de compilação (TypeScript) |
| Difícil visualizar os dados | Prisma Studio — GUI para explorar e editar dados |


> 💡 **Dica**: o Prisma não substitui o teu conhecimento de SQL — ele traduz as tuas queries para SQL otimizado por baixo dos panos. Perceber SQL continua a ajudar-te a ler o `EXPLAIN ANALYZE` quando uma query fica lenta.


**Regra de ouro**: o `schema.prisma` é a fonte única da verdade do teu modelo de dados — muda o schema primeiro, gera a migration a seguir, nunca ao contrário.


---


## 2. Pré-requisitos e instalação


| Requisito | Versão mínima |
|---|---|
| Node.js | 20.19 LTS (recomendado 22 LTS) |
| PostgreSQL | 14+ (recomendado 16 ou 17) |
| Gestor de pacotes | pnpm (preferido), npm ou yarn |
| TypeScript | 5.1+ |

```bash
pnpm add prisma --save-dev
pnpm add @prisma/client
```


> ⚠️ **Atenção**: desde a versão 7, o Prisma passou a exigir versões mais recentes de Node e TypeScript do que as versões 6.x. Se estiveres a atualizar um projeto antigo, confirma as versões instaladas antes de correres `prisma init`.


Se ainda não tens uma base de dados PostgreSQL, tens três caminhos comuns:


| Opção | Perfil |
|---|---|
| **PostgreSQL local** (Docker ou instalação nativa) | Controlo total, ideal para aprender e para dev |
| **Prisma Postgres** | Gerido pela própria Prisma, pooling de ligações incluído, integração nativa |
| **Neon / Supabase / Vercel Postgres** | Serviços geridos populares, todos compatíveis via `postgresql://` |


Para correr PostgreSQL localmente com Docker:

```bash
docker run --name meu-postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres:17
```


---


## 3. Inicializar o Prisma num projeto Next.js


Dentro do teu projeto Next.js já existente:

```bash
npx prisma init --db
```

Este comando cria:


| Ficheiro/Pasta | Função |
|---|---|
| `prisma/schema.prisma` | O schema com a ligação à base de dados e os teus models |
| `.env` | Variáveis de ambiente, incluindo `DATABASE_URL` |
| `prisma.config.ts` | Configuração do Prisma (novidade da versão 7) |


O `prisma.config.ts` gerado tem este aspeto:

```ts
import "dotenv/config"
import { defineConfig, env } from "prisma/config"

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
})
```


> 💡 **Dica**: antes da versão 7, esta configuração vivia espalhada entre o `schema.prisma` e o `package.json`. Agora tens um único ficheiro TypeScript, o que significa que podes usar lógica dinâmica (ex: `dotenv`, condicionais por ambiente) para decidir a `datasource url`.


No `.env`, define a tua connection string:

```bash
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/meu_projeto?schema=public"
```


> ⚠️ **Atenção**: nunca comitas o `.env` no Git. Adiciona-o ao `.gitignore` (o Next.js já o faz por defeito) e usa um `.env.example` sem valores reais para documentar as variáveis que o projeto precisa.


**Regra de ouro**: o `prisma init` não te obriga a nada — podes correr `npx prisma init` num projeto Next.js já criado, sem interferir com o `app/` ou o `next.config.ts`.


---


## 4. O Prisma Schema: datasource, generator e models


O `schema.prisma` tem sempre três partes:

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client"
  output   = "../app/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model Produto {
  id    Int     @id @default(autoincrement())
  nome  String
  preco Decimal
}
```


| Bloco | Função |
|---|---|
| `generator client` | Diz ao Prisma para gerar o Client TypeScript, e onde colocar o código gerado |
| `datasource db` | Diz qual a base de dados (aqui, `postgresql`) e de onde vem a connection string |
| `model` | Cada `model` mapeia para uma tabela na base de dados |


> ⚠️ **Atenção**: desde a versão 7, o `provider` do generator passou de `"prisma-client-js"` para `"prisma-client"`. É o Client Rust-free, gerado como ESM diretamente dentro do teu código-fonte (ex: `app/generated/prisma`) em vez de ir parar ao `node_modules`. Isto significa que o `output` **deixou de ser opcional na prática** — define sempre um caminho dentro do teu projeto.

> 💡 **Dica**: gerar o Client dentro de `app/generated/prisma` (em vez de escondido no `node_modules`) significa que o teu editor, o teu linter e as tuas ferramentas de build passam a "ver" o código gerado como parte normal do projeto. Adiciona essa pasta ao `.gitignore` — ela é recriada a cada `prisma generate`.


**Regra de ouro**: o `generator` decide *como* o código é gerado, a `datasource` decide *onde* os dados vivem, e os `model` decidem *o que* existe. Nunca misturas estas três responsabilidades no mesmo bloco.


---


## 5. Modelagem de dados: tipos, relações, enums e índices


### Tipos de campo comuns


| Tipo Prisma | Equivalente PostgreSQL |
|---|---|
| `String` | `text` / `varchar` |
| `Int` | `integer` |
| `BigInt` | `bigint` |
| `Float` | `double precision` |
| `Decimal` | `decimal` (usa para dinheiro, nunca `Float`) |
| `Boolean` | `boolean` |
| `DateTime` | `timestamp` |
| `Json` | `jsonb` |

```prisma
model Produto {
  id          Int      @id @default(autoincrement())
  nome        String
  descricao   String?
  preco       Decimal  @db.Decimal(10, 2)
  emStock     Boolean  @default(true)
  criadoEm    DateTime @default(now())
  atualizadoEm DateTime @updatedAt
}
```


> ⚠️ **Atenção**: nunca uses `Float` para dinheiro — erros de arredondamento em ponto flutuante acumulam-se. Usa sempre `Decimal` com `@db.Decimal(precisão, escala)`.


### Relações

```prisma
model Utilizador {
  id     Int     @id @default(autoincrement())
  email  String  @unique
  nome   String?
  posts  Post[]
}

model Post {
  id           Int        @id @default(autoincrement())
  titulo       String
  conteudo     String?
  publicado    Boolean    @default(false)
  autor        Utilizador @relation(fields: [autorId], references: [id])
  autorId      Int
}
```


| Tipo de relação | Como modelar |
|---|---|
| 1:N (um utilizador, vários posts) | Campo de lista no lado "um" (`posts Post[]`), chave estrangeira no lado "muitos" |
| 1:1 (um utilizador, um perfil) | `@relation` com `@unique` na chave estrangeira |
| N:N (posts e tags) | Tabela implícita (`Tag[]` dos dois lados) ou explícita com um model intermédio |

```prisma
model Post {
  id   Int   @id @default(autoincrement())
  tags Tag[]
}

model Tag {
  id    Int    @id @default(autoincrement())
  nome  String @unique
  posts Post[]
}
```


> 💡 **Dica**: usa a relação N:N implícita (como acima) quando não precisas de guardar dados extra na relação. Se precisares (ex: "data em que a tag foi adicionada"), cria um model intermédio explícito com os dois `@relation`.


### Enums

```prisma
enum Estado {
  RASCUNHO
  PUBLICADO
  ARQUIVADO
}

model Post {
  id     Int    @id @default(autoincrement())
  titulo String
  estado Estado @default(RASCUNHO)
}
```


### Índices e constraints

```prisma
model Produto {
  id       Int    @id @default(autoincrement())
  sku      String @unique
  categoria String

  @@index([categoria])
  @@unique([sku, categoria])
}
```


| Diretiva | Uso |
|---|---|
| `@unique` | Constraint de unicidade num único campo |
| `@@unique([...])` | Constraint de unicidade composta (vários campos) |
| `@@index([...])` | Índice para acelerar queries que filtram por esses campos |
| `@@id([...])` | Chave primária composta |


**Regra de ouro**: modela primeiro pensando nas perguntas que o teu negócio vai fazer aos dados ("quais os posts publicados deste utilizador?") — os índices e relações certos nascem dessas perguntas, não o contrário.


---


## 6. Migrations: criar, aplicar e evoluir a base de dados


Depois de escreveres ou alterares um `model`, cria e aplica uma migration:

```bash
npx prisma migrate dev --name criar_produto
```

Este comando:


| Passo | O que faz |
|---|---|
| 1 | Compara o `schema.prisma` com o estado atual da base de dados |
| 2 | Gera um ficheiro SQL em `prisma/migrations/<timestamp>_criar_produto/` |
| 3 | Aplica essa migration na tua base de dados de desenvolvimento |
| 4 | Corre `prisma generate` automaticamente, atualizando o Client |


> 💡 **Dica**: dá sempre nomes descritivos às migrations (`--name adicionar_campo_estado`, não `--name update1`) — daqui a uns meses vais agradecer-te a ti próprio ao leres o histórico em `prisma/migrations/`.


Para aplicar migrations já criadas num ambiente de produção (sem gerar novas, sem prompts interativos):

```bash
npx prisma migrate deploy
```


> ⚠️ **Atenção**: `migrate dev` é só para desenvolvimento — pode resetar a base de dados se detetar drift. Em produção usa sempre `migrate deploy`, tipicamente como parte do teu pipeline de CI/CD, nunca à mão num servidor.


### Baseline num projeto com base de dados já existente


Se já tens uma base de dados PostgreSQL com tabelas e estás a introduzir o Prisma agora:

```bash
npx prisma db pull
```

Isto faz *introspection* — lê a tua base de dados e gera os `model` correspondentes automaticamente no `schema.prisma`. Depois, "baseline" a migration inicial para o Prisma saber que essas tabelas já existem:

```bash
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > prisma/migrations/0_init/migration.sql
```

**Regra de ouro**: nunca editas ficheiros de migration já aplicados em produção. Se precisares de corrigir algo, cria uma nova migration — o histórico de migrations é um registo, não um rascunho.


---


## 7. Gerar o Prisma Client

```bash
npx prisma generate
```

Este comando lê o `schema.prisma` e gera o código TypeScript do Client no caminho definido em `output`. Desde a versão 7, este Client:


| Característica | Detalhe |
|---|---|
| Rust-free | Sem binários nativos — tudo em TypeScript/WebAssembly |
| ESM-first | Compatível nativamente com `import`/`export`, sem CommonJS |
| Gerado no teu código-fonte | Vive dentro do projeto (ex: `app/generated/prisma`), não no `node_modules` |
| Mais leve e mais rápido | Bundle até 90% mais pequeno, execução de queries até 3× mais rápida (face à era pré-7) |

```tsx
import { PrismaClient } from "@/app/generated/prisma/client"

const prisma = new PrismaClient()

const produtos = await prisma.produto.findMany()
```


> 💡 **Dica**: por ser Rust-free e mais leve, o Client da versão 7 lida muito melhor com ambientes serverless e edge (Vercel, Cloudflare Workers) do que as versões anteriores, que dependiam de um binário nativo pesado.

> ⚠️ **Atenção**: sempre que alteras o `schema.prisma` sem correr `migrate dev` (ex: só estás a testar algo), lembra-te de correr `npx prisma generate` manualmente — caso contrário o Client fica desatualizado face ao schema.


**Regra de ouro**: o Client é sempre um artefacto gerado — nunca o editas à mão. Se precisares de comportamento extra, usa Client Extensions em vez de mexeres no código gerado.


---


## 8. Conectar ao PostgreSQL: connection string, singleton e driver adapters


### Connection string

```bash
DATABASE_URL="postgresql://utilizador:password@host:5432/nome_da_bd?schema=public"
```


| Parte | Significado |
|---|---|
| `utilizador:password` | Credenciais de acesso |
| `host:5432` | Endereço e porta do servidor PostgreSQL |
| `nome_da_bd` | Nome da base de dados |
| `?schema=public` | Schema do PostgreSQL a usar (opcional, `public` por defeito) |


### Singleton do Prisma Client (essencial no Next.js)


Em desenvolvimento, o Next.js recarrega módulos a cada alteração (Fast Refresh). Sem cuidado, isso cria uma instância nova de `PrismaClient` a cada reload, esgotando as ligações disponíveis à base de dados.

```ts
// lib/prisma.ts
import { PrismaClient } from "@/app/generated/prisma/client"

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma
}
```


> ⚠️ **Atenção**: importa sempre `prisma` a partir deste ficheiro singleton (`@/lib/prisma`) em todo o projeto. Nunca faças `new PrismaClient()` diretamente dentro de uma rota ou componente — isso reintroduz o problema que o singleton resolve.


### Driver adapters


Para ambientes edge, para usares um pool de ligações gerido externamente (ex: `pg.Pool`), ou para tirares partido de otimizações específicas de cada base de dados, o Prisma usa *driver adapters*:

```bash
pnpm add @prisma/adapter-pg pg
```

```ts
// lib/prisma.ts
import "dotenv/config"
import { PrismaPg } from "@prisma/adapter-pg"
import { PrismaClient } from "@/app/generated/prisma/client"

const connectionString = process.env.DATABASE_URL as string
const adapter = new PrismaPg({ connectionString })

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter })

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma
}
```


| | Sem driver adapter | Com driver adapter (`@prisma/adapter-pg`) |
|---|---|---|
| Uso típico | Node.js tradicional, deploy num servidor persistente | Serverless, edge, ou quando precisas de controlar o pool tu próprio |
| Configuração | `new PrismaClient()` simples | `new PrismaClient({ adapter })` |
| Controlo do pool | Gerido internamente pelo Prisma | Controlas o `pg.Pool` diretamente |


**Regra de ouro**: começa sempre com o Client simples (sem adapter) — só introduzes um driver adapter quando tiveres um motivo concreto (deploy edge, necessidade de controlar o pool manualmente, integração com `@vercel/functions`).


---


## 9. Queries CRUD básicas

```ts
// Criar
const novoProduto = await prisma.produto.create({
  data: {
    nome: "Teclado mecânico",
    preco: 45000,
  },
})

// Ler (vários)
const produtos = await prisma.produto.findMany()

// Ler (um, por chave única)
const produto = await prisma.produto.findUnique({
  where: { id: 1 },
})

// Atualizar
const produtoAtualizado = await prisma.produto.update({
  where: { id: 1 },
  data: { preco: 42000 },
})

// Apagar
await prisma.produto.delete({
  where: { id: 1 },
})
```


| Método | Uso |
|---|---|
| `create` | Insere um novo registo |
| `createMany` | Insere vários registos de uma vez |
| `findMany` | Lê vários registos, com filtros opcionais |
| `findUnique` | Lê um registo pela chave única (`id`, `email`, etc.) |
| `findFirst` | Lê o primeiro registo que corresponda a um filtro (não precisa de campo único) |
| `update` | Atualiza um registo existente |
| `updateMany` | Atualiza vários registos que correspondam a um filtro |
| `delete` | Apaga um registo |
| `upsert` | Cria se não existir, atualiza se já existir |


> 💡 **Dica**: usa `findUnique` sempre que estiveres a procurar por um campo `@id` ou `@unique` — o Prisma otimiza essa query de forma diferente de um `findFirst`, e o TypeScript avisa-te em tempo de compilação se o campo não for de facto único.


**Regra de ouro**: cada método do Client é assíncrono — usa sempre `await`, e trata os erros (secção 17) em vez de deixares a Promise rejeitar silenciosamente.


---


## 10. Relações: include, select e nested writes


### Buscar dados relacionados com `include`

```ts
const utilizadorComPosts = await prisma.utilizador.findUnique({
  where: { id: 1 },
  include: { posts: true },
})
```


### Escolher só os campos que precisas com `select`

```ts
const posts = await prisma.post.findMany({
  select: {
    titulo: true,
    autor: {
      select: { nome: true },
    },
  },
})
```


> ⚠️ **Atenção**: `include` e `select` são mutuamente exclusivos no mesmo nível de uma query — ou pedes tudo mais as relações que queres (`include`), ou escolhes exatamente os campos que precisas (`select`). Misturar os dois no mesmo nível dá erro de tipo.


### Nested writes: criar registo com relação numa única query

```ts
const novoPost = await prisma.post.create({
  data: {
    titulo: "Como aprender Prisma",
    autor: {
      connect: { id: 1 },
    },
    tags: {
      connectOrCreate: [
        { where: { nome: "prisma" }, create: { nome: "prisma" } },
        { where: { nome: "postgresql" }, create: { nome: "postgresql" } },
      ],
    },
  },
})
```


| Operação de relação | Uso |
|---|---|
| `connect` | Liga a um registo já existente |
| `create` | Cria um novo registo relacionado, ao mesmo tempo |
| `connectOrCreate` | Liga se existir, cria se não existir |
| `disconnect` | Remove a ligação (não apaga o registo) |
| `set` | Substitui toda a lista de relações por uma nova |


**Regra de ouro**: `select` é sempre a opção mais eficiente para produção — só traz da base de dados o que vais mesmo usar. Reserva `include` para quando genuinamente precisas de todos os campos da relação.


---


## 11. Filtros, paginação e ordenação

```ts
const produtos = await prisma.produto.findMany({
  where: {
    preco: { gte: 10000, lte: 50000 },
    nome: { contains: "teclado", mode: "insensitive" },
  },
  orderBy: { criadoEm: "desc" },
  skip: 0,
  take: 20,
})
```


| Filtro | Significado |
|---|---|
| `equals` | Igual a |
| `not` | Diferente de |
| `in` / `notIn` | Está / não está numa lista de valores |
| `lt` / `lte` / `gt` / `gte` | Menor que / menor ou igual / maior que / maior ou igual |
| `contains` / `startsWith` / `endsWith` | Correspondência parcial de texto |
| `mode: "insensitive"` | Torna a comparação de texto insensível a maiúsculas/minúsculas |


### Combinar condições

```ts
const resultados = await prisma.produto.findMany({
  where: {
    AND: [
      { emStock: true },
      { OR: [{ categoria: "eletronica" }, { categoria: "informatica" }] },
    ],
  },
})
```


### Paginação baseada em cursor (recomendada para listas grandes)

```ts
const proximaPagina = await prisma.produto.findMany({
  take: 20,
  skip: 1,
  cursor: { id: ultimoIdDaPaginaAnterior },
  orderBy: { id: "asc" },
})
```


> 💡 **Dica**: paginação por `skip`/`take` simples é suficiente para listas pequenas e páginas de admin. Para feeds grandes e infinite scroll, prefere paginação por cursor — evita o custo crescente de um `OFFSET` grande em tabelas com muitas linhas.


**Regra de ouro**: nunca faças `findMany()` sem `take` numa tabela que pode crescer sem limite — define sempre um teto razoável, mesmo que seja alto.


---


## 12. Transactions


Quando várias operações precisam de acontecer todas juntas, ou nenhuma:

```ts
const [produtoAtualizado, movimento] = await prisma.$transaction([
  prisma.produto.update({
    where: { id: 1 },
    data: { emStock: false },
  }),
  prisma.movimentoStock.create({
    data: { produtoId: 1, tipo: "SAIDA", quantidade: 1 },
  }),
])
```

Para lógica mais complexa (condicionais entre passos), usa a forma interativa:

```ts
const resultado = await prisma.$transaction(async (tx) => {
  const produto = await tx.produto.findUnique({ where: { id: 1 } })

  if (!produto || produto.preco.lessThan(0)) {
    throw new Error("Produto inválido para esta operação")
  }

  return tx.produto.update({
    where: { id: 1 },
    data: { preco: produto.preco.mul(0.9) },
  })
})
```


> ⚠️ **Atenção**: dentro de uma transaction interativa, usa sempre o `tx` que recebes como argumento (não o `prisma` global) — caso contrário as queries corretas fora da transaction, e o atomic rollback deixa de funcionar como esperado.


**Regra de ouro**: usa a forma em array (`$transaction([...])`) sempre que conseguires — é mais rápida e mais simples. Só passa para a forma interativa (`async (tx) => {...}`) quando precisares mesmo de lógica condicional entre os passos.


---


## 13. Prisma no Next.js: Server Components, Server Actions e Route Handlers


### Em Server Components

```tsx
// app/produtos/page.tsx
import { prisma } from "@/lib/prisma"

export default async function ProdutosPage() {
  const produtos = await prisma.produto.findMany({
    orderBy: { criadoEm: "desc" },
  })

  return (
    <ul>
      {produtos.map((produto) => (
        <li key={produto.id}>{produto.nome}</li>
      ))}
    </ul>
  )
}
```


### Em Server Actions

```tsx
// app/actions/produtos.ts
"use server"

import { prisma } from "@/lib/prisma"
import { updateTag } from "next/cache"

export async function criarProduto(formData: FormData) {
  const nome = formData.get("nome") as string
  const preco = formData.get("preco") as string

  await prisma.produto.create({
    data: { nome, preco: Number(preco) },
  })

  updateTag("produtos")
}
```


### Em Route Handlers (`/api`)

```tsx
// app/api/produtos/route.ts
import { NextRequest, NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"

export async function GET() {
  const produtos = await prisma.produto.findMany()
  return NextResponse.json(produtos)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const produto = await prisma.produto.create({ data: body })
  return NextResponse.json(produto, { status: 201 })
}
```


> ⚠️ **Atenção**: nunca importes `prisma` dentro de um Client Component (`"use client"`). O Client do Prisma só corre em Node.js/edge no servidor — se tentares importá-lo num componente cliente, o build falha, e bem, porque essa dependência nunca deveria chegar ao browser.


**Regra de ouro**: em Server Actions e Route Handlers que fazem `create`/`update`/`delete`, valida sempre o input (ex: com Zod, como no guia de Next.js) antes de passares os dados ao Prisma — o Prisma protege-te de SQL injection, mas não de dados de negócio inválidos.


---


## 14. Connection pooling em produção e serverless


Em ambientes serverless (Vercel, AWS Lambda), cada invocação de função pode abrir uma ligação nova à base de dados. Sob carga, isso esgota rapidamente o limite de ligações do PostgreSQL.


| Estratégia | Quando usar |
|---|---|
| **Prisma Postgres** | Pooling incluído de fábrica, zero configuração extra — recomendado para novos projetos |
| **PgBouncer** (via Neon, Supabase, etc.) | Já vem com o provider — usa a *pooled connection string* (geralmente porta `6543` ou com `?pgbouncer=true`) |
| **`directUrl`** no schema | Necessário quando usas um pooler — as migrations precisam de uma ligação direta, sem pooling |

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")     // ligação com pooling, usada pelo Client
  directUrl = env("DIRECT_URL")       // ligação direta, usada pelas migrations
}
```

Para Vercel Fluid Compute, evita ligações penduradas quando uma função é suspensa:

```ts
import { Pool } from "pg"
import { attachDatabasePool } from "@vercel/functions"
import { PrismaPg } from "@prisma/adapter-pg"
import { PrismaClient } from "@/app/generated/prisma/client"

const pool = new Pool({ connectionString: process.env.DATABASE_URL })
attachDatabasePool(pool)

export const prisma = new PrismaClient({ adapter: new PrismaPg(pool) })
```


> 💡 **Dica**: se usares Prisma Postgres, este problema praticamente desaparece — o pooling já vem incluído e a mesma `DATABASE_URL` funciona em dev local, preview deployments e produção sem configuração adicional.


**Regra de ouro**: se vais fazer deploy num ambiente serverless, decide a tua estratégia de pooling **antes** de escreveres a primeira query — corrigir isto depois de "esgotamento de ligações" em produção é sempre mais doloroso que configurar bem desde o início.


---


## 15. Seed de dados


Cria um script de seed:

```ts
// prisma/seed.ts
import { PrismaClient } from "../app/generated/prisma/client"

const prisma = new PrismaClient()

async function main() {
  await prisma.produto.createMany({
    data: [
      { nome: "Teclado mecânico", preco: 45000 },
      { nome: "Rato sem fios", preco: 12000 },
    ],
  })
}

main()
  .then(async () => {
    await prisma.$disconnect()
  })
  .catch(async (erro) => {
    console.error(erro)
    await prisma.$disconnect()
    process.exit(1)
  })
```

Configura o comando no `prisma.config.ts`:

```ts
export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
    seed: "tsx prisma/seed.ts",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
})
```

Corre com:

```bash
npx prisma db seed
```


> 💡 **Dica**: `prisma migrate dev` corre o seed automaticamente sempre que a base de dados é resetada — útil para garantires que o ambiente de dev de qualquer pessoa na equipa arranca sempre com os mesmos dados de exemplo.


**Regra de ouro**: o seed é para dados de desenvolvimento e demonstração — nunca uses o mesmo script para popular produção com dados reais de clientes.


---


## 16. Prisma Studio


O Prisma Studio é uma interface gráfica para explorares e editares os teus dados diretamente:

```bash
npx prisma studio
```

Abre por defeito em `http://localhost:5555`, mostra todos os teus models como tabelas navegáveis, com filtros, edição inline e suporte para relações.


> 💡 **Dica**: desde a versão 7, o Prisma Studio tem uma versão renovada com dark mode e edição multi-célula — útil para popular ou corrigir dados de teste rapidamente sem escreveres SQL nem scripts.


**Regra de ouro**: o Prisma Studio é uma ferramenta de desenvolvimento — nunca o deixes exposto a apontar para uma base de dados de produção sem autenticação e rede restrita.


---


## 17. Erros comuns e como tratá-los

```ts
import { Prisma } from "@/app/generated/prisma/client"

try {
  await prisma.produto.create({
    data: { nome: "Teclado", preco: 45000, sku: "TEC-001" },
  })
} catch (erro) {
  if (erro instanceof Prisma.PrismaClientKnownRequestError) {
    if (erro.code === "P2002") {
      throw new Error("Já existe um produto com este SKU")
    }
  }
  throw erro
}
```


| Código | Situação |
|---|---|
| `P2002` | Violação de constraint única (`@unique`) |
| `P2003` | Violação de chave estrangeira |
| `P2025` | Registo não encontrado (ex: `update`/`delete` num `id` inexistente) |
| `P2021` | A tabela não existe na base de dados atual (migration não aplicada) |


> ⚠️ **Atenção**: nunca exponhas a mensagem de erro crua do Prisma diretamente ao utilizador final numa API pública — ela pode revelar detalhes da estrutura da tua base de dados. Traduz o código de erro para uma mensagem de negócio, como no exemplo acima.


**Regra de ouro**: trata os erros do Prisma pelo `code`, não pela mensagem de texto — a mensagem pode mudar entre versões, o código é a parte estável da API de erros.


---


## 18. Arquitetura e boas práticas


### Separação de camadas (mesma filosofia do guia de Next.js)

```
src/
├── app/
│   ├── generated/prisma/   # Client gerado (não editar à mão)
│   └── api/
├── lib/
│   └── prisma.ts           # singleton do Client
├── repositories/           # queries Prisma isoladas por domínio
├── services/                # lógica de negócio, chama os repositories
└── actions/                 # Server Actions, chamam os services
```

```ts
// repositories/produto-repository.ts
import { prisma } from "@/lib/prisma"

export const produtoRepository = {
  listarDisponiveis: () =>
    prisma.produto.findMany({ where: { emStock: true } }),

  criar: (dados: { nome: string, preco: number }) =>
    prisma.produto.create({ data: dados }),
}
```


### Boas práticas gerais


- Mantém todas as queries Prisma dentro de `repositories/` — nunca chames `prisma.*` diretamente de dentro de um componente React.

- Usa `select` em produção sempre que não precisares do registo completo.

- Define sempre `take` em `findMany` para evitar carregar tabelas inteiras por engano.

- Usa `Decimal` para dinheiro, nunca `Float`.

- Corre `prisma format` antes de commitares alterações ao schema — mantém o alinhamento consistente.

- Mantém as migrations no controlo de versão (Git) — nunca as adiciones ao `.gitignore`.

```bash
npx prisma format
```

**Regra de ouro**: o Prisma dá-te tipos e segurança, mas a arquitetura continua a ser tua responsabilidade — isola o acesso a dados numa camada própria, do mesmo modo que farias com qualquer outra dependência externa.


---


## 19. Roadmap: como te tornares expert


| Fase | Foco |
|---|---|
| **Fundamentos** | Schema, models, tipos, migrations básicas, CRUD simples |
| **Intermédio** | Relações (1:N, N:N), filtros avançados, transactions, singleton no Next.js |
| **Avançado** | Driver adapters, connection pooling em serverless, Client Extensions, seed automatizado |
| **Expert** | Introspection de bases de dados legadas, otimização de queries com `EXPLAIN ANALYZE`, arquitetura repository/service em apps grandes, migração incremental entre versões major |


> 💡 **Dica**: a melhor forma de fixar isto é pegar num dos teus projetos (como o Eminus, que já usa Prisma + PostgreSQL) e forçar cada query nova a passar por esta pergunta: "estou a trazer só os dados que preciso, ou estou a confiar no `include: true` por preguiça?"


**Regra de ouro final**: o Prisma muda rápido — este guia reflete a versão 7.x de agosto de 2026, já com o Client Rust-free como comportamento por defeito. O Prisma 8 já está em early access como reescrita completa em TypeScript — sempre que voltares a este guia daqui a uns meses, confirma no changelog oficial (`prisma.io/changelog`) se algo mudou.
