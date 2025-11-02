# lambda-dio
Último projeto DIO
🚀 Projeto: Automação da Configuração do S3 Object Lambda com AWS CloudFormation
🧩 Descrição

Este projeto cria automaticamente:

Um S3 Bucket (armazenamento original);

Um Lambda Function (para processar ou modificar objetos ao serem acessados);

Um S3 Access Point (para acesso padrão ao bucket);

Um S3 Object Lambda Access Point, que invoca a função Lambda ao ler objetos do bucket.

Tudo isso é feito via Infrastructure as Code (IaC) usando AWS CloudFormation.

🧱 Arquitetura Simplificada
[S3 Bucket] ───► [Access Point] ───► [S3 Object Lambda Access Point] ───► [Lambda Function]
📌 O cliente solicita um objeto através do Object Lambda Access Point,
📌 a função Lambda intercepta a requisição, processa o conteúdo,
📌 e o objeto modificado é retornado ao cliente.

⚙️ Recursos Criados
| Recurso                       | Função                                    |
| ----------------------------- | ----------------------------------------- |
| **S3Bucket**                  | Armazena objetos originais                |
| **LambdaFunction**            | Processa ou transforma objetos            |
| **S3AccessPoint**             | Fornece acesso controlado ao bucket       |
| **S3ObjectLambdaAccessPoint** | Acessa o objeto via Lambda transformadora |
| **IAMRole**                   | Permite que a Lambda acesse o bucket      |

📄 Template CloudFormation — s3-object-lambda.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Automação do S3 Object Lambda - Projeto CloudFormation simples"

Resources:
  # 1️⃣ Bucket S3 principal
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "object-lambda-demo-${AWS::AccountId}-${AWS::Region}"
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Project
          Value: S3ObjectLambdaDemo

  # 2️⃣ Função Lambda que processa os objetos
  ObjectLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub "ObjectLambdaProcessor-${AWS::Region}"
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 10
      Code:
        ZipFile: |
          import boto3
          import json

          def lambda_handler(event, context):
              s3 = boto3.client('s3')
              # O Object Lambda envia detalhes no evento
              get_object_context = event["getObjectContext"]
              request_route = get_object_context["outputRoute"]
              request_token = get_object_context["outputToken"]

              # Simples transformação: adiciona um prefixo ao conteúdo
              response = s3.get_object(
                  Bucket=event["userRequest"]["url"].split("/")[3],
                  Key=event["userRequest"]["url"].split("/")[-1]
              )
              data = response["Body"].read().decode("utf-8")

              transformed = f"*** Processado via Object Lambda ***\n{data}"

              # Retorna o objeto modificado
              s3.write_get_object_response(
                  Body=transformed.encode("utf-8"),
                  RequestRoute=request_route,
                  RequestToken=request_token
              )

              return {"status_code": 200}

  # 3️⃣ IAM Role para a função Lambda
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "ObjectLambdaRole-${AWS::Region}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: ObjectLambdaPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                  - s3-object-lambda:WriteGetObjectResponse
                Resource: "*"
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: "*"

  # 4️⃣ S3 Access Point
  S3AccessPoint:
    Type: AWS::S3::AccessPoint
    Properties:
      Bucket: !Ref S3Bucket
      Name: !Sub "base-accesspoint-${AWS::AccountId}"

  # 5️⃣ S3 Object Lambda Access Point
  ObjectLambdaAccessPoint:
    Type: AWS::S3ObjectLambda::AccessPoint
    Properties:
      Name: !Sub "object-lambda-accesspoint-${AWS::Region}"
      ObjectLambdaConfiguration:
        SupportingAccessPoint: !GetAtt S3AccessPoint.Arn
        TransformationConfigurations:
          - Actions:
              - GetObject
            ContentTransformation:
              AwsLambda:
                FunctionArn: !GetAtt ObjectLambdaFunction.Arn

Outputs:
  BucketName:
    Description: Nome do bucket criado
    Value: !Ref S3Bucket
  ObjectLambdaAP:
    Description: Nome do S3 Object Lambda Access Point
    Value: !Ref ObjectLambdaAccessPoint
  LambdaName:
    Description: Nome da função Lambda
    Value: !Ref ObjectLambdaFunction
🚀 Como Executar
1️⃣ Criar o arquivo

Salve o código acima como s3-object-lambda.yaml.

2️⃣ Fazer upload no CloudFormation

No Console AWS:

Vá em CloudFormation → Create Stack → With new resources (standard)

Escolha Upload a template file e envie s3-object-lambda.yaml

Clique em Next → Next → Create Stack

3️⃣ Aguardar a criação

O processo leva 1–2 minutos.
Após CREATE_COMPLETE, verifique:

O bucket criado (em S3)

A função Lambda (em Lambda Console)

O Access Point (em S3 → Access Points)

O Object Lambda Access Point (em S3 → Object Lambda Access Points)

🧪 Teste Rápido

Faça upload de um arquivo simples no bucket:
echo "Arquivo original no S3" > teste.txt
aws s3 cp teste.txt s3://object-lambda-demo-<ACCOUNT>-<REGION>/
Acesse o arquivo via o Object Lambda Access Point:
aws s3api get-object \
  --bucket arn:aws:s3-object-lambda:<REGION>:<ACCOUNT>:accesspoint/object-lambda-accesspoint-<REGION> \
  --key teste.txt saida.txt
Abra saida.txt e veja o conteúdo modificado:
*** Processado via Object Lambda ***
Arquivo original no S3
Pronto! O arquivo foi interceptado e transformado automaticamente pela Lambda.

🧹 Limpeza (Evitar Custos)

Para remover todos os recursos criados:
aws cloudformation delete-stack --stack-name S3ObjectLambdaDemo
💡 Insights e Aprendizados

O S3 Object Lambda permite modificar objetos em tempo real sem duplicar dados.

É útil para anonimizar dados, redimensionar imagens, converter formatos, etc.

Tudo é criado e gerenciado automaticamente via CloudFormation, garantindo reprodutibilidade.
