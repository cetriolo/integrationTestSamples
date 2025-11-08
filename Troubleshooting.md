# 🔧 Troubleshooting Guide

Guida completa per risolvere i problemi comuni durante setup e testing.

## 📑 Indice

1. [Problemi Setup Iniziale](#problemi-setup-iniziale)
2. [Problemi Go](#problemi-go)
3. [Problemi Newman/Postman](#problemi-newmanpostman)
4. [Problemi k6](#problemi-k6)
5. [Problemi Playwright](#problemi-playwright)
6. [Problemi Hurl](#problemi-hurl)
7. [Problemi Docker](#problemi-docker)
8. [Problemi GitHub Actions](#problemi-github-actions)
9. [Problemi Rete/Connessione](#problemi-reteconnessione)

---

## Problemi Setup Iniziale

### ❌ "File environment.json non trovato"

**Problema**: Newman non trova il file environment.json

**Soluzione**:
```bash
# Verifica che il file esista
ls -la tests/postman/environment.json

# Se manca, crealo
cat > tests/postman/environment.json << 'EOF'
{
  "id": "test-environment",
  "name": "Test Environment",
  "values": [
    {
      "key": "baseUrl",
      "value": "http://localhost:8080",
      "enabled": true
    }
  ]
}
EOF
```

### ❌ "Directory reports/ non esiste"

**Problema**: Test falliscono perché manca la directory reports

**Soluzione**:
```bash
# Crea tutte le directory necessarie
mkdir -p reports/{newman,k6,playwright,hurl}
```

### ❌ "Permission denied" su script

**Problema**: Gli script non sono eseguibili

**Soluzione**:
```bash
# Rendi eseguibili gli script
chmod +x setup.sh
chmod +x check-structure.sh

# Su Windows con Git Bash
git update-index --chmod=+x setup.sh
```

---

## Problemi Go

### ❌ "go: cannot find main module"

**Problema**: go.mod non inizializzato

**Soluzione**:
```bash
# Inizializza il modulo
go mod init github.com/yourusername/go-api-test

# Scarica dipendenze
go mod tidy
```

### ❌ "package github.com/gorilla/mux: cannot find package"

**Problema**: Dipendenze non installate

**Soluzione**:
```bash
# Installa tutte le dipendenze
go get github.com/gorilla/mux
go get github.com/stretchr/testify/assert
go mod download
go mod tidy
```

### ❌ "bind: address already in use"

**Problema**: Porta 8080 già occupata

**Soluzione**:
```bash
# Trova il processo sulla porta 8080
lsof -i :8080

# Oppure su Linux
netstat -tulpn | grep 8080

# Termina il processo
kill -9 <PID>

# Oppure usa una porta diversa
PORT=8081 ./go-api-test
```

### ❌ Test falliscono con "connection refused"

**Problema**: L'API non è avviata o non raggiungibile

**Soluzione**:
```bash
# Avvia l'API in background
go run main.go &

# Aspetta che sia pronta
sleep 2

# Verifica che risponda
curl http://localhost:8080/health

# Esegui i test
make test-smoke
```

---

## Problemi Newman/Postman

### ❌ "newman: command not found"

**Problema**: Newman non installato

**Soluzione**:
```bash
# Installa Newman globalmente
npm install -g newman newman-reporter-htmlextra

# Verifica installazione
newman --version

# Alternative: usa Docker
docker run -v $(pwd):/etc/newman postman/newman:latest \
  run /etc/newman/tests/postman/collection.json
```

### ❌ "Could not get any response" in Postman

**Problema**: API non raggiungibile da Postman

**Soluzione**:
```bash
# 1. Verifica che l'API sia in esecuzione
curl http://localhost:8080/health

# 2. In Postman, disabilita SSL verification:
# Settings → General → SSL certificate verification → OFF

# 3. Verifica le variabili environment
# Collection → Variables → baseUrl = http://localhost:8080

# 4. Verifica firewall/antivirus
```

### ❌ "Cannot read property 'json' of undefined"

**Problema**: Response non è JSON

**Soluzione**:
```javascript
// Nei test Postman, controlla prima se la risposta è JSON
pm.test("Response is JSON", function () {
    pm.response.to.be.json;
});

// Poi accedi al JSON
if (pm.response.headers.get("Content-Type").includes("application/json")) {
    var jsonData = pm.response.json();
}
```

### ❌ Newman reports non generati

**Problema**: Report HTML non creato

**Soluzione**:
```bash
# Assicurati che la directory esista
mkdir -p reports

# Usa il path completo nel comando
newman run tests/postman/collection.json \
  --environment tests/postman/environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export $(pwd)/reports/newman-report.html
```

---

## Problemi k6

### ❌ "k6: command not found"

**Problema**: k6 non installato

**Soluzione**:

**Linux (Debian/Ubuntu)**:
```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

**macOS**:
```bash
brew install k6
```

**Windows**:
```powershell
choco install k6
```

**Alternative - Docker**:
```bash
docker run -i grafana/k6 run - <tests/k6/load-test.js
```

### ❌ "context deadline exceeded" in k6

**Problema**: Timeout dei test

**Soluzione**:
```javascript
// Aumenta i timeout in load-test.js
export const options = {
  thresholds: {
    'http_req_duration': ['p(95)<5000'], // aumenta a 5s
  },
};

// Oppure riduci il carico
stages: [
  { duration: '30s', target: 5 },  // riduci VUs
]
```

### ❌ k6 "too many open files"

**Problema**: Limite sistema di file aperti

**Soluzione**:
```bash
# Verifica limite attuale
ulimit -n

# Aumenta temporaneamente
ulimit -n 10000

# Permanente (Linux) - aggiungi a /etc/security/limits.conf
* soft nofile 10000
* hard nofile 10000
```

---

## Problemi Playwright

### ❌ "Playwright not found"

**Problema**: Playwright non installato

**Soluzione**:
```bash
cd tests/playwright

# Installa dipendenze
npm install

# Installa browser
npx playwright install

# Installa dipendenze sistema (Linux)
npx playwright install-deps
```

### ❌ "browserType.launch: Executable doesn't exist"

**Problema**: Browser Chromium non installato

**Soluzione**:
```bash
# Installa tutti i browser
npx playwright install

# Solo Chromium
npx playwright install chromium

# Con dipendenze sistema
npx playwright install --with-deps chromium
```

### ❌ Test Playwright lentissimi

**Problema**: Performance scarse

**Soluzione**:
```javascript
// playwright.config.js
export default defineConfig({
  // Esegui test in parallelo
  workers: 4,
  
  // Disabilita video e screenshot per speed
  use: {
    video: 'off',
    screenshot: 'only-on-failure',
  },
  
  // Riduci timeout per test veloci
  timeout: 10000,
});
```

### ❌ "ECONNREFUSED" in Playwright

**Problema**: API non raggiungibile

**Soluzione**:
```javascript
// Aggiungi retry e wait
test.beforeEach(async ({ request }) => {
  // Retry fino a 5 volte
  let retries = 5;
  while (retries > 0) {
    try {
      await request.get('http://localhost:8080/health');
      break;
    } catch (e) {
      retries--;
      await new Promise(r => setTimeout(r, 1000));
    }
  }
});
```

---

## Problemi Hurl

### ❌ "hurl: command not found"

**Problema**: Hurl non installato

**Soluzione**:

**Linux**:
```bash
VERSION=4.3.0
curl -LO https://github.com/Orange-OpenSource/hurl/releases/download/$VERSION/hurl_${VERSION}_amd64.deb
sudo dpkg -i hurl_${VERSION}_amd64.deb
```

**macOS**:
```bash
brew install hurl
```

**Docker**:
```bash
docker run --network host -v $(pwd)/tests/hurl:/tests \
  orangeopensource/hurl:latest \
  --test /tests/api-tests.hurl
```

### ❌ Hurl "Assert failure" non chiaro

**Problema**: Errore generico senza dettagli

**Soluzione**:
```bash
# Esegui con verbose per più dettagli
hurl --verbose --test tests/hurl/api-tests.hurl

# Oppure very verbose
hurl --very-verbose --test tests/hurl/api-tests.hurl

# Con colori per meglio leggibilità
hurl --color --test tests/hurl/api-tests.hurl
```

### ❌ Hurl "variable not defined"

**Problema**: Variabile non impostata

**Soluzione**:
```bash
# Passa variabili da command line
hurl --variable base_url=http://localhost:8080 tests/hurl/api-tests.hurl

# Oppure da file
cat > vars.env << EOF
base_url=http://localhost:8080
user_id=123
EOF

hurl --variables-file vars.env tests/hurl/api-tests.hurl
```

---

## Problemi Docker

### ❌ "docker: command not found"

**Problema**: Docker non installato

**Soluzione**:
- **Linux**: https://docs.docker.com/engine/install/
- **macOS**: https://docs.docker.com/desktop/install/mac-install/
- **Windows**: https://docs.docker.com/desktop/install/windows-install/

### ❌ "permission denied" su Docker

**Problema**: User non ha permessi Docker (Linux)

**Soluzione**:
```bash
# Aggiungi user al gruppo docker
sudo usermod -aG docker $USER

# Logout e login, oppure
newgrp docker

# Verifica
docker ps
```

### ❌ "port is already allocated"

**Problema**: Porta già in uso

**Soluzione**:
```bash
# Trova container che usa la porta
docker ps | grep 8080

# Ferma il container
docker stop <container-id>

# Oppure cambia porta in docker-compose.yml
ports:
  - "8081:8080"
```

### ❌ Docker build fallisce con "no space left"

**Problema**: Disco pieno

**Soluzione**:
```bash
# Pulisci immagini non usate
docker system prune -a

# Pulisci volumi
docker volume prune

# Verifica spazio
docker system df
```

### ❌ "health check failed" in Docker Compose

**Problema**: Container non passa health check

**Soluzione**:
```bash
# Vedi log del container
docker-compose logs api

# Esegui health check manualmente
docker exec <container-id> wget --spider http://localhost:8080/health

# Aumenta timeout in docker-compose.yml
healthcheck:
  interval: 10s
  timeout: 10s  # aumenta questo
  retries: 10   # e questo
```

---

## Problemi GitHub Actions

### ❌ Workflow non parte

**Problema**: Workflow non si attiva su push

**Soluzione**:
```yaml
# Verifica trigger in .github/workflows/main.yml
on:
  push:
    branches: [ main, develop ]  # aggiungi branch corrente
  pull_request:
    branches: [ main ]

# Forza run manuale
# GitHub → Actions → Select workflow → Run workflow
```

### ❌ "Error: Process completed with exit code 1"

**Problema**: Test falliscono in CI

**Soluzione**:
```bash
# 1. Verifica localmente prima
make test-all

# 2. Controlla i log in GitHub Actions
# Actions → Failed workflow → Job → Step

# 3. Debug con tmate (SSH nel runner)
- name: Setup tmate session
  uses: mxschmitt/action-tmate@v3
  if: failure()
```

### ❌ "Service api unhealthy"

**Problema**: API container non parte in GitHub Actions

**Soluzione**:
```yaml
# Aumenta health check retries
services:
  api:
    image: ghcr.io/${{ github.repository }}:latest
    options: >-
      --health-cmd "wget --spider http://localhost:8080/health"
      --health-interval 10s
      --health-timeout 5s
      --health-retries 10  # aumenta da 5 a 10
      --health-start-period 30s  # aggiungi start period
```

### ❌ "Image not found" in GitHub Actions

**Problema**: Immagine Docker non disponibile

**Soluzione**:
```yaml
# Assicurati che il build job sia completato
jobs:
  test:
    needs: build  # aggiungi questa riga
    services:
      api:
        image: ${{ needs.build.outputs.image }}  # usa output del build
```

---

## Problemi Rete/Connessione

### ❌ "connection timeout" generale

**Soluzione**:
```bash
# 1. Verifica che l'API risponda
curl -v http://localhost:8080/health

# 2. Verifica firewall
sudo ufw status  # Linux
# Windows: Firewall settings

# 3. Verifica DNS (se usi hostname)
ping localhost
ping 127.0.0.1

# 4. Usa IP invece di localhost
curl http://127.0.0.1:8080/health
```

### ❌ Test passano localmente ma falliscono in Docker

**Problema**: Network isolation in Docker

**Soluzione**:
```yaml
# docker-compose.yml - usa network condiviso
services:
  api:
    networks:
      - test-network

  newman:
    networks:
      - test-network
    # Usa nome del servizio come hostname
    command: run collection.json --env-var baseUrl=http://api:8080

networks:
  test-network:
    driver: bridge
```

---

## 🆘 Debug Generale

### Workflow Debug Completo

```bash
# 1. Verifica ambiente
go version
docker --version
node --version
make --version

# 2. Pulisci tutto
make clean-all
docker system prune -af

# 3. Setup da zero
./setup.sh

# 4. Build
make build

# 5. Test singoli uno alla volta
make test-smoke
make test-newman
make test-k6-smoke
make test-playwright
make test-hurl

# 6. Se un test fallisce, debug con verbose
go test -v ./...
newman run --verbose collection.json
k6 run --verbose load-test.js
npx playwright test --debug
hurl --very-verbose test.hurl
```

### Log Utilities

```bash
# Go logs
go test -v 2>&1 | tee test.log

# Docker logs
docker-compose logs -f

# Sistema logs (Linux)
journalctl -xe

# Network debug
netstat -tulpn
ss -tulpn
lsof -i :8080
```

---

## 📞 Dove Chiedere Aiuto

Se i problemi persistono:

1. **GitHub Issues**: Apri issue nel repository
2. **Stack Overflow**: Tag `go`, `newman`, `k6`, `playwright`, `hurl`
3. **Discord/Slack**: Community dei vari tool
4. **Documentation**:
    - Go: https://go.dev/doc/
    - Newman: https://learning.postman.com/
    - k6: https://community.k6.io/
    - Playwright: https://playwright.dev/docs/
    - Hurl: https://hurl.dev/

---

## ✅ Checklist Pre-Support

Prima di chiedere aiuto, raccogli queste info:

- [ ] Sistema operativo e versione
- [ ] Versioni di Go, Node, Docker
- [ ] Output completo dell'errore
- [ ] Comandi eseguiti per riprodurre
- [ ] File di configurazione rilevanti
- [ ] Log completi