# DocFlow — Plataforma SaaS de Gestão de Documentos Empresariais
### Documento de arquitetura técnica

Este documento descreve a arquitetura recomendada para colocar a plataforma em produção com isolamento rigoroso entre empresas (multi-tenant), armazenamento seguro e caminho claro para as evoluções futuras (OCR avançado, assinatura digital, alertas de vencimento e cobrança).

Junto com este documento, foi entregue um **protótipo funcional interativo** (`docflow-saas.html`) que implementa, no navegador, todo o fluxo de uso: login por empresa, upload, o módulo de escaneamento de documentos (detecção de bordas, correção de perspectiva, múltiplas páginas, OCR, geração de PDF), organização por categorias/pastas, busca, visualização/download/exclusão e um painel administrativo. Ele usa armazenamento local do navegador para simular o banco de dados e já segrega os dados por "empresa" com as mesmas chaves que serão usadas no backend real — é a referência viva da experiência que o backend abaixo precisa sustentar, não o backend em si.

---

## 1. Princípio central: isolamento por empresa (multi-tenancy)

Isolamento de dados entre clientes **nunca deve depender apenas do código da aplicação**. A recomendação é aplicar isolamento em três camadas independentes, para que uma falha em uma camada não exponha dados de outra empresa:

1. **Banco de dados — Row-Level Security (RLS) no PostgreSQL.**
   Toda tabela que guarda dado de cliente tem uma coluna `company_id`. Uma política RLS garante que toda consulta só enxerga linhas onde `company_id = current_setting('app.current_company_id')`. Essa variável de sessão é definida pelo backend logo após validar o token do usuário, a cada requisição — nunca vem do cliente.

2. **Camada de aplicação — filtro explícito por tenant.**
   Todo repositório/serviço recebe `company_id` do contexto autenticado (nunca do corpo da requisição) e o aplica em toda query, mesmo com RLS ativo. É a defesa redundante contra bugs de configuração do banco.

3. **Armazenamento de arquivos — particionamento físico por empresa.**
   Caminho do objeto: `s3://docflow-files/{company_id}/{document_id}/{filename}`. Cada download é servido por URL assinada (curta duração, ex. 5 min), gerada só depois de checar que o `company_id` do documento bate com o do usuário autenticado. Nunca expor URLs públicas permanentes.

Um usuário "Administrador da plataforma" (equipe DocFlow) é um papel à parte, com acesso auditado a metadados agregados (não ao conteúdo dos arquivos por padrão), usado apenas pelo painel administrativo descrito na seção 6.

---

## 2. Modelo de dados

```
companies
  id, name, cnpj, plan, status (active/suspended),
  storage_used_bytes, storage_limit_bytes, created_at

users
  id, company_id (FK), name, email, password_hash,
  role (company_admin | member), status, created_at

categories
  id, company_id (FK, null = padrão do sistema), name,
  color, is_system_default, created_at

folders
  id, company_id (FK), parent_folder_id (nullable), name, created_at

documents
  id, company_id (FK), folder_id (nullable), category_id (FK),
  uploaded_by (FK users), file_name, storage_path, mime_type,
  size_bytes, page_count, ocr_text, ocr_status,
  expires_at (nullable — usado no futuro alerta de vencimento),
  signature_status (nullable — usado na futura assinatura digital),
  deleted_at (soft delete), created_at, updated_at

audit_logs
  id, company_id, user_id, action, target_type, target_id,
  ip_address, created_at

subscriptions   (preparado para cobrança futura)
  id, company_id, plan, status, provider_customer_id,
  current_period_end
```

`ocr_text` é indexado com `tsvector` do PostgreSQL (busca full-text em português), permitindo pesquisar pelo **conteúdo** do documento digitalizado, não só pelo nome do arquivo.

---

## 3. Stack recomendada

| Camada | Tecnologia | Motivo |
|---|---|---|
| Frontend | Next.js (React) + Tailwind | mesma base de componentes do protótipo entregue, SSR facilita SEO da área de marketing/login |
| Backend/API | Node.js (NestJS) ou Next API Routes | tipagem compartilhada com o frontend |
| Banco de dados | PostgreSQL (RLS ativado) | isolamento nativo por tenant, full-text search |
| Armazenamento de arquivos | S3 / Backblaze B2 (compatível S3) | criptografia SSE-KMS, versionamento, ciclo de vida |
| Fila/jobs assíncronos | Redis + BullMQ | processar OCR e miniaturas fora do request principal |
| OCR em produção | Tesseract self-hosted (custo zero) ou AWS Textract/Google Document AI (maior precisão) | o protótipo já usa Tesseract.js no navegador como prova de conceito |
| Autenticação | JWT (access + refresh) com claims `company_id`, `role`, `user_id` | validado em toda requisição, nunca confiar em claims vindas do cliente sem verificar assinatura |
| Cobrança futura | Stripe Billing | webhooks atualizam `subscriptions` e `companies.plan` |
| Infra | Docker + um provedor gerenciado (Render, Fly.io, AWS ECS) | simples de operar para um time pequeno |

---

## 4. Upload e o módulo "Escanear documento"

Fluxo do scanner (implementado no protótipo, ponta a ponta):

1. Usuário aciona **Escanear documento** (celular ou desktop com webcam).
2. Cada página é capturada pela câmera nativa do aparelho.
3. O sistema sugere automaticamente o recorte da folha (heurística de contraste de bordas) e desenha 4 alças ajustáveis — o usuário pode corrigir manualmente ângulo, rotação e enquadramento quando a sugestão não for perfeita.
4. Correção de perspectiva (transformação projetiva) endireita a folha como se fotografada de frente.
5. Realce automático: contraste, brilho e nitidez ajustáveis; modo "documento" (preto e branco de alto contraste) para reduzir sombras.
6. A página processada entra numa fila visual de páginas: dá para reordenar, excluir, refazer (tirar de novo) ou adicionar mais páginas antes de finalizar.
7. Ao concluir, todas as páginas são unidas em um único PDF.
8. OCR roda sobre cada página e o texto reconhecido é salvo junto ao documento, tornando-o pesquisável.
9. Tela de revisão final antes de salvar: nome do documento, categoria, pasta e data.

**Caminho de evolução:** trocar a heurística de bordas do protótipo por um modelo de detecção de documento mais robusto (ex. OpenCV.js/WASM ou um modelo de visão treinado) é uma troca isolada no módulo de captura — o resto do pipeline (fila de páginas, geração de PDF, OCR, metadados) não muda.

---

## 5. Preparado para o futuro (sem redesenho)

O schema e a API já reservam os pontos de extensão:

- **OCR avançado / leitura automática de campos:** coluna `ocr_text` já existe; adicionar `extracted_fields JSONB` para dados estruturados (CPF/CNPJ, datas, valores) extraídos por um modelo depois do OCR bruto.
- **Assinatura digital:** `documents.signature_status` + tabela `signature_requests` (signatários, status por signatário, provedor — ex. integração com Clicksign/DocuSign/ICP-Brasil).
- **Alertas de vencimento:** `documents.expires_at` já existe; um job diário verifica documentos próximos do vencimento e dispara notificação (e-mail/painel).
- **Cobrança por assinatura:** tabela `subscriptions` + webhook Stripe; `companies.storage_limit_bytes` e contagem de usuários já dão a base para planos com limites.

---

## 6. Painel administrativo (visão da equipe DocFlow)

Acesso restrito a um papel `platform_admin`, separado dos papéis de empresa, com:

- Lista de empresas: plano, status (ativa/suspensa), armazenamento usado vs. limite, data de criação.
- Drill-down por empresa: usuários, quantidade de documentos, categorias criadas.
- Ação de suspender/reativar uma empresa (bloqueia login, mantém dados).
- Métricas agregadas: armazenamento total da plataforma, empresas ativas, crescimento de uploads.
- Todo acesso do `platform_admin` a conteúdo de documento (não apenas metadados) fica registrado em `audit_logs`, porque é acesso sensível entre tenants e precisa de rastreabilidade.

---

## 7. Segurança — checklist de produção

- Senhas com hash Argon2id (nunca texto puro nem hash reversível).
- MFA opcional para `company_admin`.
- Rate limiting e bloqueio de força bruta no login.
- Upload validado por tipo real do arquivo (magic bytes), não só pela extensão.
- Verificação antivírus assíncrona antes de liberar o arquivo para outros usuários da empresa.
- Backups automáticos do banco (point-in-time recovery) e replicação do bucket de arquivos.
- Logs de auditoria imutáveis para toda ação sensível (login, download, exclusão, mudança de permissão).
- Testes automatizados específicos de isolamento: para cada endpoint, um teste tenta acessar um recurso de outra empresa e espera 403/404.
