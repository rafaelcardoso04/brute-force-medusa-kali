# Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux

Projeto prático desenvolvido para o lab da [DIO](https://www.dio.me) — *"Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux"*, parte do Bootcamp de Cibersegurança (DIO).

## 🎯 Objetivo

Implementar, documentar e compartilhar um estudo prático sobre ataques de força bruta (*brute force*) a credenciais, utilizando o **Kali Linux** e a ferramenta **Medusa**, em ambientes propositalmente vulneráveis (**Metasploitable 2** e **DVWA**), com o objetivo de:

- Entender como esse tipo de ataque funciona na prática;
- Simular cenários de autenticação em diferentes serviços (FTP, SMB, formulário web);
- Propor e documentar medidas de mitigação e prevenção.

> ⚠️ **Aviso de ética e legalidade**: técnicas de brute force só devem ser aplicadas em ambientes de laboratório controlados, de sua propriedade ou com autorização explícita. Usar essas técnicas contra sistemas de terceiros sem permissão é crime (Lei 12.737/2012 no Brasil). Todo o conteúdo aqui é estritamente educacional.

---

## 🧠 Conceitos estudados

### O que é Brute Force?

Ataque de força bruta é uma técnica em que o atacante tenta, de forma sistemática e automatizada, todas as combinações possíveis de usuário/senha (ou apenas senha) até encontrar a credencial correta. Pode ser:

- **Brute force clássico**: testa combinações exaustivas (ou uma wordlist grande) contra **um único usuário**.
- **Dictionary attack**: usa uma wordlist de senhas comuns/vazadas em vez de combinações aleatórias — muito mais eficiente, pois pessoas reutilizam senhas previsíveis.

### Password Spraying

Em vez de testar muitas senhas contra um usuário (o que pode gerar bloqueio de conta), o **password spraying** testa **poucas senhas muito comuns** (ex: `123456`, `Senha@123`) contra **muitos usuários diferentes**. Isso evita gatilhos de lockout por tentativas excessivas em uma única conta e é muito usado contra ambientes corporativos (Active Directory, VPNs, webmail).

### Credential Stuffing

Utiliza pares de usuário/senha **vazados em outros serviços** (de vazamentos de dados conhecidos) e os testa em massa contra o sistema alvo, explorando o fato de que muitas pessoas reutilizam a mesma senha em vários lugares.

---

## 🛠️ Ambiente de laboratório (proposto)

| Componente | Função |
|---|---|
| Kali Linux | Máquina atacante, com o Medusa instalado |
| Metasploitable 2 | VM alvo intencionalmente vulnerável (serviços FTP, SMB, etc.) |
| DVWA (Damn Vulnerable Web Application) | Aplicação web vulnerável para teste de brute force em formulário de login |
| VirtualBox | Virtualização das duas VMs, em rede interna (*host-only*), isolada da internet |

A rede **host-only** garante que o tráfego de ataque fique isolado, sem risco de atingir redes externas.

---

## 🧰 Sobre o Medusa

O **Medusa** é uma ferramenta de força bruta paralela (multi-thread) para testes de login em diversos serviços de rede: FTP, SSH, HTTP, SMB, Telnet, MySQL, entre outros.

### Sintaxe básica

```bash
medusa -h <alvo> -u <usuario> -P <wordlist_senhas> -M <modulo>
```

Principais parâmetros:

| Parâmetro | Significado |
|---|---|
| `-h` | Host/IP alvo |
| `-u` | Usuário único |
| `-U` | Arquivo com lista de usuários |
| `-p` | Senha única |
| `-P` | Arquivo com lista de senhas (wordlist) |
| `-M` | Módulo/serviço (ex: `ftp`, `smbnt`, `http`) |
| `-t` | Número de threads paralelas |
| `-f` | Para no primeiro sucesso encontrado |

### Cenário 1 — Força bruta em FTP

```bash
medusa -h 192.168.56.102 -u msfadmin -P wordlists/senhas.txt -M ftp -t 4 -f
```

- `-h 192.168.56.102`: IP da VM Metasploitable2 na rede host-only
- `-u msfadmin`: usuário conhecido/testado
- `-P wordlists/senhas.txt`: wordlist de senhas
- `-M ftp`: módulo de ataque ao serviço FTP
- `-f`: interrompe assim que encontrar a senha correta

### Cenário 2 — Password spraying em SMB com enumeração de usuários

Primeiro, enumeram-se usuários válidos do serviço SMB (ex: com `enum4linux` ou `smbclient`), gerando a wordlists/usuarios.txt. Depois:

```bash
medusa -h 192.168.56.102 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M smbnt -t 2
```

> Aqui o objetivo é simular o spraying: poucas tentativas de senha por usuário, para múltiplos usuários, reduzindo o risco de lockout.

### Cenário 3 — Força bruta em formulário web (DVWA)

Para formulários web, o Medusa usa o módulo `http` indicando os campos do formulário e a mensagem de erro de login inválido:

```bash
medusa -h 192.168.56.102 -u admin -P wordlists/senhas.txt \
  -M http -m DIR:/dvwa/login.php -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
  -m DENY_SIGNAL:"Login failed"
```

- `DIR`: caminho do formulário de login
- `FORM`: campos do formulário, com `^USER^` e `^PASS^` sendo substituídos pelas credenciais testadas
- `DENY_SIGNAL`: texto que aparece na página quando o login falha (usado pelo Medusa para saber que a tentativa falhou)

---

## 📁 Estrutura deste repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt      # lista de usuários de exemplo
│   └── senhas.txt        # lista de senhas comuns de exemplo
└── images/               # (opcional) capturas de tela do laboratório
```

---

## 🛡️ Recomendações de mitigação

Com base no estudo, as principais defesas contra brute force / password spraying / credential stuffing são:

1. **Política de senhas fortes**: exigir senhas longas, com complexidade, e bloquear as mais comuns (ex: via listas como *Have I Been Pwned Passwords*).
2. **Autenticação multifator (MFA)**: mesmo que a senha seja descoberta, o atacante não consegue autenticar sem o segundo fator.
3. **Rate limiting / lockout controlado**: limitar tentativas por IP e por conta, com atraso progressivo (*exponential backoff*), evitando bloqueios fáceis de explorar (DoS) mas dificultando automação.
4. **Monitoramento e alertas**: ferramentas como **Fail2ban** (Linux) bloqueiam IPs após X tentativas falhas em determinado tempo.
5. **CAPTCHA** em formulários de login expostos à internet.
6. **Não reutilizar senhas** entre serviços diferentes (mitiga credential stuffing).
7. **Desabilitar/restringir serviços desnecessários** (FTP, Telnet) e preferir protocolos com criptografia (SFTP, SSH).
8. **Geolocalização/comportamento anômalo**: alertar logins de locais ou horários incomuns.

---

## 📚 Referências de estudo

- Material do lab "Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux" — DIO
- Documentação oficial do Medusa (`man medusa`)
- Documentação do Metasploitable 2 e DVWA

---

## ✅ Conclusão

Este projeto demonstrou, de forma teórica e com exemplos práticos de comando, como ataques de força bruta funcionam contra diferentes serviços (FTP, SMB, aplicações web), e reforçou a importância de medidas de mitigação em camadas (senha forte + MFA + rate limiting + monitoramento) para reduzir o risco desse tipo de ataque em ambientes reais.

---

*Projeto desenvolvido como parte do Bootcamp de Cibersegurança .*
