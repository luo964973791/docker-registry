### 提前创建证书

```javascript
registry_domain='registry.docker.com'
yum install openssl -y
mkdir -p /root/certs && cd /root/certs

# 生成 CA
openssl genrsa -out ${registry_domain}.ca.key 4096
openssl req -x509 -new -nodes -key ${registry_domain}.ca.key -sha256 -days 36500 -out ${registry_domain}.ca.crt -subj "/CN=MyRegistryCA"

# 生成 registry 私钥和 CSR
openssl genrsa -out ${registry_domain}.key 4096
openssl req -new -key ${registry_domain}.key -out ${registry_domain}.csr -subj "/CN=$registry_domain"

# 用 CA 签发 registry 证书，包含 SAN
openssl x509 -req -in ${registry_domain}.csr \
  -CA ${registry_domain}.ca.crt -CAkey ${registry_domain}.ca.key -CAcreateserial \
  -out ${registry_domain}.crt -days 36500 -sha256 \
  -extfile <(printf "subjectAltName=DNS:$registry_domain")

mkdir -p /etc/docker/certs.d/$registry_domain
# docker客户端配置证书
cp /root/certs/${registry_domain}.crt /etc/docker/certs.d/$registry_domain/${registry_domain}.crt

# 服务器证书信任
cp /root/certs/${registry_domain}.crt /etc/pki/ca-trust/source/anchors/${registry_domain}.crt && update-ca-trust

#绑定hosts
echo "$(hostname -I |awk '{print $1}') localhost $registry_domain" >> /etc/hosts


#docker添加域名信任.
#cat <<EOF > /etc/docker/daemon.json
#{
#  "insecure-registries": ["$registry_domain"]
#}
#EOF

mkdir -p /etc/systemd/system/docker.service.d

#如果有代理，需要把域名写到不使用代理里面.
cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf 
[Service]
Environment="HTTP_PROXY=http://172.27.0.88:22"
Environment="HTTPS_PROXY=http://172.27.0.88:22"
Environment="NO_PROXY=localhost,127.0.0.1,$registry_domain"
EOF

systemctl daemon-reload
systemctl restart docker
```

### registry没密码方式
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
#./mc quota info myminio/docker-registry #查看是否有配额限制桶大小,0代表没限制.
#./mc du myminio/docker-registry   #查看大小
#./mc cp /root/kubespray myminio/docker-registry/ --recursive    #复制目录命令
#./mc anonymous set download myminio/test  #设置可以下载文件权限
#wget http://172.27.0.6:9000/test/index.html #测试是否可以下载

#启动镜像仓库,由于是测试，就懒得设置用户密码.
docker run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v /root/certs:/certs \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/${registry_domain}.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/${registry_domain}.key \
  -e REGISTRY_STORAGE=s3 \
  -e REGISTRY_STORAGE_S3_REGION=us-east-1 \
  -e REGISTRY_STORAGE_S3_BUCKET=docker-registry \
  -e REGISTRY_STORAGE_S3_ACCESSKEY=admin \
  -e REGISTRY_STORAGE_S3_SECRETKEY=Test@123 \
  -e REGISTRY_STORAGE_S3_REGIONENDPOINT=http://$(hostname -I |awk '{print $1}'):9000 \
  -e REGISTRY_STORAGE_S3_FORCEPATHSTYLE=true \
  registry:2
```



### registry有密码方式
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

#启动镜像仓库,使用密码
mkdir -p /root/auth
docker run --name htpasswd --entrypoint htpasswd httpd:2.4 -Bbn admin MyPassword123 > /root/auth/htpasswd
docker rm -f htpasswd
docker run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v /root/certs:/certs \
  -v /root/auth:/auth \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -e REGISTRY_STORAGE=s3 \
  -e REGISTRY_STORAGE_S3_REGION=us-east-1 \
  -e REGISTRY_STORAGE_S3_BUCKET=docker-registry \
  -e REGISTRY_STORAGE_S3_ACCESSKEY=admin \
  -e REGISTRY_STORAGE_S3_SECRETKEY=Test@123 \
  -e REGISTRY_STORAGE_S3_REGIONENDPOINT=http://$(hostname -I |awk '{print $1}'):9000 \
  -e REGISTRY_STORAGE_S3_FORCEPATHSTYLE=true \
  registry:2
```

### 删除镜像
```javascript
curl -sk https://registry.docker.com/v2/nginx/tags/list | python3 -m json.tool
curl -sk https://registry.docker.com/v2/_catalog | python3 -m json.tool
curl -s -k \
  --header "Accept:application/vnd.docker.distribution.manifest.v2+json" \
  -I -XGET https://registry.docker.com/v2/busybox/manifests/latest | \
  grep "docker-content-digest" | \
  cut -d ':' -f3

curl -X DELETE \
  https://registry.docker.com/v2/busybox/manifests/sha256:91c66c844e6bba57e92e10e755e73a816d0b99edd17eb5297d9ac519ab3a8c81 \
  -k  
```





### containerd运行时部署方式
```
registry_domain='registry.containerd.com'
yum install openssl -y
mkdir -p /root/certs && cd /root/certs

openssl genrsa -out ${registry_domain}.ca.key 4096
openssl req -x509 -new -nodes -key ${registry_domain}.ca.key -sha256 -days 36500 -out ${registry_domain}.ca.crt -subj "/CN=MyRegistryCA"

openssl genrsa -out ${registry_domain}.key 4096
openssl req -new -key ${registry_domain}.key -out ${registry_domain}.csr -subj "/CN=$registry_domain"

openssl x509 -req -in ${registry_domain}.csr \
  -CA ${registry_domain}.ca.crt -CAkey ${registry_domain}.ca.key -CAcreateserial \
  -out ${registry_domain}.crt -days 36500 -sha256 \
  -extfile <(printf "subjectAltName=DNS:$registry_domain")
  
mkdir -p /etc/containerd/certs.d/$registry_domain
cp /root/certs/${registry_domain}.crt /etc/containerd/certs.d/$registry_domain/ca.crt
cat > /etc/containerd/certs.d/$registry_domain/hosts.toml <<EOF
server = "https://${registry_domain}"

[host."https://${registry_domain}"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = false
  override_path = false
EOF

cp /root/certs/${registry_domain}.crt /etc/pki/ca-trust/source/anchors/${registry_domain}.crt && update-ca-trust

echo "$(hostname -I |awk '{print $1}') localhost $registry_domain" >> /etc/hosts

systemctl restart containerd

nerdctl run -d \
  --name minio \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=Test@123 \
  --restart always \
  quay.io/minio/minio server /data --console-address :9001


nerdctl cp minio:/usr/bin/mc .
./mc alias set myminio http://localhost:9000 admin Test@123
./mc mb myminio/docker-registry


nerdctl run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v /root/certs:/certs \
  -v /root/auth:/auth \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -e REGISTRY_STORAGE=s3 \
  -e REGISTRY_STORAGE_S3_REGION=us-east-1 \
  -e REGISTRY_STORAGE_S3_BUCKET=docker-registry \
  -e REGISTRY_STORAGE_S3_ACCESSKEY=admin \
  -e REGISTRY_STORAGE_S3_SECRETKEY=Test@123 \
  -e REGISTRY_STORAGE_S3_REGIONENDPOINT=http://$(hostname -I |awk '{print $1}'):9000 \
  -e REGISTRY_STORAGE_S3_FORCEPATHSTYLE=true \
  registry:2
```
