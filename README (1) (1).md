# API de Denúncias de Notícias

Back end em Node.js (Express + SQLite) para denunciar notícias, contabilizar as denúncias e exibir o ranking das notícias mais denunciadas.

## Objetivos atendidos

| Objetivo | Onde está |
|---|---|
| Criar o sistema de denúncias das notícias | `POST /noticias/:id/denuncias` |
| Registrar e contabilizar as denúncias recebidas | Tabela `denuncias` + contador `total_denuncias` em `noticias`, atualizados na mesma transação |
| Desenvolver o ranking das notícias mais denunciadas | `GET /ranking` |
| Ordenar as notícias pela quantidade de denúncias | `GET /noticias?ordenar=denuncias` e `GET /ranking` |
| Criar as rotas da API para denúncias e ranking | Seção [Rotas](#rotas) |

## Requisitos

- Node.js **22.13 ou superior** (usa o módulo nativo `node:sqlite`, sem banco externo)

## Como rodar

```bash
npm install
npm start          # http://localhost:3000
npm run dev        # reinicia ao salvar arquivos
npm test           # roda os testes
```

Variáveis de ambiente:

| Variável | Padrão | Descrição |
|---|---|---|
| `PORT` | `3000` | Porta do servidor |
| `DB_FILE` | `denuncias.db` | Arquivo do banco SQLite (use `:memory:` para banco temporário) |

## Rotas

### Denúncias

#### `POST /noticias/:id/denuncias`
Registra uma denúncia e atualiza o contador da notícia.

```json
{
  "denunciante": "ana",
  "motivo": "fake_news",
  "descricao": "Informação falsa (opcional, até 500 caracteres)"
}
```

Resposta `201`:

```json
{
  "denuncia": { "id": 1, "noticia_id": 1, "denunciante": "ana", "motivo": "fake_news", "descricao": null, "criada_em": "2026-10-31 19:01:59" },
  "total_denuncias": 1
}
```

Erros: `400` (campo obrigatório ausente ou motivo inválido), `404` (notícia inexistente), `409` (o mesmo denunciante já denunciou esta notícia).

#### `GET /noticias/:id/denuncias`
Lista as denúncias de uma notícia, com o total e a contagem por motivo.

```json
{
  "total": 3,
  "por_motivo": [{ "motivo": "spam", "quantidade": 2 }, { "motivo": "fake_news", "quantidade": 1 }],
  "denuncias": [ { "id": 3, "noticia_id": 1, "denunciante": "carlos", "motivo": "spam", "descricao": null, "criada_em": "..." } ]
}
```

#### `GET /denuncias/motivos`
Retorna os motivos aceitos: `fake_news`, `discurso_de_odio`, `conteudo_improprio`, `spam`, `violacao_de_direitos_autorais`, `outro`.

### Ranking

#### `GET /ranking`
Notícias mais denunciadas, da que tem mais denúncias para a que tem menos.

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `ordem` | `desc` | `desc` (mais denunciadas primeiro) ou `asc` |
| `limite` | `10` | Itens por página (máximo 100) |
| `pagina` | `1` | Página |
| `minimo` | `1` | Só inclui notícias com pelo menos N denúncias |

Resposta `200`:

```json
{
  "pagina": 1, "limite": 10, "total": 3,
  "itens": [
    { "posicao": 1, "id": 2, "titulo": "Y", "total_denuncias": 3, "criada_em": "..." },
    { "posicao": 2, "id": 3, "titulo": "Z", "total_denuncias": 2, "criada_em": "..." }
  ]
}
```

Em caso de empate, a notícia mais antiga (menor `id`) aparece primeiro.

### Notícias

#### `GET /noticias`
Lista as notícias com ordenação.

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `ordenar` | `denuncias` | `denuncias`, `recentes` ou `titulo` |
| `ordem` | `desc` (`asc` para `titulo`) | `asc` ou `desc` |
| `limite` | `20` | Itens por página (máximo 100) |
| `pagina` | `1` | Página |

Valores inválidos em `ordenar` ou `ordem` retornam `400`.

#### `POST /noticias`
Cria uma notícia (rota de apoio, para haver o que denunciar). Corpo: `{ "titulo": "...", "conteudo": "..." }`. Resposta `201`.

#### `GET /noticias/:id`
Retorna uma notícia, incluindo `total_denuncias`. `404` se não existir.

### Erros

Todas as respostas de erro seguem o formato `{ "erro": "mensagem" }`. Rotas inexistentes retornam `404` e JSON malformado retorna `400`.

## Exemplo rápido

```bash
curl -X POST localhost:3000/noticias -H 'content-type: application/json' \
  -d '{"titulo":"Título","conteudo":"Texto"}'

curl -X POST localhost:3000/noticias/1/denuncias -H 'content-type: application/json' \
  -d '{"denunciante":"ana","motivo":"fake_news"}'

curl 'localhost:3000/ranking?limite=5'
```

## Regras de negócio

- **Uma denúncia por pessoa:** o par (`noticia_id`, `denunciante`) é único; repetir retorna `409`.
- **Contagem consistente:** inserir a denúncia e incrementar `total_denuncias` acontecem na mesma transação; se algo falhar, nada é gravado.
- **Ranking rápido:** índice em `(total_denuncias DESC, id)`.
- **Ordenação segura:** os campos de ordenação são mapeados por uma lista fixa, sem concatenar texto do usuário no SQL.

## Estrutura do projeto

```
src/
├── server.js               # inicialização (PORT)
├── app.js                  # Express, middlewares e tratamento de erros
├── db.js                   # criação do banco e das tabelas
├── routes/index.js         # rotas da API
└── services/denuncias.js   # regras de negócio (denunciar, listar, ranking)
test/
└── api.test.js             # testes de API (node:test)
```

## Observações

- O `denunciante` é um texto livre. Se o projeto tiver autenticação, troque por `req.user.id` para impedir que a mesma pessoa use nomes diferentes.
- A lista de motivos fica na constante `MOTIVOS`, em `src/services/denuncias.js`.
