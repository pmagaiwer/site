# Deploy do Site - Pierre Santos

Este documento descreve, passo-a-passo, como publicar o site estático (arquivos `index_tailwind_css.html`, `style.css`, `head.jpg`) em **GitHub Pages** **(opcional)** e em **Amazon S3** (hospedagem estática). Inclui também comandos para uso com **AWS CLI** e instruções para domínio personalizado via **CloudFront** + **ACM**.

---

## Pré-requisitos
- Git instalado e configurado (nome e e-mail).
- Conta no GitHub.
- (Para S3) Conta AWS e credenciais configuradas localmente: `aws configure` com Access Key ID, Secret Access Key, região padrão e formato (json).
- (Opcional) AWS CLI v2 instalada para upload via terminal.
- Arquivos prontos na pasta local: `index_tailwind_css.html`, `style.css`, `head.jpg`

---

## 1) Publicar no GitHub (repositório público)

### 1.1 Criar repositório
1. No GitHub, clique em **New repository**.  
2. Nome sugerido: `pierremsantos.github.io` (se quiser usar GitHub Pages com domínio padrão do GitHub).  
3. Torne o repositório **Public**. Não é necessário inicializar com README (você já tem arquivos).

### 1.2 Subir os arquivos locais para o repositório
No terminal (ajuste o nome do repositório/URL):

```bash
# criar pasta local e entrar nela (se ainda não estiver)
mkdir site-pierre && cd site-pierre

# copiar/mover os arquivos gerados para esta pasta:
# index_tailwind_css.html  -> renomeie para index.html caso queira que seja a página inicial do GitHub Pages
cp /caminho/para/index_tailwind_css.html ./index.html
cp /caminho/para/style.css ./style.css
cp /caminho/para/head.jpg ./head.jpg

# inicializar git e adicionar remoto
git init
git add .
git commit -m "Initial site commit - Pierre Santos"
git branch -M main
git remote add origin https://github.com/<seu-usuario>/pierremsantos.github.io.git
git push -u origin main
```

> Observação: se você escolheu outro nome de repositório (não `username.github.io`), pode usar GitHub Pages nas configurações do repositório (Settings → Pages → Source → `gh-pages` ou `main`/`/root`) e o site ficará em `https://<seu-usuario>.github.io/<repo>/`

### 1.3 Ativar GitHub Pages (se aplicável)
1. Vá para **Settings -> Pages** do repositório.  
2. Em **Source**, selecione `main` branch e `/ (root)` como pasta. Salve.  
3. Aguarde alguns minutos; o GitHub fornecerá a URL pública do site.

---

## 2) Hospedar no Amazon S3 (recomendado para site estático profissional)

### 2.1 Criar bucket S3
1. Acesse o Console AWS → **S3** → **Create Bucket**.  
2. Nome do bucket: escolha único globalmente (ex: `pierre-portfolio-2025-<seunome>`).  
3. Região: escolha a mais próxima dos seus usuários (ex: `sa-east-1` para Brasil).  
4. **Uncheck** (desmarque) `Block all public access` (ou configure políticas depois). Confirme que entende os riscos.  
5. Criar o bucket.

### 2.2 Configurar hospedagem de site estático no S3
1. Dentro do bucket, vá em **Properties** → **Static website hosting**.  
2. Enable e insira `index.html` como **Index document**. (Opcionalmente, `error.html` como documento de erro).  
3. Salve. O console exibirá o endpoint do site (HTTP), ex: `http://<bucket>.s3-website-<region>.amazonaws.com`

### 2.3 Subir os arquivos (via Console ou AWS CLI)

#### Via console (UI)
- Clique em **Upload** → arraste `index.html`, `style.css`, `head.jpg` → Upload.  
- Após subir, selecione os objetos e torne-os públicos via **Permissions → Make public** (se sua organização permitir).

#### Via AWS CLI (recomendado automatizar)
```bash
# configure aws cli antes (aws configure)

# exemplo: sincronizar pasta local 'site-pierre' com o bucket S3
aws s3 sync ./site-pierre s3://nome-do-seu-bucket --acl public-read --delete
```
> Observação: `--acl public-read` torna objetos públicos. Melhor prática: manter bucket privado e usar política de bucket que permita leitura pública apenas para arquivos estáticos, ou usar CloudFront com política restrita.

### 2.4 Definir Content-Type correto (se necessário)
O `aws s3 sync` normalmente define `Content-Type` automaticamente. Se precisar forçar, use `--content-type` ao copiar um único arquivo:
```bash
aws s3 cp index.html s3://nome-do-seu-bucket/index.html --acl public-read --content-type "text/html; charset=utf-8"
```

### 2.5 Política de bucket para leitura pública (alternativa ao ACL)
Coloque esta policy em **Permissions → Bucket Policy** (substitua `BUCKET_NAME`):
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"PublicReadGetObject",
      "Effect":"Allow",
      "Principal": "*",
      "Action":["s3:GetObject"],
      "Resource":["arn:aws:s3:::BUCKET_NAME/*"]
    }
  ]
}
```

> Atenção: política torna todo o conteúdo do bucket público de leitura. Use somente para sites estáticos públicos.

### 2.6 Acessar o site
- Use o endpoint mostrado em **Static website hosting**, por exemplo:  
`http://nome-do-bucket.s3-website-sa-east-1.amazonaws.com`

---

## 3) (Opcional) Usar CloudFront e HTTPS com domínio personalizado
Para HTTPS e caching global, use CloudFront e ACM (certificado SSL gratuito).

### 3.1 Passos resumidos:
1. Criar certificado no **AWS Certificate Manager (ACM)** na região `us-east-1` (obrigatório para CloudFront). Solicitar certificado para `www.seudominio.com` e `seudominio.com`. Validar via DNS.  
2. Criar distribuição **CloudFront**:
   - Origin: endpoint do bucket S3 (ou s3 bucket origin `nome-do-bucket.s3.amazonaws.com`)
   - Viewer Protocol Policy: Redirect HTTP to HTTPS
   - Configure Alternate Domain Names (CNAMEs) com seu domínio e associe o certificado ACM.  
3. No seu DNS (Route 53 ou outro provedor), crie registros `A` (alias) ou `CNAME` apontando para o domínio CloudFront.  
4. Aguarde a propagação (pode demorar ~10–30 minutos ou mais).

---

## 4) Dicas e Boas Práticas
- Para deploys automáticos, crie workflow com **GitHub Actions** para build e `aws s3 sync` após push na branch `main`.  
- Use `Content-Type` correto para arquivos (HTML, CSS, JPG).  
- Configure `Cache-Control` em objetos estáticos se for usar CloudFront.  
- Para atualização frequente, automatize com CI/CD (GitHub Actions, GitLab CI).  
- Evite usar `--acl public-read` quando possível; prefira políticas de bucket ou CloudFront com origens privadas e assinadas.

---

## 5) Exemplo rápido: GitHub Actions para publicar no S3
Arquivo `.github/workflows/deploy-s3.yml` (exemplo mínimo):
```yaml
name: Deploy to S3

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: sa-east-1
      - name: Sync to S3
        run: |
          aws s3 sync ./ s3://nome-do-seu-bucket --delete --acl public-read
```
> Guarde `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` como Secrets do repositório no GitHub (Settings → Secrets).

---

## 6) Checklist final antes de publicar
- [ ] index.html (renomeado) presente e com referências corretas a `style.css` e `head.jpg`.  
- [ ] Verificar links absolutos/relativos (use caminhos relativos: `./style.css`, `./head.jpg`).  
- [ ] Arquivos foram testados localmente (abra `index.html` no navegador).  
- [ ] AWS CLI configurado se for usar CLI.  
- [ ] Políticas de segurança e privacidade revisadas (dados sensíveis não devem ficar públicos).

---

Se quiser, posso:
- Gerar o arquivo `deploy-s3.sh` com os comandos prontos para executar localmente.  
- Preparar o workflow do GitHub Actions adaptado ao seu bucket.  
- Gerar o arquivo `index.html` final (já renomeado) pronto para upload.

Basta me dizer qual dessas opções prefere. Boa sorte — seu site vai ficar ótimo no ar! 👏
