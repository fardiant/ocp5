# doc
```
https://docs.google.com/document/d/1JEH4WHQwmLxtmt0TSo1w82SiBqKmRamRx1f6nHui24M/edit?tab=t.5a138yhxuqcy
https://docs.google.com/document/d/1LoW-jRjN69Ypj_55QYKY1sfiQOP9DmszKgrXYsXr4E8/edit?tab=t.0#heading=h.32uxpyazbo5c
https://github.com/containers/kubernetes-mcp-server


# for IDE
https://developers.redhat.com/articles/2026/02/04/how-connect-openshift-lightspeed-mcp-your-ide#exposing_the_openshift_lightspeed_service
```
# MCP deployment
```
curl -fsSL https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz -o /tmp/helm.tar.gz && tar -xzf /tmp/helm.tar.gz -C /tmp && mv /tmp/linux-amd64/helm /usr/local/bin/helm && rm -rf /tmp/helm.tar.gz /tmp/linux-amd64

helm upgrade -i -n kubernetes-mcp-server --create-namespace kubernetes-mcp-server \
  oci://ghcr.io/containers/charts/kubernetes-mcp-server \
  --set openshift=true \
  --set ingress.host=kubernetes-mcp-server.apps.hub.5g-deployment.lab \
  --set 'config.toolsets={core,config,helm,kubevirt}'

helm list -A
```


```
oc adm policy add-cluster-role-to-user admin  system:serviceaccount:openshift-lightspeed:lightspeed-service-api
oc adm policy add-cluster-role-to-user cluster-admin -z kubernetes-mcp-server -n kubernetes-mcp-server
oc auth can-i create namespaces --as=system:serviceaccount:kubernetes-mcp-server:kubernetes-mcp-server

`````````

# Github MCP
```
# secret
oc create secret generic github-mcp-token --from-literal=header="xxxxxxxxxxxxxxx" -n openshift-lightspeed

# deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-mcp-server
  namespace: git-mcp
  labels:
    app.kubernetes.io/name: github-mcp-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: github-mcp-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: github-mcp-server
    spec:
      containers:
        - name: github-mcp-server
          image: ghcr.io/github/github-mcp-server:latest
          args:
            - http
          ports:
            - name: http
              containerPort: 8082
              protocol: TCP
          env:
            - name: GITHUB_PERSONAL_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: github-mcp-pat
                  key: GITHUB_PERSONAL_ACCESS_TOKEN
            - name: GITHUB_READ_ONLY
              value: "1"
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: "1"
              memory: 512Mi
          readinessProbe:
            tcpSocket:
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            capabilities:
              drop:
                - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: github-mcp-server
  namespace: git-mcp
  labels:
    app.kubernetes.io/name: github-mcp-server
spec:
  selector:
    app.kubernetes.io/name: github-mcp-server
  ports:
    - name: http
      port: 8082
      targetPort: http
      protocol: TCP
  type: ClusterIP

# OLSconfig
kind: OLSConfig
metadata:
  name: cluster
spec:
  featureGates:
    - MCPServer
  mcpServers:
    - name: github-mcp-server
      url: http://github-mcp-server.git-mcp.svc.cluster.local:8082
      timeout: 30
      headers:
        - name: Authorization
          valueFrom:
            type: secret
            secretRef:
              name: github-mcp-token
```
# create cert and cm
```
# 1st cert
openssl s_client -showcerts -servername models.corp.redhat.com -connect models.corp.redhat.com:443 </dev/null 2>/dev/null   | sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p' > /tmp/full.crt

# 2nd cert
python3 - <<'EOF'
import subprocess, glob, re

files = glob.glob("/etc/pki/ca-trust/source/anchors/*") + ["/etc/pki/tls/certs/ca-bundle.crt"]
found = []
for f in files:
    try:
        text = open(f).read()
    except Exception:
        continue
    for pem in re.findall(r"-----BEGIN CERTIFICATE-----.*?-----END CERTIFICATE-----", text, re.S):
        out = subprocess.check_output(["openssl", "x509", "-noout", "-subject", "-issuer"], input=pem.encode()).decode()
        if "2023 Certificate Authority RHCSv2" in out or "Red Hat" in out:
            print("Found in", f, ":\n", out.strip())
            found.append(pem)

open("/tmp/rh-root.pem", "w").write("\n".join(dict.fromkeys(found)) + "\n")
EOF

# join cert
cat full.crt > join.pem
cat rh-root.pem >> join.pem


# test the cert
openssl crl2pkcs7 -nocrl -certfile join.pem | openssl pkcs7 -print_certs -text -noout | grep -E 'Subject:|Issuer:'|sort|uniq
        Issuer: CN=Internal Root CA/emailAddress=xxxxxxxc@xxxxxxxx
        Issuer:  Certificate Authority 
        Subject: Internal Root CA/emailAddress=xxxxxxxxx@xxxxxxx
        Subject: 
        Subject: 3 Certificate Authority 


# create cm 
oc create configmap redhat-it-ca   --from-file=ca-bundle.crt=/tmp/join.pem   -n openshift-lightspeed

```

# create secret
```
oc create secret generic credentials  --from-literal=apitoken="xxxxxxxxxx"  -n openshift-lightspeed

```

# OLSconfig
```
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  # 1. Enable MCP Server Integration Feature Gate
  featureGates:
    - MCPServer

  # 2. Custom MCP Server Definition
  mcpServers:
    - name: kubernetes-mcp-server
      url: http://kubernetes-mcp-server.kubernetes-mcp-server.svc.cluster.local:8080/mcp
      timeout: 30

  # 3. LLM Provider (Qwen3.8-27B-FP8)
  llm:
    providers:
      - name: corp-vllm
        type: rhoai_vllm
        url: https://models.corp.redhat.com/oai/v1
        credentialsSecretRef:
          name: credentials
        models:
          - name: Qwen3.8-27B-FP8
            contextWindowSize: 262144

  # 4. Core Platform & Tool Approvals
  ols:
    defaultProvider: corp-vllm
    defaultModel: Qwen3.8-27B-FP8
    logLevel: INFO
    conversationCache:
      type: postgres
      postgres:
        maxConnections: 2000
        sharedBuffers: 256MB
    additionalCAConfigMapRef:
      name: redhat-it-ca
    toolsApprovalConfig:
      approvalType: tool_annotations
      approvalTimeout: 600
    maxIterations: 5
    mcpKubeServerConfig:
      timeout: 60
    guardrails:
      toolResultInspection:
        enabled: false
````

