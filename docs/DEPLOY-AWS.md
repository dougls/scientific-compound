# Deploy na AWS (S3 + CloudFront)

Pipeline de deploy do SPA para **Amazon S3** com entrega via **CloudFront**.

## Visão geral

- **GitHub Actions** faz o build (Angular production) e publica no S3.
- O **CloudFront** serve o site e faz cache; cada deploy invalida o cache (`/*`).
- **Triggers:** push na branch `main` ou execução manual (_workflow_dispatch_).

## 1. Pré-requisitos na AWS

### 1.1 Bucket S3

1. Crie um bucket (ex.: `scientific-compound-app`).
2. **Block Public Access:** desbloqueie o acesso público (o CloudFront acessa via OAC ou Origin Access Identity).
3. **Política do bucket:** o CloudFront precisa ler os objetos. Se usar **Origin Access Control (OAC)**:
   - Em CloudFront → sua distribuição → Origins → editar a origem S3 → criar/editar OAC.
   - Em S3 → bucket → Permissions → Bucket policy: use a política sugerida pelo CloudFront ( “Copy policy from CloudFront” na configuração da origem).

Exemplo de bucket policy (substitua `BUCKET_NAME`, `DISTRIBUTION_ARN` e `OAC_ARN`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "DISTRIBUTION_ARN"
        }
      }
    }
  ]
}
```

### 1.2 Distribuição CloudFront

1. Crie uma distribuição com **Origin** apontando para o bucket S3 (e OAC se configurado).
2. **Default Root Object:** `index.html`.
3. **Error pages (opcional):** para SPA com roteamento no cliente, adicione:
   - HTTP 403 e 404 → resposta customizada: `index.html`, código 200.
4. Anote o **Distribution ID** (ex.: `E1ABC2DEF3GHI`).

### 1.3 Usuário/role para o GitHub Actions

O workflow precisa de permissão para:

- **S3:** `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket` no bucket.
- **CloudFront:** `cloudfront:CreateInvalidation`.

**Opção A – Chaves de acesso (IAM User)**

1. Crie um usuário IAM (ex.: `github-actions-scientific-compound`).
2. Anexe uma policy inline ou customizada com as permissões acima.
3. Crie **Access Key** e guarde **Access Key ID** e **Secret Access Key**.

**Opção B – OIDC (recomendado)**

1. Configure um IdP OIDC do GitHub no IAM (Identity provider para `token.actions.githubusercontent.com`).
2. Crie uma role que permita `AssumeRoleWithWebIdentity` vindo desse IdP, restrita ao seu repositório (e opcionalmente à branch `main`).
3. Anexe à role a policy com permissões S3 e CloudFront acima.
4. No workflow, troque o passo “Configure AWS credentials” para usar `role-to-assume: arn:aws:iam::ACCOUNT_ID:role/NOME_DA_ROLE` e remova `aws-access-key-id` e `aws-secret-access-key`.

## 2. Configuração no GitHub

No repositório: **Settings → Secrets and variables → Actions**.

### Secrets (credenciais)

- `AWS_ACCESS_KEY_ID` – Access Key do usuário IAM (opção A).
- `AWS_SECRET_ACCESS_KEY` – Secret Access Key (opção A).

Se usar OIDC (opção B), não use esses secrets; use apenas `role-to-assume` no workflow.

### Variables (configuração)

- `AWS_S3_BUCKET` – nome do bucket (ex.: `scientific-compound-app`).
- `AWS_CLOUDFRONT_DISTRIBUTION_ID` – ID da distribuição CloudFront (ex.: `E1ABC2DEF3GHI`).
- `AWS_REGION` (opcional) – região do S3; padrão do workflow: `us-east-1`.

## 3. Comportamento do workflow

1. **Checkout** do repositório.
2. **Setup Node.js** e instalação de dependências (`npm ci`).
3. **Build** em modo production (`npm run build`).
4. **Sync para S3:**
   - Arquivos com hash (JS/CSS) → `Cache-Control: max-age=31536000, immutable`.
   - `index.html` (e `.json`) → `Cache-Control: max-age=0, must-revalidate`.
   - `--delete` para remover arquivos antigos do bucket.
5. **Invalidation no CloudFront** para `/*`.

## 4. Execução

- **Automático:** push na branch `main`.
- **Manual:** Actions → “Deploy to AWS (S3 + CloudFront)” → “Run workflow”.

## 5. URL do site

Após o primeiro deploy, a URL do site é a **CloudFront URL** da distribuição (ex.: `https://d1234abcd.cloudfront.net`) ou o domínio customizado configurado no CloudFront (CNAME + certificado ACM, se usar).
