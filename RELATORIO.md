# Relatorio de evidencias - Trabalho Pratico Integrador 01-06

Nome: Jorge Fernandes   Data: 24/11/2026
URL do repositorio Git: https://github.com/rafafernandes888/tp-integrador

## E01 - Ambiente (1 valor)
Comandos e versões observadas:

```powershell
docker --version
docker compose version
git --version
docker info
```

Comecei por ver se tinha tudo instalado. Deu Docker 29.9.2, Docker Compose v5.6.1 e Git 2.56.1. No `docker info` apareceu a parte do `Server` com `Server Version: 29.9.2` e `OSType: linux`, o que quer dizer que o daemon do Docker está a responder e a correr containers Linux.

Depois vi o que vinha no kit. A API está no `app.py`, os testes no `tests/test_app.py`, o Compose no `compose.yaml` (o `incidente.yaml` fica para o exercício do incidente) e a pipeline no `Jenkinsfile`.

**O ambiente está pronto e o Docker responde, por isso posso avançar.**

## E02 - Dockerfile e container (3 valores)
Tag da imagem; docker ps; HTTP /health; utilizador efetivo do container:

Completei os cinco TODO do `Dockerfile`. Primeiro copio o `requirements.txt` e instalo as dependências com `pip --no-cache-dir`, e só depois copio o `app.py`. Assim, se mudar o código, o Docker aproveita a cache e não volta a instalar tudo. A seguir criei um utilizador sem privilégios (`appuser`), criei a pasta `/data` com ele como dono, para conseguir escrever lá, e meti `USER appuser`. No `.dockerignore` deixei de fora o `.git`, o `.venv`, o `__pycache__`, o `.pytest_cache`, os `*.pyc`, o `.env`, os `*.log` e a pasta `evidencias/`.

```powershell
docker build -t natixis-support:1.0 .
docker run -d --name tp-api -p 127.0.0.1:18080:8000 natixis-support:1.0
docker ps
curl.exe -i http://127.0.0.1:18080/health
docker logs --tail 15 tp-api
docker exec tp-api id
docker image ls natixis-support
docker port tp-api
docker rm -f tp-api
```

A imagem `natixis-support:1.0` foi construída sem erros (ID `2b3fa3807f1a`). O container ficou `Up` com a porta `127.0.0.1:18080->8000/tcp`, ou seja, a 18080 do meu PC vai dar à 8000 do container, onde o uvicorn está a escutar. O `/health` respondeu:

```
HTTP/1.1 200 OK
{"status":"ok","service":"support-api","version":"1.0"}
```

Nos logs aparece `Uvicorn running on http://0.0.0.0:8000` e o meu pedido `"GET /health HTTP/1.1" 200 OK`. O `id` deu `uid=999(appuser) gid=999(appgroup)`, por isso o container não está a correr como root. No fim removi o container com `docker rm -f`, mas a imagem continua lá.

**A imagem funciona, a API responde com 200 e `status="ok"` e corre com um utilizador sem privilégios, como era pedido.**

## E03 - Compose, DNS, healthcheck e volume (4 valores)
Resultado de docker compose config; health; teste checker -> api; contador antes/depois da recriacao:

No `compose.yaml` acrescentei ao serviço `api` um healthcheck que usa Python/urllib para ir a `http://127.0.0.1:8000/health` de 5 em 5 segundos. No `checker` meti `depends_on` com `condition: service_healthy`, para ele só arrancar quando a API estiver mesmo saudável.

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose --profile diagnostico run --rm checker -fsS http://api:8000/health
curl.exe http://127.0.0.1:18080/visits
curl.exe -X POST http://127.0.0.1:18080/visits
curl.exe http://127.0.0.1:18080/visits
docker compose ps -q api
docker compose up -d --force-recreate
docker compose ps
docker compose ps -q api
curl.exe http://127.0.0.1:18080/visits
docker volume ls
```

O `config --quiet` não mostrou nada, o que quer dizer que o ficheiro está válido. O `up` criou a rede `natixis-integrador_default`, o volume `natixis-integrador_app-data` e o container da API, que ficou `Up 13 seconds (healthy)`.

Para testar o DNS corri o `checker`. Ele ficou à espera (`api-1 Waiting` → `Healthy`) e depois chegou à API pelo nome `api`:

```
{"status":"ok","service":"support-api-integrador","version":"1.0"}
```

No contador, comecei com `{"visits":0}`, fiz um POST e passou a `{"visits":1}`. Depois recriei o container: o ID mudou de `5602bfc87a29` para `ddf0f168a5f5`, ficou outra vez `healthy` e o contador continuava em `{"visits":1}`.

**Os containers falam uns com os outros pelo nome do serviço, graças ao DNS interno do Compose. O contador não se perdeu porque o ficheiro `/data/visitas.txt` está no volume `app-data`, que existe à parte do container e sobrevive à recriação.**

## E04 - Incidente Docker (3 valores)
Sintoma, comandos de investigacao, causa-raiz, alteracao realizada, prova da recuperacao e contador preservado:

Apliquei o `incidente.yaml` sem lhe mexer:

```powershell
docker compose -f compose.yaml -f incidente.yaml up -d --force-recreate
```

**Sintoma:** passado algum tempo o container estava `Up 37 seconds (unhealthy)` e o `curl.exe -i http://127.0.0.1:18080/health` dava `curl: (52) Empty reply from server`. Ou seja, o container estava a correr, mas a API não respondia.

**Comandos de diagnóstico:**

```powershell
docker compose -f compose.yaml -f incidente.yaml ps
docker compose -f compose.yaml -f incidente.yaml logs --tail 30 api
docker compose -f compose.yaml -f incidente.yaml port api 8000
docker inspect --format '{{json .State.Health.Log}}' (docker compose ps -q api)
docker compose -f compose.yaml -f incidente.yaml config
docker compose exec api python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8001/health').read())"
```

Os logs mostravam `Uvicorn running on http://0.0.0.0:8001`, mas o `port` mostrava que o host continua a apontar para a 8000 (`127.0.0.1:18080`). O histórico do healthcheck tinha várias falhas seguidas com `ExitCode 1` e `Connection refused` ao tentar `http://127.0.0.1:8000/health`. No `config` vi que o overlay muda o `command` para `--port "8001"`, mas o mapeamento (`target: 8000`) e o healthcheck continuam na 8000. Para confirmar, fiz um pedido à 8001 dentro do container e a API respondeu `{"status":"ok",...}`.

**Causa:** o `incidente.yaml` põe a API a escutar na porta 8001, enquanto o mapeamento de portas e o healthcheck estão à espera dela na 8000. A aplicação em si está boa; o problema é só a configuração da porta.

**Correção:** voltei à configuração base (sem o `incidente.yaml`) e recriei só o serviço `api`, sem mexer no volume:

```powershell
docker compose up -d --force-recreate api
docker compose ps
curl.exe -i http://127.0.0.1:18080/health
curl.exe http://127.0.0.1:18080/visits
docker compose logs --tail 5 api
```

**Validação:** o container voltou a `Up 15 seconds (healthy)`, o `/health` deu `HTTP/1.1 200 OK` e nos logs voltaram a aparecer os pedidos do healthcheck com `200 OK`. O `/visits` continuava em `{"visits":1}`, por isso os dados não se perderam.

**Um container `Up` não quer dizer que a aplicação esteja a funcionar. Aqui bastou uma porta errada para a API ficar inacessível. Para evitar isto, convém correr o `docker compose config` antes de aplicar uma alteração e confirmar que a porta do `command` bate certo com o mapeamento e com o healthcheck.**

## E05 - Git (3 valores)
Branch, hashes de commits, merge e tag v1.0.0:

(a preencher)

## E06 - Jenkins SUCCESS (4 valores)
URL/numero do build, etapas e identificador da imagem construida:

(a preencher)

## E07 - Jenkins FAILURE e recuperacao (2 valores)
Build FAILURE, etapa que falhou e excerto relevante do log; build SUCCESS apos correcao:

(a preencher)

## Verificacao final
[ ] Dockerfile [ ] compose.yaml [ ] app.py [ ] tests/ [ ] Jenkinsfile
[ ] Historico Git [ ] Tag v1.0.0 [ ] Evidencias E01-E07
