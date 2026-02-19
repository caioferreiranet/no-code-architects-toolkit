# Plano: Adicionar timestamps e tempo decorrido ao job status

## Contexto

O endpoint `POST /v1/toolkit/job/status` retorna o conteudo do arquivo JSON de status do job, mas nao inclui nenhum timestamp de quando a execucao comecou nem o tempo decorrido. O objetivo e adicionar `started_at` (ISO 8601), `completed_at` (ISO 8601) e `elapsed_time` (segundos) para que o usuario saiba quando o processamento iniciou e quanto tempo ja passou ou levou.

## Abordagem

Passar os timestamps via data dict em cada chamada `log_job_status()`, em vez de ler o arquivo anterior (evita I/O extra e race conditions). O `elapsed_time` e calculado dinamicamente no endpoint de consulta.

## Alteracoes

### 1. `app.py` — Adicionar import e timestamps nos 6 call sites relevantes

**Import** (linha 25, junto ao `import time`):
```python
from datetime import datetime, timezone
```

**3 code paths "running" → adicionar `started_at`:**

| Linhas | Code path | Escopo da variavel |
|--------|-----------|-------------------|
| 48-55 | Fila (process_queue) | `started_at` usada na mesma iteracao do `while True` |
| 112-119 | Cloud Run Job | `started_at` usada no mesmo bloco `if CLOUD_RUN_JOB` |
| 247-254 | Sync/bypass | `started_at` usada no mesmo bloco `elif` |

Em cada um: criar `started_at = datetime.now(timezone.utc).isoformat()` antes do `log_job_status` e incluir `"started_at": started_at` no dict.

**3 code paths "done" → adicionar `started_at` + `completed_at`:**

| Linhas | Code path |
|--------|-----------|
| 78-84 | Fila (process_queue) |
| 143-149 | Cloud Run Job |
| 276-282 | Sync/bypass |

Em cada um: criar `completed_at = datetime.now(timezone.utc).isoformat()` antes do `log_job_status` e incluir `"started_at": started_at, "completed_at": completed_at` no dict.

**Sem alteracao** nos status `queued`, `submitted`, `failed` e `done` (queue overflow 429) — nesses casos o job nao chegou a executar.

### 2. `routes/v1/toolkit/job_status.py` — Calcular `elapsed_time` dinamicamente

Adicionar import `from datetime import datetime, timezone` no topo do arquivo.

Apos ler o JSON do arquivo, antes de retornar:
- Se `started_at` existe e `job_status == "running"`: `elapsed_time = now - started_at`
- Se `started_at` e `completed_at` existem: `elapsed_time = completed_at - started_at`
- Adicionar `elapsed_time` (arredondado a 3 casas) ao dict de resposta

### Arquivos modificados

- `app.py` — 1 import + 6 blocos `log_job_status` alterados
- `routes/v1/toolkit/job_status.py` — import + calculo dinamico de `elapsed_time`

### Arquivos NAO modificados

- `app_utils.py` — `log_job_status()` permanece inalterada
- `routes/v1/toolkit/jobs_status.py` — endpoint batch nao retorna timing
- Nenhum service/route de processamento precisa ser alterado

## Formato da resposta apos implementacao

**Job running:**
```json
{
  "job_status": "running",
  "job_id": "abc-123",
  "started_at": "2026-02-19T14:30:00.123456+00:00",
  "elapsed_time": 12.345,
  "response": null
}
```

**Job done:**
```json
{
  "job_status": "done",
  "job_id": "abc-123",
  "started_at": "2026-02-19T14:30:00.123456+00:00",
  "completed_at": "2026-02-19T14:30:05.678901+00:00",
  "elapsed_time": 5.555,
  "response": { "..." : "..." }
}
```

**Job queued (sem alteracao):**
```json
{
  "job_status": "queued",
  "job_id": "abc-123",
  "response": null
}
```

## Compatibilidade

- Jobs criados antes da mudanca nao tem `started_at` → endpoint retorna sem `elapsed_time` (comportamento identico ao atual)
- Nenhum campo existente e removido ou renomeado

## Verificacao

1. Iniciar servidor local: `python app.py`
2. Enviar request com `webhook_url` para enfileirar job
3. Consultar `POST /v1/toolkit/job/status` com o `job_id` retornado
4. Verificar que `started_at` aparece quando status e "running"
5. Verificar que `started_at`, `completed_at` e `elapsed_time` aparecem quando status e "done"
6. Enviar request sem `webhook_url` (sync) e verificar que o status "done" tambem tem os campos
