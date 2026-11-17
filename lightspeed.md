# MCP deployment
```
curl -fsSL https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz -o /tmp/helm.tar.gz && tar -xzf /tmp/helm.tar.gz -C /tmp && mv /tmp/linux-amd64/helm /usr/local/bin/helm && rm -rf /tmp/helm.tar.gz /tmp/linux-amd64

helm upgrade -i -n kubernetes-mcp-server --create-namespace kubernetes-mcp-server \
  oci://ghcr.io/containers/charts/kubernetes-mcp-server \
  --set openshift=true \
  --set ingress.host=kubernetes-mcp-server.apps.hub.5g-deployment.lab \
  --set 'config.toolsets={core,config,helm,kubevirt}'
```


```
oc adm policy add-cluster-role-to-user admin  system:serviceaccount:openshift-lightspeed:lightspeed-service-api
oc adm policy add-cluster-role-to-user cluster-admin -z kubernetes-mcp-server -n kubernetes-mcp-server
oc auth can-i create namespaces --as=system:serviceaccount:kubernetes-mcp-server:kubernetes-mcp-server

`````````



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

