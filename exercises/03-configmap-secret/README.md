# Exercise 03 — ConfigMap and Secret

> Related: [ConfigMap/Secret skeleton](../../skeletons/configmap-secret.yaml) | [README — Workloads & Scheduling](../../README.md#domain-3--workloads--scheduling-15)

Create ConfigMaps and Secrets, then inject them into a pod as environment variables and mounted files.

## Tasks

1. Create a namespace called `exercise-03`
2. Create a ConfigMap named `app-config` with:
   - Key `APP_MODE` = `production`
   - Key `LOG_LEVEL` = `debug`
3. Create a Secret named `db-creds` with:
   - Key `DB_USER` = `admin`
   - Key `DB_PASS` = `s3cretP@ss`
4. Create a pod named `app` that:
   - Uses image `busybox:1.36`, command `sleep 3600`
   - Loads `APP_MODE` and `LOG_LEVEL` from the ConfigMap as env vars
   - Loads `DB_USER` and `DB_PASS` from the Secret as env vars
   - Mounts the entire ConfigMap as files at `/etc/config/`
5. Verify the env vars are set inside the pod
6. Verify the mounted files exist at `/etc/config/`

## Hints

<details>
<summary>Stuck? Click to reveal hints</summary>

- `k create configmap app-config --from-literal=APP_MODE=production --from-literal=LOG_LEVEL=debug`
- `k create secret generic db-creds --from-literal=DB_USER=admin --from-literal=DB_PASS=s3cretP@ss`
- Use `envFrom` to load all keys from a ConfigMap or Secret
- Use `volumes` + `volumeMounts` to mount ConfigMap as files

</details>

## What tripped me up

> I referenced a ConfigMap name that didn't exist yet (`app-conf` instead of `app-config` — typo). The pod went into `CreateContainerConfigError` and I spent 3 minutes staring at the YAML before checking `k describe pod` events. The event message says exactly which ConfigMap is missing. Check events first, always.
>
> Also confused `envFrom` vs `env.valueFrom`. `envFrom` loads ALL keys from a ConfigMap/Secret as env vars. `env.valueFrom.configMapKeyRef` loads a single key. On the exam, if the question says "load all keys," use `envFrom`. If it says "load KEY_X as MY_VAR," use `valueFrom`. Getting them backwards doesn't error — you just get wrong variable names.

## Verify

```bash
# Check env vars
k exec app -n exercise-03 -- env | grep -E "APP_MODE|LOG_LEVEL|DB_USER|DB_PASS"

# Check mounted files
k exec app -n exercise-03 -- ls /etc/config/
k exec app -n exercise-03 -- cat /etc/config/APP_MODE
```

## Cleanup

```bash
k delete ns exercise-03
```

<details>
<summary>Solution</summary>

```bash
k create ns exercise-03

k create configmap app-config -n exercise-03 \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=debug

k create secret generic db-creds -n exercise-03 \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=s3cretP@ss
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: exercise-03
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-creds
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

```bash
k apply -f app.yaml

# Verify
k exec app -n exercise-03 -- env | grep -E "APP_MODE|LOG_LEVEL|DB_USER|DB_PASS"
k exec app -n exercise-03 -- ls /etc/config/
```

</details>


### English Explanation

In Kubernetes, **`volumes`** and **`volumeMounts`** work together as a two-step process to attach storage to your containers.

#### Real-world Analogy:
* **`volumes`** = Buying an external hard drive or USB pen drive. You have the storage ready at the computer (Pod) level, but no folder is using it yet.
* **`volumeMounts`** = Plugging that USB drive into a specific port and mapping it to a folder path (like `D:\files` or `/etc/config`) so a specific software (Container) can read or write files.

---

#### Key Differences:

| Feature | `spec.volumes` | `spec.containers[].volumeMounts` |
| :--- | :--- | :--- |
| **Level** | **Pod level** | **Container level** |
| **Question it answers** | *What* storage exists and *where* does data come from? | *Where* inside the container filesystem should it appear? |
| **Scope** | Available to **all** containers in the Pod | Specific to **one** container |
| **Typical Fields** | `name`, storage source (`configMap`, `secret`, `pvc`, `emptyDir`, etc.) | `name` (matching the volume), `mountPath`, `readOnly` |

---

#### Example from [app.yaml](file:///home/rupesh/CKA-Certified-Kubernetes-Administrator/exercises/03-configmap-secret/app.yaml):

```yaml
spec:
  containers:
    - name: app
      image: busybox:1.36
      # 2. volumeMounts (Container Level):
      # Mounts the volume named "config-vol" into this container at path "/etc/config"
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config

  # 1. volumes (Pod Level):
  # Declares the storage source (here, a ConfigMap named app-config) and names it "config-vol"
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

> **Why are they separated?**
> A single Pod can run multiple containers. By defining the `volume` once at the Pod level, multiple containers can mount the exact same volume at different paths (e.g., container A mounts it at `/data` as read-write, while container B mounts it at `/backup` as read-only).

---

---

### తెలుగు వివరణ (Telugu Explanation)

కుబెర్‌నెటిస్ (Kubernetes) లో **`volumes`** మరియు **`volumeMounts`** అనేవి కంటైనర్‌కి స్టోరేజ్ (డేటా/ఫైల్స్) అందించడానికి వాడే రెండు ముఖ్యమైన భాగాలు.

#### నిజ జీవిత ఉదాహరణ (Real-world Analogy):
* **`volumes`**: మీ చేతిలో ఒక **పెన్‌డ్రైవ్ (Pen Drive)** లేదా ఎక్స్‌టర్నల్ హార్డ్ డిస్క్ ఉండటం వంటిది. అంటే స్టోరేజ్ సిస్టమ్ (Pod) దగ్గర సిద్ధంగా ఉంది.
* **`volumeMounts`**: ఆ పెన్‌డ్రైవ్‌ను కంప్యూటర్‌కు కనెక్ట్ చేసి, ఒక ప్రత్యేకమైన ఫోల్డర్ (Path - ఉదాహరణకు `/etc/config`) లో ఓపెన్ చేసి వాడటం వంటిది.

---

#### ప్రధాన తేడాలు (Key Differences):

1. **`volumes` (Pod Level - పాడ్ స్థాయి):**
   - ఇది Pod డెఫినిషన్‌లో (`spec.volumes`) ఉంటుంది.
   - ఇది **"స్టోరేజ్ మూలం ఏంటి?"** అని చెబుతుంది (అది ConfigMap ఆ? Secret ఆ? లేక PersistentVolumeClaim ఆ?).
   - ఆ స్టోరేజ్ మొత్తానికి ఒక పేరు (`name`) ఇస్తుంది.
   - పాడ్ లోపల ఉన్న అన్ని కంటైనర్లకి ఈ వాల్యూమ్ అందుబాటులో ఉంటుంది.

2. **`volumeMounts` (Container Level - కంటైనర్ స్థాయి):**
   - ఇది Container డెఫినిషన్‌లో (`spec.containers[].volumeMounts`) ఉంటుంది.
   - ఇది **"ఆ స్టోరేజ్ కంటైనర్ లోపల ఏ డైరెక్టరీ/పాత్‌లో (`mountPath`) కనిపించాలి?"** అని నిర్ణయిస్తుంది.
   - `volumes` లో ఇచ్చిన పేరును రిఫరెన్స్ చేసి, ఆ కంటైనర్ ఫైల్ సిస్టమ్‌కి అటాచ్ చేస్తుంది.

---

#### ముఖ్యమైన ప్రయోజనం:
ఒక Pod లో రెండు లేదా అంతకంటే ఎక్కువ కంటైనర్లు ఉన్నప్పుడు, `volumes` కింద **ఒకేసారి** స్టోరేజ్ డిఫైన్ చేసి:
- కంటైనర్ 1 ఆ డేటాను `/app/data` లో మౌంట్ చేసుకోవచ్చు (Read-Write).
- కంటైనర్ 2 అదే డేటాను `/var/log` లో రీడ్-ఓన్లీగా (`readOnly: true`) మౌంట్ చేసుకోవచ్చు.