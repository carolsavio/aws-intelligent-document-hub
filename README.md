# AWS Intelligent Document Hub

> Solução de arquitetura serverless na AWS para centralização, organização,
> pesquisa e gerenciamento seguro de documentos corporativos.

# Visão Geral

Este projeto apresenta uma proposta arquitetural para resolver um problema comum
em empresas que armazenam milhares de documentos (PDF, DOCX, XLSX, imagens,
contratos, notas fiscais e relatórios) de forma descentralizada.

A solução utiliza serviços gerenciados da AWS para criar uma plataforma
escalável, segura e de baixa manutenção.

# Problema

A empresa possui documentos espalhados entre computadores, servidores locais,
Google Drive, OneDrive e compartilhamentos de rede.

Consequências:

- Dificuldade para localizar documentos
- Ausência de versionamento
- Falta de auditoria
- Ausência de classificação
- Risco de perda de arquivos
- Permissões inconsistentes

# Objetivos

- Centralizar documentos
- Automatizar classificação
- Permitir pesquisa rápida
- Garantir segurança
- Possibilitar auditoria
- Reduzir custo operacional

# Requisitos Funcionais

- Upload de documentos
- OCR automático
- Indexação
- Pesquisa
- Controle de acesso
- Versionamento
- Backup

# Arquitetura

```text
Usuário
   |
CloudFront
   |
Frontend (S3)
   |
API Gateway
   |
Lambda (gera presigned URL)
   |
Amazon S3  <-- upload direto do cliente via presigned URL
   |
Evento S3
   |
Lambda Trigger (inicia job assíncrono)
   |
Textract (API assíncrona: Start*) --> SNS --> Lambda de callback 
   |
Amazon Comprehend Custom Classification 
   |
DynamoDB  --(DynamoDB Streams)-->  Lambda  -->  OpenSearch 
             |
        Consulta
```
# Fluxo da Solução

## 1 - Upload 

O usuário autentica via Amazon Cognito.

A API Gateway aciona uma Lambda que **gera uma presigned URL do S3** e a
devolve ao cliente. O cliente faz o upload **diretamente para o S3** usando
essa URL.

O documento é salvo em um bucket S3 com Versioning habilitado.

## 2 - Processamento

O evento `ObjectCreated` dispara uma Lambda "trigger", que inicia um job de
extração usando a **API assíncrona do Textract** (`StartDocumentTextDetection`
ou `StartDocumentAnalysis`).

Ao receber o callback, a Lambda:

- envia o texto extraído para o **Amazon Comprehend Custom Classification**
- obtém a categoria do documento
- grava metadados no DynamoDB

## 3 - Sincronização DynamoDB → OpenSearch

Em vez de a Lambda gravar diretamente em dois lugares (DynamoDB e
OpenSearch), o DynamoDB grava os metadados e um **DynamoDB Stream** aciona
uma Lambda dedicada que replica o dado no OpenSearch.

## 4 - Pesquisa

Usuário informa palavras-chave.

A API consulta o OpenSearch.

Os metadados retornam rapidamente.

O documento é recuperado diretamente do S3.
# Serviços AWS

## Amazon S3

Armazenamento principal.

Motivos:

- 11 noves de durabilidade
- baixo custo
- versionamento
- lifecycle
- integração nativa com upload direto via presigned URL 

## AWS Lambda

Executa toda a orquestração do pipeline.

Benefícios:

- sem servidores
- cobrança por uso
- escala automática

## API Gateway

Expõe API REST.

Responsável por:

- autenticação
- rate limit
- integração Lambda (para gerar URLs e consultar metadados)

## Amazon Textract

Extrai texto, formulários e tabelas.

Usa a **API assíncrona** para suportar documentos multi-página, com
notificação de conclusão via SNS. A API síncrona fica reservada para casos
pontuais de página única.

## Amazon Comprehend

Classificação de domínio (RH, Financeiro, Jurídico, Comercial) requer um
**modelo Custom Classification treinado**, não o Comprehend padrão. O
Comprehend padrão continua útil para extração de entidades e metadados
automáticos (datas, valores, nomes), que podem enriquecer o registro no
DynamoDB.

## DynamoDB

Armazena metadados.

Exemplo:

- Nome
- Categoria
- Autor
- Bucket
- Hash
- Data
- Versão

Alimenta o OpenSearch via DynamoDB Streams.

## OpenSearch

Indexa conteúdo textual permitindo pesquisas rápidas.

## Cognito

Autenticação de usuários.

## IAM

Controle de permissões seguindo princípio do menor privilégio.

## KMS

O KMS gerencia as **chaves de criptografia** usadas pelo S3 para proteger os
**dados** armazenados.

## CloudWatch

Logs e métricas.

## CloudTrail

Auditoria das ações realizadas na conta.

## SNS

Notifica a conclusão de jobs assíncronos do Textract e pode ser reutilizado
para outras notificações do pipeline.

# Segurança

- Bucket privado
- HTTPS obrigatório
- Criptografia SSE-KMS
- IAM Least Privilege
- MFA para administradores
- Logs centralizados
- Auditoria completa
- AWS WAF na frente do CloudFront/API Gateway
- Fila de Dead Letter (DLQ) nas Lambdas do pipeline assíncrono, para
  capturar falhas de processamento sem perder o evento

# Backup

- Versionamento do S3
- Lifecycle para Glacier
- Deep Archive para retenção longa

# Escalabilidade

Como todos os componentes são gerenciados:

- Lambda escala automaticamente
- API Gateway escala automaticamente
- S3 praticamente ilimitado
- DynamoDB on-demand
- OpenSearch escalável

