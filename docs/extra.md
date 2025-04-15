## 1. Rede Interna

### 1.1 Configuração de Rede nas VMs

Alteramos o **Adaptador 2** das VMs Backend e Database para `Rede Interna` e nomeamos a rede como llw

![Rede Interna](images/redeinterna1.png)

Para a VM Frontend incluímos um terceiro adaptador como `rede interna` e também nomeamos como llw

![Rede Interna](images/redeinterna2.png)

### 1.2 Configuração dos IPs 

O IP escolhido para a rede interna foi 10.10.10.0/29, então foi necessário configurar o arquivo `interfaces` e o arquivo de `hosts` das VMs. 

Em cada uma delas usamos o comando de editar o arquivo 

```bash
vim /etc/network/interfaces 
```

VM Frontend 

```bash
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.168.0.3
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 10.10.10.3
    netmask 255.255.255.248
    network 10.10.10.0 
```

VM Backend 

```bash
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 10.10.10.1
    netmask 255.255.255.248
    network 10.10.10.0
```

VM Database 

```bash
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 10.10.10.2
    netmask 255.255.255.248
    network 10.10.10.0
```

E para os hosts usamos o comando de edição do arquivo:

```bash
vim /etc/hosts 
```

Resolvemos os nomes para os IPs da rede interna:

```bash
10.10.10.1 backend.llw
10.10.10.2 database.llw
10.10.10.3 frontend.llw
```

### 2.4 Configuração do nginx 

Editamos o arquivo de configuração do nginx para configurar o **proxy reverso**:

```bash
vim /etc/nginx/http.d/default.conf
```

O arquivo foi editado com essas informações:

```bash 
server {
    listen 80;
    listen 8080;
    listen [::]:80;

    server_name backend.llw;

    access_log /var/log/nginx/frontend_access.log;
    error_log /var/log/nginx/frontend_error.log;

    location / {
        root /opt/frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend.llw:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass_request_headers on;
        proxy_pass_request_body on;
    }

    location = /404.html {
        internal;
    }
}
```


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