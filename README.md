# 🎬 CineZap-Bot

Bot de Telegram que mostra os **filmes em cartaz**, **sessões, horários e preços** dos cinemas da sua cidade, além de **trailer**, **nota do IMDb** e **bilheteria** de cada filme, tudo navegando por botões, sem digitar nada.

Os dados são coletados automaticamente por scrapers (Ingresso.com, IMDb, YouTube e Box Office Mojo), armazenados no **Supabase (PostgreSQL)** e consumidos pelo bot.

> Atualmente a cidade suportada é **Uberlândia - MG**, mas a arquitetura já é preparada para múltiplas cidades.

---

## 📋 Sumário

- [Funcionalidades](#-funcionalidades)
- [Como funciona (arquitetura)](#-como-funciona-arquitetura)
- [Tecnologias e bibliotecas](#-tecnologias-e-bibliotecas)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Modelo de dados](#-modelo-de-dados)
- [Como executar](#-como-executar)
- [Como foi feito](#-como-foi-feito)
- [Competências desenvolvidas](#-competências-desenvolvidas)
- [Limitações conhecidas e próximos passos](#-limitações-conhecidas-e-próximos-passos)

---

## ✨ Funcionalidades

### Bot do Telegram

- **Menu interativo com botões inline**: toda a navegação acontece editando a mesma mensagem, sem poluir o chat.
- **Escolha de cidade** a partir do comando `/start`.
- **Lista de filmes em cartaz** na cidade escolhida, em ordem alfabética.
- **Ficha do filme**: sinopse, duração, classificação indicativa e gêneros.
- **Sessões em 4 passos**: data → tipo de sessão (Dublado, Legendado, XD, VIP...) → cinema e horário → link direto de compra.
- **Preço da inteira** exibido no próprio botão de cada horário.
- **Trailer**: link do YouTube com pré-visualização grande do vídeo no chat.
- **Curiosidades**: diretor, elenco principal, nota e popularidade no IMDb e bilheteria (Box Office Mojo).
- **Botão "Voltar"** em todas as telas.
- **Filtro inteligente de sessões**: oculta sessões que começaram há mais de 15 minutos.
- **Conversão de fuso horário**: horários guardados em UTC e exibidos em Brasília (UTC-3).

### Coleta de dados (scrapers)

- Lista de filmes em cartaz via **Playwright** no Ingresso.com.
- Dados estruturados do filme lidos do **JSON-LD** embutido na página.
- Sessões, tipos e preços obtidos pelas **APIs internas** do Ingresso.com (mais rápido e estável que raspar o HTML).
- Nota, popularidade e duração via **IMDb**.
- Primeiro trailer encontrado no **YouTube** e sua miniatura.
- Bilheteria no **Box Office Mojo**.
- **Persistência idempotente** (upsert): rodar o scraper várias vezes não duplica filmes, cinemas nem sessões.
- **Limpeza prévia das sessões** da cidade antes de repovoar, evitando sessões antigas ou canceladas.

---

## 🧩 Como funciona (arquitetura)

O projeto é dividido em duas partes independentes que se comunicam apenas pelo banco de dados:

<img width="900" height="250" alt="grava" src="https://github.com/user-attachments/assets/1fd60d08-274c-4b04-8512-a87f2997f9ce" />



**Fluxo do scraper (`main.py`):**

1. Limpa as sessões da cidade.
2. Lista os filmes em cartaz.
3. Para cada filme: coleta dados do Ingresso.com e do IMDb → salva → busca trailer no YouTube → busca bilheteria no Box Office Mojo → atualiza o registro.

**Fluxo do bot:**

<img width="1140" height="140" alt="grava" src="https://github.com/user-attachments/assets/af62f9e9-4ac4-48bc-b398-7fba4f69b8a5" />

---

## 🛠 Tecnologias e bibliotecas

| Categoria | Tecnologia | Uso no projeto |
|---|---|---|
| Linguagem | **Python 3.10+** | Todo o projeto (usa a sintaxe `str \| None`) |
| Bot | **python-telegram-bot** (v22, assíncrono) | Handlers de comandos, botões inline e edição de mensagens |
| Banco de dados | **Supabase** (`supabase-py`) + PostgreSQL | Armazenamento e consultas com `upsert`, `in_`, `gte`, `order` |
| Scraping (navegador) | **Playwright** | Ingresso.com (páginas dinâmicas) e IMDb |
| Scraping (navegador) | **Selenium** + **webdriver-manager** | Busca de trailers no YouTube |
| Scraping (HTTP) | **Requests** + **BeautifulSoup4** | Box Office Mojo |
| APIs | **API interna do Ingresso.com** (via `requests`) | Cidades, eventos, datas, sessões e preços |
| Configuração | **python-dotenv** | Variáveis de ambiente (tokens e chaves) |
| Modelagem | **dataclasses** | Classe `Movie` para os dados do IMDb |
| Outros | `re`, `json`, `datetime`, `urllib.parse` | Regex, JSON-LD, fusos horários e URLs |

---

## 📁 Estrutura do projeto

```
CineZap-Bot/
├── main.py                     # Orquestra a coleta de dados
├── requirements.txt
├── .env                        # Variáveis de ambiente (não versionar!)
│
├── bot/
│   └── telegram_bot.py         # Bot do Telegram (handlers e menus)
│
├── database/
│   └── supabase_client.py      # Conexão e funções de persistência
│
└── scraper/
    ├── ingresso.py             # Ingresso.com (Playwright + API interna)
    ├── youtube.py              # Trailer no YouTube (Selenium)
    ├── mojo.py                 # Bilheteria (Requests + BeautifulSoup)
    ├── imdb/
    │   ├── __init__.py         # Função pública buscar_dados_imdb()
    │   ├── browser.py          # Criação/fechamento do navegador
    │   ├── imdb_search.py      # Busca o link do filme
    │   └── imdb_movie.py       # Extrai dados da página do filme
    └── models/
        └── movie.py            # Dataclass Movie
```

---

## 🗄 Modelo de dados

Três tabelas no Supabase. Os `on_conflict` usados no código definem as chaves únicas necessárias:

| Tabela | Campos principais | Chave única (upsert) |
|---|---|---|
| `filmes` | `id`, `titulo`, `sinopse`, `duracao`, `classificacao`, `diretor`, `generos[]`, `elenco[]`, `imagem_url`, `url_ingresso`, `imdb_id`, `nota_imdb`, `popularidade_imdb`, `trailer_url_youtube`, `imagem_url_youtube`, `bilheteria_mojo` | `titulo` |
| `cinemas` | `id`, `nome`, `cidade` | `nome, cidade` |
| `sessoes` | `id`, `filme_id`, `cinema_id`, `data_horario`, `tipo_sessao`, `preco_sem_taxa`, `preco_com_taxa`, `link_compra` | `filme_id, cinema_id, data_horario` |

<details>
<summary>Exemplo de SQL para criar as tabelas</summary>

```sql
create table filmes (
  id bigint generated always as identity primary key,
  titulo text unique not null,
  sinopse text,
  duracao text,
  classificacao text,
  diretor text,
  generos text[],
  elenco text[],
  imagem_url text,
  url_ingresso text,
  imdb_id text,
  nota_imdb text,
  popularidade_imdb text,
  trailer_url_youtube text,
  imagem_url_youtube text,
  bilheteria_mojo text
);

create table cinemas (
  id bigint generated always as identity primary key,
  nome text not null,
  cidade text not null,
  unique (nome, cidade)
);

create table sessoes (
  id bigint generated always as identity primary key,
  filme_id bigint references filmes(id) on delete cascade,
  cinema_id bigint references cinemas(id) on delete cascade,
  data_horario timestamptz not null,
  tipo_sessao text,
  preco_sem_taxa numeric,
  preco_com_taxa numeric,
  link_compra text,
  unique (filme_id, cinema_id, data_horario)
);
```

> Ajuste os tipos conforme o seu banco; este é só um ponto de partida baseado no que o código lê e grava.

</details>

---

## 🚀 Como executar

### Pré-requisitos

- Python 3.10 ou superior
- Google Chrome instalado (usado pelo Selenium)
- Um projeto no [Supabase](https://supabase.com) com as tabelas acima
- Um bot criado no Telegram via [@BotFather](https://t.me/BotFather)

### 1. Clonar e instalar

```bash
git clone https://github.com/athinags/CineZap-Bot.git
cd CineZap-Bot

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
playwright install chromium
```

### 2. Configurar variáveis de ambiente

Crie um arquivo `.env` na raiz:

```env
TELEGRAM_TOKEN=seu_token_do_botfather
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_KEY=sua_chave_do_supabase
```

> ⚠️ Nunca suba o `.env` para o GitHub. Adicione-o ao `.gitignore`.

### 3. Popular o banco (scraper)

```bash
python main.py
```

### 4. Iniciar o bot

Execute a partir da raiz do projeto, para que o import `database.supabase_client` funcione:

```bash
python -m bot.telegram_bot
```

Depois é só abrir o bot no Telegram e enviar `/start`.

---

## 🔨 Como foi feito

1. **Análise das fontes**: inspeção do Ingresso.com com as ferramentas do navegador, descobrindo que a página embute dados em **JSON-LD** e que o site consome **APIs internas** para sessões e preços.
2. **Scraper do Ingresso.com**: Playwright para renderizar a lista de filmes (conteúdo carregado via JavaScript) e `requests` nas APIs internas para as sessões, mais rápido e confiável que raspar HTML. O fluxo é *slug da cidade → ID da cidade*, *slug do filme → ID do evento*, *datas disponíveis* e, por fim, *sessões por data agrupadas por tipo*, com o preço de cada sessão em chamada separada.
3. **Enriquecimento dos dados**: IMDb (nota, popularidade, duração), YouTube (trailer e miniatura) e Box Office Mojo (bilheteria com regex em ordem de prioridade: worldwide → total → domestic → international).
4. **Modelagem do banco** no Supabase com chaves únicas para tornar a carga **idempotente** via `upsert`.
5. **Normalização de dados**: duração ISO 8601 (`PT2H53M`) convertida para texto legível (`2h 53min`), e remoção do tipo "Normal" dos rótulos de sessão (`["Normal", "Dublado"]` vira `Dublado`).
6. **Bot do Telegram**: cada tela é um handler assíncrono ligado por `CallbackQueryHandler` e padrões regex. O estado da navegação vai no `callback_data` (ex.: `tipo:12:2026-10-26:Dublado`) e em `context.user_data`.
7. **Tratamento de fuso horário**: timestamps em UTC no banco, convertidos para UTC-3 na hora de exibir.

---

## 🎓 Competências desenvolvidas

**Web scraping e automação**
- Raspagem de sites dinâmicos (JavaScript) com Playwright e Selenium.
- Raspagem de HTML estático com Requests e BeautifulSoup.
- Extração de dados estruturados (JSON-LD) e identificação de endpoints de APIs internas via DevTools.
- Uso de headers, User-Agent, timeouts e pausas para uma coleta mais robusta.
- Organização de scrapers em módulos independentes por fonte.

**Backend e banco de dados**
- Modelagem relacional (filmes, cinemas, sessões) com chaves estrangeiras e restrições de unicidade.
- Operações de `upsert`, filtros, ordenação e consultas com `IN` usando o Supabase.
- Cargas idempotentes e estratégia de limpeza/repovoamento de dados.

**Desenvolvimento de bots**
- Programação assíncrona (`async`/`await`) com python-telegram-bot.
- Design de UX conversacional com teclados inline e navegação hierárquica.
- Gerenciamento de estado via `callback_data` e `user_data`.

**Engenharia de software**
- Estruturação em pacotes e separação de responsabilidades (scraper, banco, bot).
- Tratamento de exceções com *fallbacks* (o scraper segue mesmo se uma fonte falhar).
- Type hints modernos e `dataclasses`.
- Gerenciamento de segredos com variáveis de ambiente e ambientes virtuais.

**Manipulação de dados**
- Expressões regulares para extração e limpeza de texto.
- Conversão de formatos (ISO 8601) e de fusos horários.
- Deduplicação de dados.

---

## 🔭 Limitações conhecidas e próximos passos

**Limitações atuais**
- Apenas Uberlândia está habilitada no menu do bot.
- A coleta é manual (`python main.py`), sem agendamento.
- Os scrapers dependem da estrutura dos sites; mudanças no HTML ou nas APIs internas podem quebrá-los.
- A busca por título no IMDb, no YouTube e no Box Office Mojo pega o primeiro resultado, o que pode gerar correspondências incorretas para títulos ambíguos.
- Os preços consideram apenas o ingresso **inteira**.
- O scraper abre um navegador novo por filme, o que deixa a coleta lenta.

**Ideias futuras**
- [ ] Agendar a coleta (cron, GitHub Actions ou similar).
- [ ] Adicionar mais cidades e detecção de localização.
- [ ] Notificações de estreias e de novas sessões.
- [ ] Favoritar filmes e cinemas.
- [ ] Reaproveitar uma única instância de navegador e paralelizar a coleta.
- [ ] Testes automatizados e logging estruturado.
- [ ] Deploy do bot (Railway, Render, VPS ou Docker) com *webhook* no lugar de *polling*.

---

## ⚖️ Aviso legal

Projeto desenvolvido **para fins educacionais e de portfólio**. Os dados pertencem às respectivas plataformas (Ingresso.com, IMDb, YouTube e Box Office Mojo). Verifique os termos de uso de cada site antes de usar scraping em larga escala ou fins comerciais.

---

## 👤 Autores

Feito por **[athinags](https://github.com/athinags)**, **[OCesarCardoso](https://github.com/OCesarCardoso)**, **[onlyjpx](https://github.com/onlyjpx)** e **[MatheusRRocha22](https://github.com/MatheusRRocha22)**.
