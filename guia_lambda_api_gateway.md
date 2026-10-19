# Guia Prático: AWS Lambda e API Gateaway

Este guia documenta o passo a passo para provisionar, configurar e integrar aplicações serverless usando o AWS Lambda e o Amazon API Gateway. Serão demonstradas a criação de funções, a configuração de rotas HTTP e a conexão entre os serviços para expor sua aplicação.

> Aviso: Este laboratório foi desenhado para o ambiente acadêmico (AWS Academy / Learner Lab). As configurações de acesso público e a ausência de autenticação (como chaves de API ou autorizadores) demonstradas aqui não são recomendadas para ambientes corporativos de produção, onde a segurança e o controle de acesso rigorosos são essenciais.

## Criando uma função Lambda por evento

### Usando a UI

Nesta etapa, vamos criar a função Lambda `guardrail-stop-by-tag` diretamente pelo console. O objetivo é criar um "GuardRail" (protetor de custos) que será acionado a cada 5 minutos para buscar e desligar instâncias EC2 que possuam a tag `AutoStop=true`.

#### Pré-requisito: Preparar a Instância EC2
1. Vá ao console do **Amazon EC2** e inicie uma instância (pode ser a `t2.micro` padrão).
2. Após a criação, selecione a instância, vá em **Tags** (ou **Ações > Gerenciamento de instâncias > Gerenciar tags**) e adicione:
   - **Chave:** `AutoStop`
   - **Valor:** `true`

#### Passo 1: Criar a Role do IAM (Permissões)
A função Lambda precisará de permissão para listar e parar instâncias.
1. No console da AWS, pesquise por **IAM** e abra o serviço.
2. No menu lateral, clique em **Roles** (Funções) e depois em **Create role** (Criar função).
3. Selecione **AWS service** (Serviço da AWS) e escolha **Lambda** na lista. Clique em **Next**.
4. Na busca por políticas, adicione a política gerenciada **AmazonEC2FullAccess** (Nota: Em produção, use o princípio do privilégio mínimo, mas para este laboratório, esta política já cobre o necessário). Clique em **Next**.
5. Dê o nome de `GuardRailLambdaRole` e clique em **Create role**.

#### Passo 2: Criar a Função Lambda
1. No console da AWS, pesquise por **Lambda** e abra o serviço.
2. Clique em **Create function** (Criar função).
3. Selecione **Author from scratch** (Criar do zero).
4. Preencha:
   - **Function name:** `guardrail-stop-by-tag`
   - **Runtime:** `Python 3.11`
   - Em **Permissions** (Permissões), expanda **Change default execution role** e escolha **Use an existing role** (Usar uma função existente).
   - Selecione a role `GuardRailLambdaRole` que você criou no Passo 1.
5. Clique em **Create function**.

#### Passo 3: Inserir o Código
1. Na tela da função, role até o bloco **Code source**.
2. Apague o código padrão e cole o código abaixo:

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Filtra instâncias em execução que tenham a tag AutoStop=true
    response = ec2.describe_instances(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']},
            {'Name': 'tag:AutoStop', 'Values': ['true']}
        ]
    )
    
    instance_ids = []
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instance_ids.append(instance['InstanceId'])
    
    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f'Instancias paradas com sucesso: {instance_ids}')
        return f'Paradas: {instance_ids}'
    else:
        print('Nenhuma instancia com a tag AutoStop=true foi encontrada.')
        return 'Nada a fazer.'

### Usando CloudFormation

1. Abra o laboratorio AWS;
2. Abra o CloudShell ('>_');
3. Crie o arquivo template:
  - Cole o seguinte no termina:
```bash
cat > guardrail-cost.yaml << 'EOF'
# Cole aqui o conteúdo completo do YAML
EOF
```

**Temaplet YAML:**
```
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: GuardRail de Custos - Para instancias EC2 com a tag AutoStop=true'

Resources:
  # 1. Instância EC2 de teste (já nasce com a tag que o Lambda vai buscar)
  MinhaInstanciaDeTeste:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0c55b159cbfafe1f0 # Amazon Linux 2
      Tags:
        - Key: Name
          Value: Instancia-GuardRail
        - Key: AutoStop
          Value: 'true' # O Lambda vai procurar por esta tag

  # 2. Role do IAM para o Lambda (permissao para descrever e parar instancias)
  GuardRailLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: GuardRailPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:DescribeInstances
                  - ec2:StopInstances
                Resource: '*'
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: 'arn:aws:logs:*:*:*'

  # 3. Funcao Lambda que varre as instancias pela tag
  GuardRailFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: 'guardrail-stop-by-tag'
      Runtime: 'python3.11'
      Handler: 'index.lambda_handler'
      Role: !GetAtt GuardRailLambdaRole.Arn
      Timeout: 30
      MemorySize: 128
      Code:
        ZipFile: |
          import boto3
          
          def lambda_handler(event, context):
              ec2 = boto3.client('ec2')
              # Filtra instancias em execucao que tenham a tag AutoStop=true
              response = ec2.describe_instances(
                  Filters=[
                      {'Name': 'instance-state-name', 'Values': ['running']},
                      {'Name': 'tag:AutoStop', 'Values': ['true']}
                  ]
              )
              
              instance_ids = []
              for reservation in response['Reservations']:
                  for instance in reservation['Instances']:
                      instance_ids.append(instance['InstanceId'])
              
              if instance_ids:
                  ec2.stop_instances(InstanceIds=instance_ids)
                  print(f'Instancias paradas com sucesso: {instance_ids}')
                  return f'Paradas: {instance_ids}'
              else:
                  print('Nenhuma instancia com a tag AutoStop=true foi encontrada.')
                  return 'Nada a fazer.'

  # 4. Permissao para o EventBridge invocar o Lambda
  PermissionForEvents:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref GuardRailFunction
      Action: 'lambda:InvokeFunction'
      Principal: 'events.amazonaws.com'

  # 5. Regra do EventBridge (Agendamento)
  ScheduleRule:
    Type: AWS::Events::Rule
    Properties:
      Name: 'GuardRailSchedule'
      Description: 'Dispara o GuardRail a cada 5 minutos'
      ScheduleExpression: 'rate(5 minutes)'
      State: 'ENABLED'
      Targets:
        - Arn: !GetAtt GuardRailFunction.Arn
          Id: 'GuardRailTarget'
```
4. Valide o arquivo: `aws cloudformation validate-template --template-body file://guardrail-cost.yaml`;
5. Criando o sctak do Formation:
```bash
aws cloudformation create-stack \
  --stack-name GuardRailLab \
  --template-body file://guardrail-cost.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

Para acompanhar a criação:
```bash
aws cloudformation wait stack-create-complete --stack-name GuardRailLab
```

``` bash
aws cloudformation describe-stacks --stack-name GuardRailLab --query "Stacks[0].StackStatus"
```

## Criando APIs com API Gateway

O API Gateway é um serviço totalmente gerenciado que permite criar, publicar, manter, monitorar e proteger APIs em qualquer escala. Ele suporta três tipos principais de API.

### Usando a UI: HTTP API

**HTTP API** integrada a uma função **AWS Lambda**.


#### Passo 1: Criar a Função Lambda (O Backend)
1. No console da AWS, na barra de busca superior, pesquise por **Lambda** e abra o serviço.
2. Clique no botão laranja **Create function** (Criar função).
3. Selecione **Author from scratch** (Criar do zero).
4. Em **Function name**, digite um nome, como `minha-funcao-api`.
5. Em **Runtime**, escolha **Python 3.11** (ou a versão mais recente disponível).
6. Em **Architecture**, mantenha `x86_64`.
7. Em **Permissions**, deixe a opção padrão **Create a new role with basic Lambda permissions** (o CloudFormation/Console criará a role automaticamente).
8. Clique em **Create function**.
9. Na tela da função, role até o bloco **Code source** e substitua o código padrão pelo seguinte (para retornar uma resposta JSON amigável):
   ```python
   def lambda_handler(event, context):
       return {
           "statusCode": 200,
           "body": "Olá do API Gateway + Lambda!"
       }
   ```
10. Clique em **Deploy** para salvar o código.

> **💡 Dica para a apresentação:** Mostre o código e explique que o objeto `event` (não usado aqui, mas disponível) conteria os dados da requisição HTTP, e o `statusCode` + `body` são exatamente o que o API Gateway devolverá ao cliente.


#### Passo 2: Criar a HTTP API no API Gateway
1. Na barra de busca superior, pesquise por **API Gateway** e abra o serviço.
2. Na página inicial do API Gateway, clique em **Create API** (Criar API).
3. No cartão **HTTP API**, clique em **Build** (Construir).
4. Em **API name**, digite um nome, como `minha-http-api`.
5. Em **Integrations**, clique em **Add integration** (Adicionar integração).
6. Selecione **Lambda**.
7. No campo **Lambda function**, digite o nome da função criada no Passo 1 (`minha-funcao-api`) e selecione-a na lista.
8. Clique em **Next** (Avançar).
9. Na tela **Configure routes** (Configurar rotas):
   - **Method**: deixe `GET`.
   - **Path**: deixe `/` (a rota padrão "catch-all").
10. Clique em **Next**.
11. Na tela **Define stages** (Definir estágios):
    - **Stage name**: digite `prod` (ou `$default`).
    - Deixe a opção **Auto-deploy** (Implantação automática) **marcada**.
12. Clique em **Next**.
13. Revise as configurações e clique em **Create** (Criar).


#### Passo 3: Testar a API
1. Após a criação, você será redirecionado para a página de detalhes da API.
2. Localize o campo **Invoke URL** (URL de invocação). Ele terá um formato como:
   ```
   https://abc123.execute-api.us-east-1.amazonaws.com/prod
   ```
3. Copie essa URL.
4. Abra uma nova aba do navegador e cole a URL (ou use o comando `curl` no CloudShell: `curl https://SUA-URL`).
5. Você deverá ver a resposta: `"Olá do API Gateway + Lambda!"`.

#### Passo 4: Limpeza
Para evitar custos desnecessários no Learner Lab:
1. No console do API Gateway, selecione a API criada.
2. Clique em **Delete** (Excluir) e confirme.
3. No console do Lambda, selecione a função `minha-funcao-api` e exclua-a (a role do IAM criada automaticamente também pode ser removida, se desejar).


### Usando CloudFormation: API Rest

```bash
Resources:
  # 1. A Função Lambda (Backend)
  MinhaFuncaoHello:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.handler
      Runtime: python3.11
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              return {
                  "statusCode": 200,
                  "body": "Olá do API Gateway + Lambda!"
              }

  # 2. A API no API Gateway
  MinhaApiRest:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: "MinhaPrimeiraAPI"

  # 3. O Recurso (ex: /hello)
  RecursoHello:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref MinhaApiRest
      ParentId: !GetAtt MinhaApiRest.RootResourceId
      PathPart: "hello"

  # 4. O Método (GET) com Integração Proxy para o Lambda
  MetodoGetHello:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref MinhaApiRest
      ResourceId: !Ref RecursoHello
      HttpMethod: "GET"
      AuthorizationType: "NONE"
      Integration:
        Type: "AWS_PROXY" # O modo de integração que repassa tudo para o Lambda
        IntegrationHttpMethod: "POST"
        Uri: !Sub 
          - "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaArn}/invocations"
          - LambdaArn: !GetAtt MinhaFuncaoHello.Arn

  # 5. Permissão para o API Gateway invocar o Lambda
  PermissaoApiInvocaLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt MinhaFuncaoHello.Arn
      Action: "lambda:InvokeFunction"
      Principal: "apigateway.amazonaws.com"
      SourceArn: !Sub "arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${MinhaApiRest}/*"
```

