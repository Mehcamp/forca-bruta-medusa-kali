# Ataques de Força Bruta com Medusa e Kali Linux: estudo de caso

Documentação de estudo sobre ataques de força bruta em FTP, em um formulário web (DVWA) e em SMB, usando o Kali Linux e a ferramenta Medusa contra ambientes propositalmente vulneráveis (Metasploitable 2 / DVWA). Feito como desafio do bootcamp de Cybersecurity da DIO.

> **Nota sobre este repositório:** este material é um **estudo teórico e de preparação**, os comandos abaixo são escritos na sintaxe correta do Medusa, mas as saídas descritas são o **resultado esperado**, explicado com base na documentação oficial da ferramenta e no comportamento observado — não capturas de tela de um ataque real. Não há, portanto, pasta `/images` com prints de execução.

## O que é um ataque de força bruta

Um ataque de força bruta tenta, de forma automática, muitas combinações de usuário e senha contra um serviço (FTP, um login web, SMB etc.) até encontrar uma que funcione. Quando o atacante já tem uma lista de usuários prováveis e testa poucas senhas comuns contra todos eles, chama-se **password spraying** — é mais lento para ser bloqueado por mecanismos que travam a conta após X tentativas de um mesmo usuário, porque cada usuário recebe só uma ou duas tentativas.

## Ambiente proposto pelo desafio

| Papel | Máquina | Função |
|---|---|---|
| Atacante | Kali Linux | Tem o Medusa e outras ferramentas de auditoria instaladas |
| Alvo | Metasploitable 2 | Linux propositalmente vulnerável, com serviços como FTP e SMB mal configurados |
| Alvo (web) | DVWA (Damn Vulnerable Web Application) | Aplicação web propositalmente vulnerável, roda sobre um servidor web |

As duas VMs ficam em uma **rede host-only** no VirtualBox: uma rede isolada, visível só entre as VMs e o computador hospedeiro, sem acesso à internet nem exposição para fora da máquina. Isso é importante por segurança: um ataque de força bruta nunca deve ser direcionado a um serviço que não seja seu ou que você não tenha autorização explícita para testar.

## Wordlists usadas neste estudo

Arquivos em `wordlists/`, pensados para serem pequenos e ilustrarem a lógica, não para serem exaustivos:

- `usuarios.txt`: nomes de usuário comuns e prováveis em ambientes de teste, incluindo `msfadmin` (usuário padrão conhecido do Metasploitable 2).
- `senhas.txt`: senhas fracas e previsíveis, incluindo `msfadmin` (a senha padrão correspondente).

Em um ataque real, listas maiores vêm de vazamentos conhecidos (como o `rockyou.txt`, que acompanha o Kali Linux) ou são geradas com regras (datas, variações de nome da empresa, etc.).

## Cenário 1: força bruta em FTP

**Comando:**
```bash
medusa -h <IP_DO_ALVO> -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp
```

**O que cada parte faz:**
- `-h <IP_DO_ALVO>`: endereço IP da máquina-alvo (seria o IP do Metasploitable 2 na rede host-only, descoberto antes com `nmap -sV <faixa_de_ip>`).
- `-U wordlists/usuarios.txt`: lista de usuários a testar.
- `-P wordlists/senhas.txt`: lista de senhas a testar contra cada usuário.
- `-M ftp`: módulo do Medusa que fala o protocolo FTP.

**Resultado esperado:** o Metasploitable 2 tem um serviço FTP (vsftpd) com credenciais padrão fracas. O Medusa testaria cada combinação de usuário e senha e marcaria como `ACCOUNT FOUND` a combinação `msfadmin` / `msfadmin`, que é a credencial real conhecida dessa imagem. Esse é o motivo de ela aparecer nas duas wordlists.

**Validação de acesso (como seria feita):** depois de o Medusa indicar a credencial válida, o próprio Kali conectaria com `ftp <IP_DO_ALVO>` e faria login com esse usuário e senha, confirmando o acesso na prática.

## Cenário 2: automação de tentativas em formulário web (DVWA)

Formulários web não falam um protocolo padrão como o FTP, então o ataque precisa descrever exatamente a requisição HTTP.

**Comando:**
```bash
medusa -h <IP_DO_ALVO> -U wordlists/usuarios.txt -P wordlists/senhas.txt -M http -m DIR:/dvwa/login.php -m FORM:"username=^USER^&password=^PASS^&Login=Login" -m FAIL:"Login failed"
```

**O que cada parte faz:**
- `-M http`: módulo HTTP do Medusa.
- `-m DIR:/dvwa/login.php`: caminho da página de login no servidor.
- `-m FORM:"..."`: o corpo da requisição enviada, com `^USER^` e `^PASS^` substituídos por cada combinação testada.
- `-m FAIL:"Login failed"`: um texto que só aparece na página quando o login falha. É assim que o Medusa sabe diferenciar uma tentativa que deu certo de uma que não deu.

**Por que esse cenário é mais delicado:** o nível de segurança configurado no DVWA muda o resultado. No nível "Low", não há limite de tentativas nem captcha, então o ataque teria sucesso rápido com a senha padrão do DVWA (`admin` / `password`). No nível "Medium" ou "High", o DVWA já simula proteções (atraso entre tentativas, token anti-automação), que tornam esse mesmo ataque mais lento ou inviável — e é justamente essa comparação que mostra, na prática, o valor de cada mitigação.

## Cenário 3: password spraying em SMB com enumeração de usuários

**Passo prévio — enumeração de usuários**, com outra ferramenta do Kali:
```bash
enum4linux -U <IP_DO_ALVO>
```
Esse comando tenta listar os nomes de usuário existentes no servidor SMB, aproveitando configurações permissivas comuns em versões antigas do Samba (presentes no Metasploitable 2). O resultado alimentaria o `usuarios.txt` usado a seguir.

**Ataque com Medusa:**
```bash
medusa -h <IP_DO_ALVO> -U wordlists/usuarios.txt -P wordlists/senhas.txt -M smbnt
```
- `-M smbnt`: módulo do Medusa para autenticação SMB/NetBIOS.

**Por que "password spraying" aqui:** em vez de testar muitas senhas contra um único usuário, testa-se uma lista pequena de senhas contra muitos usuários enumerados. Isso reduz a chance de travar uma conta por excesso de tentativas, e é uma técnica comum porque muita gente reutiliza a mesma senha fraca.

## Mitigações (como se proteger de cada cenário)

| Cenário | Recomendações |
|---|---|
| FTP | Desativar contas e serviços padrão não usados; exigir senhas fortes; preferir SFTP/FTPS no lugar de FTP puro, que trafega credenciais sem criptografia; bloquear a conta após algumas tentativas falhas. |
| Formulário web (DVWA) | Limitar tentativas de login por IP/usuário (rate limiting); usar captcha após algumas falhas; atrasar a resposta em logins incorretos; exigir autenticação multifator (MFA); nunca expor mensagens de erro que digam qual campo (usuário ou senha) está errado. |
| SMB | Desativar enumeração anônima de usuários; manter o Samba atualizado; segmentar a rede para que SMB não fique acessível de fora do necessário; aplicar política de bloqueio de conta e de senha forte; monitorar tentativas de autenticação repetidas (SIEM/logs). |
| Geral | Autenticação multifator sempre que possível; gestor de senhas para evitar senhas fracas ou reaproveitadas; monitoramento e alertas para picos de tentativas de login. |

## O que eu aprendi

- Força bruta não é um único ataque, e sim uma técnica que muda de forma conforme o protocolo: em FTP e SMB o Medusa fala diretamente o protocolo, enquanto em um formulário web é preciso descrever a requisição HTTP inteira, incluindo um texto que identifique falha de login.
- Password spraying é uma variação pensada para não disparar bloqueios de conta, testando poucas senhas contra muitos usuários em vez de muitas senhas contra um usuário só.
- Boa parte da eficácia desses ataques, nos ambientes propositalmente vulneráveis do curso, vem de senhas padrão conhecidas e da falta de limite de tentativas — que são exatamente os dois pontos mais citados nas mitigações.
- Testar isso só é ético e legal em ambiente próprio e isolado (como a rede host-only entre Kali e Metasploitable 2) ou com autorização explícita por escrito.

## Referências

- Documentação oficial do Medusa: http://www.foofus.net/jmk/medusa/medusa.html
- Kali Linux: https://www.kali.org/
- DVWA: http://www.dvwa.co.uk/
- Nmap: https://nmap.org/book/
