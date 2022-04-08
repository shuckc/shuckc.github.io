---
title: Testing minio redis notifications
date: 2020-03-17 17:00:00
tags:
    - redis
    - minio
categories:
    - tech
keywords:
    - markdown
    - code block
---

Documented at [docs.min.io](https://docs.min.io/docs/minio-bucket-notification-guide.html) "Publish MinIO events via Redis"

Terminal 1 runs up local redis, enables password, binds to all if

	% docker run --name some-redis -d redis
	% docker exec -it some-redis bash
	root@800973395e9f:/data# redis-cli
	127.0.0.1:6379> config set requirepass yoursecret
	OK
	127.0.0.1:6379> set bind 0.0.0.0
	OK
	127.0.0.1:6379> save
	OK
	127.0.0.1:6379> monitor


Terminal two runs minio server:

	% docker run -p 9000:9000 -e "MINIO_ACCESS_KEY=EXAMPLE" -e "MINIO_SECRET_KEY=wJalrXUtnFE" minio/minio server /data

Temrinal three fishes out the docker network ip for redis

	% docker ps
	CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                    NAMES
	800973395e9f   redis         "docker-entrypoint.s…"   7 minutes ago    Up 7 minutes    6379/tcp                 some-redis
	7f76659ad36a   minio/minio   "/usr/bin/docker-ent…"   18 minutes ago   Up 18 minutes   0.0.0.0:9000->9000/tcp   gracious_jang
	% docker inspect --format '{{ .NetworkSettings.IPAddress }}' 800973395e9f
	172.17.0.4

Then runs minio mc client:

    % docker run -it --entrypoint=/bin/sh minio/mc
    sh-4.4# mc alias set dock http://172.17.0.2:9000/ EXAMPLE wJalrXUtnFE
    Added `dock` successfully.
    sh-4.4# mc mb dock/test123
    Bucket created successfully `dock/test123`.
    sh-4.4# mc admin config set dock/ notify_redis:1 address="172.17.0.4:6379" format="namespace" key="bucketevents" password="yoursecret" queue_dir="" queue_limit="0"
    Successfully applied new settings.
    Please restart your server 'mc admin service restart dock/'.
    sh-4.4# mc admin service restart dock/
    Restart command successfully sent to `dock/`.
    Restarted `dock/` successfully.

Running `mc admin config set <instance> notify_redis` causes the server to open up a socket to redis and test the auth, so check server logs.

This causes the service in T1 to reload, this time printing a URN on startup

	Restarting on service signal

	 You are running an older version of MinIO released 1 month ago
	 Update: Run `mc admin update`

	Endpoint: http://172.17.0.2:9000  http://127.0.0.1:9000
	SQS ARNs: arn:minio:sqs::1:redis

	Browser Access:
	   http://172.17.0.2:9000  http://127.0.0.1:9000

	Object API (Amazon S3 compatible):
	   Go:         https://docs.min.io/docs/golang-client-quickstart-guide
	   Java:       https://docs.min.io/docs/java-client-quickstart-guide
	   Python:     https://docs.min.io/docs/python-client-quickstart-guide
	   JavaScript: https://docs.min.io/docs/javascript-client-quickstart-guide
	   .NET:       https://docs.min.io/docs/dotnet-client-quickstart-guide
	Drive /data does not support O_DIRECT for reads, proceeding to use the drive without O_DIRECT

Continuing with minio client

	sh-4.4# mc mb dock/images
	Bucket created successfully `dock/images`.
	sh-4.4# mc event add dock/images arn:minio:sqs::1:redis
	Successfully added arn:minio:sqs::1:redis
	sh-4.4# mc event list dock/images
	arn:minio:sqs::1:redis   s3:ObjectCreated:*,s3:ObjectRemoved:*,s3:ObjectAccessed:*   Filter:
	sh-4.4# touch silly.txt
	sh-4.4# mc cp silly.txt dock/images
	 0 B / ? ┃░░░░░░░░░░░

Which causes the redis cli monitor to emit:

	1616017807.403088 [0 172.17.0.2:50754] "HSET" "bucketevents" "images/silly.txt" "{\"Records\":[{\"eventVersion\":\"2.0\",\"eventSource\":\"minio:s3\",\"awsRegion\":\"\",\"eventTime\":\"2021-03-17T21:50:07.400Z\",\"eventName\":\"s3:ObjectCreated:Put\",\"userIdentity\":{\"principalId\":\"EXAMPLE\"},\"requestParameters\":{\"accessKey\":\"EXAMPLE\",\"region\":\"\",\"sourceIPAddress\":\"172.17.0.3\"},\"responseElements\":{\"content-length\":\"0\",\"x-amz-request-id\":\"166D3FA2406CF4BC\",\"x-minio-deployment-id\":\"a1991d17-c7a8-4259-8bbf-fb0906f1f5d6\",\"x-minio-origin-endpoint\":\"http://172.17.0.2:9000\"},\"s3\":{\"s3SchemaVersion\":\"1.0\",\"configurationId\":\"Config\",\"bucket\":{\"name\":\"images\",\"ownerIdentity\":{\"principalId\":\"EXAMPLE\"},\"arn\":\"arn:aws:s3:::images\"},\"object\":{\"key\":\"silly.txt\",\"eTag\":\"d41d8cd98f00b204e9800998ecf8427e\",\"contentType\":\"text/plain\",\"userMetadata\":{\"content-type\":\"text/plain\"},\"sequencer\":\"166D3FA240CF0AE4\"}},\"source\":{\"host\":\"172.17.0.3\",\"port\":\"\",\"userAgent\":\"MinIO (linux; amd64) minio-go/v7.0.9 mc/RELEASE.2021-02-19T05-34-40Z\"}}]}"


