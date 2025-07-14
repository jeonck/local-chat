# Ollama Web UI 연동 및 배포 문제 해결 전체 절차서

## 1. Ollama systemd 서비스 환경변수(CORS 등) 적용

### 1-1. systemd 오버라이드 파일 생성 및 환경변수 추가
```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```
아래와 같이 입력:
```
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_ORIGINS=*"
```

### 1-2. systemd 데몬 리로드 및 Ollama 서비스 재시작
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 1-3. 적용 여부 확인
```bash
sudo systemctl show ollama | grep OLLAMA_ORIGINS
# OLLAMA_ORIGINS=* 출력 확인
```
또는 Ollama 로그에서 환경변수 적용 확인:
```bash
sudo journalctl -u ollama -f
```

## 2. Kubernetes Pod에서 Ollama API 접근 확인

### 2-1. 네트워크/방화벽 정책 점검
Pod에서 호스트 IP로 curl 테스트:
```bash
kubectl exec -it [pod명] -- apk add curl
kubectl exec -it [pod명] -- curl -v http://10.30.1.10:11434/api/tags
```
iptables, UFW 등 방화벽에서 11434 포트 허용 여부 확인.

## 3. Nginx 프록시 및 정적 파일 서빙 문제 해결

### 3-1. Nginx 프록시 및 CORS 설정 (nginx.conf)
```nginx
events {}

http {
    include mime.types;
    server {
        listen 80;

        # Ollama 프록시 설정 (CORS 처리 강화)
        location /api/ {
            proxy_pass http://10.30.1.10:11434/;

            # CORS 핵심 헤더
            add_header 'Access-Control-Allow-Origin' "$http_origin" always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;

            # 프리플라이트 요청 처리
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Length' 0;
                return 204;
            }

            # 헤더 전파 설정
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # 기본 정적 파일 서빙
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### 3-2. Dockerfile 확인
```dockerfile
# test.Dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf

# 정적 파일을 Nginx의 기본 디렉터리에 복사
COPY . /usr/share/nginx/html

# 80번 포트 노출
EXPOSE 80

# Nginx 실행
CMD ["nginx", "-g", "daemon off;"]
```

### 3-3. Dockerfile 및 파일 복사 확인
```dockerfile
# Dockerfile에서 Nginx 설정 확인
COPY nginx.conf /etc/nginx/nginx.conf
COPY . /usr/share/nginx/html
```
정적 파일(styles.css, script.js, index.html 등)이 정상적으로 복사되었는지 컨테이너 내부에서 확인.

## 4. Kubernetes 배포 설정 (deploy-svc.yaml)

### 4-1. 네임스페이스 및 서비스 어카운트 설정
```yaml
---
# 네임스페이스
apiVersion: v1
kind: Namespace
metadata:
  name: local-chatbot
---
# 레지스트리 인증 Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: local-chatbot
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJiYXJlLWlvLnRyZWFsLnh5ejozMTk1MyI6eyJ1c2VybmFtZSI6ImR0cCIsInBhc3N3b3JkIjoiZGlnaXRhbHR3aW4xMiFAIn19fQ==
---
# 서비스 어카운트
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-chatbot
  namespace: local-chatbot
---
# 클러스터 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: local-chatbot-role
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
---
# 클러스터 역할 바인딩
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-chatbot-binding
subjects:
- kind: ServiceAccount
  name: local-chatbot
  namespace: local-chatbot
roleRef:
  kind: ClusterRole
  name: local-chatbot-role
  apiGroup: rbac.authorization.k8s.io
---
# 디플로이먼트
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-chatbot
  namespace: local-chatbot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-chatbot
  template:
    metadata:
      labels:
        app: local-chatbot
    spec:
      serviceAccountName: local-chatbot
      imagePullSecrets:
      - name: registry-credentials
      nodeSelector:
        app-local-chatbot: local-chatbot  # wnode2 노드의 라벨과 매칭
      containers:
      - name: local-chatbot
        image: bare-io.treal.xyz:31953/local-chatbot:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80 # port 주의
        env:
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: OLLAMA_HOST
          value: "0.0.0.0"
        - name: OLLAMA_ORIGINS
          value: "http://*"
---
# 서비스
apiVersion: v1
kind: Service
metadata:
  name: local-chatbot-service
  namespace: local-chatbot
spec:
  selector:
    app: local-chatbot
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001
  type: LoadBalancer
```

## 5. Web UI 코드 및 CORS 우회 적용

### 5-1. 프록시 경유로 Ollama API 호출
```javascript
const url = "/api/generate";
const response = await fetch(url, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-Requested-With': 'XMLHttpRequest'
    },
    credentials: 'same-origin',
    body: JSON.stringify(data)
});
```

## 6. 정적 파일(styles.css) 미반영 문제 해결

### 6-1. Nginx 설정 및 파일 복사 확인
- `include mime.types;` 포함 여부 확인
- `location` 설정이 정상인지 확인
- 파일이 실제로 존재하는지 확인

### 6-2. HTML 내 CSS 링크 경로 확인
```html
<link rel="stylesheet" href="styles.css"> 
<!-- 또는 -->
<link rel="stylesheet" href="/styles.css">
```

### 6-3. 브라우저 캐시 강제 새로고침
- Ctrl+Shift+R 또는 캐시 비우고 새로고침
- 네트워크 탭에서 200 OK, Content-Type: text/css 확인.

## 7. 최종 점검 및 검증
- 웹 UI에서 채팅 요청 시 CORS 에러 없이 정상 동작 확인.
- styles.css 등 정적 파일이 정상적으로 반영되는지 확인.
- Pod에서 Ollama API로 직접 curl 호출 시 200 OK 응답 확인.
- Ollama 서비스가 재기동 후에도 환경변수(특히 CORS 관련)가 남아있는지 확인.
