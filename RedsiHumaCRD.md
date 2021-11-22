## Huma Redis CRD

Owner: Anand Totala (anandt-huma)
Reviwer:Compose Operator Maintainers
Status: Draft Revision, 0.1

### Problem statement:
- External resources creation is vey tightly coupled with application CRD, they need to be decoupled so that external resource creation can be called from App composition CRD or Application CRD.
- Need to provide a way by which end users can refer to already creatd external resources, just in case they want to create the external resource separately before hand.
- Need to provide a way to end user by which they can create external resources automatically by specifying details in App CRD specs or Application Compositon CRD specs. 

### Solution:
```
Note: I am using Redis here for explaining solution, 
same solution or strategy can be applied for any external resource.
```
- Need to create a CRD which will specify the specs for the external resources.
- In case of redis it is storage, tlsEnabled and certsMountPath as shown below.
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
- when the RedisClaim will be applied, compose-operator should be able to create the external resource(kubedb's Redis)
based on config provided.
- Below is the state of RedisClaim after it is successfully applied.
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
- Under status section of the RedisClaim, we can have all the env variables and their secretRef.

### Using Reference
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

- Below is the state of Application composition after it is successfully applied.
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
### Creation of external resource automatically
- In case where user will not have external resources pre-created, end user can specify the config of external resources directly in App or Application composition CRD, the way it does today.
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: test
spec:
  image: nginx:1.20-alpine
  ports:
    - 80:80
  redis:
    storage: 1Gi
    tlsEnabled: true
    certsMountPath: /certs
    envVarName: REDIS_URL
```
- compose-operator will be able to create RedisClaim based on config provided above
```yaml
apiVersion: operator.huma.com/v1alpha1
kind: RedisClaim
metadata:
  name: test-app
  namespace: test
  ownerReferences:
  - apiVersion: operator.huma.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Application
    name: test-app
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
- This RedisClaim controller will create the necessary external resource specified here and can be utilised by App CR.

## High Level Design Diagram of proposed solution.
 
![diagram](RedisClaim.png)
