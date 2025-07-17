# ✅ Configuração de Servidor DNS com BIND no RHEL 10 para OpenShift

Este guia apresenta um passo a passo completo para configurar um servidor DNS local utilizando o BIND no RHEL 10. A configuração é voltada para ambientes de laboratório com OpenShift, permitindo a resolução de nomes internos como `api.ocp.lab.local` e `*.apps.ocp.lab.local`.

---

## 🔧 Etapas de Configuração

### 1. Instale os pacotes necessários

```bash
sudo dnf install bind bind-utils -y
```

---

### 2. Edite o arquivo principal de configuração do BIND

```bash
sudo vim /etc/named.conf
```

Altere ou adicione as diretivas abaixo:

```conf
listen-on port 53 { any; };
allow-query     { any; };
recursion yes;
```

Adicione as zonas DNS ao final do arquivo:

```conf
zone "ocp.lab.local" IN {
    type master;
    file "/var/named/ocp.lab.local.db";
    allow-update { none; };
};

zone "10.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/192.168.10.rev";
    allow-update { none; };
};
```

---

### 3. Crie os arquivos de zona

#### Zona direta (ocp.lab.local):

```bash
sudo vim /var/named/ocp.lab.local.db
```

```dns
$TTL 1D
@       IN SOA  dns.ocp.lab.local. root.ocp.lab.local. (
                            2025071701 ; Serial
                            1D         ; Refresh
                            1H         ; Retry
                            1W         ; Expire
                            3H )       ; Minimum

        IN NS   dns.ocp.lab.local.

dns     IN A    192.168.10.10
api     IN A    192.168.10.100
apps    IN A    192.168.10.200
*.apps  IN A    192.168.10.200
```

#### Zona reversa (192.168.10.rev):

```bash
sudo vim /var/named/192.168.10.rev
```

```dns
$TTL 1D
@       IN SOA  dns.ocp.lab.local. root.ocp.lab.local. (
                            2025071701
                            1D
                            1H
                            1W
                            3H )
        IN NS   dns.ocp.lab.local.

10      IN PTR  dns.ocp.lab.local.
100     IN PTR  api.ocp.lab.local.
200     IN PTR  apps.ocp.lab.local.
```

---

### 4. Ajuste permissões e contexto SELinux

```bash
sudo chown root:named /var/named/*.db
sudo restorecon -v /var/named/*.db
```

---

### 5. Inicie e habilite o serviço BIND

```bash
sudo systemctl enable --now named
systemctl status named
```

---

### 6. Libere a porta DNS no firewall

```bash
sudo firewall-cmd --add-service=dns --permanent
sudo firewall-cmd --reload
```

---

### 7. Teste a resolução DNS

Consulta direta:

```bash
dig @localhost api.ocp.lab.local
```

Consulta reversa:

```bash
dig -x 192.168.10.100 @localhost
```

---

### 8. Configure os clientes para usar este DNS

Edite `/etc/resolv.conf`:

```conf
nameserver 192.168.10.10
search ocp.lab.local
```

Para tornar permanente com NetworkManager:

```bash
nmcli con mod <nome-da-conexao> ipv4.dns "192.168.10.10"
nmcli con mod <nome-da-conexao> ipv4.ignore-auto-dns yes
nmcli con up <nome-da-conexao>
```

---

## 📌 Considerações finais

- Os dados inseridos  nos arquivos foram gerados de acordo com o ambiente de testes, reuna todas informações necessárias para o preenchimento dos arquivos de zonas.
- Este servidor DNS local é ideal para laboratórios OpenShift em ambientes fechados.
- Use `*.apps.ocp.lab.local` caso ainda não tenha definido um domínio para sua aplicação, servidor, cluster, etc... 
- Verifique logs com:
- Neste exemplo foi realizado a configuração de DNS para um Cluster Openshift.

```bash
journalctl -u named
```

- Valide os arquivos de zona:

```bash
named-checkzone ocp.lab.local /var/named/ocp.lab.local.db
named-checkzone 10.168.192.in-addr.arpa /var/named/192.168.10.rev
```