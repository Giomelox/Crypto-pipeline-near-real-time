# Crypto Pipeline Near Real Time 

## Este projeto implementa um pipeline de dados orientado a eventos para ingestão e processamento de dados de mercado de criptomoedas, simulando um cenário near real-time em ambiente AWS. Em um ambiente produtivo, a ingestão contínua seria realizada com Amazon Kinesis e processamento streaming dedicado. Neste projeto, o comportamento foi simulado utilizando agendamentos periódicos e eventos do S3, mantendo os princípios arquiteturais de um pipeline real-time.

# Fluxo Geral

EventBridge (2 min schedule) → Lambda (API ingestion) → S3 Bronze (raw JSON) → EventBridge (via S3 Event) → Glue Workflow → Glue Job Silver (typed + partitioned parquet) → Glue Job Gold (hourly & daily analytics) → AWS Glue Data Catalog + AWS Athena

# Decisões Arquiteturais

- A ingestão foi implementada via polling (EventBridge 1 min) para simular streaming em ambiente gratuito.

- O processamento Silver é orientado a evento (S3 → EventBridge → Glue Workflow).

- A camada Gold é processada em batch controlado (Hourly/Daily) para simular agregações near real-time.

- O uso de Parquet foi escolhido por otimização de leitura analítica e compressão columnar.

- O particionamento é baseado no timestamp da fonte (e não da ingestão) para manter consistência temporal.

# Serviços Utilizados:

- AWS IAM Role + IAM Policy
- AWS S3
- AWS Lambda
- AWS EventBridge
- AWS Glue Job
- AWS Glue Workflow
- AWS Glue Data Catalog
- AWS Athena
- AWS CloudWatch

# O projeto segue o padrão:

- Bronze → dados brutos

- Silver → dados tratados e tipados

- Gold → dados agregados e prontos para analytics

# Esse modelo garante:

- Governança

- Reprocessamento simples

- Separação clara de responsabilidades entre armazenamentos

- Escalabilidade futura

# Características Técnicas:

- Arquitetura orientada a eventos

- Particionamento temporal

- Escrita em formato columnar (Parquet)

- Workflow para orquestração

- Controle explícito de tipos

# Idempotência

Os jobs da camada Gold são idempotentes. Cada execução sobrescreve apenas a partição correspondente ao período processado, garantindo consistência caso o job seja reexecutado.

# Processamento Incremental

Os jobs Gold processam apenas as partições necessárias da camada Silver, evitando leitura completa do dataset e reduzindo custo computacional.

# Modelo Medalhão:

## Camada Bronze

A camada Bronze armazena os dados brutos exatamente como retornados pela API pública da CoinGecko.

• Características:

- Formato: JSON

- Particionamento por data (year/month/day)

- Nenhuma transformação aplicada

- Dados imutáveis

A ingestão é realizada por uma função Lambda acionada a cada 1 minuto por EventBridge.

• Responsabilidades da Lambda:

- Consumir API externa

- Criar partição temporal

- Persistir JSON bruto no S3

- Garantir rastreabilidade por timestamp

Esse modelo simula ingestão contínua (stream-like) via polling controlado.

## Camada Silver

• A camada Silver é responsável por:

- Tipagem explícita das colunas

- Renomeação padronizada

- Conversão de timestamps

- Particionamento por data da fonte

- Escrita em formato Parquet

O processamento é realizado por um Glue Job acionado por um Glue Workflow, disparado por evento de criação de objeto no S3.

• Transformações Aplicadas:

- Cast para Double, Integer e Long

- Conversão de last_updated para timestamp

- Criação de coluna processed_at

- Particionamento por year/month/day

O objetivo dessa camada é fornecer dados consistentes e prontos para consumo analítico.

## Camada Gold

• A camada Gold é responsável por:

- Agregações por hora

- Agregações por dia

- Métricas consolidadas

- Base otimizada para BI e dashboards

O processamento é realizado por 2 Glues Jobs distintos, com schedules acionados por hora e por dia, para manter métricas relevantes para análise posterior.

# AWS S3 Buckets Estruturados por Camada

Estruturação do S3 com modelo medalhão:

<img width="1052" height="448" alt="image" src="https://github.com/user-attachments/assets/56e2b334-5428-427f-9899-8e44dc9f97e1" />

# AWS Lambda 

<img width="1087" height="455" alt="image" src="https://github.com/user-attachments/assets/773b6f72-fc26-42cc-a6fa-1b68cacbf137" />

Código de exemplo para extração do arquivo JSON e inserção na camada bronze do S3:

`````

import json
import boto3
import os
from datetime import datetime
import urllib.request

s3 = boto3.client("s3")
BUCKET_NAME = os.environ.get("Digite o nome da variável de ambiente que você salvar no lambda")

# URL da API
API_URL = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd" # URL publica da API

def lambda_handler(event, context):
    """
    Lambda disparada pelo EventBridge.
    - Faz requisição à API externa
    - Salva os dados brutos na camada bronze do S3
    """

    try:
        # Chama a API externa
        with urllib.request.urlopen(API_URL) as response:
            payload = json.loads(response.read().decode())

        # Cria partição por data
        now = datetime.utcnow()
        partition_path = now.strftime("year=%Y/month=%m/day=%d")
        file_name = f"crypto_{now.strftime('%Y%m%dT%H%M%S%f')}.json"
        s3_key = f"bronze/crypto/{partition_path}/{file_name}"

        # Serializa e envia para o S3
        body = json.dumps(payload, ensure_ascii = False)
        s3.put_object(
            Bucket = BUCKET_NAME,
            Key = s3_key,
            Body = body.encode("utf-8"),
            ContentType = "application/json"
        )

        return {
            "statusCode": 200,
            "message": f"Arquivo salvo em s3://{BUCKET_NAME}/{s3_key}"
        }

    except Exception as e:
        print('Ocorreu um Erro:', e)
        return {
            "statusCode": 500,
            "error": str(e)
        }
        
`````

## Métricas do Lambda:

<img width="1600" height="697" alt="image" src="https://github.com/user-attachments/assets/b733425c-c3b4-4844-8395-2afbbd4f7c1b" />

## JSON Registrado na camada bronze:

<img width="1600" height="363" alt="image" src="https://github.com/user-attachments/assets/b5979d35-06b7-4fd5-8b0d-82c1abc48df0" />

## Configurações do EventBridge Schedule:

<img width="1564" height="691" alt="image" src="https://github.com/user-attachments/assets/c2d0a136-316b-456d-964e-ad6f484fbe6b" />

# AWS S3 Events

Configurações de propriedades do bucket bronze:

<img width="1600" height="377" alt="image" src="https://github.com/user-attachments/assets/4a713020-576b-45f3-8621-33a285198a10" />

## Event Pattern configurado no EventBridge Rule:

<img width="1545" height="715" alt="image" src="https://github.com/user-attachments/assets/c5203c56-e509-4869-ae2c-f0be962c0845" />

## Configurações do Target dentro do EventBridge Rule:

<img width="1577" height="576" alt="image" src="https://github.com/user-attachments/assets/86c56489-c02f-4493-9f8e-ec53aa015ce7" />

# AWS Glue Workflow

Configurações do trigger para a orquestração funcionar corretamente:

<img width="1600" height="692" alt="image" src="https://github.com/user-attachments/assets/013910f0-ecb5-4685-ba5a-97ecf8c6c7f0" />

# AWS Glue Job (Silver)

Código utilizado para o funcionamento do processamento do JSON:

`````

import sys
from pyspark.sql.functions import (
    col,
    to_timestamp,
    year,
    month,
    dayofmonth,
    current_timestamp
)
from pyspark.sql.types import DoubleType, IntegerType, LongType, StringType
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext


# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session


# =========================
# Caminhos
# =========================
BRONZE_PATH = "s3://main-bronze-bucket/bronze/crypto/"
SILVER_PATH = "s3://main-silver-bucket/silver/crypto/"


# =========================
# Leitura Bronze
# =========================
df = spark.read.json(BRONZE_PATH)


# =========================
# Transformações (renomeação + cast de tipos)
# =========================
df_transformed = (
    df.select(
        col("id").cast(StringType()).alias("name"),
        col("current_price").cast(DoubleType()).alias("price_usd"),
        col("market_cap").cast(LongType()).alias("market_cap_usd"),
        col("market_cap_rank").cast(IntegerType()).alias("market_rank"),
        col("high_24h").cast(DoubleType()).alias("high_24h_usd"),
        col("low_24h").cast(DoubleType()).alias("low_24h_usd"),
        col("price_change_24h").cast(DoubleType()).alias("price_change_24h_usd"),
        col("price_change_percentage_24h").cast(DoubleType()).alias("price_change_pct_24h"),
        col("ath").cast(DoubleType()).alias("all_time_high_usd"),
        col("ath_change_percentage").cast(DoubleType()).alias("all_time_high_change_pct"),
        col("atl").cast(DoubleType()).alias("all_time_low_usd"),
        col("atl_change_percentage").cast(DoubleType()).alias("all_time_low_change_pct"),
        to_timestamp(
            col("last_updated"),
            "yyyy-MM-dd'T'HH:mm:ss.SSSX"
        ).alias("update_timestamp"),
        current_timestamp().alias("processed_at")
    )
)

# Remove registros inválidos
df_transformed = df_transformed.filter(
    col("update_timestamp").isNotNull()
)


# =========================
# Particionamento por data da fonte
# =========================
df_partitioned = (
    df_transformed
        .withColumn("year", year(col("update_timestamp")))
        .withColumn("month", month(col("update_timestamp")))
        .withColumn("day", dayofmonth(col("update_timestamp")))
)


# =========================
# Escrita Silver em Parquet
# =========================
(
    df_partitioned
        .write
        .mode("append")
        .partitionBy("year", "month", "day")
        .parquet(SILVER_PATH)
)

print("Job finalizado com sucesso.")

`````

## Run do Job rodando com sucesso:

<img width="1570" height="712" alt="image" src="https://github.com/user-attachments/assets/feeb996a-8146-426c-942a-c4772572792b" />

## Arquivo criado com sucesso na camada silver:

<img width="1600" height="348" alt="image" src="https://github.com/user-attachments/assets/aa32ed88-0dc1-403f-a92e-88448cadc766" />

# AWS Glue (Gold - p/hour)

Código utilizado para o processamento da camada prata a cada hora:

`````

import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql.functions import (
    col,
    avg,
    max,
    min,
    stddev,
    year,
    month,
    dayofmonth,
    date_trunc,
    first,
    count,
    input_file_name,
    hour
)
from pyspark.sql.types import DecimalType
from pyspark.sql.window import Window
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from datetime import datetime, timedelta
import boto3
import time

# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

# =========================
# Caminhos
# =========================
SILVER_PATH = "s3://main-silver-bucket/silver/crypto"
GOLD_PATH = "s3://main-gold-bucket/gold/crypto/hourly"

# =========================
# Data, ano, mês, dias e horas atuais
# =========================
target_time = datetime.utcnow() - timedelta(hours = 1)

target_year = target_time.year
target_month = target_time.month
target_day = target_time.day
target_hour = target_time.hour

# =========================
# Leitura Silver filtrando por dia atual (para não gerar lentidão nem custo lendo todo o silver)
# =========================
silver_path_today = (
    f"{SILVER_PATH}/year={target_year}/"
    f"month={target_month}/"
    f"day={target_day}/"
)

df = spark.read.parquet(silver_path_today)

# =========================
# Truncar e filtrar apenas hora atual
# =========================
df = df.filter(
    hour(col("update_timestamp")) == target_hour
)

df = df.withColumn(
    "hour_timestamp",
    date_trunc("hour", col("update_timestamp"))
)

# =========================
# Window para OHLC real
# =========================
window_spec_open = Window.partitionBy("name", "hour_timestamp") \
                          .orderBy(col("update_timestamp").asc())

window_spec_close = Window.partitionBy("name", "hour_timestamp") \
                           .orderBy(col("update_timestamp").desc())

df = df.withColumn("price_open_hour",
                   first("price_usd").over(window_spec_open))

df = df.withColumn("price_close_hour",
                   first("price_usd").over(window_spec_close))
                   
# =========================
# Agregação por ativo + hora
# =========================
df_gold = (
    df.groupBy("name", "hour_timestamp")
    .agg(
         max("price_open_hour").alias("price_open_hour"),
         max("price_close_hour").alias("price_close_hour"),
         max("price_usd").alias("price_high_hour"),
         min("price_usd").alias("price_low_hour"),
         avg("price_usd").alias("price_avg_hour"),
         stddev("price_usd").alias("price_volatility_hour"),
         avg("market_cap_usd").alias("market_cap_avg_hour"),
         max("market_cap_usd").alias("market_cap_max_hour"),
         min("market_cap_usd").alias("market_cap_min_hour"),
         avg("price_change_24h_usd").alias("avg_price_change_24h_usd"),
         avg("price_change_pct_24h").alias("avg_price_change_pct_24h"),
         count("*").alias("records_in_hour")
    )
)

df_gold = df_gold.select(
    col("name"),
    col("hour_timestamp"),

    col("price_open_hour").cast(DecimalType(20,10)),
    col("price_close_hour").cast(DecimalType(20,10)),
    col("price_high_hour").cast(DecimalType(20,10)),
    col("price_low_hour").cast(DecimalType(20,10)),
    col("price_avg_hour").cast(DecimalType(20,10)),
    col("price_volatility_hour").cast(DecimalType(20,10)),

    col("market_cap_avg_hour").cast(DecimalType(20,2)),
    col("market_cap_max_hour"),
    col("market_cap_min_hour"),

    col("avg_price_change_24h_usd").cast(DecimalType(20,10)),
    col("avg_price_change_pct_24h").cast(DecimalType(10,6)),

    col("records_in_hour")
)

# =========================
# Particionamento por data da fonte
# =========================
df_gold = (
    df_gold
        .withColumn("year", year(col("hour_timestamp")))
        .withColumn("month", month(col("hour_timestamp")))
        .withColumn("day", dayofmonth(col("hour_timestamp")))
        .withColumn("hour", hour(col("hour_timestamp")))
)

# =========================
# Escrita Gold
# =========================
(
    df_gold
        .write
        .mode("overwrite")
        .partitionBy("year", "month", "day", "hour")
        .parquet(GOLD_PATH)
)

print("Job finalizado com sucesso.")

`````

## Configuração do Glue Schedule:

<img width="1578" height="383" alt="image" src="https://github.com/user-attachments/assets/7bcfec51-9f02-4dcb-a66d-bd9823ca03f4" />

## Run do Job rodando com sucesso:

<img width="1573" height="709" alt="image" src="https://github.com/user-attachments/assets/243a45e2-a2af-4d02-a6b9-dea93dd599c5" />

## Arquivo criado com sucesso na camada gold:

<img width="1600" height="354" alt="image" src="https://github.com/user-attachments/assets/8886ac94-7de5-41cd-8717-a671262c74ca" />

# AWS Glue (Gold p/day)

Código utilizado para o processamento da camada prata a cada dia:

`````

import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql.functions import (
    col,
    avg,
    max,
    min,
    stddev,
    year,
    month,
    dayofmonth,
    date_trunc,
    first,
    count
)
from pyspark.sql.types import DecimalType
from pyspark.sql.window import Window
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from datetime import datetime, timedelta
import boto3
import time

# =========================
# Inicialização
# =========================
args = getResolvedOptions(sys.argv, [])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

# =========================
# Caminhos
# =========================
SILVER_PATH = "s3://main-silver-bucket/silver/crypto/"
GOLD_PATH = "s3://main-gold-bucket/gold/crypto/daily"

# =========================
# Dia fechado anterior
# =========================
target_date = (datetime.utcnow() - timedelta(days = 1)).date()

start_day = datetime.combine(target_date, datetime.min.time())
end_day = start_day + timedelta(days = 1)

target_year = start_day.year
target_month = start_day.month
target_day = start_day.day

# =========================
# Leitura apenas da partição do dia
# =========================
silver_path_target = (
    f"{SILVER_PATH}/year={target_year}/"
    f"month={target_month}/"
    f"day={target_day}/"
)

df = spark.read.parquet(silver_path_target)

# =========================
# Filtro por intervalo do dia (robusto)
# =========================
df = df.filter(
    (col("update_timestamp") >= start_day) &
    (col("update_timestamp") < end_day)
)

df = df.withColumn(
    "day_timestamp",
    date_trunc("day", col("update_timestamp"))
)

# =========================
# Window para OHLC diário real
# =========================
window_spec_open = Window.partitionBy("name", "day_timestamp") \
                         .orderBy(col("update_timestamp").asc())

window_spec_close = Window.partitionBy("name", "day_timestamp") \
                          .orderBy(col("update_timestamp").desc())

df = df.withColumn(
    "price_open_day",
    first("price_usd").over(window_spec_open)
)

df = df.withColumn(
    "price_close_day",
    first("price_usd").over(window_spec_close)
)

# =========================
# Agregação diária
# =========================
df_gold = (
    df.groupBy("name", "day_timestamp")
      .agg(
          max("price_open_day").alias("price_open_day"),
          max("price_close_day").alias("price_close_day"),
          max("price_usd").alias("price_high_day"),
          min("price_usd").alias("price_low_day"),
          avg("price_usd").alias("price_avg_day"),
          stddev("price_usd").alias("price_volatility_day"),
          avg("market_cap_usd").alias("market_cap_avg_day"),
          max("market_cap_usd").alias("market_cap_max_day"),
          min("market_cap_usd").alias("market_cap_min_day"),
          avg("price_change_24h_usd").alias("avg_price_change_24h_usd"),
          avg("price_change_pct_24h").alias("avg_price_change_pct_24h"),
          count("*").alias("records_in_day")
      )
)

df_gold = df_gold.select(
    col("name"),
    col("day_timestamp"),

    col("price_open_day").cast(DecimalType(20,10)),
    col("price_close_day").cast(DecimalType(20,10)),
    col("price_high_day").cast(DecimalType(20,10)),
    col("price_low_day").cast(DecimalType(20,10)),
    col("price_avg_day").cast(DecimalType(20,10)),
    col("price_volatility_day").cast(DecimalType(20,10)),

    col("market_cap_avg_day").cast(DecimalType(20,2)),
    col("market_cap_max_day"),
    col("market_cap_min_day"),

    col("avg_price_change_24h_usd").cast(DecimalType(20,10)),
    col("avg_price_change_pct_24h").cast(DecimalType(10,6)),

    col("records_in_day")
)

# =========================
# Particionamento
# =========================
df_gold = (
    df_gold
        .withColumn("year", year(col("day_timestamp")))
        .withColumn("month", month(col("day_timestamp")))
        .withColumn("day", dayofmonth(col("day_timestamp")))
)

# =========================
# Escrita idempotente
# =========================
(
    df_gold
        .write
        .mode("overwrite")
        .partitionBy("year", "month", "day")
        .parquet(GOLD_PATH)
)

print('Job diário finalizado com sucesso.')

`````

## Configuração do Glue Schedule:

<img width="1585" height="364" alt="image" src="https://github.com/user-attachments/assets/e4cad315-cc3f-4bdd-9d68-083550561785" />

## Run do Job rodando com sucesso:

<img width="1582" height="719" alt="image" src="https://github.com/user-attachments/assets/eea87e3e-1604-4b34-929a-420e66f29914" />

## Arquivo criado com sucesso na camada gold:

<img width="1859" height="419" alt="image" src="https://github.com/user-attachments/assets/6c2a5e62-caca-47c9-8823-828e12f128fc" />

## Ambas as partições criadas (Daily e Hourly):

<img width="1862" height="515" alt="image" src="https://github.com/user-attachments/assets/a1131e30-f1e7-4534-8f0e-cab8fa4c3581" />

# AWS Athena + AWS Glue Data Catalog

## Código utilizado para criar o schema da camada gold (por hora)

`````

CREATE EXTERNAL TABLE hourly (
  name string,
  hour_timestamp timestamp,
  price_open_hour decimal(20,10),
  price_close_hour decimal(20,10),
  price_high_hour decimal(20,10),
  price_low_hour decimal(20,10),
  price_avg_hour decimal(20,10),
  price_volatility_hour decimal(20,10),
  market_cap_avg_hour decimal(20,2),
  market_cap_max_hour bigint,
  market_cap_min_hour bigint,
  avg_price_change_24h_usd decimal(20,10),
  avg_price_change_pct_24h decimal(10,6),
  records_in_hour bigint
)
PARTITIONED BY (
  year int,
  month int,
  day int,
  hour int
)
STORED AS PARQUET
LOCATION 's3://main-gold-bucket/gold/crypto/hourly/';

`````

Partition projection configurado via código:

`````

ALTER TABLE hourly
SET TBLPROPERTIES (
  'projection.enabled'='true',

  'projection.year.type'='integer',
  'projection.year.range'='2024,2030',

  'projection.month.type'='integer',
  'projection.month.range'='1,12',

  'projection.day.type'='integer',
  'projection.day.range'='1,31',

  'projection.hour.type'='integer',
  'projection.hour.range'='0,23',

  'storage.location.template'='s3://main-gold-bucket/gold/crypto/hourly/year=${year}/month=${month}/day=${day}/hour=${hour}/'
);

`````

## Código utilizado para criar o schema da camada gold (por dia)

`````

CREATE EXTERNAL TABLE daily (
  name string,
  day_timestamp timestamp,
  price_open_day decimal(20,10),
  price_close_day decimal(20,10),
  price_high_day decimal(20,10),
  price_low_day decimal(20,10),
  price_avg_day decimal(20,10),
  price_volatility_day decimal(20,10),
  market_cap_avg_day decimal(20,2),
  market_cap_max_day bigint,
  market_cap_min_day bigint,
  avg_price_change_24h_usd decimal(20,10),
  avg_price_change_pct_24h decimal(10,6),
  records_in_day bigint
)
PARTITIONED BY (
  year int,
  month int,
  day int
)
STORED AS PARQUET
LOCATION 's3://main-gold-bucket/gold/crypto/daily/';

`````

Partition projection configurado via código:

`````

ALTER TABLE daily
SET TBLPROPERTIES (
  'projection.enabled'='true',

  'projection.year.type'='integer',
  'projection.year.range'='2024,2030',

  'projection.month.type'='integer',
  'projection.month.range'='1,12',

  'projection.day.type'='integer',
  'projection.day.range'='1,31',

  'storage.location.template'='s3://main-gold-bucket/gold/crypto/daily/year=${year}/month=${month}/day=${day}/'
);

`````

## DataBase criado dentro do Data Catalog e tabelas criadas com sucesso após o run do Athena:

<img width="1584" height="420" alt="image" src="https://github.com/user-attachments/assets/6cfc7307-8547-4162-86c9-9b8159f5a197" />

## Schemas padronizados dentro da tabela daily criada:

<img width="1560" height="751" alt="image" src="https://github.com/user-attachments/assets/616b4968-3859-4977-a657-5b03c87082ca" />

## Schemas padronizados dentro da tabela hourly criada:

<img width="1571" height="780" alt="image" src="https://github.com/user-attachments/assets/bc6cd398-9449-4792-ae50-542d337050be" />

## SELECT utilizado para testar consulta da tabela hourly no Athena:

`````

SELECT *
FROM hourly
WHERE year = 2026 AND month = 3 AND day = 4 AND hour = 22;

`````

## SELECT utilizado para testar consulta da tabela daily no Athena:

`````

SELECT *
FROM daily
WHERE year = 2026 AND month = 3 AND day = 17;

`````

# Fim do Projeto
