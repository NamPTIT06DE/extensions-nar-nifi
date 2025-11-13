Triển khai NiFi cluster (2 node)
=================================

1) Triển khai ZooKeeper (host 10.9.3.129)

- Tạo file `docker-compose.yml` với nội dung sau:

''' 
version: '3.8'

services:
  zookeeper:
    image: zookeeper:3.8.3
    container_name: zookeeper
    restart: unless-stopped
    environment:
      ZOO_MY_ID: 1
      ZOO_PORT: 2181
      ZOO_SERVERS: server.1=10.9.3.129:2888:3888;2181
      ZOO_TICK_TIME: 4000
      ZOO_INIT_LIMIT: 15
      ZOO_SYNC_LIMIT: 10
    ports:
      - "2181:2181"
    volumes:
      - ./zoo_data:/data
      - ./zoo_datalog:/datalog
'''
  -  ZOO_SERVERS: server.1=10.9.3.129:2888:3888;2181 ->  định nghĩa Server ZooKeeper có ID là 1 tại IP 10.9.3.129, sử dụng cổng 2888 và 3888 để giao tiếp và bầu cử trong cluster, và cổng 2181 để client (ứng dụng) kết nối.

- Khởi động ZooKeeper:

```
docker compose up
```

2) Triển khai NiFi node1 (host 10.9.3.129)

- Tạo file `docker-compose.yml` cho node1, thêm `hostname: nifi.local`:


version: '3.8'

services:
  nifi:
    image: apache/nifi:2.6.0
    container_name: nifi
    hostname: nifi.local
    ports:
      - "8012:8012"
      - "11443:11443"
    env_file:
      - nifi.env
    volumes:
      - ./extensions:/opt/nifi/nifi-current/nar_extensions
      - ./lib:/opt/nifi/nifi-current/ex_lib
      - ./data:/opt/nifi/nifi-current/ex_data
      - ./start.sh:/opt/nifi/scripts/start.sh
    restart: unless-stopped


- Tạo file `nifi.env` với nội dung sau:
'''
NIFI_WEB_HTTP_PORT=8012
NIFI_WEB_HTTP_HOST=nifi
#NIFI_WEB_HTTPS_PORT=
#NIFI_WEB_HTTPS_HOST=
SINGLE_USER_CREDENTIALS_USERNAME=admin
SINGLE_USER_CREDENTIALS_PASSWORD=admin@12345678910

NIFI_CLUSTER_IS_NODE=true
NIFI_CLUSTER_NODE_ADDRESS=nifi
NIFI_CLUSTER_NODE_PROTOCOL_PORT=11443
NIFI_ZK_CONNECT_STRING=10.9.3.129:2181
NIFI_SENSITIVE_PROPS_KEY=myUltraSecretKey
NIFI_ELECTION_MAX_WAIT=1 min
'''
   - lưu ý: NIFI_ZK_CONNECT_STRING=10.9.3.129:2181 (địa chỉ host chạy zookeeper)
- Khởi động node:

```
docker-compose up -d
```

- Nếu gặp lỗi sau khi start container:

```
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: exec: "../scripts/start.sh": permission denied: unknown
```

  - Fix nhanh: trên host chạy:

  ```
  chmod +x start.sh
  ```

- phân giải hostname giữa các node để join cluster, bạn có thể thêm record vào `/etc/hosts` trong container (ví dụ):

```
docker exec -u 0 -it nifi bash
echo "10.9.3.129 nifi" >> /etc/hosts
echo "10.9.2.78 nifi1" >> /etc/hosts
```

3) Triển khai node2 (host 10.9.2.78)

- Các bước giống hệt node1: tạo `docker-compose.yml`, `nifi.env`, mount volumes và `start.sh` như trên, sau đó `docker-compose up -d` và phân giải hostname.

