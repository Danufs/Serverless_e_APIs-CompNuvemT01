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
   - Selecione a role `LabRole`.
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
            {'Name': 'tag:AutoStop', 'Values': ['true']},
            {'Name': 'instance-type', 'Values': ['t3.micro']}
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
        print('Nenhuma instancia t3.small com a tag AutoStop=true foi encontrada.')
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
   - **Schedule expression:** `rate(1 minutes)`
5. Clique em **Add**.

#### 🧹 Limpeza (Evitar custos no Academy)
Para não consumir créditos, apague tudo o que foi criado:
1. **EventBridge:** Vá em EventBridge > Rules, selecione `GuardRailSchedule` e clique em **Delete**.
2. **Lambda:** Vá em Lambda, selecione `guardrail-stop-by-tag` e clique em **Actions > Delete**.
3. **EC2:** Vá em EC2 > Instances, selecione a instância de teste, clique em **Instance State > Terminate instance**.

---

### Opção B: Usando CloudFormation

1. Abra o laboratório AWS e inicie o **CloudShell** (`>_`).

2. Crie o arquivo template executando o comando abaixo. **Cole o Template YAML (mostrado logo depois) entre as linhas `cat` e `EOF`** — o `EOF` só fecha no fim:

```bash
cat > guardrail-cost.yaml << 'EOF'
# Cole o Template YAML aqui dentro (entre esta linha e o EOF abaixo)
EOF
```

**Template YAML:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: GuardRail de Custos - Para instancias EC2 com a tag AutoStop=true'

Parameters:
  LabRoleName:
    Type: String
    Default: 'LabRole'
    Description: 'Nome da role pre-criada no AWS Academy Learner Lab'
  AmiId:
    Type: AWS::EC2::Image::Id
    Default: 'ami-0292859281a0c8e80'
    Description: 'AMI Amazon Linux'
  LambdaTimeout:
    Type: Number
    Default: 30
    MinValue: 3
    MaxValue: 900
    Description: 'Timeout da Lambda em segundos'

Resources:
  # 1. Instancia EC2 de teste (ja nasce com a tag que o Lambda vai buscar)
  MinhaInstanciaDeTeste:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref AmiId
      Tags:
        - Key: Name
          Value: Instancia-GuardRail
        - Key: AutoStop
          Value: 'true'

  # 2. Funcao Lambda - usa a LabRole existente
  GuardRailFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: 'guardrail-stop-by-tag'
      Runtime: 'python3.11'
      Handler: 'index.lambda_handler'
      Role: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${LabRoleName}'
      Timeout: !Ref LambdaTimeout
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
                      {'Name': 'tag:AutoStop', 'Values': ['true']},
                      {'Name': 'instance-type', 'Values': ['t3.micro']}
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
                  print('Nenhuma instancia t3.micro com a tag AutoStop=true foi encontrada.')
                  return 'Nada a fazer.'

  # 3. Permissao para o EventBridge invocar o Lambda
  PermissionForEvents:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref GuardRailFunction
      Action: 'lambda:InvokeFunction'
      Principal: 'events.amazonaws.com'

  # 4. Regra do EventBridge (Agendamento)
  ScheduleRule:
    Type: AWS::Events::Rule
    Properties:
      Name: 'GuardRailSchedule'
      Description: 'Dispara o GuardRail a cada 1 minuto'
      ScheduleExpression: 'rate(1 minute)'
      State: 'ENABLED'
      Targets:
        - Arn: !GetAtt GuardRailFunction.Arn
          Id: 'GuardRailTarget'
```

> **Dica**: se preferir, cole o comando inteiro de uma vez (com o YAML já dentro). Só é essencial que a linha `EOF` fique **abaixo** da última linha do YAML — se ela ficar logo depois do `cat`, o arquivo sai vazio.

3. Confirme que o arquivo foi salvo com o conteúdo (deve ter cerca de 75 linhas, não 1 nem 2):

```bash
wc -l guardrail-cost.yaml
```

4. Valide o template:

```bash
aws cloudformation validate-template --template-body file://guardrail-cost.yaml
```

5. Crie a stack:

```bash
aws cloudformation create-stack \
  --stack-name GuardRailLab \
  --template-body file://guardrail-cost.yaml
```

6. Acompanhe a criação e o status:

```bash
aws cloudformation wait stack-create-complete --stack-name GuardRailLab
aws cloudformation describe-stacks --stack-name GuardRailLab --query "Stacks[0].StackStatus"
```

O status esperado é `CREATE_COMPLETE`.

#### 🧹 Limpeza (Evitar custos no Academy)

Como o CloudFormation gerencia tudo, a limpeza é feita com um único comando (ele apagará a EC2, a Lambda, a permissão e a regra do EventBridge):

```bash
aws cloudformation delete-stack --stack-name GuardRailLab
```

Para aguardar a exclusão terminar:

```bash
aws cloudformation wait stack-delete-complete --stack-name GuardRailLab
```

---

Encontrei os erros! O principal motivo do seu `403 Forbidden` anterior está justamente aqui no texto. 

O seu arquivo tem **três erros graves** que impedem a API de funcionar e o teste de passar. Vou listá-los e depois te dar o texto corrigido.

### 🚨 Os Erros no arquivo:

1. **Incompatibilidade de Rota (O motivo do seu erro 403)**:
   * No **Passo 2**, você manda criar a rota como `GET /` (apenas a barra).
   * No **Passo 5**, você manda testar a URL `.../status`. 
   * **O problema:** A rota `/status` não existe na sua API (você criou a `/`). Quando você chama uma rota que não existe, o API Gateway retorna `403 Forbidden`.
   * **Solução:** Ou você cria a rota como `/status` no Passo 2, ou testa apenas a URL base no Passo 5.

2. **Confusão no IAM (Passo 3)**:
   * O título fala em "Lambda Authorizer", mas o texto manda selecionar o "IAM". 
   * Além disso, **não existe** essa opção "Manage authorizers > selecionar o IAM" no console do HTTP API. O IAM não é um "Authorizer" na interface; ele é ativado na própria rota.
   * **Solução:** Para usar IAM no HTTP API, você precisa ir em **Routes**, selecionar a rota `GET /`, clicar em **Edit**, e mudar o *Authorization type* para **AWS_IAM**.

3. **Código do Lambda (Passo 1)**:
   * O código no texto diz `"mensagem": "Olá do HTTP API + Lambda!"`.
   * Mas no Passo 5, a explicação diz que o retorno será "o corpo JSON com a lista de instâncias do seu Lambda".
   * **O problema:** O código não busca instâncias, só devolve um "Olá". É só uma inconsistência de texto, mas confunde na hora de explicar.

---

## Parte 2: Criando APIs com API Gateway (HTTP API)**

**Opção A: Usando a UI (HTTP API)**
Integração de uma HTTP API com uma função AWS Lambda, incluindo configurações de segurança (IAM) e monitoramento.

**Passo 1: Criar a Função Lambda (O Backend)**
1. No console da AWS, acesse o **Lambda** e clique em **Create function**.
2. Selecione **Author from scratch**.
3. Em *Function name*, digite `minha-funcao-api`.
4. Em *Runtime*, escolha `Python 3.12`.
5. Em *Permissions*, selecione **Use an existing role** e escolha a `LabRole`.
6. Clique em **Create function**.
7. No bloco *Code source*, substitua o código pelo seguinte:
```python
import json

def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"mensagem": "Olá do HTTP API + Lambda!"})
    }
```
8. Clique em **Deploy**.

**Passo 2: Criar a HTTP API no API Gateway**
1. Na barra de busca, acesse o **API Gateway** e clique em **Create API**.
2. No cartão **HTTP API**, clique em **Build**.
3. Em *API name*, digite `minha-http-api`.
4. Em *Integrations*, clique em **Add integration** e selecione **Lambda**. Busque a `minha-funcao-api`.
5. Clique em **Next**.
6. Em *Configure routes*, defina:
   - **Method:** `GET`
   - **Path:** `/status` *(Atenção: mude de `/` para `/status` para bater com o teste!)*
7. Clique em **Next**.
8. Em *Define stages*, deixe o Stage name como `$default` e o **Auto-deploy** marcado.
9. Clique em **Next** e depois em **Create**.

**Passo 3: Configurar Autenticação (IAM)**
*No HTTP API, o IAM não fica no menu "Authorizers", ele é aplicado direto na rota.*
1. No menu lateral esquerdo da sua API, clique em **Routes**.
2. Selecione a sua rota `GET /status`.
3. Clique em **Edit**.
4. Em *Authorization type*, mude de `NONE` para **AWS_IAM**.
5. Clique em **Save**.

**Passo 4: Configurar Throttling (Controle de Tráfego)**
1. No menu lateral, clique em **Monitor** > **Throttling**.
2. Selecione o stage `$default` e clique em **Edit**.
3. Defina:
   - **Rate limit:** 5 (requisições por segundo).
   - **Burst limit:** 10 (picos temporários).
4. Clique em **Save**.

**Passo 5: Testar a API Protegida (com IAM)**
1. Copie a **Invoke URL** gerada no topo da tela (ex: `https://k5029wdurj.execute-api.us-east-1.amazonaws.com`).
2. **Teste sem credenciais (deve falhar):**
   - No navegador, tente acessar: `https://SUA-URL-DA-API/status`.
   - **Resultado esperado:** `{"message":"Forbidden"}`. Isso prova que a API está bloqueando acessos anônimos.
3. **Teste com credenciais via CloudShell (deve funcionar):**
   - Vá para o **CloudShell** e execute o comando abaixo. Ele usa a flag `--aws-sigv4` para assinar a requisição usando as suas credenciais do laboratório.
   ```bash
   curl --aws-sigv4 "aws:amz:us-east-1:execute-api" \
        --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
        -H "x-amz-security-token: $AWS_SESSION_TOKEN" \
        -i "https://SUA-URL-DA-API/status"
   ```
   - **Resultado esperado:** `HTTP/2 200` e o corpo JSON `{"mensagem": "Olá do HTTP API + Lambda!"}`.

**🧹 Limpeza (Evitar custos no Academy)**
1. **API Gateway:** Selecione a API `minha-http-api`, clique em **Actions > Delete**.
2. **Lambda:** No console do Lambda, exclua a função `minha-funcao-api`.

---

## Opção B: Usando CloudFormation (HTTP API com Segurança)

No CloudShell, crie o arquivo `api-gateway-http.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Lab: HTTP API integrada ao Lambda com Lambda Authorizer'

Parameters:
  RoleName:
    Type: String
    Default: LabRole # Usando LabRole para facilitar no Academy

Resources:
  # 1. A Função Lambda (Backend)
  MinhaFuncaoHello:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: minha-funcao-api
      Handler: index.lambda_handler
      Runtime: python3.12
      Role: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${RoleName}'
      Code:
        ZipFile: |
          import json
          def lambda_handler(event, context):
              return {
                  "statusCode": 200,
                  "headers": {"Content-Type": "application/json"},
                  "body": json.dumps({"mensagem": "Olá do HTTP API + Lambda!"})
              }

  # 2. A Função Lambda (Authorizer)
  FuncaoAutorizadora:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: minha-funcao-autorizadora
      Handler: index.lambda_handler
      Runtime: python3.12
      Role: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${RoleName}'
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              token = (event.get("headers") or {}).get("authorization", "")
              return {"isAuthorized": token == "segredo123"}

  # 3. A HTTP API
  MinhaHttpApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: "MinhaPrimeiraHTTPAPI"
      ProtocolType: HTTP

  # 4. Integração do Backend
  IntegracaoLambda:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref MinhaHttpApi
      IntegrationType: AWS_PROXY
      IntegrationUri: !GetAtt MinhaFuncaoHello.Arn
      PayloadFormatVersion: '2.0'

  # 5. O Authorizer
  MeuAuthorizer:
    Type: AWS::ApiGatewayV2::Authorizer
    Properties:
      ApiId: !Ref MinhaHttpApi
      Name: meu-autorizador-cfn
      AuthorizerType: REQUEST
      AuthorizerUri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${FuncaoAutorizadora.Arn}/invocations'
      AuthorizerPayloadFormatVersion: '2.0'
      EnableSimpleResponses: true
      IdentitySource:
        - '$request.header.Authorization'

  # 6. A Rota Protegida (GET /)
  RotaPrincipal:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref MinhaHttpApi
      RouteKey: 'GET /'
      Target: !Sub 'integrations/${IntegracaoLambda}'
      AuthorizationType: CUSTOM
      AuthorizerId: !Ref MeuAuthorizer

  # 7. Stage $default com Auto-deploy e Throttling
  StageDefault:
    Type: AWS::ApiGatewayV2::Stage
    Properties:
      ApiId: !Ref MinhaHttpApi
      StageName: '$default'
      AutoDeploy: true
      DefaultRouteSettings:
        ThrottlingRateLimit: 5
        ThrottlingBurstLimit: 10

  # 8. Permissão para o API Gateway invocar o Backend
  PermissaoApiGateway:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref MinhaFuncaoHello
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${MinhaHttpApi}/*/*'

  # 9. Permissão para o API Gateway invocar o Authorizer
  PermissaoAuthorizer:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref FuncaoAutorizadora
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${MinhaHttpApi}/authorizers/${MeuAuthorizer}'

Outputs:
  InvokeURL:
    Description: URL base da HTTP API
    Value: !Sub 'https://${MinhaHttpApi}.execute-api.${AWS::Region}.amazonaws.com'
```

Crie a stack:
```bash
aws cloudformation create-stack \
  --stack-name ApiGatewayLab \
  --template-body file://api-gateway-http.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```
Acompanhe a criação:
```bash
aws cloudformation wait stack-create-complete --stack-name ApiGatewayLab
```
Obtenha a URL para testar:
```bash
aws cloudformation describe-stacks --stack-name ApiGatewayLab --query "Stacks[0].Outputs[0].OutputValue" --output text
```

Teste via CloudShell:
```bash
# Teste sem o token (deve dar 401 Unauthorized)
curl -i https://SUA-URL-DA-API

# Teste com o token (deve dar 200 OK)
curl -H "Authorization: segredo123" https://SUA-URL-DA-API
```

### 🧹 Limpeza (Evitar custos no Academy)
Para apagar TUDO (API, Lambdas, Logs) de uma só vez, basta deletar a stack:
```bash
aws cloudformation delete-stack --stack-name ApiGatewayLab
aws cloudformation wait stack-delete-complete --stack-name ApiGatewayLab
```ma só vez, basta deletar a stack:
```bash
aws cloudformation delete-stack --stack-name ApiGatewayLab
aws cloudformation wait stack-delete-complete --stack-name ApiGatewayLab
```
