# Guia Prático: AWS Lambda e API Gateaway

Este guia documenta o passo a passo para provisionar, configurar e integrar aplicações serverless usando o AWS Lambda e o Amazon API Gateway. Serão demonstradas a criação de funções, a configuração de rotas HTTP e a conexão entre os serviços para expor sua aplicação.

> Aviso: Este laboratório foi desenhado para o ambiente acadêmico (AWS Academy / Learner Lab). As configurações de acesso público e a ausência de autenticação (como chaves de API ou autorizadores) demonstradas aqui não são recomendadas para ambientes corporativos de produção, onde a segurança e o controle de acesso rigorosos são essenciais.

## Criando uma função Lambda ativa por agendamento

### Usando a UI


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

## Usando função lambda por reação a evento junto com API Gateway
