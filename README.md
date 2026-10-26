# 📚 Biblioteca — Sistema de Gerenciamento de Biblioteca

Aplicativo **desktop para Linux** que controla acervo, leitores, empréstimos, devoluções e renovações de uma biblioteca de porte médio (universitária ou municipal), com relatórios exportáveis, importação de acervo e backup em HDD externo.

Funciona **100% offline**, em um único computador, operado pelo administrador. Dados em **SQLite**, interface em **React**, regras de negócio em **Python (FastAPI)**.

---

## 📑 Sumário

1. [Visão geral](#-visão-geral)
2. [Funcionalidades](#-funcionalidades)
3. [Como o sistema funciona](#-como-o-sistema-funciona)
4. [Stack tecnológica](#-stack-tecnológica)
5. [Arquitetura](#-arquitetura)
6. [Estrutura de pastas](#-estrutura-de-pastas)
7. [Modelo de dados](#-modelo-de-dados)
8. [Regras de negócio](#-regras-de-negócio)
9. [Plano de desenvolvimento por fases](#-plano-de-desenvolvimento-por-fases)
10. [Como rodar em desenvolvimento](#-como-rodar-em-desenvolvimento)
11. [Como gerar e instalar o aplicativo no Linux](#-como-gerar-e-instalar-o-aplicativo-no-linux)
12. [Testes](#-testes)
13. [Segurança](#-segurança)
14. [Fora do escopo e evoluções futuras](#-fora-do-escopo-e-evoluções-futuras)

---

## 🎯 Visão geral

| Item | Definição |
|---|---|
| Público | Biblioteca de porte médio (universitária ou municipal) |
| Operador | Somente o **administrador**, com login |
| Plataforma | Desktop, **Linux** |
| Máquinas simultâneas | 1 computador |
| Conexão | Offline, sem dependência de internet |
| Front-end | React |
| Back-end | Python + FastAPI |
| Banco de dados | SQLite (arquivo local) |
| Backup | HDD externo |

**Objetivo:** substituir controles manuais (cadernos e planilhas) por um sistema que aplica as regras de empréstimo automaticamente, avisa sobre atrasos, bloqueia leitores pendentes e gera estatísticas prontas para análise.

---

## ✨ Funcionalidades

### Acervo
- Cadastro de **obras** (livros, periódicos e mídias) com título, autor, editora, ano, ISBN e categoria.
- Cadastro de **exemplares** com código de barras/tombo único e status (disponível, emprestado, perdido, danificado).
- Busca por título, autor, ISBN ou código.
- **Importação** do acervo existente via CSV ou XLSX, com prévia e relatório de erros.

### Leitores
- Cadastro de leitores por tipo: **aluno**, **professor** e **público**.
- Alunos com matrícula, **ano** e **sala**.
- Histórico completo de empréstimos e situação atual (regular, em atraso, bloqueado).

### Empréstimos
- **Empréstimo, devolução e renovação** com validação automática das regras.
- Regras **diferentes por tipo de leitor** (prazo, limite de itens e renovações).
- **Alerta de atraso** e **bloqueio automático** de novos empréstimos até a devolução.

### Relatórios e estatísticas
- Livros **mais emprestados** e **menos emprestados**.
- **Alunos com mais empréstimos**, com a quantidade, filtrável por **ano/sala**.
- Empréstimos por período, atrasos atuais e histórico.
- **Exportação em CSV e XLSX**.

### Administração
- Login do administrador (único usuário).
- Configuração das políticas por tipo de leitor.
- **Backup e restauração** em HDD externo.

---

## ⚙️ Como o sistema funciona

### 1. Inicialização do aplicativo

1. O usuário clica no ícone **Biblioteca** no menu do Linux.
2. O `launcher` escolhe uma porta livre e sobe o **FastAPI** em `127.0.0.1`.
3. O FastAPI abre (ou cria) o banco em `~/.local/share/biblioteca/biblioteca.db` e aplica migrações pendentes.
4. O **PyWebView** abre uma janela nativa apontando para o servidor local, que entrega o **React** já compilado.
5. A tela de **login** é exibida.

### 2. Fluxo de empréstimo

```
Admin informa leitor + exemplar (ou lê o código de barras)
        │
        ▼
Leitor está ativo?  ──não──▶ recusa com mensagem
        │ sim
        ▼
Leitor tem item em atraso?  ──sim──▶ BLOQUEIA e mostra os itens atrasados
        │ não
        ▼
Atingiu o limite do tipo de leitor?  ──sim──▶ recusa com mensagem
        │ não
        ▼
Exemplar está disponível?  ──não──▶ recusa com mensagem
        │ sim
        ▼
Calcula data prevista (hoje + prazo do tipo de leitor)
Registra empréstimo, guarda ano/sala do leitor
Exemplar passa a "emprestado"
```

### 3. Fluxo de devolução

1. O admin informa o exemplar (ou lê o código de barras).
2. O sistema localiza o empréstimo em aberto, registra a data de devolução e libera o exemplar.
3. Se o leitor não tiver mais itens atrasados, o **bloqueio é removido automaticamente**.

### 4. Fluxo de renovação

1. Permitida apenas se o empréstimo **não está atrasado** e o limite de renovações do tipo de leitor não foi atingido.
2. O novo prazo é contado a partir da data da renovação, usando o prazo do tipo de leitor.

### 5. Controle de atrasos

Um empréstimo está em atraso quando **não foi devolvido e a data prevista é anterior a hoje**. O cálculo é feito em tempo de consulta (sem tarefas em segundo plano), então o painel sempre reflete a situação real. O sistema:

- exibe um **alerta** no painel inicial;
- marca o leitor como **em atraso** nas telas de leitores e empréstimos;
- **impede novos empréstimos** até a devolução.

### 6. Importação do acervo

1. O admin escolhe um arquivo CSV ou XLSX.
2. O sistema valida colunas obrigatórias e formatos e mostra uma **prévia** (linhas válidas e com erro).
3. Ao confirmar, a gravação ocorre em **uma única transação**: se algo falhar, nada é gravado pela metade.
4. Um relatório final lista as linhas importadas e os erros.

### 7. Estatísticas

As consultas agregam a tabela de empréstimos. O **ano e a sala do aluno são gravados no momento do empréstimo**, então as estatísticas por turma continuam corretas mesmo que o aluno mude de sala depois. Cada relatório pode ser exportado em CSV ou XLSX.

### 8. Backup

1. O admin escolhe a pasta de destino no HDD externo (uma vez, em Configurações).
2. O sistema usa a **API de backup do SQLite** (seguro mesmo com o banco aberto), gera um arquivo com data e hora no nome e **valida a integridade** da cópia.
3. Se o HDD não estiver conectado, o sistema avisa e registra a falha no histórico.
4. A **restauração** substitui o banco atual, mantendo antes uma cópia de segurança do estado anterior.

---

## 🧰 Stack tecnológica

| Camada | Tecnologias |
|---|---|
| Front-end | React, Vite, React Router, Recharts (gráficos) |
| Back-end | Python 3.11+, FastAPI, Uvicorn, SQLAlchemy 2, Alembic, Pydantic |
| Banco | SQLite (modo WAL, chaves estrangeiras ativas) |
| Segurança | argon2/bcrypt para senhas, token de sessão com expiração |
| Planilhas | CSV (stdlib) e openpyxl (XLSX) |
| Desktop | PyWebView, PyInstaller, pacote `.deb` |
| Testes | pytest, Vitest/React Testing Library |

---

## 🏗️ Arquitetura

Aplicação **monolítica local, em camadas**, empacotada como app desktop.

```
┌──────────────────────────────────────────────┐
│  Janela desktop (PyWebView)                  │
│  ┌────────────────────────────────────────┐  │
│  │ React (build estático)                 │  │
│  └──────────────────┬─────────────────────┘  │
│                     │ HTTP 127.0.0.1         │
│  ┌──────────────────▼─────────────────────┐  │
│  │ FastAPI                                │  │
│  │ routers → services → repositories      │  │
│  └──────────────────┬─────────────────────┘  │
│                     │ SQLAlchemy             │
│  ┌──────────────────▼─────────────────────┐  │
│  │ SQLite (biblioteca.db)                 │  │
│  └────────────────────────────────────────┘  │
└──────────┬─────────────────────┬─────────────┘
           │ backup / restauração│ importação / exportação
           ▼                     ▼
      HDD externo           CSV / XLSX
```

| Camada | Responsabilidade |
|---|---|
| `routers` | Endpoints HTTP e validação de entrada/saída; sem regra de negócio |
| `services` | Regras de negócio (empréstimo, atraso, importação, relatórios, backup) |
| `repositories` | Acesso ao banco via SQLAlchemy |
| `models` | Entidades do banco |
| `schemas` | Modelos Pydantic |
| `core` | Configuração, banco, segurança e dependências |

---

## 📂 Estrutura de pastas

```
biblioteca/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── core/            # config, database, security, deps
│   │   ├── models/          # usuario_admin, tipo_leitor, leitor, obra, exemplar, emprestimo, logs
│   │   ├── schemas/         # auth, leitor, obra, exemplar, emprestimo, relatorio
│   │   ├── repositories/    # *_repo.py
│   │   ├── services/        # auth, acervo, leitor, emprestimo, atraso,
│   │   │                    # importacao, relatorio, exportacao, backup
│   │   ├── routers/         # auth, obras, exemplares, leitores, tipos_leitor,
│   │   │                    # emprestimos, importacao, relatorios, backup
│   │   └── static/          # build do React (gerado)
│   ├── migrations/          # Alembic
│   ├── tests/
│   ├── alembic.ini
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── pages/           # Login, Dashboard, Acervo, Leitores, Emprestimos,
│   │   │                    # Importacao, Relatorios, Configuracoes
│   │   ├── components/
│   │   ├── services/        # api.js + um arquivo por módulo
│   │   ├── hooks/
│   │   ├── routes/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── package.json
│   └── vite.config.js
├── desktop/
│   ├── launcher.py
│   ├── biblioteca.spec
│   ├── biblioteca.desktop
│   ├── icone.png
│   └── build/               # build.sh, install.sh, make_deb.sh
├── docs/
├── .gitignore
└── README.md
```

**Dados do usuário (fora da pasta do programa):**

```
~/.local/share/biblioteca/
├── biblioteca.db
├── logs/
└── exportacoes/
```

> O banco nunca fica dentro da pasta do programa, então atualizar ou reinstalar o app **não apaga os dados**.

---

## 🗄️ Modelo de dados

| Tabela | Campos principais |
|---|---|
| `usuario_admin` | id, usuario, senha_hash, criado_em |
| `tipo_leitor` | id, nome, prazo_dias, limite_emprestimos, max_renovacoes |
| `leitor` | id, nome, matricula, tipo_leitor_id, ano, sala, contato, ativo |
| `obra` | id, titulo, autor, editora, ano_publicacao, isbn, categoria, tipo_material, ativo |
| `exemplar` | id, obra_id, codigo_barras (único), status |
| `emprestimo` | id, exemplar_id, leitor_id, data_emprestimo, data_prevista, data_devolucao, renovacoes, ano_leitor, sala_leitor |
| `log_backup` | id, data, destino, status, mensagem |
| `log_importacao` | id, data, arquivo, linhas_ok, linhas_erro, detalhes |

**Relações:** uma obra tem vários exemplares · um leitor tem vários empréstimos · um exemplar tem vários empréstimos ao longo do tempo (apenas um em aberto) · um tipo de leitor tem vários leitores.

---

## 📏 Regras de negócio

### Políticas iniciais (editáveis em Configurações)

| Tipo de leitor | Prazo | Limite de itens | Renovações |
|---|---|---|---|
| Aluno | 7 dias | 3 | 2 |
| Professor | 15 dias | 5 | 3 |
| Público | 7 dias | 2 | 1 |

> Valores sugeridos; ajuste conforme a realidade da biblioteca.

### Resumo das regras

- **Empréstimo:** leitor ativo + exemplar disponível + sem atraso + abaixo do limite do tipo.
- **Renovação:** sem atraso + abaixo do máximo de renovações do tipo.
- **Atraso:** não devolvido e data prevista anterior a hoje → alerta e bloqueio.
- **Desbloqueio:** automático quando não resta item atrasado.
- **Sem multa financeira** na versão 1.

---

## 🚀 Plano de desenvolvimento por fases

Cada fase termina com uma entrega funcional e testável. Não avance de fase sem cumprir os **critérios de aceite**.

---

### Fase 0 — Preparação do ambiente

**Objetivo:** deixar o projeto pronto para desenvolver sem retrabalho.

**Tarefas**
- Criar o repositório Git, o `.gitignore` e a estrutura de pastas.
- Criar o ambiente virtual Python e instalar FastAPI, Uvicorn, SQLAlchemy, Alembic, Pydantic, passlib e pytest.
- Criar o projeto React com Vite e instalar React Router.
- Definir convenções: nomes de arquivos, formato de commits e estilo de código (ruff/black no Python, ESLint/Prettier no React).
- Escrever o README inicial com instruções de execução.

**Entregáveis:** repositório organizado; `uvicorn` sobe um endpoint `/api/saude`; `npm run dev` abre uma página inicial.

**Critérios de aceite**
- Front e back sobem em desenvolvimento sem erros.
- O front consegue chamar o endpoint de saúde do back-end.

---

### Fase 1 — Fundação: banco, configuração e login

**Objetivo:** ter a base técnica e a autenticação do administrador funcionando.

**Tarefas**
- Implementar `core/config.py` com os caminhos de dados (`~/.local/share/biblioteca`), criando a pasta se não existir.
- Implementar `core/database.py`: engine do SQLite com **WAL** e **chaves estrangeiras** ativas.
- Configurar o **Alembic** e criar a primeira migração (`usuario_admin`, `tipo_leitor`).
- Popular os três tipos de leitor (aluno, professor, público) com as políticas iniciais.
- Implementar `security.py`: hash de senha e geração/validação de token com expiração.
- Criar o fluxo de **primeiro acesso** (definir a senha do administrador quando não existir usuário).
- Criar os endpoints `POST /api/auth/login` e `POST /api/auth/trocar-senha`.
- Criar a dependência que protege todas as rotas, exceto o login.
- Front: tela de login, armazenamento do token e rotas protegidas; layout base com menu lateral.

**Entregáveis:** login funcionando de ponta a ponta; banco criado e versionado por migrações.

**Critérios de aceite**
- Sem login, nenhuma rota da API (exceto login) responde dados.
- Senha nunca é gravada em texto puro.
- Rodar as migrações em um banco vazio produz o esquema esperado.
- Testes de autenticação passando.

---

### Fase 2 — Cadastros: acervo e leitores

**Objetivo:** permitir cadastrar e consultar tudo o que alimenta os empréstimos.

**Tarefas**
- Migração e modelos de `obra`, `exemplar` e `leitor`.
- Repositories, services e routers de **obras**, **exemplares** e **leitores** (criar, listar com busca e paginação, editar, inativar).
- Validações: código de barras único, ISBN com formato válido (quando informado), matrícula única, campos obrigatórios por tipo de leitor (alunos exigem ano e sala).
- **Inativar em vez de excluir** registros que já tiveram movimentação, para preservar o histórico.
- Tela de **políticas por tipo de leitor** (editar prazo, limite e renovações).
- Front: telas **Acervo** e **Leitores** com tabela, busca, formulários em modal e mensagens de erro claras.

**Entregáveis:** cadastro completo de obras, exemplares, leitores e políticas.

**Critérios de aceite**
- Não é possível cadastrar dois exemplares com o mesmo código de barras.
- Obra com exemplares emprestados não pode ser inativada.
- Busca retorna resultados corretos e paginados.
- Testes dos services de cadastro passando.

---

### Fase 3 — Núcleo do sistema: empréstimos, devolução e atrasos

**Objetivo:** implementar as regras centrais da biblioteca. Esta é a fase mais importante e a mais testada.

**Tarefas**
- Migração e modelo de `emprestimo` (incluindo `ano_leitor` e `sala_leitor`).
- `emprestimo_service`: **emprestar**, **devolver** e **renovar**, com todas as validações descritas em [Regras de negócio](#-regras-de-negócio).
- `atraso_service`: calcular atrasos, listar leitores em atraso e informar o bloqueio.
- Executar cada operação em **transação**, para que o status do exemplar e o registro do empréstimo nunca fiquem inconsistentes.
- Mensagens de recusa específicas (ex.: "Leitor atingiu o limite de 3 itens", "Leitor possui 2 itens em atraso").
- Endpoints de empréstimo, devolução, renovação e lista de atrasados.
- Front: tela de **balcão** com campo de leitura de código de barras (foco automático), busca de leitor, lista de empréstimos em aberto, botões de devolver e renovar.
- **Painel (dashboard):** empréstimos do dia, itens a vencer e alerta de atrasos.
- Destaque visual para leitores em atraso/bloqueados.

**Entregáveis:** fluxo completo de balcão funcionando.

**Critérios de aceite**
- Leitor em atraso **não consegue** novo empréstimo.
- Devolver o último item atrasado **libera** o leitor automaticamente.
- Os limites e prazos mudam conforme o tipo de leitor.
- Renovação é recusada para item atrasado e quando o máximo é atingido.
- Cobertura de testes alta em `emprestimo_service` e `atraso_service` (casos de borda: prazo vencendo hoje, limite exato, exemplar emprestado, leitor inativo).

---

### Fase 4 — Importação do acervo

**Objetivo:** permitir carregar um acervo já existente sem digitar tudo.

**Tarefas**
- Definir o **layout do arquivo** (colunas: título, autor, editora, ano, ISBN, categoria, tipo de material, código de barras/tombo, quantidade de exemplares).
- Disponibilizar um **arquivo-modelo** para download (CSV e XLSX).
- `importacao_service`: leitura de CSV (com detecção de separador e codificação UTF-8/Latin-1) e XLSX (openpyxl).
- Validação linha a linha, gerando lista de erros com número da linha e motivo.
- Endpoint de **prévia** (não grava) e endpoint de **confirmação** (grava em transação única).
- Evitar duplicidade: obras já existentes (por ISBN ou título + autor) recebem novos exemplares em vez de duplicar.
- Registrar o resultado em `log_importacao`.
- Front: tela de importação em três passos — enviar arquivo, ver prévia com erros, confirmar e ver resumo.

**Entregáveis:** importação de planilhas com prévia e relatório de erros.

**Critérios de aceite**
- Arquivo com erros mostra todos os problemas antes de gravar.
- Falha durante a gravação não deixa dados parciais.
- Importar o mesmo arquivo duas vezes não duplica obras.
- Limite de tamanho e tipo de arquivo validados.

---

### Fase 5 — Relatórios, estatísticas e exportação

**Objetivo:** transformar o histórico de empréstimos em informação útil.

**Tarefas**
- `relatorio_service` e `relatorio_repo` com consultas agregadas:
  - livros **mais emprestados** e **menos emprestados** (inclusive os nunca emprestados), por período;
  - **alunos com mais empréstimos**, com quantidade, filtros por **ano** e **sala**;
  - empréstimos por período (dia, mês, ano);
  - atrasos atuais e histórico.
- Criar **índices** no banco para as colunas usadas nos filtros (data, leitor, exemplar, ano, sala).
- `exportacao_service`: gerar **CSV** (UTF-8 com BOM, para abrir corretamente no LibreOffice/Excel) e **XLSX** formatado (cabeçalho, largura de colunas).
- Front: tela de **Relatórios** com filtros, tabelas, gráficos simples (Recharts) e botões **Exportar CSV/XLSX**.

**Entregáveis:** relatórios com filtros e exportação.

**Critérios de aceite**
- Os números batem com dados de teste conhecidos.
- Trocar o aluno de sala **não altera** as estatísticas de empréstimos passados.
- Arquivos exportados abrem sem erros de acentuação.
- Relatórios continuam rápidos com dezenas de milhares de empréstimos.

---

### Fase 6 — Backup e restauração

**Objetivo:** garantir que os dados não se percam.

**Tarefas**
- `backup_service` usando a **API de backup do SQLite** (nunca copiar o arquivo diretamente).
- Nome dos arquivos com data e hora (ex.: `biblioteca_2026-10-26_1430.db`).
- Validação de integridade da cópia (`PRAGMA integrity_check`).
- Detectar se o HDD/pasta de destino está disponível e avisar de forma clara quando não estiver.
- **Backup agendado:** ao fechar o aplicativo e/ou diariamente (configurável), com política de retenção (manter os N backups mais recentes).
- **Restauração:** escolher um backup, validar o arquivo, criar uma cópia de segurança do banco atual e só então substituir.
- Registrar tudo em `log_backup` e exibir o histórico.
- Front: seção de backup em **Configurações** (escolher pasta, fazer backup agora, ver histórico, restaurar) e aviso no painel quando o último backup for antigo.

**Entregáveis:** backup manual e agendado, e restauração segura.

**Critérios de aceite**
- Backup gerado com o app aberto é válido e restaurável.
- Restaurar um arquivo inválido é recusado sem alterar o banco atual.
- Com o HDD desconectado, o sistema não trava e informa o problema.

---

### Fase 7 — Empacotamento como aplicativo Linux

**Objetivo:** entregar o sistema como um programa instalável, com ícone no menu.

**Tarefas**
- `desktop/launcher.py`: escolher porta livre, subir o FastAPI, **aguardar a porta responder** e abrir a janela PyWebView; encerrar o servidor ao fechar a janela (e disparar o backup de saída, se configurado).
- Script de build: `npm run build` → copiar para `backend/app/static` → PyInstaller em modo `--onedir`.
- Aplicar migrações automaticamente na inicialização.
- Criar o arquivo `biblioteca.desktop` e o ícone.
- Gerar o **pacote `.deb`** (declarando as dependências `python3-gi` e `gir1.2-webkit2-4.1`) e/ou o script `install.sh` (instala em `/opt/biblioteca` e registra o atalho).
- Tratar erros de inicialização com mensagem amigável e arquivo de log em `~/.local/share/biblioteca/logs`.
- Plano alternativo documentado: abrir com `chromium --app=http://127.0.0.1:PORTA` caso o WebKit apresente problemas.

**Entregáveis:** instalador `.deb` (ou script) e aplicativo abrindo pelo menu.

**Critérios de aceite**
- Instalação limpa em Linux sem Python/Node instalados para o usuário final.
- Atualizar a versão preserva o banco de dados.
- Fechar a janela encerra todos os processos.
- Aplicativo abre em poucos segundos.

---

### Fase 8 — Qualidade, testes finais e documentação

**Objetivo:** validar o sistema inteiro e deixá-lo pronto para uso real.

**Tarefas**
- Revisar a cobertura de testes e completar os casos de borda.
- Teste manual completo com roteiro: instalação → login → importação → cadastros → empréstimos/devoluções/renovações → atrasos → relatórios → backup → restauração.
- Teste de carga com dados fictícios (ex.: 20 mil obras e 100 mil empréstimos) para validar desempenho de busca e relatórios.
- Revisão de segurança (rotas protegidas, validação de entradas, senha).
- Revisar mensagens, textos e acessibilidade básica (navegação por teclado no balcão).
- Escrever o **manual do usuário** (passo a passo com capturas de tela) e um guia de **recuperação de desastre** (como restaurar em outro computador).
- Congelar a versão 1.0, criar a tag no Git e o pacote final.

**Entregáveis:** versão 1.0 estável, manual do usuário e guia de recuperação.

**Critérios de aceite**
- Nenhum erro crítico no roteiro de testes manual.
- Todos os testes automatizados passando.
- Um novo usuário consegue operar o balcão seguindo apenas o manual.

---

### Visão resumida das fases

| Fase | Foco | Resultado |
|---|---|---|
| 0 | Ambiente | Projeto pronto para desenvolver |
| 1 | Banco e login | Base técnica e autenticação |
| 2 | Cadastros | Acervo, leitores e políticas |
| 3 | Empréstimos e atrasos | Núcleo do sistema |
| 4 | Importação | Acervo carregado por planilha |
| 5 | Relatórios | Estatísticas e exportação |
| 6 | Backup | Segurança dos dados |
| 7 | Empacotamento | Aplicativo instalável no Linux |
| 8 | Qualidade | Versão 1.0 e documentação |

---

## 💻 Como rodar em desenvolvimento

**Pré-requisitos:** Python 3.11+, Node.js 20+, Git.

```bash
# 1. Clonar
git clone <url-do-repositorio> biblioteca
cd biblioteca

# 2. Back-end
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

# 3. Front-end (outro terminal)
cd frontend
npm install
npm run dev
```

O front em desenvolvimento acessa a API em `http://127.0.0.1:8000` (configure o proxy no `vite.config.js`).

---

## 📦 Como gerar e instalar o aplicativo no Linux

**Dependências do sistema (Debian/Ubuntu):**

```bash
sudo apt install python3-gi gir1.2-webkit2-4.1 python3-venv
```

**Gerar o executável:**

```bash
./desktop/build/build.sh      # compila o React e gera o executável com PyInstaller
```

**Instalar:**

```bash
sudo ./desktop/build/install.sh   # copia para /opt/biblioteca e cria o atalho no menu
# ou, se gerou o pacote:
sudo apt install ./biblioteca_1.0.0_amd64.deb
```

Depois é só abrir **Biblioteca** pelo menu de aplicativos.

---

## 🧪 Testes

```bash
# Back-end
cd backend && pytest

# Front-end
cd frontend && npm test
```

Prioridade de cobertura: regras de **empréstimo, renovação, devolução e atraso**, **importação**, **relatórios** e **backup/restauração**.

---

## 🔒 Segurança

- API escuta **somente em `127.0.0.1`**, em porta livre escolhida na inicialização.
- Senha do administrador armazenada com **hash** (argon2/bcrypt).
- Token de sessão com **expiração**; todas as rotas protegidas, exceto o login.
- Todas as entradas validadas com **Pydantic**.
- Arquivos importados validados por **tipo e tamanho**.

---

## 🔭 Fora do escopo e evoluções futuras

**Fora da versão 1:** multas financeiras, reservas, autoatendimento de leitores, acesso por rede.

**Ideias para depois:**
- Multas e reservas de obras.
- Perfis adicionais (atendente) e auditoria de ações.
- Consulta pública do catálogo.
- Operação em rede com vários computadores (migrando para PostgreSQL).
- Impressão de carteirinhas e etiquetas de código de barras.

---

## 📄 Licença

Defina a licença do projeto (ex.: MIT) antes de publicar.