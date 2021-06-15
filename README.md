# ibmmq-container

```
docker run \
-e LICENSE=accept \
-e MQ_QMGR_NAME=QMGR1  \
-e LOG_FORMAT=basic \
-e MQ_ENABLE_METRICS=false \
-e MQ_ADMIN_PASSWORD=12345678 \
--volume /tmp/mq/mqm:/mnt/mqm \
-p 1414:1414 \
-p 9443:9443 \
--name mq \
-d ibmcom/mq:9.2.2.0-r1-amd64 
```
