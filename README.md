# 🏛️ Monitor Proposições CMT-PI — Câmara Municipal de Teresina

Monitora automaticamente o portal da Câmara Municipal de Teresina-PI e envia email quando há proposições novas nos tipos selecionados. Roda **4x por dia** via GitHub Actions (8h, 12h, 17h e 21h, horário de Brasília).

> **Atenção:** este monitor roda em **self-hosted runner** (VPS própria), não nos servidores do GitHub. O portal usa o sistema Agape, que bloqueia IPs de datacenter como os da AWS/GitHub.

---

## Tipos monitorados

- Projeto de Lei Ordinária
- Projeto de Lei Complementar
- Projeto de Decreto Legislativo
- Projeto de Resolução
- Projeto de Resolução Administrativa
- Proposta de Emenda a Lei Orgânica
- Veto
- Indicação
- Requerimento

> **Atenção sobre Requerimento:** este é um tipo genérico em Teresina. No primeiro run pode vir com volume alto. Se necessário, remova `'requerimento'` do array `TIPOS_MONITORADOS` no `monitor.js` e faça commit.

---

## Como funciona

1. GitHub Actions dispara o job nos horários configurados, usando o runner instalado na VPS
2. O script acessa `teresina.camarasempapel.com.br/spl/consulta-producao.aspx`
3. Navega pelas páginas via postback ASP.NET UpdatePanel (sistema Agape)
4. Filtra pelos tipos monitorados e compara com os IDs já registrados em `estado.json`
5. Se há proposições novas → envia email organizado por tipo
6. Salva o estado atualizado no repositório

---

## Estrutura do repositório

```
monitor-proposicoes-cmt-pi/
├── monitor.js
├── package.json
├── estado.json
├── README.md
└── .github/
    └── workflows/
        └── monitor.yml
```

---

## Setup

### PARTE 1 — Gmail (App Password)

> Se já tem App Password configurado para outros monitores, pode reutilizar a mesma senha.

1. Acesse [myaccount.google.com/security](https://myaccount.google.com/security)
2. Ative **Verificação em duas etapas** se ainda não estiver ativa
3. Busque **"Senhas de app"** → Criar → nome: `monitor-cmt-pi`
4. Copie a senha de **16 letras** (aparece só uma vez)

---

### PARTE 2 — Criar repositório no GitHub

1. [github.com](https://github.com) → **+ → New repository**
2. Nome: `monitor-proposicoes-cmt-pi` | Visibilidade: **Private**
3. Clique em **Create repository**

---

### PARTE 3 — Upload dos arquivos

1. Na página do repositório: **"uploading an existing file"**
2. Faça upload de: `monitor.js`, `package.json`, `README.md`
3. Commit changes

4. Para o workflow: **Add file → Create new file**
5. Nome: `.github/workflows/monitor.yml`
6. Cole o conteúdo do `monitor.yml` e commit

---

### PARTE 4 — Secrets

**Settings → Secrets and variables → Actions → New repository secret**

| Name | Valor |
|------|-------|
| `EMAIL_REMETENTE` | flavia@monitorlegislativo.com.br |
| `EMAIL_SENHA` | App Password de 16 letras (sem espaços) |
| `EMAIL_DESTINO` | tramitacao@monitorlegislativo.com.br |

---

### PARTE 5 — Registrar o self-hosted runner

> Se já existe um runner registrado na VPS para outro monitor, **não instale um novo runner**. Basta registrar este repositório em uma nova pasta com `config.sh`.

1. No repositório: **Settings → Actions → Runners → New self-hosted runner**
2. Selecione: **Linux → x64**
3. Execute os comandos na VPS:

```bash
mkdir actions-runner-cmt-pi && cd actions-runner-cmt-pi

# Baixar e extrair (use o comando exato gerado pelo GitHub)
curl -o actions-runner-linux-x64-X.X.X.tar.gz -L https://github.com/actions/runner/releases/...
tar xzf ./actions-runner-linux-x64-X.X.X.tar.gz

# Configurar com o token gerado pelo GitHub
./config.sh --url https://github.com/SEU_USUARIO/monitor-proposicoes-cmt-pi --token TOKEN_GERADO

# Instalar como serviço
sudo ./svc.sh install
sudo ./svc.sh start
```

4. O runner deve aparecer com status **Idle** (verde) no GitHub

---

### PARTE 6 — Testar

1. **Actions → Monitor Proposições CMT-PI → Run workflow → Run workflow**
2. Aguarde ~30-60 segundos
3. Verde = funcionou

---

## Resetar o estado

1. No repositório, clique em `estado.json` → ícone de edição (lápis)
2. Substitua todo o conteúdo por:
```json
{"proposicoes_vistas":[],"ultima_execucao":""}
```
3. Commit → rode manualmente

---

## Problemas comuns

**Runner aparece como Offline**
```bash
cd actions-runner-cmt-pi
sudo ./svc.sh status
sudo ./svc.sh start
```

**`fetch failed` ou erro de conexão**
```bash
curl -I https://teresina.camarasempapel.com.br/spl/consulta-producao.aspx
```

**`0 proposições novas` mas deveria ter**
→ Verifique se o nome do tipo bate exatamente com o array `TIPOS_MONITORADOS` no `monitor.js`.

**Erro de autenticação Gmail**
→ Verifique se `EMAIL_SENHA` foi colado sem espaços.
