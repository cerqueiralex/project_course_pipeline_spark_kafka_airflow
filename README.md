# Pipeline de extração de dados em Tempo real através da API Random User Generator com Kafka, Spark e Airflow

Este projeto implementa um pipeline ETL utilizando Airflow e Docker para extrair dados meteorológicos de uma API pública e armazená-los em um banco de dados SQLite. A aplicação realiza requisições periódicas para capturar informações como temperatura, velocidade do vento, condição do tempo, umidade e visibilidade, estruturando e salvando os dados em formato CSV para posterior análise. O ambiente é orquestrado com Docker Compose, e a configuração segue os padrões recomendados pelo Apache Airflow, garantindo fácil reprodutibilidade e escalabilidade do fluxo de dados.

<img src="https://i.imgur.com/bHA1YzQ.png" style="width:100%;height:auto"/>

Tecnologias:  
* Apache Airflow
* Apache Kafka
* Apache Spark
* Confluent Control Center
* Schema Registry for Confluent Platform
* Docker
* DB Cassandra
* Python

## API Random User Generator

A free, open-source API for generating random user data. Like Lorem Ipsum, but for people.

https://randomuser.me/

## Configurações da Stack Docker 

### 🖥️ 01. Máquina Servidor

Documentação: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html

Acessando pasta do projeto com cd/path

```
docker compose up -d
```

Imagens Docker (Microserviços):

* Apache Kafka (image: bitnami/kafka:3.9.0)  
* Confluent Schema Registry (image: confluentinc/cp-schema-registry:7.4.0)  
* Confluent Control Center (image: confluentinc/cp-enterprise-control-center:7.4.0)  
* Apache Airflow Webserver (apache/airflow:2.10.4-python3.11)  
* Apache Airflow Scheduler (image: apache/airflow:2.10.4-python3.11)
* Banco de dados PostgreSQL usado pelo Airflow (image: postgres:14.0)
* Cluster Spark
  * spark-master: image: bitnami/spark:3.5.4
  * spark-worker-1: image: bitnami/spark:3.5.4
* Cassandra DB (image: cassandra:5.0.2)

### 🖥️ 02. Máquina Cliente

**Dockerfile**

* Linux com Interpretador Python (python:3.11-slim)
* Dependências:
  * pyspark==3.5.4
  * kafka-python==2.0.2
  * cassandra-driver==3.29.2

Acessando pasta do projeto com cd/path

01. Criar imagem Cassandra
```
docker build -t kafka-spark-cassandra-consumer .
```
02. Criar Container
```
docker run --name dsa_cliente -dit --network dsaservidor_dsacademy kafka-spark-cassandra-consumer
```

Dentro do container, o comando abaixo inicia o Kafka Consumer (conecta ao spark, kafka e cassandra):
```
python dsa_consumer_stream.py --mode initial
```


> Nota: No final do Dockerfile ao descomentar a última linha e recriar imagem e container, não será necessário a ativação manual da dag e o container irá processar os dados automaticamente.

```
# Comando de entrada padrão
#CMD ["python", "dsa_consumer_stream.py", "--mode", "append"]
```


## Pipeline Scripts
```
dsa_consumer_stream.py
```

Esse código implementa um pipeline de ingestão e processamento de dados em tempo real com Apache Spark, Kafka e Cassandra.  

Primeiro, ele configura logging, importa bibliotecas necessárias e define funções auxiliares para criar o keyspace e a tabela no Cassandra, além de formatar e inserir dados.  

Em seguida, o programa principal aceita um argumento de linha de comando (--mode) para definir se os dados do Kafka devem ser lidos desde o início ou somente os novos.  

Ele cria conexões com o Cassandra e Spark, configura a leitura de um tópico Kafka, e define um schema para os dados JSON consumidos.  

Esses dados são transformados em um DataFrame estruturado, filtrados por e-mails válidos, e então inseridos no Cassandra lote a lote com foreachBatch, mantendo o processamento contínuo até ser encerrado. 

```
dsa_kafka_stream.py
```

Esse código define uma DAG no Apache Airflow chamada dsa-real-time-etl-stack, que agenda diariamente a execução de uma tarefa Python (streaming_task) responsável por fazer streaming em tempo real de dados de usuários aleatórios obtidos via API pública. 

O processo começa com a função dsa_extrai_dados_api(), que consome dados da API RandomUser.me, seguida da função dsa_formata_dados(), que transforma os dados brutos em um formato estruturado com campos como nome, endereço, e-mail e imagem de perfil, atribuindo também um UUID único para cada usuário. 

Em seguida, a função dsa_stream_dados() estabelece uma conexão com um broker Kafka (em broker:29092), e por 60 segundos envia os dados formatados para o tópico dsa_kafka_topic, registrando logs de sucesso ou erro. 

Todo esse fluxo é encapsulado em uma DAG do Airflow, permitindo a orquestração e agendamento automatizado da pipeline de ETL em tempo real.


## Resultado

Acessar o Cassandra e verificar o resultado do armazenamento:  

Esse conjunto de comandos serve para acessar o banco de dados Cassandra em um container Docker e consultar dados de uma tabela.  

Abre o terminal interativo do Cassandra (CQLSH) dentro do container chamado cassandra.
```
docker exec -it cassandra cqlsh
```
Define o keyspace (similar ao schema em bancos relacionais) chamado dsa_dados_usuarios como o contexto atual para execução dos próximos comandos:
```
USE dsa_dados_usuarios;
```
Realiza uma consulta SQL/CQL para retornar todos os registros da tabela tb_usuarios dentro do keyspace selecionado.
```
SELECT * FROM tb_usuarios;
```
<img src="https://i.imgur.com/3mywWWV.png" style="width:100%;height:auto"/>

