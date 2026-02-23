# Prova Técnica — Pipeline End-to-End (ANA → Tratamento → Agendamento → API → Análise)

Este repositório é uma prova em formato **Git (fork + PR)**.

Você vai implementar e/ou corrigir um pipeline completo:

1) **Extract** (ANA Medição SIN — snapshot determinístico e live best-effort)  
2) **Transform** (parsing/normalização/validação/dedup)  
3) **Load** (persistência em SQLite + idempotência)  
4) **Schedule** (rotina recorrente “cron-like” dentro do container, com frequência consistente)  
5) **Serve** (FastAPI para consulta + gatilho)  
6) **Analyze** (métricas simples e interpretação)

Fonte pública:
- https://www.ana.gov.br/sar0/MedicaoSin

---

## Regras importantes (scraping responsável)
- **Não é permitido** contornar login, CAPTCHA, WAF/Cloudflare ou anti-bot.
- Use `timeout`, `User-Agent`, `retry/backoff` para 429/5xx e **rate limit** (ex.: 1 req/s).
- O modo `live` pode falhar por razões externas; o código deve retornar erro claro e continuar operável.

## Reprodutibilidade
Os testes oficiais rodam em **snapshot local** (arquivo `data/ana_snapshot.html`) para não depender do site.

---

# Como rodar

## Local
```bash
python -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
pytest -q

uvicorn app.api.main:app --reload --port 8000
# http://localhost:8000/docs
```

## Docker
```bash
docker compose up --build
# API em http://localhost:8000/docs

docker compose run --rm tests
```

O scheduler roda no serviço `scheduler` e grava em `data/out/ana.db`.

---

# Variáveis de ambiente

- `APP_DATA_DIR` (default: `data`) — onde ficam snapshot e outputs
- `ANA_MODE` (`snapshot|live`, default: `snapshot`)
- `PIPELINE_INTERVAL_SECONDS` (default: `60`) — frequência do job (scheduler)
- `ANA_RESERVATORIO` (default: `19091`)
- `ANA_DATA_INICIAL` (default: `2025-10-01`) — ISO
- `ANA_DATA_FINAL` (default: `2026-02-22`) — ISO

---

# Questões (7)

## Q1 — Bugfix: parsing de datas e números pt-BR (legado)
Arquivo: `src/app/core/parsing.py` + `tests/test_parsing.py`

Corrigir parsing de:
- datas com/sem hora; ISO com `T`
- números pt-BR com milhar/decimal
- tokens ausentes (`""`, `—`, `NaN`, `inf`)

Adicione **3 testes** cobrindo edge cases.

## Q2 — Refatoração: I/O + logging + responsabilidades
Arquivo: `src/app/core/io_utils.py`

Separar leitura/validação/transformação e substituir `print` por `logging`.

## Q3 — Normalização + validação de schema
Arquivo: `src/app/core/transforms.py`

Implementar `normalize_record()` e `validate_record()` e adicionar **5 testes**.

## Q4 — Performance: dedupe/lookup O(n)
Arquivo: `src/app/core/transforms.py`

Substituir o dedupe lento O(n²) por O(n) e comentar trade-off em 2–3 linhas.

## Q5 — Scraping ANA: client + parser robustos
Arquivos: `src/app/ana/client.py`, `src/app/ana/parser.py`, `tests/test_ana_parser.py`

- Montar URL correta (`button=Buscar`, datas DD/MM/YYYY, reservatório)
- `fetch_ana_html()` com retry/backoff 429/5xx + rate limit
- `parse_ana_records()` tolerante a pequena variação de headers

## Q6 — Pipeline idempotente + persistência (SQLite)
Arquivo: `src/app/core/storage.py`, `tests/test_storage.py`

Persistir registros em SQLite (`data/out/ana.db`) com chave natural `record_id`.
`upsert_many()` deve retornar `{inserted, existing}` e ser idempotente.

## Q7 — Agendamento + API + Análise
- Scheduler: `src/app/jobs/scheduler.py` (frequência consistente, não “driftar”)
- Job: `src/app/jobs/extract_job.py` (executa 1 rodada: extract→transform→load)
- API: `src/app/api/main.py` (consulta + gatilho `/extract/ana`)
- Análise: `src/app/analysis/ana_analysis.py` + endpoint `/ana/analysis`

---

## Entrega
- Fork + PR
- Descrever decisões técnicas e como rodar
- (Opcional) melhorias adicionais, desde que justificadas

Boa Sorte e Boa Prova!
