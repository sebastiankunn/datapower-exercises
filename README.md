# 🏎️ IBM&nbsp;DataPower&nbsp;Sprint&nbsp;Lab&nbsp;(2‑3&nbsp;giorni)

> **Focus:** costruire velocemente un flusso OAuth2&nbsp;+&nbsp;JWT end‑to‑end, con DataPower come *Token Issuer* e *Security&nbsp;Gateway*. Il laboratorio è pensato per essere completato in **massimo 3&nbsp;giorni**.

---

## 🗺️ Architettura complessiva

```
┌──────────┐        ┌────────────┐        ┌────────────┐
│  Client  │ ──▶── │ DataPower   │ ──▶── │ Echo&nbsp;Srv&nbsp;  │
│ (curl)   │       │  Gateways   │       │   5000     │
└──────────┘        │ 8080‑82    │        └────────────┘
        ▲           │            │               ▲
        │           ▼            ▼               │
        │     ┌────────────┐  ┌────────────┐     │
        │     │ JWT&nbsp;B/E    │  │ Auth&nbsp;Prov. │     │
        └──── │   5001     │  │   5002     │ ◀───┘
              └────────────┘  └────────────┘
```

* **simple\_echo.py&nbsp;(5000)** – server che riflette qualunque richiesta.
* **jwt\_backend.py&nbsp;(5001)** – genera chiavi RSA, espone JWKS e valida JWT.
* **auth\_provider.py&nbsp;(5002)** – valida utenti & client per il token endpoint.
* **DataPower** – tre Multi‑Protocol&nbsp;Gateway (8080‑82).

---

## 📦 Sorgenti completi dei backend

### 1️⃣ Echo Server – `simple_echo.py`

```python
from flask import Flask, request, jsonify
import time

app = Flask(__name__)

@app.route('/echo', methods=['GET', 'POST', 'PUT', 'DELETE'])
def echo():
    """Restituisce metodo, header e corpo ricevuti."""
    return jsonify({
        'method': request.method,
        'path': request.path,
        'headers': dict(request.headers),
        'body': request.get_data(as_text=True),
        'timestamp': time.time()
    })

@app.route('/basic-echo', methods=['GET', 'POST'])
def basic_echo():
    auth = request.headers.get('Authorization', '')
    return jsonify({
        'received_auth': auth,
        'method': request.method,
        'message': 'Basic auth received'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

### 2️⃣ JWT Backend – `jwt_backend.py`

> All’avvio genera automaticamente **RSA‑2048**. Espone JWKS **e anche la chiave privata** (solo per laboratorio!).

```python
from flask import Flask, request, jsonify
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
import jwt, time, base64

app = Flask(__name__)

# 🗝️ Generazione chiavi
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# Serializza in PEM
private_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()
)
public_pem = public_key.public_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PublicFormat.SubjectPublicKeyInfo
)

# Helpers
def int_to_base64url(val: int) -> str:
    byte_length = (val.bit_length() + 7) // 8
    return base64.urlsafe_b64encode(val.to_bytes(byte_length, 'big')).decode().rstrip('=')

# ----------------  ROUTES  ----------------
@app.route('/.well-known/jwks.json')
def jwks():
    """Serve la chiave pubblica in formato JWKS."""
    pub_num = public_key.public_numbers()
    jwk = {
        'kty': 'RSA', 'use': 'sig', 'kid': 'test-key-1', 'alg': 'RS256',
        'n': int_to_base64url(pub_num.n),
        'e': int_to_base64url(pub_num.e)
    }
    return jsonify({'keys': [jwk]})

# ⚠️ SOLO PER LAB – ESPONE CHIAVE PRIVATA!
@app.route('/private-key.pem')
def private_key_pem():
    resp = app.response_class(private_pem, mimetype='application/x-pem-file')
    resp.headers['Content-Disposition'] = 'attachment; filename=private_key.pem'
    return resp

@app.route('/protected', methods=['GET', 'POST'])
def protected():
    auth = request.headers.get('Authorization')
    if not auth or not auth.startswith('Bearer '):
        return jsonify({'error': 'JWT required'}), 401
    token = auth[7:]
    try:
        payload = jwt.decode(token, public_pem, algorithms=['RS256'])
        return jsonify({'message': 'JWT valid!', 'payload': payload})
    except jwt.ExpiredSignatureError:
        return jsonify({'error': 'Token expired'}), 401
    except jwt.InvalidTokenError as e:
        return jsonify({'error': f'Invalid token: {e}'}), 401

@app.route('/info')
def info():
    return jsonify({
        'message': 'JWT Backend Ready',
        'jwks_url': '/.well-known/jwks.json',
        'protected_url': '/protected'
    })

if __name__ == '__main__':
    print('🔑 JWT Backend avviato su :5001')
    app.run(host='0.0.0.0', port=5001, debug=True)
```

---

### 3️⃣ Auth Provider – `auth_provider.py`

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# Database “in‑memory”
USERS = {'alice': 'pass123', 'bob': 'pass456'}
CLIENTS = {'app1': 'secret1', 'app2': 'secret2'}

@app.route('/validate', methods=['POST'])
def validate():
    data = request.json or {}

    # 👤 User flow
    if 'username' in data:
        user, pwd = data.get('username'), data.get('password')
        if USERS.get(user) == pwd:
            return jsonify({'valid': True, 'type': 'user', 'subject': user, 'scopes': ['read', 'write']})

    # 🤖 Client Credentials flow
    if 'client_id' in data:
        cid, secret = data.get('client_id'), data.get('client_secret')
        if CLIENTS.get(cid) == secret:
            return jsonify({'valid': True, 'type': 'client', 'subject': cid, 'scopes': ['api_access']})

    return jsonify({'valid': False}), 401

@app.route('/users')
def list_users():
    return jsonify({'users': list(USERS), 'clients': list(CLIENTS)})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5002, debug=True)
```

---

# 📝&nbsp;Esercizi

## 📅 Giorno&nbsp;1&nbsp;– Token&nbsp;Issuer

### 🎯 Obiettivo

Creare l’endpoint **`/oauth/token` (porta&nbsp;8080)** che emette JWT.

### 🔨 Task

1. Basic&nbsp;Auth → chiama backend&nbsp;5002.
2. Client&nbsp;Creds → chiama backend&nbsp;5002.
3. Firma JWT con chiave scaricata da `http://localhost:5001/private-key.pem`.
4. Response OAuth2 standard (`access_token`, `token_type`, `expires_in`, `scope`).

### 👩‍💻 Implementazione

* Solo **GatewayScript**.
* Preleva JWK per `kid`.
* Claim minimi: `sub`, `iss`, `exp`, `scope`.

### ✅ Test

```bash
curl -X POST http://datapower:8080/oauth/token \
  -H "Authorization: Basic $(echo -n 'alice:pass123' | base64)" \
  -d "grant_type=password"

curl -X POST http://datapower:8080/oauth/token \
  -d "grant_type=client_credentials&client_id=app1&client_secret=secret1"
```

---

## 📅 Giorno&nbsp;2‑3&nbsp;– Security&nbsp;Gateways

### 2A&nbsp;– JWT&nbsp;→&nbsp;Echo&nbsp;(8081)

* Valida JWT via JWKS 5001.
* Forward a&nbsp;`http://localhost:5000/echo`.
* Header `X-User: <subject>`.

### 2B&nbsp;– Echo&nbsp;→&nbsp;JWT&nbsp;(8082)

* Riceve chiamate senza auth.
* Genera JWT (sub&nbsp;=`gateway-user`).
* Forward a&nbsp;`http://localhost:5001/protected` con header Bearer.

### ✅ Test Rapidi

```bash
token=$(curl -s -X POST ... | jq -r .access_token)
curl -H "Authorization: Bearer $token" http://datapower:8081/echo

curl http://datapower:8082/protected
```

---

## ✨ Bonus – PKCE (facoltativo)

```javascript
function validatePKCE(verifier, challenge) {
  var crypto = require('crypto');
  var hash = crypto.createHash('sha256').update(verifier).digest('base64url');
  return hash === challenge;
}
```

---

# ⚡ Setup rapido

| Shell | Comando                   |
| ----- | ------------------------- |
| 1     | `python simple_echo.py`   |
| 2     | `python jwt_backend.py`   |
| 3     | `python auth_provider.py` |

Quick&nbsp;check:

```bash
curl http://localhost:5000/echo
curl http://localhost:5001/info
curl http://localhost:5002/users
```

---

## 📋 Checklist

| Day | Attività          | ✔ |
| --- | ----------------- | - |
| 1   | Token endpoint    | ⬜ |
|     | BasicAuth&nbsp;→&nbsp;JWT   | ⬜ |
|     | ClientCreds&nbsp;→&nbsp;JWT | ⬜ |
| 2‑3 | Gateway&nbsp;8081      | ⬜ |
|     | Gateway&nbsp;8082      | ⬜ |
|     | Test & fix        | ⬜ |
| 💫  | PKCE bonus        | ⬜ |

---

## 📦 Deliverables

1. Export dominio DataPower `OAuth_Lab`.
2. Script di test (bash).
3. Screenshot esiti.

---

### ☠️ Security Disclaimer

La route `/private-key.pem` va **rimossa** o protetta prima di qualsiasi ambiente reale.
