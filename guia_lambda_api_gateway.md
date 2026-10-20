# Guia Prático: AWS Lambda e API Gateway

Este guia documenta o passo a passo para provisionar, configurar e integrar aplicações serverless usando o AWS Lambda e o Amazon API Gateway. Serão demonstradas a criação de funções, a configuração de rotas HTTP e a conexão entre os serviços para expor sua aplicação.

> **Aviso:** Este laboratório foi desenhado para o ambiente acadêmico (AWS Academy / Learner Lab). As configurações de acesso público e a ausência de autenticação (como chaves de API ou autorizadores) demonstradas aqui não são recomendadas para ambientes corporativos de produção, onde a segurança e o controle de acesso rigorosos são essenciais.

---

## Parte 1: Criando uma função Lambda por agendamento (EventBridge)

Nesta etapa, vamos criar um "GuardRail" (protetor de custos). A função será acionada a cada 5 minutos para buscar e desligar instâncias EC2 que possuam a tag `AutoStop=true`.

### Opção A: Usando a Interface (UI)

#### Pré-requisito: Preparar a Instância EC2
1. Vá ao console do **Amazon EC2** e inicie uma instância (pode ser a `t3.micro` padrão).
2. Após a criação, selecione a instância, vá em **Tags** e adicione:
   - **Chave:** `AutoStop`
   - **Valor:** `true`

#### Passo 1: Criar a Função Lambda
1. No console da AWS, pesquise por **Lambda** e abra o serviço.
2. Clique em **Create function**.
3. Selecione **Author from scratch**.
4. Preencha:
   - **Function name:** `guardrail-stop-by-tag`
   - **Runtime:** `Python 3.11`
   - Em **Permissions**, expanda **Change default execution role** e escolha **Use an existing role**.
   - Selecione a role `LabRole` criada no Passo 1.
5. Clique em **Create function**.

#### Passo 2: Inserir o Código
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

#### Passo 3: Configurar o Gatilho (EventBridge)
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

Integração de uma **HTTP API** com uma função **AWS Lambda**, incluindo configurações de segurança e monitoramento.

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

#### Passo 3: Configurar Autenticação (API Key)
Para evitar que qualquer pessoa acesse sua API, vamos exigir uma chave de acesso.
1. Na sua API (`minha-http-api`), clique em **API Keys** no menu lateral esquerdo e depois em **Create API key**.
2. Dê o nome de `MinhaChaveDeAcesso` e clique em **Create**.
3. Agora, vá em **Usage Plans** > **Create**.
4. Dê o nome de `PlanoBasico`. Na seção **API Stages**, adicione a sua API (`minha-http-api`) e o estágio `prod`.
5. Na seção **Associated API Keys**, adicione a `MinhaChaveDeAcesso` e clique em **Create**.

#### Passo 4: Configurar Throttling (Controle de Tráfego)
Ainda dentro do `PlanoBasico` que você acabou de criar:
1. Clique no plano `PlanoBasico` e vá até a aba **Throttling**.
2. Clique em **Configure** e defina:
   - **Rate Limit:** `5` (limite de 5 requisições por segundo).
   - **Burst Limit:** `10` (picos temporários de até 10 requisições).
3. **O que isso significa?** Se você fizer mais de 5 requisições no mesmo segundo, a API retornará o erro `429 Too Many Requests`, protegendo seu Lambda.

#### Passo 5: Habilitar Observabilidade (Logs e Métricas)
1. Na sua API (`minha-http-api`), clique em **Monitor** no menu lateral esquerdo.
2. Clique em **Logging** e depois em **Manage logging**.
3. Selecione a sua API e o estágio `prod`. Ative a opção **Detailed metrics**.
4. Para habilitar os **Logs de Execução**:
   - Vá em **Stages** > selecione o estágio `prod`.
   - Na aba **Logs and tracing**, habilite o **CloudWatch Logs** (formato JSON).

#### Passo 6: Testar a API Protegida
1. Copie a **Invoke URL** gerada (ex: `https://abc123.execute-api.us-east-1.amazonaws.com/prod`).
2. **Teste sem a chave:** Abra no navegador. Você receberá um erro `403 Forbidden`.
3. **Teste com a chave:** No CloudShell, execute o comando (substitua a URL e a chave):
```bash
curl -H "x-api-key: SUA_CHAVE_AQUI" https://SUA-URL-DA-API
```
4. Você deverá ver a mensagem `"Olá do API Gateway + Lambda!"`.
5. **Verifique os logs:** Vá no console do **CloudWatch** > **Log groups** e abra o log group `aws/apigateway/minha-http-api` para ver os detalhes da requisição bem-sucedida.

#### 🧹 Limpeza (Evitar custos no Academy)
Para apagar tudo o que foi criado:
1. **API Gateway:** Vá em **Usage Plans** e exclua o `PlanoBasico`. Vá em **API Keys** e exclua a `MinhaChaveDeAcesso`.
2. **API Gateway:** Selecione a API `minha-http-api`, clique em **Actions > Delete**.
3. **Lambda:** No console do Lambda, exclua a função `minha-funcao-api`.
4. **IAM:** A role criada automaticamente pelo Lambda pode ser excluída (busque pelo nome da função no IAM > Roles).
5. **CloudWatch:** (Opcional) Exclua o Log Group `aws/apigateway/minha-http-api` para limpar os logs gerados.

---

### Opção B: Usando CloudFormation (REST API com Segurança)

Esta é a forma "completa" para produção. O template abaixo cria a Role, a Função Lambda, a REST API, o método GET, o deploy, **a API Key, o Plano de Uso com Throttling** e as permissões necessárias.

1. No CloudShell, crie o arquivo `api-gateway.yaml`:
```bash
cat > api-gateway.yaml << 'EOF'
# Cole o template YAML abaixo
EOF
```

**Template YAML:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: API REST integrada ao Lambda com API Key e Throttling'

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
                  "body": "Olá do API Gateway + Lambda (Protegido)!"
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

  # 5. O Método (GET) com Integração Proxy e exigindo API Key
  MetodoGetHello:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref MinhaApiRest
      ResourceId: !Ref RecursoHello
      HttpMethod: "GET"
      AuthorizationType: "NONE"
      ApiKeyRequired: true # <--- Exige a API Key
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

  # 7. Deploy da API
  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: MetodoGetHello
    Properties:
      RestApiId: !Ref MinhaApiRest
      StageName: "prod"

  # 8. Criação da API Key
  MinhaApiKey:
    Type: AWS::ApiGateway::ApiKey
    DependsOn: ApiDeployment
    Properties:
      Name: "MinhaChaveDeAcessoCFN"
      Enabled: true
      StageKeys:
        - RestApiId: !Ref MinhaApiRest
          StageName: "prod"

  # 9. Criação do Usage Plan com Throttling
  MeuUsagePlan:
    Type: AWS::ApiGateway::UsagePlan
    DependsOn: ApiDeployment
    Properties:
      UsagePlanName: "PlanoBasicoCFN"
      ApiStages:
        - ApiId: !Ref MinhaApiRest
          Stage: "prod"
      Throttle:
        BurstLimit: 10
        RateLimit: 5

  # 10. Associação da API Key ao Usage Plan
  MinhaApiKeyUsagePlan:
    Type: AWS::ApiGateway::UsagePlanKey
    Properties:
      KeyId: !Ref MinhaApiKey
      KeyType: API_KEY
      UsagePlanId: !Ref MeuUsagePlan
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
4. Obtenha a URL e o ID da API Key para testar:
```bash
# Ver a URL da API
aws apigateway get-stages --rest-api-id $(aws apigateway get-rest-apis --query "items[?name=='MinhaPrimeiraAPI'].id" --output text) --query "item[0].invokeUrl"

# Ver o ID da API Key (para pegar o valor real, precisa usar get-api-key)
aws apigateway get-api-keys --query "items[?name=='MinhaChaveDeAcessoCFN'].id" --output text
```
*(Dica: Para pegar o valor real da chave via CLI, use `aws apigateway get-api-key --api-key ID_DA_CHAVE --include-value`)*

5. **Teste via CloudShell** (lembre-se de substituir a URL e o valor da chave):
```bash
curl -H "x-api-key: SUA_CHAVE_AQUI" https://SUA-URL-DA-API/hello
```

#### 🧹 Limpeza (Evitar custos no Academy)
Para apagar TUDO (API, Lambda, Role, API Key, Usage Plan) de uma só vez, basta deletar a stack:
```bash
aws cloudformation delete-stack --stack-name ApiGatewayLab
aws cloudformation wait stack-delete-complete --stack-name ApiGatewayLab
```
