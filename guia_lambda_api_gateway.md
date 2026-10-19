# Guia Prático: AWS Lambda e API Gateway

Este guia documenta o passo a passo para provisionar, configurar e integrar aplicações serverless usando o AWS Lambda e o Amazon API Gateway. Serão demonstradas a criação de funções, a configuração de rotas HTTP e a conexão entre os serviços para expor sua aplicação.

> **Aviso:** Este laboratório foi desenhado para o ambiente acadêmico (AWS Academy / Learner Lab). As configurações de acesso público e a ausência de autenticação (como chaves de API ou autorizadores) demonstradas aqui não são recomendadas para ambientes corporativos de produção, onde a segurança e o controle de acesso rigorosos são essenciais.

---

## Parte 1: Criando uma função Lambda por agendamento (EventBridge)

Nesta etapa, vamos criar um "GuardRail" (protetor de custos). A função será acionada a cada 5 minutos para buscar e desligar instâncias EC2 que possuam a tag `AutoStop=true`.

### Opção A: Usando a Interface (UI)

#### Pré-requisito: Preparar a Instância EC2
1. Vá ao console do **Amazon EC2** e inicie uma instância (pode ser a `t2.micro` padrão).
2. Após a criação, selecione a instância, vá em **Tags** e adicione:
   - **Chave:** `AutoStop`
   - **Valor:** `true`

#### Passo 1: Criar a Role do IAM (Permissões)
A função Lambda precisará de permissão para listar e parar instâncias.
1. No console da AWS, pesquise por **IAM** e abra o serviço.
2. No menu lateral, clique em **Roles** e depois em **Create role**.
3. Selecione **AWS service** e escolha **Lambda** na lista. Clique em **Next**.
4. Na tela de permissões, clique em **Create policy** (abrirá em outra aba) ou adicione uma política inline com as seguintes permissões (JSON):
   - Ações: `ec2:DescribeInstances`, `ec2:StopInstances`, `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`.
5. Dê o nome de `GuardRailLambdaRole` e clique em **Create role**.

#### Passo 2: Criar a Função Lambda
1. No console da AWS, pesquise por **Lambda** e abra o serviço.
2. Clique em **Create function**.
3. Selecione **Author from scratch**.
4. Preencha:
   - **Function name:** `guardrail-stop-by-tag`
   - **Runtime:** `Python 3.11`
   - Em **Permissions**, expanda **Change default execution role** e escolha **Use an existing role**.
   - Selecione a role `GuardRailLambdaRole` criada no Passo 1.
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
```
3. Clique em **Deploy**.

#### Passo 4: Configurar o Gatilho (EventBridge)
1. Na página da função, clique em **+ Add trigger**.
2. Selecione **EventBridge (CloudWatch Events)**.
3. Em **Rule**, escolha **Create a new rule**.
4. Preencha:
   - **Rule name:** `GuardRailSchedule`
   - **Rule type:** `Schedule expression`
   - **Schedule expression:** `rate(5 minutes)`
5. Clique em **Add**.

#### 🧹 Limpeza (Evitar custos no Academy)
Para não consumir créditos, apague tudo o que foi criado:
1. **EventBridge:** Vá em EventBridge > Rules, selecione `GuardRailSchedule` e clique em **Delete**.
2. **Lambda:** Vá em Lambda, selecione `guardrail-stop-by-tag` e clique em **Actions > Delete**.
3. **IAM:** Vá em IAM > Roles, busque por `GuardRailLambdaRole`, selecione e clique em **Delete**.
4. **EC2:** Vá em EC2 > Instances, selecione a instância de teste, clique em **Instance State > Terminate instance**.

---

### Opção B: Usando CloudFormation

1. Abra o laboratório AWS e inicie o **CloudShell** (`>_`).
2. Crie o arquivo template executando o comando abaixo no terminal (cole o YAML dentro do bloco `EOF`):
```bash
cat > guardrail-cost.yaml << 'EOF'
# Cole o conteúdo do YAML abaixo
EOF
```

**Template YAML:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: GuardRail de Custos - Para instancias EC2 com a tag AutoStop=true'

Resources:
  # 1. Instância EC2 de teste (já nasce com a tag que o Lambda vai buscar)
  MinhaInstanciaDeTeste:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0c55b159cbfafe1f0 # Amazon Linux 2 (verifique se esta AMI existe na sua região)
      Tags:
        - Key: Name
          Value: Instancia-GuardRail
        - Key: AutoStop
          Value: 'true'

  # 2. Role do IAM para o Lambda
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

  # 3. Funcao Lambda
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
                  print(f'Instancias paradas: {instance_ids}')
                  return f'Paradas: {instance_ids}'
              else:
                  print('Nenhuma instancia encontrada.')
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

3. Valide o arquivo:
```bash
aws cloudformation validate-template --template-body file://guardrail-cost.yaml
```
4. Crie a stack (o `CAPABILITY_NAMED_IAM` é obrigatório pois criamos uma Role com nome específico):
```bash
aws cloudformation create-stack \
  --stack-name GuardRailLab \
  --template-body file://guardrail-cost.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```
5. Acompanhe a criação e o status:
```bash
aws cloudformation wait stack-create-complete --stack-name GuardRailLab
aws cloudformation describe-stacks --stack-name GuardRailLab --query "Stacks[0].StackStatus"
```

#### 🧹 Limpeza (Evitar custos no Academy)
Como o CloudFormation gerencia tudo, a limpeza é feita com um único comando (ele apagará a EC2, Lambda, IAM e EventBridge):
```bash
aws cloudformation delete-stack --stack-name GuardRailLab
```
Para aguardar a exclusão terminar:
```bash
aws cloudformation wait stack-delete-complete --stack-name GuardRailLab
```

---

## Parte 2: Criando APIs com API Gateway

O API Gateway é um serviço totalmente gerenciado que permite criar, publicar, manter, monitorar e proteger APIs em qualquer escala.

### Opção A: Usando a UI (HTTP API)

Integração de uma **HTTP API** com uma função **AWS Lambda**.

#### Passo 1: Criar a Função Lambda (O Backend)
1. No console da AWS, acesse o **Lambda** e clique em **Create function**.
2. Selecione **Author from scratch**.
3. Em **Function name**, digite `minha-funcao-api`.
4. Em **Runtime**, escolha `Python 3.11`.
5. Deixe as permissões padrão (**Create a new role...**).
6. Clique em **Create function**.
7. No bloco **Code source**, substitua o código pelo seguinte:
```python
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Olá do API Gateway + Lambda!"
    }
```
8. Clique em **Deploy**.

#### Passo 2: Criar a HTTP API no API Gateway
1. Na barra de busca, acesse o **API Gateway**.
2. Clique em **Create API**.
3. No cartão **HTTP API**, clique em **Build**.
4. Em **API name**, digite `minha-http-api`.
5. Em **Integrations**, clique em **Add integration** e selecione **Lambda**.
6. No campo **Lambda function**, digite `minha-funcao-api` e selecione-a.
7. Clique em **Next**.
8. Em **Configure routes**, deixe o **Method** como `GET` e o **Path** como `/`.
9. Clique em **Next**.
10. Em **Define stages**, digite `prod` no **Stage name** e mantenha o **Auto-deploy** marcado.
11. Clique em **Next** e depois em **Create**.

#### Passo 3: Testar a API
1. Copie a **Invoke URL** gerada (ex: `https://abc123.execute-api.us-east-1.amazonaws.com/prod`).
2. Abra no navegador ou use o CloudShell: `curl https://SUA-URL`.
3. Você deverá ver a mensagem `"Olá do API Gateway + Lambda!"`.

#### 🧹 Limpeza (Evitar custos no Academy)
1. **API Gateway:** No console do API Gateway, selecione a API `minha-http-api`, clique em **Actions > Delete**.
2. **Lambda:** No console do Lambda, selecione a função `minha-funcao-api` e exclua-a (**Actions > Delete**).
3. **IAM:** A role criada automaticamente pelo Lambda pode ser excluída (busque pelo nome da função no IAM > Roles).

---

### Opção B: Usando CloudFormation (REST API)

Esta é a forma "completa" para produção. O template abaixo cria a Role, a Função Lambda, a REST API, o método GET, a integração, o deploy para o estágio `prod` e a permissão do API Gateway.

1. No CloudShell, crie o arquivo `api-gateway.yaml`:
```bash
cat > api-gateway.yaml << 'EOF'
# Cole o template YAML abaixo
EOF
```

**Template YAML:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: API REST integrada ao Lambda'

Resources:
  # 1. Role de execução para o Lambda
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

  # 2. A Função Lambda (Backend)
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

  # 3. A API no API Gateway
  MinhaApiRest:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: "MinhaPrimeiraAPI"

  # 4. O Recurso (/hello)
  RecursoHello:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref MinhaApiRest
      ParentId: !GetAtt MinhaApiRest.RootResourceId
      PathPart: "hello"

  # 5. O Método (GET) com Integração Proxy
  MetodoGetHello:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref MinhaApiRest
      ResourceId: !Ref RecursoHello
      HttpMethod: "GET"
      AuthorizationType: "NONE"
      Integration:
        Type: "AWS_PROXY"
        IntegrationHttpMethod: "POST"
        Uri: !Sub 
          - "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaArn}/invocations"
          - LambdaArn: !GetAtt MinhaFuncaoHello.Arn

  # 6. Permissão para o API Gateway invocar o Lambda
  PermissaoApiInvocaLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt MinhaFuncaoHello.Arn
      Action: "lambda:InvokeFunction"
      Principal: "apigateway.amazonaws.com"
      SourceArn: !Sub "arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${MinhaApiRest}/*"

  # 7. Deploy da API (Obrigatório para gerar a URL pública)
  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: MetodoGetHello
    Properties:
      RestApiId: !Ref MinhaApiRest
      StageName: "prod"
EOF
```

2. Crie a stack:
```bash
aws cloudformation create-stack \
  --stack-name ApiGatewayLab \
  --template-body file://api-gateway.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```
3. Acompanhe a criação:
```bash
aws cloudformation wait stack-create-complete --stack-name ApiGatewayLab
```
4. Obtenha a URL da API no console do API Gateway (Stages > prod) ou via CLI.

#### 🧹 Limpeza (Evitar custos no Academy)
Para apagar a API, a Lambda e a Role do IAM de uma só vez, basta deletar a stack:
```bash
aws cloudformation delete-stack --stack-name ApiGatewayLab
```
Para aguardar a exclusão terminar:
```bash
aws cloudformation wait stack-delete-complete --stack-name ApiGatewayLab
```
