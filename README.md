# Integration Open Policy Agent with Istio Ingress Gateway

## Install istio

https://istio.io/latest/docs/setup/ required version 1.9+

## Add Open Policy Agent sidecar

### Create simple rego rules

```bash
kubectl apply -f - <<<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-gateway-policy
  namespace: istio-system
data:
  policy.rego: |
    package envoy.authz
    import input.attributes.request.http as http_request
    default allow = true
EOF
```

### Manually add open policy agent sidecar for istio-ingressgateway deployment

Before Istio 1.11 version we need manually patch `istio-ingressgateway` deployment with patch:

```bash
cat > istio-ingressgateway-patch.yaml <<<EOF
spec:
  template:
    spec:
      containers:
      - name: opa
        image: openpolicyagent/opa:0.29.1-envoy
        securityContext:
          runAsUser: 1111
        volumeMounts:
          - readOnly: true
            mountPath: /policy
            name: opa-policy
        args:
          - "run"
          - "--server"
          - "--addr=localhost:8181"
          - "--diagnostic-addr=0.0.0.0:8282"
          - "--set=plugins.envoy_ext_authz_grpc.addr=:9191"
          - "--set=plugins.envoy_ext_authz_grpc.query=data.envoy.authz.allow"
          - "--set=decision_logs.console=true"
          - "--ignore=.*"
          - "/policy/"
        livenessProbe:
          httpGet:
            path: /health?plugins
            scheme: HTTP
            port: 8282
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /health?plugins
            scheme: HTTP
            port: 8282
          initialSeconds: 5
          periodSeconds: 5
      volumes:
      - name: opa-policy
        configMap:
          name: opa-gateway-policy
```

Apply this patch:

```bash
kubectl -n istio-system patch deployment istio-ingressgateway --patch "$(cat istio-ingressgatewat-patch.yaml)"
```

### Create Istio ServiceEntry for communication ingress with embedded Open Policy Agent

```bash
kubectl apply -f - <<<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: opa-gateway-sidecar
  namespace: istio-system
spec:
  hosts:
  - external-authz-grpc.local
  location: MESH_INTERNAL
  ports:
  - number: 9191
    name: grpc
    protocol: GRPC
  resolution: STATIC
  endpoints:
  - address: 127.0.0.1
EOF
```

### Add extensionProvider in Istio configuration

```bash
kubectl -n istio-system edit configmap istio
```

And modify data:
```bash
data:
  mesh: |-
    extensionProviders:
    - name: "gateway.opa.local"
      envoyExtAuthzGrpc:
        service: "istio-system/external-authz-grpc.local"
        port: "9191"
````

### Create Istio CUSTOM AuthorizationPolicy

```bash
kubectl apply -f - <<<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: gateway-opa
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: CUSTOM
  provider:
    name: "gateway.opa.local"
  rules:
  - to:
    - operation:
        notPaths: ["/ip"]
EOF
```

And now after create AuthorizationPolicy for all incoming requests to Istio Gateway with path not matched `/ip` OPA evaluate checks from opa-gateway-policy configmap.

## Useful links

Main manual from Istio blog: https://istio.io/latest/blog/2021/better-external-authz/

AuthorizationPolicy documentation: https://istio.io/latest/docs/reference/config/security/authorization-policy/

Open Policy Agent Language: https://www.openpolicyagent.org/docs/latest/policy-language/

More examples of builtin functions available in rego rules: https://www.openpolicyagent.org/docs/latest/policy-reference/

Online decode JWT token: https://jwt.io

## Additional example of Open Policy Agent rules

### JWT token check

If you want to check the JWT token for an incoming requests, then this can be done using the following REGO rules:

```bash
    package envoy.authz
    import input.attributes.request.http as http_request
    default allow = false
 
    token = {"payload": payload} {
      [_, payload, _] := io.jwt.decode(http_request.headers.authn_token)
    }
    allow {
        is_company_empl
        is_company_idp
        action_allowed
    }
    is_company_empl {
      endswith(token.payload.email, "@company.com")
    }
    is_company_idp {
      token.payload.iss == "https://id.company.com/auth/realms/Company"
    }
```

In this example, the JWT token is taken from the authn_token header and decrypted, after which we perform 2 checks:
- Use has email on `company.com` domain
- This JWT token was issued by Company IDP

