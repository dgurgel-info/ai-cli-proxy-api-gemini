# CLIProxyAPI + Gemini + Hermes no Ubuntu

Tutorial copy-and-paste para instalar o CLIProxyAPI em uma VPS Ubuntu, autenticar uma conta Google/Gemini por OAuth e usar os modelos no Hermes Agent.

> **Estado validado:** Ubuntu 24.04 x86_64, CLIProxyAPI 7.3.11, serviço systemd persistente e Hermes conectado por `http://127.0.0.1:8317/v1`.
>
> **Importante:** este tutorial não publica tokens, senhas, cookies ou arquivos OAuth. Substitua todos os valores marcados com `SEU_...`.

## Índice

1. [O que será instalado](#1-o-que-será-instalado)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Instalar o CLIProxyAPI](#3-instalar-o-cliproxyapi)
4. [Criar a configuração local](#4-criar-a-configuração-local)
5. [Criar o serviço persistente](#5-criar-o-serviço-persistente)
6. [Verificar o serviço](#6-verificar-o-serviço)
7. [Autenticar o Google/Gemini](#7-autenticar-o-googlegemini)
8. [Confirmar os modelos](#8-confirmar-os-modelos)
9. [Configurar o Hermes](#9-configurar-o-hermes)
10. [Testar tudo](#10-testar-tudo)
11. [Comandos de manutenção](#11-comandos-de-manutenção)
12. [Problemas comuns](#12-problemas-comuns)
13. [Segurança](#13-segurança)

---

## 1. O que será instalado

```text
Conta Google/Gemini
        ↓ OAuth
CLIProxyAPI local: 127.0.0.1:8317
        ↓ API compatível com OpenAI
Hermes Agent
```

O serviço fica limitado à própria VPS. O Hermes e o CLIProxyAPI podem conversar localmente sem abrir a porta para a internet.

Componentes:

```text
Binário:       /usr/local/bin/cli-proxy-api
Configuração:  /root/.cli-proxy-api/config.yaml
OAuth:         /root/.cli-proxy-api/
Serviço:       cliproxyapi.service
Porta:         127.0.0.1:8317
```

---

## 2. Pré-requisitos

Execute como `root` ou use `sudo` nos comandos equivalentes:

```bash
id -un
. /etc/os-release && printf '%s %s\n' "$NAME" "$VERSION_ID"
uname -m
command -v curl
command -v systemctl
```

O procedimento foi validado em:

```text
Ubuntu 24.04 x86_64
```

---

## 3. Instalar o CLIProxyAPI

### 3.1 Definir a versão

```bash
mkdir -p /cache/cliproxyapi-install
cd /cache/cliproxyapi-install

VERSION=7.3.11
BASE="https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}"
```

### 3.2 Baixar a release e o checksum

```bash
curl -fL --retry 3 \\
  -o "CLIProxyAPI_${VERSION}_linux_amd64.tar.gz" \\
  "${BASE}/CLIProxyAPI_${VERSION}_linux_amd64.tar.gz"

curl -fL --retry 3 \\
  -o checksums.txt \\
  "${BASE}/checksums.txt"
```

### 3.3 Validar o arquivo baixado

```bash
grep "CLIProxyAPI_${VERSION}_linux_amd64.tar.gz" checksums.txt > checksum.selected
sha256sum -c checksum.selected
```

Resultado esperado:

```text
CLIProxyAPI_7.3.11_linux_amd64.tar.gz: OK
```

Se aparecer `FAILED`, pare e não instale o arquivo.

### 3.4 Extrair e instalar

```bash
rm -rf unpack
mkdir unpack
tar -xzf "CLIProxyAPI_${VERSION}_linux_amd64.tar.gz" -C unpack

install -d -m 0750 /opt/cliproxyapi
install -m 0755 unpack/cli-proxy-api /usr/local/bin/cli-proxy-api
install -m 0644 unpack/LICENSE /opt/cliproxyapi/LICENSE
install -m 0644 unpack/README.md /opt/cliproxyapi/README.md
```

Validar:

```bash
/usr/local/bin/cli-proxy-api --version
```

---

## 4. Criar a configuração local

Crie o diretório:

```bash
install -d -m 0700 /root/.cli-proxy-api
```

Crie o arquivo:

```bash
nano /root/.cli-proxy-api/config.yaml
```

Cole exatamente:

```yaml
host: "127.0.0.1"
port: 8317
auth-dir: "/root/.cli-proxy-api"

api-keys:
  - "TROQUE-ESTA-CHAVE-LOCAL"

remote-management:
  allow-remote: false
  secret-key: ""

debug: false
request-log: false
logging-to-file: true
```

Salve no `nano`:

```text
Ctrl + O
Enter
Ctrl + X
```

Proteja os arquivos:

```bash
chmod 700 /root/.cli-proxy-api
chmod 600 /root/.cli-proxy-api/config.yaml
```

### Escolher uma chave local

Gere uma chave local forte:

```bash
openssl rand -hex 32
```

Substitua `TROQUE-ESTA-CHAVE-LOCAL` no YAML pelo valor gerado. Não publique essa chave no GitHub.

Nos exemplos abaixo, use sempre:

```bash
export CLIPROXY_KEY='SUA_CHAVE_LOCAL'
```

---

## 5. Criar o serviço persistente

Crie a unidade systemd:

```bash
nano /etc/systemd/system/cliproxyapi.service
```

Cole:

```ini
[Unit]
Description=CLIProxyAPI local OpenAI/Gemini/Claude compatible API service
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/.cli-proxy-api
ExecStart=/usr/local/bin/cli-proxy-api -config /root/.cli-proxy-api/config.yaml
Restart=always
RestartSec=5
TimeoutStopSec=20
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ReadWritePaths=/root/.cli-proxy-api

[Install]
WantedBy=multi-user.target
```

Ative imediatamente e no boot:

```bash
systemctl daemon-reload
systemctl enable --now cliproxyapi.service
```

O `enable` faz o serviço iniciar automaticamente depois de reboot. O `Restart=always` faz o systemd tentar recuperá-lo se o processo cair.

---

## 6. Verificar o serviço

```bash
systemctl is-enabled cliproxyapi.service
systemctl is-active cliproxyapi.service
systemctl status cliproxyapi.service --no-pager
```

Resultado esperado:

```text
enabled
active
```

Verificar a porta:

```bash
ss -ltnp | grep 8317
```

Resultado esperado:

```text
127.0.0.1:8317
```

Testar a API:

```bash
export CLIPROXY_KEY='SUA_CHAVE_LOCAL'

curl -sS --max-time 15 \\
  -H "Authorization: Bearer ${CLIPROXY_KEY}" \\
  http://127.0.0.1:8317/v1/models
```

Antes do login, a lista de modelos pode estar vazia. Isso é normal.

---

## 7. Autenticar o Google/Gemini

A versão validada utiliza o fluxo OAuth do Antigravity para expor os modelos Google/Gemini.

### 7.1 Parar o serviço temporariamente

```bash
systemctl stop cliproxyapi.service
```

### 7.2 Criar o túnel SSH no computador local

**Abra um terminal no computador onde está o navegador** (não na VPS) e execute:

```bash
ssh -N -o ExitOnForwardFailure=yes -L 51121:127.0.0.1:51121 root@IP_DA_VPS -p 22
```

Se você usa uma chave SSH:

```bash
ssh -i /caminho/da-chave -N -o ExitOnForwardFailure=yes \
  -L 51121:127.0.0.1:51121 root@IP_DA_VPS -p 22
```

Substitua `IP_DA_VPS` e, se necessário, a porta SSH. **Deixe esse terminal aberto** até o login terminar; com `-N` ele não exibirá um prompt remoto. O túnel pode ser criado antes de o CLIProxyAPI começar a ouvir na porta de callback.

### 7.3 Iniciar o login na VPS

Em **outro terminal, conectado à VPS**, execute:

```bash
/usr/local/bin/cli-proxy-api \
  -config /root/.cli-proxy-api/config.yaml \
  -antigravity-login \
  -no-browser
```

O programa exibirá uma URL de autorização Google e aguardará o callback. O fluxo validado usa a porta `51121`. **Se ele informar outra porta, pare o túnel com `Ctrl+C` e abra-o novamente usando a porta informada nas duas posições de `-L` antes de abrir a URL.** Não feche o terminal do login enquanto ele estiver aguardando.

### 7.4 Autorizar no navegador

1. Copie a URL exibida pelo CLIProxyAPI;
2. Abra-a no navegador local;
3. Escolha a conta Google;
4. Autorize o acesso;
5. Aguarde a mensagem de autenticação concluída.

Nunca envie o callback, token ou arquivo OAuth por mensagem.

### 7.5 Iniciar novamente o serviço

Depois que aparecer a confirmação de sucesso:

```bash
systemctl start cliproxyapi.service
systemctl is-active cliproxyapi.service
```

O arquivo OAuth será salvo dentro de:

```text
/root/.cli-proxy-api/
```

---

## 8. Confirmar os modelos

```bash
export CLIPROXY_KEY='SUA_CHAVE_LOCAL'

curl -sS --max-time 20 \\
  -H "Authorization: Bearer ${CLIPROXY_KEY}" \\
  http://127.0.0.1:8317/v1/models
```

Na validação realizada, apareceram modelos como:

```text
gemini-3.1-flash-lite
gemini-3.5-flash-lite
gemini-3.6-flash-high
gemini-3.7-flash-high
gemini-3.8-flash-high
gemini-3.1-pro-low
gemini-3-flash
gemini-pro-agent
```

Use o nome exatamente como retornado pela sua própria VPS. Os modelos disponíveis podem mudar.

### Testar uma conversa diretamente

Substitua `NOME_DO_MODELO`:

```bash
curl -sS http://127.0.0.1:8317/v1/chat/completions \\
  -H "Authorization: Bearer ${CLIPROXY_KEY}" \\
  -H "Content-Type: application/json" \\
  -d '{
    "model": "NOME_DO_MODELO",
    "messages": [
      {
        "role": "user",
        "content": "Responda apenas: CLIProxyAPI funcionando."
      }
    ],
    "stream": false
  }'
```

Só configure o Hermes depois que esse teste retornar resposta válida.

---

## 9. Configurar o Hermes

Faça backup da configuração atual:

```bash
cp ~/.hermes/config.yaml \\
  ~/.hermes/config.yaml.backup-before-cliproxyapi
```

Configure o provedor customizado:

```bash
hermes config set model.provider custom
hermes config set model.default NOME_DO_MODELO
hermes config set model.base_url http://127.0.0.1:8317/v1
hermes config set model.api_key "$CLIPROXY_KEY"
```

Exemplo com o modelo usado na validação:

```bash
hermes config set model.provider custom
hermes config set model.default gemini-3.1-pro-low
hermes config set model.base_url http://127.0.0.1:8317/v1
hermes config set model.api_key "$CLIPROXY_KEY"
```

O `base_url` deve terminar em `/v1`. Não use `/v1/chat/completions` no campo `base_url`.

Validar:

```bash
hermes config check
hermes config get model.provider
hermes config get model.default
hermes config get model.base_url
```

---

## 10. Testar tudo

### 10.1 Testar o Hermes em uma consulta

```bash
hermes chat -q "Responda apenas: Hermes conectado ao Gemini via CLIProxyAPI."
```

Resposta esperada:

```text
Hermes conectado ao Gemini via CLIProxyAPI.
```

### 10.2 Abrir o Hermes interativo

```bash
hermes
```

### 10.3 Trocar o modelo

```bash
hermes config set model.default NOME_EXATO_DO_MODELO
```

### 10.4 Verificar o serviço depois do teste

```bash
systemctl is-enabled cliproxyapi.service
systemctl is-active cliproxyapi.service
```

---

## 11. Comandos de manutenção

Reiniciar:

```bash
systemctl restart cliproxyapi.service
```

Parar:

```bash
systemctl stop cliproxyapi.service
```

Iniciar:

```bash
systemctl start cliproxyapi.service
```

Logs:

```bash
journalctl -u cliproxyapi.service -n 100 --no-pager
```

Acompanhar logs em tempo real:

```bash
journalctl -u cliproxyapi.service -f
```

Ver a versão:

```bash
/usr/local/bin/cli-proxy-api --version
```

Ver arquivos OAuth sem exibir conteúdo:

```bash
find /root/.cli-proxy-api -maxdepth 1 -type f -printf '%f\n' | sort
```

---

## 12. Problemas comuns

### `Connection refused`

```bash
systemctl status cliproxyapi.service --no-pager
journalctl -u cliproxyapi.service -n 100 --no-pager
ss -ltnp | grep 8317
```

### `401 Invalid API key`

A chave do Hermes precisa ser igual à chave em `api-keys` do CLIProxyAPI:

```bash
grep -n 'api-keys' -A2 /root/.cli-proxy-api/config.yaml
hermes config get model.base_url
```

Não publique a chave no GitHub.

### A lista de modelos está vazia

Confirme se o arquivo OAuth existe:

```bash
find /root/.cli-proxy-api -maxdepth 1 -type f -printf '%f\n' | sort
```

Depois reinicie:

```bash
systemctl restart cliproxyapi.service
```

### OAuth não retorna para o navegador

Confirme que o túnel SSH está aberto:

```bash
ssh -L 51121:127.0.0.1:51121 root@IP_DA_VPS -p 22
```

Use a porta exata exibida pelo comando de login.

### O Hermes está usando o provedor antigo

```bash
hermes config get model.provider
hermes config get model.base_url
hermes config get model.default
```

O resultado esperado é:

```text
custom
http://127.0.0.1:8317/v1
NOME_DO_MODELO
```

---

## 13. Segurança

- Mantenha `host: "127.0.0.1"` quando o Hermes estiver na mesma VPS;
- Não abra a porta `8317` no firewall sem necessidade;
- Não ative `remote-management.allow-remote` sem necessidade;
- Não publique a chave local no GitHub;
- Não publique arquivos de `/root/.cli-proxy-api/`;
- Não publique `~/.hermes/.env`;
- Não compartilhe tokens, cookies ou callbacks OAuth;
- Não use o proxy para terceiros sem revisar os termos do provedor;
- Evite automação pesada, rotação de contas ou exposição pública da API;
- Faça backup somente de arquivos sem credenciais.

Para remover a autorização OAuth local:

```bash
systemctl stop cliproxyapi.service
rm -f /root/.cli-proxy-api/antigravity-*.json
systemctl start cliproxyapi.service
```

---

## Resultado validado

```text
CLIProxyAPI: 7.3.11
Serviço: cliproxyapi.service
Boot: enabled
Estado: active
Bind: 127.0.0.1:8317
OAuth Google/Antigravity: concluído
Modelos: disponíveis após autenticação
Hermes: conectado e testado
```

## 14. Riscos, políticas e fontes para revalidação

Esta seção deve ser lida antes de usar o CLIProxyAPI com uma conta Google importante.

### Relato público relacionado

- Issue 1814 do CLIProxyAPI: https://github.com/router-for-me/CLIProxyAPI/issues/1814

Essa issue registra um relato de suspensão de conta associado ao uso do CLIProxyAPI com login Google/Antigravity. É um relato individual: não comprova, sozinho, que o CLIProxyAPI foi a causa definitiva da suspensão e não permite calcular uma probabilidade de ban.

### Regras e documentos oficiais

- Termos de IA generativa do Google: https://policies.google.com/terms/generative-ai
- Licença do plugin Gemini Code Assist: https://developers.google.com/gemini-code-assist/resources/plugin-license

Esses documentos são a referência oficial para uso, restrições, abuso, interferência, redistribuição e disponibilização dos serviços. Eles podem ser alterados pelo Google; reavalie os links antes de usar o serviço em produção ou comercialmente.

### Como o risco pode aparecer

O Google não publica o classificador completo usado para detectar abuso. Em termos práticos, o provedor pode correlacionar sinais como:

- cliente OAuth e aplicação usados na autorização;
- padrão, volume e frequência das requisições;
- IP, localização e mudanças frequentes de rede;
- muitas sessões ou chamadas simultâneas;
- automação contínua fora do cliente oficial;
- rotação de várias contas;
- tentativas de contornar limites, proteções ou controles de acesso;
- violação das políticas de conteúdo ou uso.

Isso é uma avaliação de risco, não uma afirmação de que todos esses sinais são usados em todos os casos.

### Riscos específicos desta configuração

- O OAuth do Antigravity é transformado pelo CLIProxyAPI em uma API local compatível com outros protocolos.
- O Hermes pode gerar chamadas longas, automatizadas e repetidas.
- A VPS pode produzir um padrão de uso diferente do cliente oficial.
- A porta ou a chave local podem ser expostas acidentalmente.
- O uso por terceiros, revenda ou oferta pública pode criar riscos adicionais de política e segurança.

Não há garantia de que uma conta permanecerá sem restrições ou suspensão.

### Medidas de redução de risco

- Preferir uma conta Google separada para testes, não a conta principal.
- Manter o serviço limitado a `127.0.0.1:8317`.
- Não compartilhar a chave de `api-keys`.
- Não habilitar `remote-management.allow-remote` sem necessidade.
- Não expor o CLIProxyAPI como serviço para terceiros.
- Não usar rotação automática de múltiplas contas.
- Evitar automação pesada e execução contínua sem necessidade.
- Não tentar burlar limites, bloqueios ou verificações do Google.
- Para produção, avaliar a Gemini API oficial com chave própria e faturamento configurado.

### Revogar a autorização local

```bash
systemctl stop cliproxyapi.service
rm -f /root/.cli-proxy-api/antigravity-*.json
systemctl start cliproxyapi.service
```

A remoção local não substitui a revogação na Conta Google. Para revogar completamente, remova também o acesso em **Conta Google → Segurança → Conexões com apps e serviços de terceiros**.
