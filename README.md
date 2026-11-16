# Catálogo de Produtos — CRUD com MongoDB

Atividade Prática da disciplina **IAL-215 – Banco de Dados NoSQL**.

Sistema de catálogo de produtos que **cadastra, lista, altera e exclui** produtos (CRUD), com controle de estoque, tags, fotos, relatório e dashboard. Todos os dados ficam no **MongoDB**: banco `catalogo`, coleção `produtos`.

![Dashboard do sistema](docs/prints/dashboard.jpg)

## Integrantes

- Guilherme Paiva
- João Henrique
- Marcos
- Thales

## Sumário

- [Tecnologias](#tecnologias)
- [Onde está o banco de dados](#onde-está-o-banco-de-dados)
- [Modelo do documento](#modelo-do-documento)
- [Onde está o CRUD](#onde-está-o-crud)
- [Telas](#telas)
- [Como executar](#como-executar)
- [Estrutura do projeto](#estrutura-do-projeto)

## Tecnologias

| Tecnologia | Para que serve no projeto | Arquivo |
| --- | --- | --- |
| **MongoDB** Community Server | Banco de dados. Cada produto é um documento. | servidor em `localhost:27017` |
| **Python** + **Flask** | Servidor web: telas do site e CRUD | `python-web/app.py` |
| **PyMongo** | Driver que conecta o Python ao MongoDB | `python-web/app.py` |
| **Jinja2**, HTML e CSS | Visual das telas | `python-web/templates/`, `python-web/static/` |
| **Java 17** + `mongodb-driver-sync` | Programa de terminal com o mesmo CRUD | `java-cli/src/main/java/ial215/catalogo/CatalogoCli.java` |
| **Maven** | Baixa o driver e compila o Java | `java-cli/pom.xml` |
| **mongosh** e **Compass** | Consultas direto no banco (evidências) | `mongosh/comandos.js` |

O site em Python e o programa em Java usam **o mesmo banco e a mesma coleção**: o que um grava, o outro enxerga.

## Onde está o banco de dados

O banco **não é um arquivo dentro do projeto**. O MongoDB é um programa (servidor) que fica rodando no computador e atende no endereço `mongodb://localhost:27017`. O projeto só se conecta a ele:

```python
# python-web/app.py
client = MongoClient("mongodb://localhost:27017")
db = client["catalogo"]          # banco
produtos = db["produtos"]        # coleção usada por todo o CRUD
```

Para usar outro servidor (por exemplo, o MongoDB Atlas), defina a variável de ambiente `MONGO_URI`.

Índices criados ao iniciar (`criar_indices()` em `app.py`):

| Índice | Campos | Para quê |
| --- | --- | --- |
| `sku_unico` | `sku` (único) | Não deixa cadastrar dois produtos com o mesmo código |
| `texto_nome_descricao` | `nome`, `descricao` (texto) | Busca por texto |
| `categoria_preco` | `categoria`, `preco` | Filtros e ordenação por categoria e preço |

## Modelo do documento

```json
{
  "_id": ObjectId("..."),
  "sku": "ESC-0003",
  "nome": "Cadeira ergonômica",
  "descricao": "",
  "categoria": "Escritório",
  "preco": 899.0,
  "estoque": 5,
  "tags": ["móveis"],
  "fornecedor": { "nome": "Conforto Móveis", "cidade": "Bauru" },
  "especificacoes": { "peso_maximo": "120 kg", "ajuste_altura": "sim", "cor": "preta" },
  "foto": "ESC-0003-1794421041.jpg",
  "ativo": true,
  "criado_em": ISODate("..."),
  "atualizado_em": ISODate("...")
}
```

| Recurso do MongoDB | Onde aparece |
| --- | --- |
| Documento embutido | `fornecedor` fica dentro do produto, sem tabela separada nem JOIN |
| Array | `tags`, alterado com `$addToSet` e `$pull` |
| Esquema flexível | `especificacoes` tem chaves diferentes em cada produto |
| Operador atômico | `$inc` na entrada e saída de estoque |
| Aggregation Pipeline | relatório (`$group`, `$sort`, `$unwind`) e dashboard (`$facet`, `$bucket`, `$cond`) |

## Onde está o CRUD

### Python — `python-web/app.py`

| Operação | Função | Rota | Comando no MongoDB |
| --- | --- | --- | --- |
| **C**reate | `novo()` | `POST /produtos/novo` | `insert_one` |
| **R**ead (lista e busca) | `listar()` | `GET /` | `find().sort().skip().limit()`, `count_documents`, `distinct` |
| **R**ead (um produto) | `detalhe()` | `GET /produtos/<id>` | `find_one` |
| **U**pdate (edição) | `editar()` | `POST /produtos/<id>/editar` | `update_one` com `$set` / `$unset` |
| **U**pdate (estoque) | `ajustar_estoque()` | `POST /produtos/<id>/estoque` | `update_one` com `$inc` |
| **U**pdate (tags) | `alterar_tag()` | `POST /produtos/<id>/tags` | `update_one` com `$addToSet` / `$pull` |
| **D**elete | `excluir()` | `POST /produtos/<id>/excluir` | `delete_one` |
| Relatório | `relatorio()` | `GET /relatorio` | `aggregate` |
| Dashboard | `dashboard()` | `GET /dashboard` | `aggregate` com `$facet` |

Carga de dados de exemplo: `python-web/seed.py` (`insert_many` e, com `--reset`, `delete_many`).

### Java — `java-cli/src/main/java/ial215/catalogo/CatalogoCli.java`

| Operação | Método | Comando no MongoDB |
| --- | --- | --- |
| **C**reate | `inserir()` | `insertOne` |
| **R**ead | `listar()` | `find` com `Filters` e `Sorts` |
| **U**pdate | `atualizarPreco()`, `ajustarEstoque()` | `updateOne` com `set` / `inc` |
| **D**elete | `excluir()` | `deleteOne` |
| Relatório | `relatorio()` | `aggregate` com `Aggregates.group` |

### mongosh — `mongosh/comandos.js`

Os mesmos comandos (`insertOne`, `find`, `updateOne`, `deleteOne`, `aggregate`) para rodar direto no shell do banco.

## Telas

As telas abaixo mostram o CRUD com o produto de teste "Fone de ouvido Bluetooth" (SKU `ELE-0099`).

| | |
| --- | --- |
| **Lista de produtos** ![Lista](docs/prints/lista.jpg) | **Busca (Read)** — o filtro `$regex` enviado ao banco aparece na tela ![Busca](docs/prints/busca.jpg) |
| **Cadastro (Create)** ![Formulário](docs/prints/create_form.jpg) | **Produto cadastrado**, com o `_id` gerado pelo MongoDB ![Cadastrado](docs/prints/create_ok.jpg) |
| **Edição (Update com `$set`)** — preço alterado ![Atualizado](docs/prints/update_ok.jpg) | **Estoque e tags (Update com `$inc` e `$addToSet`)** ![Estoque e tags](docs/prints/tag_ok.jpg) |
| **Exclusão (Delete)** ![Excluído](docs/prints/delete_ok.jpg) | **SKU duplicado** — o índice único impede o cadastro ![SKU duplicado](docs/prints/dup_sku.jpg) |
| **Relatório por categoria (Aggregation Pipeline)** ![Relatório](docs/prints/relatorio.jpg) | |

## Como executar

### 1. Instale

- [MongoDB Community Server](https://www.mongodb.com/try/download/community) (deixe marcado "Install MongoDB as a Service" e "Install MongoDB Compass")
- [Python 3.10+](https://www.python.org/downloads/) (marque "Add python.exe to PATH")
- Só para o cliente Java: [JDK 17+](https://adoptium.net/temurin/releases/) e [Maven](https://maven.apache.org/download.cgi)

### 2. Rode a aplicação web

No Windows: dois cliques em **`rodar-web.bat`**. Ele instala as dependências, insere os produtos de exemplo e abre o navegador.

Pelo terminal:

```bash
cd python-web
python -m venv .venv
# Windows: .venv\Scripts\activate      Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
python seed.py        # insere 12 produtos de exemplo (use --reset para recriar)
python app.py         # abre em http://localhost:5000/dashboard
```

### 3. Rode o cliente Java (opcional)

No Windows: dois cliques em **`rodar-java.bat`**. Pelo terminal:

```bash
cd java-cli
mvn compile exec:java
```

Menu: listar, buscar por nome, inserir, atualizar preço, ajustar estoque, excluir e relatório por categoria.

### 4. Veja os dados no banco

```bash
mongosh --file mongosh/comandos.js
```

Ou abra o MongoDB Compass, conecte em `mongodb://localhost:27017` e abra `catalogo > produtos`.

## Estrutura do projeto

```
catalogo-produtos-mongodb/
├── python-web/
│   ├── app.py             ← conexão com o banco, CRUD e agregações
│   ├── seed.py            ← 12 produtos de exemplo
│   ├── requirements.txt   ← Flask e PyMongo
│   ├── templates/         ← telas (dashboard, lista, formulário, detalhe, relatório)
│   └── static/            ← CSS e fotos enviadas (uploads/)
├── java-cli/
│   ├── pom.xml
│   └── src/main/java/ial215/catalogo/CatalogoCli.java   ← CRUD em Java
├── mongosh/
│   └── comandos.js        ← comandos de evidência
├── docs/prints/           ← prints usados neste README
├── rodar-web.bat          ← atalho Windows: aplicação web
├── rodar-java.bat         ← atalho Windows: cliente Java
└── README.md
```

## Problemas comuns

| Erro | Causa | Solução |
| --- | --- | --- |
| `ServerSelectionTimeoutError` / "Não foi possível conectar no MongoDB" | O serviço do MongoDB está desligado | Inicie o serviço "MongoDB" no Windows (Serviços) ou instale o MongoDB Community Server |
| `'python' não é reconhecido` | Python fora do PATH | Reinstale marcando "Add python.exe to PATH" |
| `'mvn' não é reconhecido` | Maven fora do PATH | Adicione a pasta `bin` do Maven à variável Path |
| Aparece a versão antiga do site | Outra instância usando a porta 5000 | Feche as janelas antigas e recarregue com Ctrl + F5 |
