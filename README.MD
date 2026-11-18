# 🌿 CANA — Assistente inteligente para delivery

O CANA é um agente de atendimento para pedidos de mercado. Esta versão organiza o sistema em dois serviços de aplicação: **Python** mantém a conversa, a IA, os webhooks e a interface; **Java (Spring Boot)** centraliza regras de negócio, catálogo, estoque, carrinhos, pedidos e todo o acesso ao PostgreSQL.

> **Regra arquitetural:** o serviço Python não se conecta diretamente ao PostgreSQL nem ao Redis. Toda leitura ou gravação de dados de negócio passa pela API Java.

## 1. Arquitetura

```text
Cliente / Simulador Web / WhatsApp
                │
                ▼
       Evolution API (WhatsApp)
                │ webhook
                ▼
       PYTHON — FastAPI (8000)
       • atendimento e mensagens
       • interpretação com Gemini/OpenAI (opcional)
       • fallback local de interpretação
       • simulador web, painel CANA Admin e webhooks
                │ HTTP + JSON
                ▼
       JAVA — Spring Boot (8081)
       • catálogo e consulta de produtos
       • consulta de estoque
       • persistência de carrinhos
       • criação e cálculo de pedidos
       • validações de negócio
                │ JPA / Hibernate + Flyway
                ▼
       PostgreSQL (5432)

Evolution API usa Redis para cache/sessões (6379).
```

A IA interpreta a intenção e produz os itens estruturados da conversa. O Java é a fonte de verdade para dados comerciais: produto cadastrado, preço, estoque, persistência e cálculo financeiro de pedidos. Um pedido só é gravado quando o endpoint de criação é chamado explicitamente; a conversa não transforma automaticamente uma lista em pedido confirmado.

## 2. Estrutura do projeto

```text
Agente-de-Delivery/
├── app/                         # Python: agente, mensagens e integração
│   ├── config.py                # Variáveis de ambiente
│   ├── java_api.py              # Cliente HTTP para o backend Java
│   ├── estado.py                # Fachada de carrinho (sem banco local)
│   ├── schemas.py               # Modelos Pydantic
│   ├── services.py              # Interpretação e formatação das mensagens
│   └── whatsapp.py              # Envio via Evolution API
├── backend-java/                # Java: domínio e persistência
│   ├── src/main/java/           # Controllers, serviços, entidades e repositórios
│   ├── src/main/resources/      # Configuração e migrações Flyway
│   ├── Dockerfile
│   └── pom.xml
├── database/init/               # SQL não destrutivo para banco novo
├── templates/index.html         # Simulador web
├── templates/admin.html         # Painel CANA Admin
├── tests/                       # Testes Python
├── docs/ARQUITETURA_V2.md       # Decisões e limites da arquitetura
├── docker-compose.yml           # PostgreSQL, Redis, Java, Python e Evolution API
├── Dockerfile                   # Serviço Python
├── requirements.txt             # Dependências Python de execução
├── requirements-dev.txt         # Dependências Python de desenvolvimento/testes
├── init_db.sql                  # SQL de referência (não destrutivo)
├── iniciar-cana.ps1             # Cria .env e inicia o ambiente no Windows
├── parar-cana.ps1               # Para sem remover os dados
├── configurar-webhook.ps1       # Configura o webhook da Evolution API
├── gerar-qrcode.ps1              # Abre o QR Code pelo PowerShell (sem Python no host)
├── gerar_qrcode.py               # Alternativa para ambiente Python local
├── listar_modelos.py             # Lista modelos acessíveis pela chave Gemini
└── .env.example                 # Modelo de configuração local
```

## 3. Iniciar no Windows (modo recomendado: Docker)

### Pré-requisitos

- Docker Desktop instalado e aberto.
- Git instalado (para clonar/atualizar o repositório; não é necessário para abrir o ZIP).
- Windows PowerShell 5.1 ou PowerShell 7.
- Conexão com a internet na primeira construção das imagens.

Java e Maven não precisam estar instalados no Windows quando você usa Docker: o contêiner compila o backend Java.

### Passo 1 — Criar o arquivo `.env`

Na pasta principal do projeto, copie `.env.example` e renomeie a cópia para `.env`. O script `iniciar-cana.ps1` faz isso automaticamente na primeira execução. Para configurar manualmente, use:

```powershell
Copy-Item .env.example .env
```

Abra `.env`. Para começar localmente, os valores de exemplo funcionam. Antes de publicar ou usar em produção, troque todas as senhas e chaves locais. Coloque sua chave Gemini em `GEMINI_API_KEY` para habilitar o modelo (padrão configurado: `gemini-3.8-flash`); sem ela, o modo local continua disponível, com interpretação mais simples.

### Passo 2 — Construir e iniciar todos os serviços

No PowerShell:

```powershell
.\iniciar-cana.ps1
```

Ou, se preferir iniciar manualmente:

```powershell
docker compose up --build -d
```

Na primeira execução, o Docker baixa as dependências e constrói os serviços. Confira o estado:

```powershell
docker compose ps
```

Acompanhe os logs:

```powershell
docker compose logs -f java-api python-api
```

### Passo 3 — Abrir o CANA

- Simulador web: http://localhost:8000
- Painel da equipe: http://localhost:8000/admin
- Saúde do Python: http://localhost:8000/health
- Prontidão Python + Java: http://localhost:8000/ready
- Saúde da API Java: http://localhost:8081/api/health
- Evolution API: http://localhost:8080

As sessões pareadas do WhatsApp ficam no volume Docker `evolution_instances`, para sobreviverem à recriação do container. A base da Evolution e os dados de negócio do CANA compartilham o mesmo serviço PostgreSQL de desenvolvimento, mas usam nomes de tabelas distintos.

### Passo 4 — Parar ou reiniciar

```powershell
# Parar os serviços sem apagar os dados
docker compose down

# Iniciar novamente
docker compose up -d

# Ver logs de um serviço
docker compose logs -f java-api
```

> **Não use `docker compose down -v` para uma parada comum.** A opção `-v` apaga os volumes e pode excluir os dados do PostgreSQL e Redis.

## 4. Executar sem Docker (modo desenvolvimento)

Para executar serviços individualmente, instale Java 21, Maven, Python 3.11+ e tenha PostgreSQL disponível. Configure `.env` com `JAVA_API_URL=http://localhost:8081` e as credenciais reais do banco.

Terminal 1 — backend Java, a partir de `backend-java`:

```powershell
mvn spring-boot:run
```

Terminal 2 — Python, na raiz do projeto:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
$env:JAVA_API_URL = "http://localhost:8081"
python -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

No modo local, defina `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `INTERNAL_API_TOKEN` e `ADMIN_TOKEN_SECRET` no ambiente do processo Java. O mesmo `INTERNAL_API_TOKEN` deve estar no `.env` do Python.


## 5. CANA Admin — painel administrativo

Acesse **http://localhost:8000/admin** depois de iniciar os serviços. O painel é servido pelo Python, mas todas as leituras e gravações de negócio passam pela API Java; o navegador não recebe o token interno nem acessa PostgreSQL.

### Contas iniciais

| Funcionário | Usuário | Papel | Senha inicial |
|---|---|---|---|
| Nicolas | `nicolas` | Administrador | `123` |
| Pedro Lucas | `pedro.lucas` | Funcionário | `123` |

As duas contas usam a senha provisória solicitada. Ela é armazenada com hash PBKDF2 no PostgreSQL. A conta Nicolas pode cadastrar e editar produtos; ambos podem consultar produtos, movimentar estoque e administrar pedidos. Não há cadastro público de novas contas. Troque a senha de demonstração antes de qualquer implantação real; o painel é uma base de desenvolvimento, não um sistema pronto para exposição pública.

### O que o painel oferece

- Dashboard com total de produtos ativos, itens abaixo do mínimo, pedidos pendentes e vendas confirmadas.
- Consulta e pesquisa de produtos por nome, categoria ou código.
- Cadastro e edição de produtos (administrador).
- Entradas, saídas e ajustes de estoque, com motivo e identificação do funcionário.
- Histórico das últimas 100 movimentações.
- Lista de pedidos e atualização de status.
- Confirmar pedido baixa o estoque em transação; cancelar um pedido já confirmado registra a reposição. A confirmação é recusada quando não há quantidade suficiente.

### Catálogo inicial

A migração `V2__cana_admin.sql` preserva os cinco itens originais e completa o catálogo com produtos comuns até **no máximo 30 produtos ativos**. Preços e quantidades são valores demonstrativos para o ambiente local; revise-os no painel antes de usar o sistema com clientes. Produtos podem ser cadastrados depois pelo administrador. A migração é não destrutiva e não apaga dados existentes.

### Segurança e configuração

- `ADMIN_TOKEN_SECRET` assina tokens administrativos com validade de 8 horas. Use uma chave própria, longa e aleatória antes de produção.
- `INTERNAL_API_TOKEN` continua sendo exigido nas chamadas Python → Java.
- O token administrativo é mantido na sessão do navegador e validado pelo Java em cada operação.
- O segredo `123` é exclusivamente para primeiro acesso/desenvolvimento. A senha não é adequada para internet ou ambiente real.
- Use HTTPS, controle de acesso à rede, backups e política de troca de senhas antes de disponibilizar o painel fora do computador local.

## 6. API Java disponível

Todas as rotas abaixo, exceto `/api/health`, aceitam o cabeçalho `X-Internal-Token` quando `INTERNAL_API_TOKEN` estiver configurado. As rotas `/api/admin/**` também exigem `Authorization: Bearer <token-administrativo>`. Na navegação normal, o painel chama o proxy Python em `/admin/api/**`; o token interno do serviço não é enviado ao navegador.

| Método | Rota | Finalidade |
|---|---|---|
| GET | `/api/health` | Saúde do serviço Java |
| GET | `/api/products?query=arroz` | Buscar produtos cadastrados |
| GET | `/api/products/{id}` | Consultar produto por ID |
| GET | `/api/products/barcode/{barcode}` | Consultar código de barras |
| GET | `/api/products/{id}/stock` | Consultar estoque |
| GET | `/api/carts/{phone}` | Carregar carrinho persistido |
| PUT | `/api/carts/{phone}` | Salvar carrinho |
| DELETE | `/api/carts/{phone}` | Limpar carrinho |
| POST | `/api/orders` | Criar pedido com preços vindos do banco |
| POST | `/api/admin/auth/login` | Autenticar funcionário e emitir token de 8 horas |
| GET | `/api/admin/dashboard` | Resumo do mercadinho |
| GET | `/api/admin/products` | Listar e pesquisar produtos com estoque |
| POST / PUT | `/api/admin/products` | Cadastrar / editar produto (administrador) |
| POST | `/api/admin/products/{id}/stock-movements` | Registrar entrada, saída ou ajuste |
| GET | `/api/admin/stock-movements` | Histórico de movimentações |
| GET | `/api/admin/orders` | Consultar pedidos e vendas |
| PATCH | `/api/admin/orders/{id}/status` | Atualizar andamento e aplicar baixa/estorno de estoque |

Exemplo de criação de pedido (somente com produto já cadastrado):

```json
{
  "cliente_telefone": "5581999999999",
  "cliente_nome": "Cliente de teste",
  "forma_pagamento": "PIX",
  "origem": "SIMULADOR",
  "itens": [
    { "produto_id": 3, "quantidade": 2, "unidade_medida": "un" }
  ]
}
```

O backend calcula o subtotal com o preço cadastrado no banco, não com um preço enviado pelo cliente. Quando o produto é vendido por KG e a solicitação vem em gramas, ou é vendido por L e a solicitação vem em mililitros, o Java converte a quantidade para a unidade cadastrada antes do cálculo. Unidades incompatíveis são rejeitadas. O exemplo acima cria um pedido de teste real no banco; use apenas quando desejar gravá-lo.

## 7. WhatsApp com Evolution API

A Evolution API fica disponível em http://localhost:8080. Para conectar um número:

1. Configure `EVOLUTION_API_KEY`, `EVOLUTION_INSTANCE` e, opcionalmente, `WEBHOOK_TOKEN` no `.env`.
2. Execute `.\gerar-qrcode.ps1` no PowerShell. O script cria ou verifica a instância, solicita o QR Code e abre a imagem no navegador. A alternativa `python gerar_qrcode.py` também está disponível para quem já configurou o Python local.
3. Leia o QR Code com o WhatsApp que será usado no atendimento.
4. Depois do pareamento, execute `configurar-webhook.ps1` no PowerShell. O script configura o evento `MESSAGES_UPSERT` e inclui o token na URL quando `WEBHOOK_TOKEN` estiver preenchido.
5. Confira `EVOLUTION_API_URL`, `EVOLUTION_LOCAL_API_URL`, `EVOLUTION_API_KEY`, `EVOLUTION_INSTANCE` e `EVOLUTION_API_VERSION` no `.env`.

O script usa o endpoint `POST /webhook/set/{instance}` da Evolution API v2. Consulte também a [documentação de configuração de webhook](https://docs.evoapicloud.com/api-reference/webhook/set).

A rota de webhook ignora mensagens enviadas pelo próprio número, grupos e eventos que não sejam mensagens recebidas. A conexão exige uma instância pareada e credenciais válidas da sua Evolution API; o repositório não contém essas credenciais.

## 8. Banco de dados e migrações

- PostgreSQL é o banco principal da aplicação e também é compartilhado com a Evolution API nesta configuração de desenvolvimento.
- Flyway executa as migrações Java no início do serviço. `baseline-on-migrate` com versão `0` permite adotar de forma controlada o schema previamente criado pelo script `database/init/01_schema.sql`.
- A migração inicial usa `CREATE TABLE IF NOT EXISTS` e inserções idempotentes; ela não executa `DROP TABLE` e não apaga tabelas existentes.
- Carrinhos são salvos na tabela `carrinhos` pelo serviço Java. O Python não usa Redis nem memória local para persistir carrinhos.
- Redis permanece como infraestrutura interna da Evolution API, não como banco de carrinhos do CANA.
- A Evolution API v2 utiliza PostgreSQL e Redis; as sessões locais ficam persistidas no volume `evolution_instances`.

**Banco já existente:** faça um backup antes de atualizar. Não apague o volume antigo. O Flyway cria a tabela `carrinhos` e os objetos que estiverem ausentes. Se seu banco tiver estrutura diferente da definida em `init_db.sql`, pare e compare o schema antes de aplicar a migração.

## 9. Testes

Testes Python:

```powershell
.\.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
pytest -q
```

Testes unitários Java (a partir de `backend-java`, com Java 21 e Maven):

```powershell
mvn test
```

Verificação de funcionamento da aplicação integrada:

```powershell
Invoke-RestMethod http://localhost:8000/ready
```

A resposta esperada contém `status = ready`, `python = ok` e `java = ok`.

## 10. Próximas etapas recomendadas

1. Ligar o fluxo de confirmação da conversa à criação dos pedidos (sem registrar pedidos antes da confirmação do cliente).
2. Implementar idempotência persistente de eventos do WhatsApp, evitando duplicação após reinício do Python.
3. Adicionar testes de integração Java + PostgreSQL e testes ponta a ponta com uma instância Evolution de homologação.
4. Implementar troca de senha pela interface administrativa e exigir senha forte antes de produção.
5. Rever autenticação, logs, backups, HTTPS, rotação de segredos e limites antes de produção.

## 11. Segurança

Não coloque chaves Gemini, tokens Evolution, senhas ou tokens internos no GitHub. O arquivo `.env` está no `.gitignore`; apenas `.env.example` deve ser versionado. Se algum segredo real já tiver sido enviado ao repositório anteriormente, revogue-o e gere outro.


## Material histórico preservado

As notas antigas em `Obsidian/` e o PDF original em `docs/hermes.delivery.pdf` foram mantidos no pacote. Eles podem descrever uma arquitetura anterior em Python; para executar e entender a versão atual, use este README e `docs/ARQUITETURA_V2.md`.
