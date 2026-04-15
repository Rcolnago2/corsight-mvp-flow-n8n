# POC n8n — Checagem Facial com FTP + Corsight Fortify

Este repositório contém um workflow **importável no n8n** para prova de conceito (POC) de checagem facial.

- Arquivo principal do workflow: `corsight_ftp_face_check_poc.n8n.json`
- Fluxo: FTP (entrada de imagens) → Detect/Analyze/Compare (Corsight) → Classificação → Movimentação no FTP

---

## 1) O que este workflow faz

Em alto nível, para cada imagem da pasta de entrada no FTP, o fluxo:

1. Autentica na API do Corsight e obtém token Bearer.
2. Lista arquivos no FTP e filtra imagens (`.jpg`, `.jpeg`, `.png`).
3. Baixa a imagem e converte para Base64.
4. Chama `detect` para detectar face(s) e obter crop(s).
5. Seleciona automaticamente a melhor detecção:
   - maior `detection_score`
   - empate: maior área de bounding box
6. Usa o **crop retornado pelo detect** (não a imagem original).
7. Chama `analyze` com `detect=false` para validar qualidade do crop.
8. Se `valid_face=true`, compara (`compare`) contra referências Base64 (POC).
9. Consolida score final e classifica:
   - `strong_match` (>= 85)
   - `manual_review` (>= 75 e < 85)
   - `no_match` (< 75)
10. Move o arquivo para a pasta FTP apropriada e gera resumo final do item.

Também há ramificações explícitas para:
- falha de login
- erro de API
- sem face detectada
- face inválida

---

## 2) Pré-requisitos

Antes de importar, garanta:

- n8n em execução (versão recente, com nós padrão: Cron, Manual Trigger, HTTP Request, FTP, IF, Code, Merge, SplitInBatches).
- Acesso de rede do n8n ao servidor FTP e às APIs do Corsight.
- Pastas FTP existentes (ou política para criação) para entrada e saídas.
- Variáveis de ambiente configuradas no ambiente onde o n8n roda.

---

## 3) Variáveis de ambiente obrigatórias

> O workflow **não hardcode** credenciais sensíveis. Tudo usa `{{$env.*}}`.

Configure as seguintes variáveis:

### Corsight
- `CORSIGHT_BASE_URL_AUTH`
- `CORSIGHT_BASE_URL_POI`
- `CORSIGHT_USERNAME`
- `CORSIGHT_PASSWORD`

### FTP
- `FTP_HOST`
- `FTP_PORT`
- `FTP_USER`
- `FTP_PASSWORD`
- `FTP_INPUT_PATH`
- `FTP_PROCESSED_PATH`
- `FTP_MATCH_PATH`
- `FTP_REVIEW_PATH`
- `FTP_NOFACE_PATH`
- `FTP_ERROR_PATH`

### Exemplo (apenas ilustrativo)

```bash
CORSIGHT_BASE_URL_AUTH=https://SEU_AUTH_HOST
CORSIGHT_BASE_URL_POI=https://SEU_POI_HOST
CORSIGHT_USERNAME=SEU_USUARIO
CORSIGHT_PASSWORD=SUA_SENHA

FTP_HOST=ftp.seudominio.com
FTP_PORT=21
FTP_USER=seu_usuario_ftp
FTP_PASSWORD=sua_senha_ftp
FTP_INPUT_PATH=/input
FTP_PROCESSED_PATH=/processed
FTP_MATCH_PATH=/match
FTP_REVIEW_PATH=/review
FTP_NOFACE_PATH=/noface
FTP_ERROR_PATH=/error
```

---

## 4) Como importar no n8n

1. Abra seu n8n.
2. Vá em **Workflows** → **Import from File**.
3. Selecione `corsight_ftp_face_check_poc.n8n.json`.
4. Salve o workflow.
5. Verifique se os nós importaram sem erro de versão/tipo.

---

## 5) Ajustes obrigatórios após importar

### 5.1 Referências faciais (POC)
No nó **Load Reference Faces**, substitua:
- `BASE64_REFERENCE_1`
- `BASE64_REFERENCE_2`

por Base64 real das faces de referência da Pessoa de Interesse.

> Para este POC, as referências estão no próprio workflow (sem consulta dinâmica a POI DB).

### 5.2 Endpoint de login
O nó **Corsight Login** usa `POST {{$env.CORSIGHT_BASE_URL_AUTH}}/login`.

Se sua instalação Corsight usar outro path/corpo de autenticação, ajuste este nó.

### 5.3 Estrutura de resposta do detect/analyze/compare
O workflow foi escrito para formato comum de resposta (`statusCode`, `body`, `match_confidence`, `valid_face`, etc.).

Se sua API retornar campos diferentes, ajuste os nós Code:
- `Check Detections + Select Best Detection`
- `Check Face Validity`
- `Aggregate Scores + Classify Result`

---

## 6) Como executar

Você pode executar de dois modos:

- **Manual**: use o nó **Manual Trigger** (ideal para testes iniciais).
- **Automático**: deixe o **Cron Trigger** ativo (a cada 5 minutos).

Recomendação de validação inicial:

1. Coloque 3–5 imagens conhecidas em `FTP_INPUT_PATH`.
2. Rode manualmente uma execução.
3. Verifique no painel de execução:
   - Detecção e seleção de crop
   - `valid_face`
   - `best_score`
   - `classification`
4. Valide se os arquivos foram movidos para as pastas esperadas.

---

## 7) Regras de classificação implementadas

- `best_score >= 85` → `strong_match` → move para `FTP_MATCH_PATH`
- `75 <= best_score < 85` → `manual_review` → move para `FTP_REVIEW_PATH`
- `best_score < 75` → `no_match` → move para `FTP_PROCESSED_PATH`

Casos especiais:
- sem face detectada → `FTP_NOFACE_PATH`
- erro ou face inválida → `FTP_ERROR_PATH`

---

## 8) Troubleshooting rápido

### Erro de login no Corsight
- Verifique `CORSIGHT_BASE_URL_AUTH`, usuário e senha.
- Confirme path `/login` da sua instância.
- Confira firewall/proxy entre n8n e Corsight.

### Não lista arquivos do FTP
- Verifique host, porta, credenciais e permissões da pasta de entrada.
- Confira se `FTP_INPUT_PATH` existe.

### Arquivos não são movidos
- Verifique permissões de rename/move no FTP para as pastas destino.
- Confirme se as pastas destino existem.

### Sempre cai em erro no detect/analyze/compare
- Capture a resposta HTTP no histórico da execução.
- Compare os campos retornados com os esperados pelos nós Code.
- Ajuste o parse dos campos em cada nó Code conforme seu payload real.

---

## 9) Boas práticas para produção (próximos passos)

Para evoluir além da POC:

- Trocar referências estáticas por busca dinâmica em base de POIs.
- Persistir logs em banco (ex.: Postgres) para auditoria.
- Adicionar alertas (Slack/Email/Webhook) para `manual_review` e `error`.
- Incluir controle de duplicidade/idempotência de arquivo.
- Incluir retentativas com backoff para falhas transitórias de API/FTP.
- Adicionar trilha de métricas (tempo por etapa, taxa de erro, throughput).

---

## 10) Resumo final

Este workflow foi desenhado para ser:
- **didático** (nomes claros + fluxo modular),
- **operacional** (processamento item a item + movimentação FTP),
- **seguro para POC** (sem segredos hardcoded),
- **fácil de evoluir** para cenário real de produção.

Se quiser, na próxima iteração posso fornecer também:
- versão com `Continue On Fail` + retry controlado por nó,
- versão com gravação em banco,
- versão com múltiplas pessoas de interesse e ranking top-N.
