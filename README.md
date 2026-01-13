PostgreSQL → Kafka → PostgreSQL CDC Pipeline (Debezium + Kafka Connect)

Bu proje, PostgreSQL üzerinde gerçekleşen veri değişikliklerinin (Change Data Capture – CDC) Apache Kafka aracılığıyla yakalanmasını ve başka bir PostgreSQL veritabanına aktarılmasını göstermektedir.

Proje tamamen Docker üzerinde çalışan bir mimari ile hazırlanmıştır ve ders / laboratuvar ortamında CDC, Kafka ve event-driven mimariyi göstermek amacıyla oluşturulmuştur.

Mimari Genel Bakış

PostgreSQL Source veritabanında yapılan INSERT, UPDATE ve DELETE işlemleri PostgreSQL WAL (Write-Ahead Log) üzerinden logical replication ile üretilir.
Debezium bu WAL kayıtlarını okuyarak Kafka topic’lerine CDC event’leri olarak yazar.
Kafka Connect JDBC Sink Connector ise bu event’leri okuyarak hedef PostgreSQL veritabanına yazar.

Akış şu şekildedir:

PostgreSQL (Source)
→ WAL / Logical Replication
→ Debezium
→ Kafka Topic
→ (Opsiyonel Python Consumer)
→ PostgreSQL (Sink)

Kullanılan Teknolojiler

PostgreSQL 15
Apache Kafka
Kafka Connect
Debezium 2.5
Docker & Docker Compose
Python (kafka-python, opsiyonel)

Docker Üzerinde Çalışan Servisler

postgres_source
CDC kaynağı PostgreSQL veritabanı

postgres_sink
CDC hedef PostgreSQL veritabanı

zookeeper
Kafka koordinasyon servisi

kafka
Apache Kafka broker

kafka-connect
Kafka Connect + Debezium connector’ları

CDC (Change Data Capture) Mantığı

Source PostgreSQL veritabanında gerçekleşen:

INSERT
UPDATE
DELETE

işlemleri WAL içerisine yazılır.

Debezium PostgreSQL Source Connector:

Trigger kullanmaz

Polling yapmaz

Doğrudan WAL’dan okuma yapar

Her değişikliği Kafka topic’lerine JSON event olarak yazar

Kafka Connect JDBC Sink Connector:

Kafka topic’lerinden gelen event’leri okur

Hedef PostgreSQL veritabanına INSERT / UPDATE olarak yazar

Ön Gereksinimler

Docker
Docker Compose
(Opsiyonel) Python 3

Projeyi Çalıştırma

Docker servislerini başlatmak için proje dizininde:

docker compose up -d

Çalışan servisleri kontrol etmek için:

docker ps

PostgreSQL Source – CDC Ayarları

CDC çalışabilmesi için source PostgreSQL üzerinde aşağıdaki ayarlar yapılır:

wal_level = logical
max_replication_slots = 10
max_wal_senders = 10

Bu ayarlar ALTER SYSTEM ile yapıldıktan sonra PostgreSQL container restart edilir.

Veritabanları ve Tablolar

Source PostgreSQL

source_db veritabanı oluşturulur ve aşağıdaki tablo tanımlanır:

customers
customer_id (SERIAL, primary key)
first_name
last_name
email
days_count
room_class
payment (numeric)

Sink PostgreSQL

sink_db veritabanı oluşturulur ve aşağıdaki tablo tanımlanır:

customers
customer_id (INTEGER, primary key – source’tan gelir)
first_name
last_name
email
days_count
room_class
payment (numeric)

Sink tarafında auto.create kapalı olduğu için tablolar manuel olarak oluşturulmuştur.

Kafka Connect Connector’ları

Source Connector:

Debezium PostgreSQL Source Connector

WAL üzerinden CDC event üretir

Sink Connector:

JDBC Sink Connector

Kafka’dan gelen event’leri PostgreSQL’e yazar

Connector’lar runtime’da REST API üzerinden oluşturulur.

Connector’ları oluşturmak için:

bash scripts/create-connectors.sh

Connector listesini kontrol etmek için:

curl http://localhost:8083/connectors

Kafka Topic Kontrolleri

Kafka üzerindeki topic’leri listelemek için:

kafka-topics --bootstrap-server kafka:9092 --list

CDC event’lerini okumak için (ilk 5 mesaj):

kafka-console-consumer
--bootstrap-server kafka:9092
--topic pgserver1.public.customers
--from-beginning
--max-messages 5

Bu mesajlar Debezium Envelope formatında JSON olarak gelir ve op alanı ile INSERT (c), UPDATE (u), DELETE (d) işlemleri ayırt edilir.

Test Senaryosu

Source PostgreSQL üzerinde INSERT, UPDATE ve DELETE işlemleri yapılır.
Her işlem için Kafka topic’inde bir CDC event oluşur.
Bu event’ler sink PostgreSQL veritabanına yansıtılır.

Bu sayede Source → Kafka → Sink veri akışı doğrulanır.

Python Kafka Consumer (Opsiyonel)

Kafka topic’lerini daha temiz bir formatta okumak için Python consumer kullanılmıştır.

Bu consumer:

Kafka’ya bağımsız bir client olarak bağlanır

Debezium event’lerinden yalnızca after alanını parse eder

INSERT ve UPDATE işlemlerini okunabilir şekilde gösterir

Python consumer demo ve gözlem amaçlıdır.

Eğitim Amaçlı Notlar

Connector’lar Docker restart sonrası otomatik gelmez, tekrar oluşturulmalıdır.
CDC yalnızca source veritabanında yapılır.
Sink veritabanında WAL veya logical replication gerekmez.
Bu mimari event-driven ve near real-time veri replikasyonunu göstermektedir.

Projenin Amacı

Bu proje, PostgreSQL CDC mantığını, Kafka event tabanlı mimariyi ve veritabanları arası veri akışını eğitim ve laboratuvar ortamında göstermek amacıyla hazırlanmıştır.
