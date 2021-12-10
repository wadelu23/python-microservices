# Microservices 與 RabbitMQ

> 此處簡易紀錄 docker microservices 與 RabbitMQ 實作
> 
> 如需完整做法與原始碼，可參考
> 
> [Microservices Tutorial || Build a Complete Vue Python Microservices Application](https://www.youtube.com/watch?v=y69VDOczkik)
> 
> https://github.com/scalablescripts/python-microservices


- [Microservices 與 RabbitMQ](#microservices-與-rabbitmq)
  - [概念簡介](#概念簡介)
    - [微服務](#微服務)
    - [Message Queue](#message-queue)
    - [微服務特性](#微服務特性)
  - [範例解說](#範例解說)
    - [環境建置](#環境建置)
    - [Message Queue流程](#message-queue流程)

---

## 概念簡介
### 微服務
微服務是一種以業務功能為主的服務設計概念，每一個服務都具有自主運行的業務功能，對外開放不受語言限制的 API (最常用的是 HTTP)，應用程式則是由一個或多個微服務組成。

---

### Message Queue
而微服務中的訊息佇列（Message Queue）
功能為廣播訊息，並傳遞到每個服務中

RabbitMQ 則是目前較為廣泛使用的 Message Queue

更詳細說明可參考[RabbitMQ 基本介紹、安裝教學](https://kucw.github.io/blog/2020/11/rabbitmq/)

此處使用 [CloudAMQP](https://www.cloudamqp.com/)

登入並建立 instance，即可於 producer 與 consumer 設定AMQP_URL

有提供介面查看各種狀態

![image](https://github.com/wadelu23/MarkdownPicture/blob/main/microservices/rabbitMQ-manager.png?raw=true)

### 微服務特性
* 每個服務都容易被替換，搭配CI/CD能快速更替某服務，不容易使整體出錯
* 服務是以能力來組織的，例如使用者介面、前端、推薦系統、帳單或是物流等。
* 由於功能被拆成多個服務，因此可以由不同的程式語言、資料庫實作。

---

## 範例解說

此處範例功能為
* 有多張圖片與其標題，可對該圖片點擊喜歡

後端微服務負責部分
* main(Flask)
  * 圖片主資料，新增、刪除、更改等。接收點擊like的API，也會透過RabbitMQ傳遞到admin中
* admin(django)
  * 各圖片like資料，而與圖片的新增、刪除、更改動作，會透過RabbitMQ傳遞到main中

### 環境建置
* Dockerfile 設定所需環境與套件
* docker-compose.yml，設定三個services
  * backend (主程式)
  * queue (Message Queue)
  * db (資料庫)
```yml
version: '3.8'
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    ports:
      - 8000:8000
    volumes:
      - .:/app
    depends_on:
      - db

  queue:
    build:
      context: .
      dockerfile: Dockerfile
    command: 'python -u consumer.py'
    depends_on:
      - db

  db:
    image: mysql:5.7.22
    restart: always
    environment:
      MYSQL_DATABASE: admin
      MYSQL_USER: root
      MYSQL_PASSWORD: root
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - .dbdata:/var/lib/mysql
    ports:
      - 33066:3306
```
---

### Message Queue流程
此處以實作的模式說明，其他模式說明請參考[RabbitMQ 基本介紹、安裝教學](https://kucw.github.io/blog/2020/11/rabbitmq/)

* Producer : 發送 message
* Exchange : 依照設定規則分配 message 給指定 Queue
* Queue : 暫存 message
* Consumer : 從對應 Queue 中接收處理 message

此處搭配使用[pika](https://pika.readthedocs.io/en/stable/index.html)

RabbitMQ相關概念或其他實作可多參考

[rabbitMQ和pika模組](https://www.796t.com/article.php?id=72890)

[RabbitMQ 訊息佇列-實作](https://medium.com/%E7%A8%8B%E5%BC%8F%E4%B9%BE%E8%B2%A8/python-rabbitmq-%E8%A8%8A%E6%81%AF%E4%BD%87%E5%88%97-%E5%AF%A6%E4%BD%9C-ff0756eac9dd)


**設定Producer**
```python
# producer.py

AMQP_URL = 'your_AMQP_URL'

params = pika.URLParameters(AMQP_URL)

connection = pika.BlockingConnection(params)

channel = connection.channel()


def publish(method, body):
    properties = pika.BasicProperties(method)
    channel.basic_publish(
        exchange='',
        routing_key='admin',
        body=json.dumps(body),
        properties=properties)
```


```python
# 在需要的地方發送
publish('product_liked', id)
```


**Comsumer**
```python
# consumer.py

AMQP_URL = 'your_AMQP_URL'
params = pika.URLParameters(AMQP_URL)

connection = pika.BlockingConnection(params)

channel = connection.channel()

channel.queue_declare(queue='admin')


def callback(ch, method, properties, body):
    print('Receive in admin')
    # 接收後續處理
    id = json.loads(body)
    print(id)
    product = Product.objects.get(id=id)
    product.likes += 1
    product.save()
    print('Product likes increased!')


channel.basic_consume(
    queue='admin', on_message_callback=callback, auto_ack=True)

print('Started Consuming')

channel.start_consuming()

channel.close()
```

可先將backend與db背景運行，接著運行queue

利用[Postman](https://www.postman.com/)工具傳送資料

便可查看檢查運作情形，如下圖

![image](https://github.com/wadelu23/MarkdownPicture/blob/main/microservices/queue-msg.png?raw=true)

