# Guia Prático: AWS Lambda

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
