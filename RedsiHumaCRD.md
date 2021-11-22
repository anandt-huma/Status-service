## Huma Redis CRD


### Input resources definition 
- Creating an App which is using Redis

```yaml
apiVersion: operator.huma.com/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: test
spec:
  redisRef:
    - name: test-redis
      envVarName: REDIS_URL
  image: nginx:1.20-alpine
  ports:
    - "80"
---
apiVersion: operator.huma.com/v1alpha1
kind: RedisClaim
metadata:
  name: test-redis
  namespace: redis
spec:
  storage: 1Gi
  tlsEnabled: true
  certsMountPath: /cert
```

### Steps:
- Create RedisClaim CRD first.
- Create Application

### Create RedisDB:
- Input CRD:
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: RedisClaim
metadata:
  name: test-redis
  namespace: redis
spec:
  storage: 1Gi
  tlsEnabled: true
  certsMountPath: /cert
```

- RedisClaim after creation and injection of required variables and data
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: RedisClaim
metadata:
  name: test-redis
  namespace: redis
spec:
  Redis:
    storage: 1Gi
    tlsEnabled: true
    certsMountPath: /cert
status:
  injectAppEnvs:
  - name: REDIS_USERNAME
    secretValue: test-redis-redis-auth@username
  - name: REDIS_PASSWORD
    secretValue: test-redis-redis-auth@password
  - name: REDIS_URL
    secretValue: redis://$(REDIS_USERNAME):$(REDIS_PASSWORD)@test-redis-redis:6379
  - name: TLS_CA_CERT_FILE
    value: /certs/ca.crt
  - name: TLS_CERT_FILE
    value: /certs/tls.crt
  - name: TLS_KEY_FILE
    value: /certs/tls.key
```

### Create Application:
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: test
spec:
  redisRef:
    - name: test-redis
      envVarName: REDIS_URL
  image: nginx:1.20-alpine
  ports:
    - "80"
```
- Application after creation and injection of secrets and env variables.
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: test
spec:
  env:
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: REDIS_USERNAME
      secretValue: test-redis-redis-auth@username
    - name: REDIS_PASSWORD
      secretValue: test-redis-redis-auth@password
    - name: REDIS_URL
      secretValue: redis://$(REDIS_USERNAME):$(REDIS_PASSWORD)@test-redis-redis:6379
    - name: REDIS_URL
      value: rediss://($REDIS_PASSWORD)@redis-tls-test-redis:6379
    - name: TLS_CA_CERT_FILE
      value: /certs/ca.crt
    - name: TLS_CERT_FILE
      value: /certs/tls.crt
    - name: TLS_KEY_FILE
      value: /certs/tls.key
  image: nginx:1.20-alpine
  ports:
    - 80:80
  redisRef:
    - name: test-redis
      envVarName: REDIS_URL
```
