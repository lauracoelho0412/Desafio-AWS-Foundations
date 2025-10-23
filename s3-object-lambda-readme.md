# Automatização do S3 Object Lambda com CloudFormation

## 📋 Índice
- [Visão Geral](#visão-geral)
- [Pré-requisitos](#pré-requisitos)
- [Arquitetura](#arquitetura)
- [Passo a Passo](#passo-a-passo)
- [Template CloudFormation](#template-cloudformation)
- [Testando a Configuração](#testando-a-configuração)
- [Limpeza de Recursos](#limpeza-de-recursos)
- [Solução de Problemas](#solução-de-problemas)

## 🎯 Visão Geral

Este guia demonstra como automatizar a configuração do **S3 Object Lambda** usando templates do **AWS CloudFormation**. O S3 Object Lambda permite adicionar seu próprio código para processar dados recuperados do Amazon S3 antes de retorná-los para uma aplicação.

### Casos de Uso Comuns:
- Redimensionar e processar imagens dinamicamente
- Filtrar ou reduzir dados sensíveis
- Converter formatos de arquivo
- Enriquecer dados com informações adicionais
- Comprimir ou descomprimir objetos

## ✅ Pré-requisitos

Antes de começar, certifique-se de ter:

- Conta AWS ativa
- AWS CLI instalado e configurado
- Permissões IAM adequadas para:
  - Criar buckets S3
  - Criar funções Lambda
  - Criar roles IAM
  - Criar stacks do CloudFormation
  - Criar S3 Access Points
- Conhecimento básico de YAML/JSON
- Conhecimento básico de Python ou Node.js (para a função Lambda)

## 🏗️ Arquitetura

```
┌─────────────┐
│   Cliente   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│ S3 Object Lambda        │
│ Access Point            │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Função Lambda           │
│ (Processa dados)        │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ S3 Access Point         │
│ (Supporting)            │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Bucket S3               │
│ (Dados originais)       │
└─────────────────────────┘
```

## 📝 Passo a Passo

### Passo 1: Preparar o Código da Função Lambda

Crie um arquivo `lambda_function.py` com o código que processará seus objetos:

```python
import boto3
import urllib.parse

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    # Obter informações da requisição
    get_context = event['getObjectContext']
    route = get_context['outputRoute']
    token = get_context['outputToken']
    s3_url = get_context['inputS3Url']
    
    # Buscar o objeto original
    response = urllib.request.urlopen(s3_url)
    original_object = response.read().decode('utf-8')
    
    # Processar o objeto (exemplo: converter para maiúsculas)
    transformed_object = original_object.upper()
    
    # Retornar o objeto transformado
    s3_client.write_get_object_response(
        RequestRoute=route,
        RequestToken=token,
        Body=transformed_object
    )
    
    return {'statusCode': 200}
```

### Passo 2: Criar o Template CloudFormation

Crie um arquivo `s3-object-lambda-stack.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Configuracao automatizada do S3 Object Lambda'

Parameters:
  BucketName:
    Type: String
    Description: Nome do bucket S3
    Default: my-object-lambda-bucket
  
  LambdaCodeBucket:
    Type: String
    Description: Bucket onde esta o codigo Lambda zipado
  
  LambdaCodeKey:
    Type: String
    Description: Chave do arquivo zip do Lambda
    Default: lambda_function.zip

Resources:
  # Bucket S3 Principal
  SourceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${BucketName}-${AWS::AccountId}'
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  # IAM Role para a Função Lambda
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: S3ObjectLambdaPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:ListBucket
                Resource:
                  - !GetAtt SourceBucket.Arn
                  - !Sub '${SourceBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - s3-object-lambda:WriteGetObjectResponse
                Resource: '*'

  # Função Lambda
  ObjectLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: S3ObjectTransformFunction
      Runtime: python3.11
      Handler: lambda_function.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        S3Bucket: !Ref LambdaCodeBucket
        S3Key: !Ref LambdaCodeKey
      Timeout: 60
      MemorySize: 256

  # S3 Access Point (Supporting Access Point)
  SupportingAccessPoint:
    Type: AWS::S3::AccessPoint
    Properties:
      Bucket: !Ref SourceBucket
      Name: !Sub 'supporting-ap-${AWS::AccountId}'

  # S3 Object Lambda Access Point
  ObjectLambdaAccessPoint:
    Type: AWS::S3ObjectLambda::AccessPoint
    Properties:
      Name: !Sub 'object-lambda-ap-${AWS::AccountId}'
      ObjectLambdaConfiguration:
        SupportingAccessPoint: !GetAtt SupportingAccessPoint.Arn
        TransformationConfigurations:
          - Actions:
              - GetObject
            ContentTransformation:
              AwsLambda:
                FunctionArn: !GetAtt ObjectLambdaFunction.Arn

  # Permissão para S3 invocar Lambda
  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref ObjectLambdaFunction
      Action: lambda:InvokeFunction
      Principal: s3.amazonaws.com
      SourceAccount: !Ref AWS::AccountId

Outputs:
  BucketName:
    Description: Nome do bucket S3
    Value: !Ref SourceBucket
  
  ObjectLambdaAccessPointArn:
    Description: ARN do S3 Object Lambda Access Point
    Value: !GetAtt ObjectLambdaAccessPoint.Arn
  
  ObjectLambdaAccessPointAlias:
    Description: Alias do Object Lambda Access Point
    Value: !GetAtt ObjectLambdaAccessPoint.Alias.Value
  
  LambdaFunctionArn:
    Description: ARN da funcao Lambda
    Value: !GetAtt ObjectLambdaFunction.Arn
```

### Passo 3: Preparar e Fazer Upload do Código Lambda

```bash
# Criar o arquivo zip
zip lambda_function.zip lambda_function.py

# Criar bucket para o código (se não existir)
aws s3 mb s3://my-lambda-code-bucket-$(aws sts get-caller-identity --query Account --output text)

# Upload do código
aws s3 cp lambda_function.zip s3://my-lambda-code-bucket-$(aws sts get-caller-identity --query Account --output text)/
```

### Passo 4: Validar o Template

```bash
aws cloudformation validate-template \
  --template-body file://s3-object-lambda-stack.yaml
```

### Passo 5: Criar a Stack do CloudFormation

```bash
aws cloudformation create-stack \
  --stack-name s3-object-lambda-stack \
  --template-body file://s3-object-lambda-stack.yaml \
  --parameters \
    ParameterKey=BucketName,ParameterValue=my-demo-bucket \
    ParameterKey=LambdaCodeBucket,ParameterValue=my-lambda-code-bucket-ACCOUNT_ID \
    ParameterKey=LambdaCodeKey,ParameterValue=lambda_function.zip \
  --capabilities CAPABILITY_IAM
```

### Passo 6: Monitorar a Criação da Stack

```bash
# Verificar status
aws cloudformation describe-stacks \
  --stack-name s3-object-lambda-stack \
  --query 'Stacks[0].StackStatus'

# Acompanhar eventos
aws cloudformation describe-stack-events \
  --stack-name s3-object-lambda-stack \
  --max-items 10
```

### Passo 7: Obter os Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name s3-object-lambda-stack \
  --query 'Stacks[0].Outputs'
```

## 🧪 Testando a Configuração

### 1. Fazer Upload de um Arquivo de Teste

```bash
# Obter o nome do bucket dos outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name s3-object-lambda-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' \
  --output text)

# Criar arquivo de teste
echo "hello world from s3 object lambda" > test.txt

# Upload
aws s3 cp test.txt s3://$BUCKET_NAME/
```

### 2. Acessar via Object Lambda Access Point

```bash
# Obter o ARN do Object Lambda Access Point
OL_ARN=$(aws cloudformation describe-stacks \
  --stack-name s3-object-lambda-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`ObjectLambdaAccessPointArn`].OutputValue' \
  --output text)

# Ler objeto através do Object Lambda
aws s3api get-object \
  --bucket $OL_ARN \
  --key test.txt \
  output.txt

# Verificar resultado (deve estar em maiúsculas)
cat output.txt
```

### 3. Comparar com o Original

```bash
# Ler diretamente do bucket
aws s3 cp s3://$BUCKET_NAME/test.txt original.txt

# Comparar
echo "Original:"
cat original.txt
echo -e "\nTransformado:"
cat output.txt
```

## 🧹 Limpeza de Recursos

Para remover todos os recursos criados:

```bash
# Esvaziar o bucket
aws s3 rm s3://$BUCKET_NAME --recursive

# Deletar a stack
aws cloudformation delete-stack \
  --stack-name s3-object-lambda-stack

# Verificar deleção
aws cloudformation wait stack-delete-complete \
  --stack-name s3-object-lambda-stack
```

## 🔧 Solução de Problemas

### Erro: "AccessDenied" ao acessar Object Lambda

**Solução:** Verifique se a role IAM da Lambda tem permissões para `s3-object-lambda:WriteGetObjectResponse`

### Lambda timeout

**Solução:** Aumente o timeout no template CloudFormation (padrão: 60 segundos)

### Erro ao criar Access Point

**Solução:** Certifique-se de que o nome é único e segue o padrão: apenas letras minúsculas, números e hífens

### Stack em estado ROLLBACK_COMPLETE

**Solução:** 
```bash
# Verificar motivo do erro
aws cloudformation describe-stack-events \
  --stack-name s3-object-lambda-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'

# Deletar e recriar
aws cloudformation delete-stack --stack-name s3-object-lambda-stack
```

## 📚 Recursos Adicionais

- [Documentação oficial do S3 Object Lambda](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transforming-objects.html)
- [CloudFormation S3ObjectLambda::AccessPoint](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3objectlambda-accesspoint.html)
- [Exemplos de código Lambda para S3 Object Lambda](https://github.com/aws-samples/amazon-s3-object-lambda-default-configuration)

## 📄 Licença

Este projeto está sob a licença MIT.

## 🤝 Contribuições

Contribuições são bem-vindas! Sinta-se à vontade para abrir issues ou pull requests.