## 1. Revisão da configuração SSH

Começamos revisando se o acesso ao root e por senha estavam bloqueados em todas as VMs

```bash
vim /etc/ssh/sshd_config
```

Garantimos que não será possível conectar-se ao root, nem conectar utilizando senha, apenas chaves RSA.

```bash
PermitRootLogin no
PasswordAuthentication no
```

---

## 2. Firewall

Utilizamos iptables para bloquear todos o acesso às portas inutilizadas.

Instalamos iptables em todas as VMs: 

```bash
apk add iptables
```

Adicionamos as seguintes regras de tráfego:

```bash
iptables -P INPUT DROP
```

>Bloqueia qualquer conexão de entrada (menos as **exceções**).

```bash
iptables -P OUTPUT ACCEPT
```

> Permite que a VM envie dados para fora, importante para podermos acessar serviços externos como a **internet**.

```bash
iptables -P FORWARD DROP
```

>Bloqueia a VM de "atuar como roteador", bloqueando o encaminhamento de pacotes entre interfaces de rede. É "opcional" nessa ocasião, mas é uma boa prática, já que essa VM apenas recebe conexões e não encaminha dados para nenhum outro dispositivo, e como isso não deveria acontecer mesmo podemos bloquear se ocorrer.

--- 

### 2.1 Excessões VM Database

```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

>Essa é crucial. Ela permite respostas a conexões que já foram estabelecidas, se não a VM só enviaria as requisições mas barraria a resposta, Para a VM Database é importante para ela responder às consultas SQL feitas pela VM Back-End.


```bash
iptables -A INPUT -p tcp --dport 3306 -j ACCEPT
```

>Essa vai liberar a porta 3306 que será utilizada para receber conexão ao mariadb da VM Back-End.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

>Essa vai liberar a porta 22 utilizada em nossas VM para conexão SSH, mas a VM Database não recebe mais conexão ssh no nosso cenário entao é **opcional**.

---

### 2.2 Excessões VM Backend

```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

>Essa é crucial. Ela permite respostas a conexões que já foram estabelecidas, se não a VM só enviaria as requisições mas barraria a resposta, Para a vm Back-End é importante para ela se conectar e receber a resposta do Banco de Dados.

```bash
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

>Essa vai liberar a porta 8080 que será utilizada para receber requisições da VM Front-End.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

>Essa vai liberar a porta 22 utilizada em nossas VM para conexão SSH, mas a VM Back-End não recebe mais conexão ssh no nosso cenário entao é **opcional**.

---

### 2.3 Excessões VM Frontend

```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

>Essa é crucial. Ela permite respostas a conexões que já foram estabelecidas, se não a VM só enviaria as requisições mas barraria a resposta, Para a vm Front-End é importante para ela se conectar e receber a resposta da API.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

>Essa vai liberar a porta 22 utilizada em nossas VM para conexão SSH, por onde ela recebe os backups por exemplo.

---