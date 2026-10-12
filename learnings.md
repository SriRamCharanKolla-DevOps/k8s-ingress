# 🚀 Kubernetes Ingress & AWS ALB Controller — Mistakes & Learnings

ఈ డాక్యుమెంట్‌లో మనం `k8s-ingress` ప్రాజెక్ట్ డెవలప్‌మెంట్ మరియు AWS Load Balancer Controller (ALB) సెటప్ చేసే సమయంలో ఎదుర్కొన్న ముఖ్యమైన తప్పులు (Mistakes), ఎర్రర్స్ (Errors), వాటికి గల కారణాలు (Root Causes), మరియు భవిష్యత్ రిఫరెన్స్ కోసం పరిష్కారాలు (Solutions & Best Practices) స్పష్టంగా పొందుపరచబడ్డాయి.

---

## 📑 విషయ సూచిక (Table of Contents)
1. [Mistake 1: Deployment లో Pod Template Indentation లోపం](#-mistake-1-deployment-లో-pod-template-indentation-లోపం)
2. [Mistake 2: Ingress `rules.http.paths` Hierarchy & Indentation లోపం](#-mistake-2-ingress-ruleshttppaths-hierarchy--indentation-లోపం)
3. [Mistake 3: ACM Certificate `Pending validation` & ALB 0 Listeners Error](#-mistake-3-acm-certificate-pending-validation--alb-0-listeners-error)
4. [Mistake 4: సర్టిఫికెట్ `ISSUED` అయినా Ingress లో పాత ఎర్రర్ కనిపించడం & Reconciliation Lag](#-mistake-4-సర్టిఫికెట్-issued-అయినా-ingress-లో-పాత-ఎర్రర్-కనిపించడం--reconciliation-lag)
5. [Route 53 Alias Record Setup (బ్రౌజర్‌లో యాక్సెస్ చేయడానికి చివరి స్టెప్)](#-route-53-alias-record-setup-బ్రౌజర్‌లో-యాక్సెస్-చేయడానికి-చివరి-స్టెప్)
6. [AWS Load Balancer Controller & Ingress Debugging Cheat Sheet](#-aws-load-balancer-controller--ingress-debugging-cheat-sheet)

---

## 📌 Mistake 1: Deployment లో Pod Template Indentation లోపం

### 🛑 Error Message
```text
$ kubectl apply -f manifest.yaml
service/app1 created
Error from server (BadRequest): error when creating "manifest.yaml": Deployment in version "v1" cannot be handled as a Deployment: strict decoding error: unknown field "spec.selector.template"
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
[manifest.yaml](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L9-L25) లో `spec.selector` మరియు `spec.template` రెండు కూడా `spec` కింద సమానమైన లెవెల్ (Siblings) లో ఉండాలి.
కానీ పొరపాటున `template:` ని `selector:` లోపల (4 spaces) రాయడం వల్ల, కుబెర్‌నెటీస్ దీనిని `spec.selector.template` గా గుర్తించి strict decoding ఎర్రర్ ఇచ్చింది.

### ❌ తప్పు కోడ్ (Wrong Code):
```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      project: roboshop
      tier: frontend
      component: app1
    template:                 # ❌ తప్పు: selector లోపల ఉంది (4 spaces)
      metadata:
        labels: ...
      spec:
        containers: ...
```

### ✅ సరైన కోడ్ (Fixed Code in [manifest.yaml:L9-L25](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L9-L25)):
```yaml
spec:
  replicas: 2
  selector:                   # Sibling 1 (2 spaces)
    matchLabels:
      project: roboshop
      tier: frontend
      component: app1
  template:                   # ✅ సరిగ్గా: spec కింద నేరుగా ఉంది (2 spaces)
    metadata:
      labels:
        project: roboshop
        tier: frontend
        component: app1
    spec:
      containers:
      - name: app1
        image: ramcharankola2/app1:1.0.0
```

---

## 📌 Mistake 2: Ingress `rules.http.paths` Hierarchy & Indentation లోపం

### 🛑 Error Message
```text
$ kubectl apply -f manifest.yaml
deployment.apps/app1 unchanged
service/app1 unchanged
Error from server (BadRequest): error when creating "manifest.yaml": Ingress in version "v1" cannot be handled as a Ingress: strict decoding error: unknown field "spec.rules[0].backend", unknown field "spec.rules[0].path", unknown field "spec.rules[0].paths"
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
Kubernetes `networking.k8s.io/v1` Ingress నియమాల ప్రకారం:
* `spec.rules` కింద కేవలం `host` మరియు `http` మాత్రమే నేరుగా ఉంటాయి.
* `paths`, `path`, `pathType`, మరియు `backend` ఫీల్డ్స్ అన్నీ కూడా తప్పనిసరిగా `http:` లోపల నెస్టెడ్ అయి ఉండాలి.
* ఇండింటేషన్ తేడా వల్ల `paths`, `path`, మరియు `backend` ఫీల్డ్స్ అన్నీ `rules[0]` కింద డైరెక్ట్ కీస్ గా పార్స్ అయ్యాయి.

### ❌ తప్పు కోడ్ (Wrong Code):
```yaml
spec:
  rules:
  -  host: "app1.aitechapp.fun"
     http:                     # ❌ కింద ఏమీ లేదు
     paths:                    # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
     - pathType: Prefix
     path: "/"                 # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
     backend:                  # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
       service:
         name: app1
         port:
           number: 80 
```

### ✅ సరైన కోడ్ (Fixed Code in [manifest.yaml:L52-L64](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L52-L64)):
```yaml
spec:
  ingressClassName: alb        # ✅ AWS Load Balancer Controller కోసం తప్పనిసరి
  rules:
  - host: "app1.aitechapp.fun"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: app1
            port:
              number: 80
```

---

## 📌 Mistake 3: ACM Certificate `Pending validation` & ALB 0 Listeners Error

### 🛑 Symptoms & Error Message
AWS EC2 Console లో Application Load Balancer (`k8s-roboshop-...`) క్రియేట్ అయ్యి **Active** స్టేట్‌లో ఉంటుంది, కానీ లోపల చూస్తే:
> **Listeners and rules (0):** *"No listener. You don't have a listener associated with this load balancer."*

మరియు `kubectl describe ingress app1` రన్ చేసినప్పుడు Warning ఈవెంట్ కనిపిస్తుంది:
```text
Warning  FailedDeployModel  ingress  Failed deploy model due to operation error Elastic Load Balancing v2: CreateListener, 
https response error StatusCode: 400, api error UnsupportedCertificate: 
The certificate 'arn:aws:acm:us-east-1:069416262340:certificate/e09fdd96-80c6-4814-a235-89c17501f08f' 
must have a fully-qualified domain name, a supported signature, and a supported key size.
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
1. [manifest.yaml:L46-L47](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L46-L47) లో కేవలం HTTPS Port 443 మాత్రమే కాన్ఫిగర్ చేయబడింది:
   ```yaml
   alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
   alb.ingress.kubernetes.io/certificate-arn: 'arn:aws:acm:us-east-1:069416262340:certificate/e09fdd96-80c6-4814-a235-89c17501f08f'
   ```
2. డొమైన్ పేరు `*.aitechapp.fun` మరియు సర్టిఫికెట్ ARN సరైనవే అయినప్పటికీ, AWS Certificate Manager (ACM) లో ఆ సర్టిఫికెట్ ఇంకా **`Pending validation`** స్టేటస్‌లో ఉంది.
3. AWS Elastic Load Balancing నియమాల ప్రకారం, **`Issued`** స్టేటస్‌లో లేని సర్టిఫికెట్‌ను ఏ లోడ్ బ్యాలెన్సర్‌కు అటాచ్ చేయలేము. అందుకే AWS `CreateListener` రిక్వెస్ట్‌ను రిజెక్ట్ చేసింది.
4. ఫలితంగా ALB క్రియేట్ అయినా కూడా 0 లిజినర్లతో ఆగిపోయింది.

### 🛠️ పరిష్కారం (Fix Steps):

#### Step 1: DNS Validation పూర్తి చేయడం (Route 53 లేదా DNS Registrar లో)
1. AWS Console లో **Certificate Manager (ACM)** కి వెళ్ళండి.
2. సంబంధిత Certificate ID (`e09fdd96-80c6-4814-a235-89c17501f08f`) పై క్లిక్ చేయండి.
3. **Domains** సెక్షన్‌లో:
   * Route 53 లో డొమైన్ ఉంటే: **"Create records in Route 53"** బటన్ క్లిక్ చేయండి.
   * వేరే ప్రొవైడర్ (GoDaddy, Namecheap) అయితే: CNAME Name & CNAME Value ని మీ DNS లో యాడ్ చేయండి.
4. 2-5 నిమిషాల్లో స్టేటస్ **`Issued`** (Green Checkmark) గా మారుతుంది.

#### Step 2: Ingress ని Re-apply చేయడం
సర్టిఫికెట్ `Issued` అయిన తర్వాత:
```bash
kubectl apply -f app1/manifest.yaml
```
ఇప్పుడు ALB లో Port 443 (HTTPS) లిజినర్ మరియు రూల్స్ ఆటోమేటిక్‌గా క్రియేట్ అవుతాయి.

#### 💡 ప్రో-టిప్ (టెస్టింగ్ కోసం క్విక్ ట్రిక్):
సర్టిఫికెట్ వాలిడేషన్ ఇంకా పూర్తి కానప్పుడు ట్రాఫిక్ టెస్ట్ చేయాలనుకుంటే, [manifest.yaml:L46-L47](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L46-L47) లో తాత్కాలికంగా HTTP Port 80 ని వాడవచ్చు:
```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
# alb.ingress.kubernetes.io/certificate-arn: ...
```

---

## 📌 Mistake 4: సర్టిఫికెట్ `ISSUED` అయినా Ingress లో పాత ఎర్రర్ కనిపించడం & Reconciliation Lag

### 🛑 Error Message / Confusion
Route 53 లో DNS CNAME రికార్డ్ యాడ్ చేసి, ACM లో సర్టిఫికెట్ **`Issued`** అయినప్పటికీ, టెర్మినల్ లో `kubectl describe ingress app1` రన్ చేస్తే ఇంకా పాత ఎర్రర్ చూపిస్తుంది:
```text
Warning  FailedDeployModel  5m14s  ingress  Failed deploy model due to operation error Elastic Load Balancing v2: CreateListener...
api error UnsupportedCertificate: The certificate '...' must have a fully-qualified domain name...
```
మరియు AWS Console లో ALB లో ఇంకా 0 Listeners కనిపిస్తాయి.

### 🔍 Root Cause (ఎందుకు ఇలా జరుగుతుంది?)
1. **Kubernetes Event Retention Policy:**
   * కుబెర్‌నెటీస్‌లో `kubectl describe` చూపించే `Events` హిస్టారికల్ లాగ్స్ మాత్రమే. ఇవి డిఫాల్ట్‌గా **1 గంట (60 నిమిషాలు)** పాటు etcd లో స్టోర్ అయ్యి ఉంటాయి (`--event-ttl=1h`).
   * ఎర్రర్ వచ్చిన సమయాన్ని గమనించండి (`Age: 5m14s`). సర్టిఫికెట్ AWS ACM లో `ISSUED` అయిన సమయానికి సరిగ్గా **కొన్ని సెకన్ల ముందు** కంట్రోలర్ చివరిసారిగా ట్రై చేసి ఫెయిల్ అయిన ఈవెంట్ అది.
2. **Kubernetes Controller Watch Mechanism (No Real-Time AWS Sync):**
   * AWS Load Balancer Controller అనేది కుబెర్‌నెటీస్ ఇన్గ్రెస్ ఆబ్జెక్ట్‌లలో మార్పులను (Events) మాత్రమే గమనిస్తుంది (Watch).
   * AWS క్లౌడ్‌లో (ACM లో) సర్టిఫికెట్ స్టేటస్ మారిన వెంటనే కుబెర్‌నెటీస్ కంట్రోలర్‌కు నోటిఫికేషన్ రాదు.
   * కుబెర్‌నెటీస్ లో ఇన్గ్రెస్ రిసోర్స్ అప్‌డేట్ అయ్యే వరకు లేదా బ్యాక్‌గ్రౌండ్ లాంగ్ పీరియాడిక్ సింక్ లూప్ జరిగే వరకు కంట్రోలర్ రీ-ట్రై చేయదు.

### 🛠️ పరిష్కారం (Controller ని రీ-ట్రిగ్గర్ చేయడం ఎలా?):
సర్టిఫికెట్ `ISSUED` అయిన వెంటనే కంట్రోలర్ కొత్త స్టేట్‌తో ALB లిజినర్‌ను రీ-క్రియేట్ చేయాలంటే:

#### Option 1: Ingress ని Annotate చేసి Reconcile ట్రిగ్గర్ చేయడం (Recommended):
```bash
kubectl annotate ingress app1 reconcile="$(date +%s)" --overwrite
```

#### Option 2: Ingress మ్యానిఫెస్ట్‌ను Re-apply చేయడం:
```bash
kubectl apply -f app1/manifest.yaml
```

#### Option 3: Controller Pod ని Restart చేయడం:
```bash
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```
> **ఫలితం:** కంట్రోలర్ వెంటనే ACM నుండి `ISSUED` సర్టిఫికెట్‌ను ఫెచ్ చేసి ALB లో **HTTPS: 443** లిజినర్ మరియు టార్గెట్ గ్రూప్‌ను క్రియేట్ చేస్తుంది. `kubectl get ingress app1` లో `ADDRESS` కాలమ్‌లో ALB DNS నేమ్ కనిపిస్తుంది.

---

## 📌 Route 53 Alias Record Setup (బ్రౌజర్‌లో యాక్సెస్ చేయడానికి చివరి స్టెప్)

ALB మరియు HTTPS లిజినర్ క్రియేట్ అయిన తర్వాత, బ్రౌజర్‌లో `https://app1.aitechapp.fun` ఓపెన్ అవ్వాలంటే Route 53 లో డొమైన్‌ను ALB కి మ్యాప్ చేయాలి.

### స్టెప్స్:
1. AWS Route 53 లో **Hosted Zones** $\rightarrow$ `aitechapp.fun` లోకి వెళ్ళండి.
2. **"Create record"** బటన్ క్లిక్ చేయండి.
3. వివరాలు ఇలా నమోదు చేయండి:
   * **Record name:** `app1` (పూర్తి పేరు: `app1.aitechapp.fun`)
   * **Record type:** `A - Routes traffic to an IPv4 address and some AWS resources`
   * **Alias:** స్విచ్ ఆన్ (Toggle ON) చేయండి.
   * **Route traffic to:** 
     * *Alias to Application and Classic Load Balancer*
     * *Region:* `us-east-1`
     * *Choose load balancer:* మీ ALB అయిన `k8s-roboshop-37213adecc` ని ఎంచుకోండి.
4. **Create records** క్లిక్ చేయండి.
5. బ్రౌజర్‌లో **`https://app1.aitechapp.fun`** ఓపెన్ చేస్తే సురక్షితమైన గ్రీన్ లాక్ సింబల్‌తో మీ `app1` కంటైనర్ లోడ్ అవుతుంది!

---

## 📌 AWS Load Balancer Controller & Ingress Debugging Cheat Sheet

భవిష్యత్తులో ఇలాంటి సమస్యలు వచ్చినప్పుడు క్షణాల్లో గుర్తించడానికి ఈ కమాండ్స్ వాడండి:

```bash
# 1. Ingress స్టేటస్ మరియు వార్నింగ్ ఈవెంట్స్ చూడటానికి:
kubectl describe ingress <ingress-name>

# 2. AWS Load Balancer Controller లైవ్ లాగ్స్ చూడటానికి:
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100 -f

# 3. ACM సర్టిఫికెట్ స్టేటస్ చెక్ చేయడానికి:
aws acm describe-certificate \
  --certificate-arn <YOUR_CERTIFICATE_ARN> \
  --region us-east-1 \
  --query "CertificateDetail.Status"

# 4. K9s ద్వారా చూడటానికి (Interactive CLI):
# Terminal లో 'k9s' టైప్ చేసి ':ingress' లేదా ':pods' (in kube-system) లోకి వెళ్లి 'l' నొక్కి లాగ్స్ చూడవచ్చు.
```
