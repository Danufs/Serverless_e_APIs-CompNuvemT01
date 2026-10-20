# Guia Prático: Amazon EventBridge + AWS Step Functions

Este laboratório demonstra a integração entre o **Amazon EventBridge** e o **AWS Step Functions**, utilizando eventos, regras de roteamento, máquinas de estados, retries e tratamento de falhas.

O objetivo é implementar a mesma solução de duas formas:

1. **Provisionamento manual utilizando AWS CLI**
2. **Provisionamento automatizado utilizando AWS CloudFormation**

> **Objetivo:** Demonstrar uma arquitetura orientada a eventos em que um evento enviado ao EventBridge inicia automaticamente uma execução do Step Functions.

> **Observação:** Este laboratório foi desenvolvido para ambientes educacionais como o AWS Academy. Algumas permissões e papéis IAM podem ser gerenciados automaticamente pelo ambiente e não fazem parte do escopo desta atividade.

---

# Provisionamento Manual (AWS CLI)

## Passo 1: Configuração Inicial

Crie variáveis de ambiente para obter automaticamente o ID da conta e montar o ARN da LabRole:

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

export LAB_ROLE_ARN="arn:aws:iam::$ACCOUNT_ID:role/LabRole"
```

Verifique o ARN gerado:

```bash
echo $LAB_ROLE_ARN
```

Exemplo de saída:

```text
arn:aws:iam::123456789012:role/LabRole
```

---

## Passo 2: Criar um Event Bus

Criar um barramento de eventos personalizado:

```bash
aws events create-event-bus \
  --name academy-bus
```

Verificar criação:

```bash
aws events list-event-buses
```

---

## Passo 3: Criar a Função Lambda

Criar e editar o arquivo:

```bash
nano lambda_function.py
```

Conteúdo:

```python
import random

def lambda_handler(event, context):

    if random.randint(1,10) <= 6:
        raise Exception("Falha simulada")

    return {
        "status": "SUCESSO",
        "event": event
    }
```

Salvar o arquivo e compactar:

```bash
zip lambda.zip lambda_function.py
```

Criar a função:

```bash
aws lambda create-function \
  --function-name process-order \
  --runtime python3.11 \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda.zip \
  --role $LAB_ROLE_ARN
```

---

## Passo 4: Obter o ARN da Lambda

```bash
export LAMBDA_ARN=$(aws lambda get-function \
  --function-name process-order \
  --query 'Configuration.FunctionArn' \
  --output text)
```

Verificar:

```bash
echo $LAMBDA_ARN
```

---

## Passo 5: Criar a State Machine

Criar e editar o arquivo:

```bash
nano state-machine.json
```

Conteúdo:

```json
{
  "StartAt": "WaitState",
  "States": {
    "WaitState": {
      "Type": "Wait",
      "Seconds": 5,
      "Next": "ProcessOrder"
    },
    "ProcessOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ARN_DA_LAMBDA",
        "Payload.$": "$"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "Failure"
        }
      ],
      "Next": "Success"
    },
    "Success": {
      "Type": "Succeed"
    },
    "Failure": {
      "Type": "Fail"
    }
  }
}
```

Substituir valo para ARN do Lambda:

```bash
sed -i "s|ARN_DA_LAMBDA|$LAMBDA_ARN|g" state-machine.json
```

Criar a State Machine:

```bash
aws stepfunctions create-state-machine \
  --name academy-order-workflow \
  --definition file://state-machine.json \
  --role-arn $LAB_ROLE_ARN
```

Obter o ARN da State Machine:

```bash
export STATE_MACHINE_ARN=$(aws stepfunctions list-state-machines \
  --query 'stateMachines[0].stateMachineArn' \
  --output text)
```

Verificar:

```bash
echo $STATE_MACHINE_ARN
```

---

## Passo 6: Criar uma Rule do EventBridge

```bash
aws events put-rule \
  --name academy-order-rule \
  --event-bus-name academy-bus \
  --event-pattern '{
    "source":["academy.demo"],
    "detail-type":["NovoPedido"]
}'
```

---

## Passo 7: Adicionar a State Machine como Target

```bash
aws events put-targets \
  --rule academy-order-rule \
  --event-bus-name academy-bus \
  --targets "Id"="1","Arn"="$STATE_MACHINE_ARN","RoleArn"="$LAB_ROLE_ARN"
```

---

# Teste da Solução

Enviar um evento para o EventBridge:

```bash
aws events put-events \
--entries '[
 {
   "Source":"academy.demo",
   "DetailType":"NovoPedido",
   "EventBusName":"academy-bus",
   "Detail":"{\"pedidoId\":\"1001\",\"cliente\":\"Aluno AWS Academy\"}"
 }
]'
```

---

## Resultado Esperado

No Step Functions será possível visualizar:

### Cenário de Sucesso

```text
WaitState
   ↓
ProcessOrder
   ↓
Success
```

### Cenário com Retry

```text
WaitState
   ↓
ProcessOrder
   ↓
Retry 1
   ↓
Retry 2
   ↓
Success
```

### Cenário de Falha

```text
WaitState
   ↓
ProcessOrder
   ↓
Retry 1
   ↓
Retry 2
   ↓
Retry 3
   ↓
Failure
```

---

# Provisionamento com CloudFormation

Em vez de criar cada recurso individualmente através da AWS CLI, também é possível provisionar toda a solução utilizando uma única Stack CloudFormation.

## Template CloudFormation

Criar e editar o arquivo:

```bash
nano eventbridge-stepfunctions.yml
```

Conteúdo:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EventBridge + Step Functions Demo

Resources:

  AcademyEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: academy-bus

  ProcessOrderFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: process-order
      Runtime: python3.11
      Handler: index.lambda_handler
      Timeout: 10
      Role: arn:aws:iam::<ACCOUNT_ID>:role/LabRole

      Code:
        ZipFile: |
          import random

          def lambda_handler(event, context):
              if random.randint(1,10) <= 6:
                  raise Exception("Falha simulada")

              return {
                  "status": "SUCESSO",
                  "event": event
              }

  OrderStateMachine:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      RoleArn: arn:aws:iam::<ACCOUNT_ID>:role/LabRole

      DefinitionString:
        !Sub |
          {
            "StartAt": "WaitState",
            "States": {
              "WaitState": {
                "Type": "Wait",
                "Seconds": 5,
                "Next": "ProcessOrder"
              },
              "ProcessOrder": {
                "Type": "Task",
                "Resource": "arn:aws:states:::lambda:invoke",
                "Parameters": {
                  "FunctionName": "${ProcessOrderFunction.Arn}",
                  "Payload.$": "$"
                },
                "Retry": [{
                  "ErrorEquals": ["States.ALL"],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 3,
                  "BackoffRate": 2
                }],
                "Catch": [{
                  "ErrorEquals": ["States.ALL"],
                  "Next": "Failure"
                }],
                "Next": "Success"
              },
              "Success": {
                "Type": "Succeed"
              },
              "Failure": {
                "Type": "Fail"
              }
            }
          }

  OrderRule:
    Type: AWS::Events::Rule
    Properties:
      Name: academy-order-rule
      EventBusName: academy-bus

      EventPattern:
        source:
          - academy.demo

        detail-type:
          - NovoPedido

      Targets:
        - Arn: !Ref OrderStateMachine
          Id: StepFunctionsTarget
```

Substituir:

```text
<ACCOUNT_ID>
```

pelo valor exibido em:

```bash
echo $ACCOUNT_ID
```

---

## Criar a Stack

```bash
aws cloudformation deploy \
  --template-file eventbridge-stepfunctions.yml \
  --stack-name academy-event-demo
```

---

# Teste Após o Deploy

```bash
aws events put-events \
--entries '[
 {
   "Source":"academy.demo",
   "DetailType":"NovoPedido",
   "EventBusName":"academy-bus",
   "Detail":"{\"pedidoId\":\"1001\",\"cliente\":\"Aluno AWS Academy\"}"
 }
]'
```

---

# Limpeza dos Recursos

## Caso tenha utilizado CloudFormation

```bash
aws cloudformation delete-stack \
  --stack-name academy-event-demo
```

## Caso tenha realizado o provisionamento manual

```bash
aws events remove-targets \
  --rule academy-order-rule \
  --event-bus-name academy-bus \
  --ids 1

aws events delete-rule \
  --name academy-order-rule \
  --event-bus-name academy-bus

aws stepfunctions delete-state-machine \
  --state-machine-arn "$STATE_MACHINE_ARN"

aws lambda delete-function \
  --function-name process-order

aws events delete-event-bus \
  --name academy-bus
```

> Importante: Remova todos os recursos ao final do laboratório para evitar consumo desnecessário dos créditos disponíveis no AWS Academy.
