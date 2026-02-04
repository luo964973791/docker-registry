### 提前创建证书

```javascript
yum install openssl -y
mkdir -p /root/certs && cd /root/certs
# 生成 CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=MyRegistryCA"

# 生成 registry 私钥和 CSR
openssl genrsa -out registry.key 4096
openssl req -new -key registry.key -out registry.csr -subj "/CN=registry.docker.com"

# 用 CA 签发 registry 证书，包含 SAN
openssl x509 -req -in registry.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out registry.crt -days 365 -sha256 -extfile <(printf "subjectAltName=DNS:registry.docker.com")

mkdir -p /etc/docker/certs.d/registry.docker.com
#docker客户端配置证书
cp /root/certs/ca.crt /etc/docker/certs.d/registry.docker.com/ca.crt

#服务器证书信任
cp /root/certs/ca.crt /etc/pki/ca-trust/source/anchors/ && update-ca-trust

#绑定域名
[root@node1 tmp]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.27.0.3  registry.docker.com


#docker添加域名信任.
[root@node1 ~]# cat /etc/docker/daemon.json 
{
  "insecure-registries": ["registry.docker.com"]
}
systemctl daemon-reload
systemctl restart docker
```

### 启动

```javascript
#启动minio
docker run -d \
  --name minio \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=Test@123 \
  --restart always \
  quay.io/minio/minio server /data --console-address :9001
  
  
#拷贝mc创建bucket
docker cp minio:/usr/bin/mc .
./mc alias set myminio http://localhost:9000 admin Test@123
./mc mb myminio/docker-registry

#启动镜像仓库
docker run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v /root/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -e REGISTRY_STORAGE=s3 \
  -e REGISTRY_STORAGE_S3_REGION=us-east-1 \
  -e REGISTRY_STORAGE_S3_BUCKET=docker-registry \
  -e REGISTRY_STORAGE_S3_ACCESSKEY=admin \
  -e REGISTRY_STORAGE_S3_SECRETKEY=Test@123 \
  -e REGISTRY_STORAGE_S3_REGIONENDPOINT=http://172.27.0.3:9000 \
  -e REGISTRY_STORAGE_S3_FORCEPATHSTYLE=true \
  registry:2
```
