> Supuestos:
>
> * **NS de `mdc.es`**: `ns1.mdc.es` (192.0.2.53 / 2001:db8:100::53) y `ns2.mdc.es` (192.0.2.54 / 2001:db8:100::54).
> * **Servicios**: `www.mdc.es` → 192.0.2.10, `mail.mdc.es` → 192.0.2.20 (MX).
> * **Subdominio delegado**: `intranet.mdc.es` con NS propios `ns1.intranet.mdc.es` (192.0.2.155) y `ns2.intranet.mdc.es` (192.0.2.156).
> * **Zonas inversas**: 192.0.2.0/24 y 2001:db8:100::/48 (extracto).

---

# 0) (Didáctico) Extracto de delegación en `.es`

> **No lo gestionas tú en la práctica**, es solo para ver cómo se delega `mdc.es` con *glue*. Si montas un lab “falso .es”, este sería el trozo.

```zone
$TTL 86400
es.  IN SOA  a.nic.es. hostmaster.nic.es. (
      2025092601 3600 900 1209600 86400 )

; Autoridad de .es (ficticio para el lab)
es.  IN NS   a.nic.es.
es.  IN NS   b.nic.es.
a.nic.es. IN A     192.0.2.200
b.nic.es. IN A     192.0.2.201

; Delegación de mdc.es a ns1/ns2.mdc.es con glue
mdc.es.   IN NS   ns1.mdc.es.
mdc.es.   IN NS   ns2.mdc.es.
ns1.mdc.es. IN A      192.0.2.53
ns2.mdc.es. IN A      192.0.2.54
ns1.mdc.es. IN AAAA   2001:db8:100::53
ns2.mdc.es. IN AAAA   2001:db8:100::54
```

---

# 1) Zona autoritativa de `mdc.es` (primaria)

**Fichero**: `/var/named/db.mdc.es`

```zone
$TTL 3600
@   IN SOA  ns1.mdc.es. admin.mdc.es. (
        2025092601 ; serial AAAAMMDDnn
        3600       ; refresh (1h)
        900        ; retry   (15m)
        1209600    ; expire  (14d)
        300 )      ; negative cache TTL (5m)

; Servidores de nombres autoritativos de la zona
    IN  NS   ns1.mdc.es.
    IN  NS   ns2.mdc.es.

; Glue internos (no estrictamente necesarios si ya están en .es,
; pero útiles dentro de la misma zona)
ns1 IN  A     192.0.2.53
ns2 IN  A     192.0.2.54
ns1 IN  AAAA  2001:db8:100::53
ns2 IN  AAAA  2001:db8:100::54

; Registros de servicio
@    IN  MX 10 mail.mdc.es.

www  IN  A     192.0.2.10
www  IN  AAAA  2001:db8:100::10

mail IN  A     192.0.2.20
mail IN  AAAA  2001:db8:100::20

; Alias de ejemplo
web  IN  CNAME www.mdc.es.

; TXT (SPF simplificado a efectos didácticos)
@    IN  TXT   "v=spf1 mx -all"

; --- Delegación del subdominio intranet.mdc.es ---
intranet       IN  NS     ns1.intranet.mdc.es.
intranet       IN  NS     ns2.intranet.mdc.es.

; Glue para los NS del subdominio (porque están DENTRO del subdominio delegado)
ns1.intranet   IN  A      192.0.2.155
ns2.intranet   IN  A      192.0.2.156
```

> **named.conf** (zona primaria):

```conf
zone "mdc.es" IN {
    type master;
    file "/var/named/db.mdc.es";
    allow-transfer { 192.0.2.54; }; // ns2
    also-notify { 192.0.2.54; };
};
```

---

# 2) Zona autoritativa de `intranet.mdc.es` (primaria del subdominio delegado)

**Fichero**: `/var/named/db.intranet.mdc.es`

```zone
$TTL 3600
@   IN SOA  ns1.intranet.mdc.es. admin.mdc.es. (
        2025092601
        3600
        900
        1209600
        300 )

    IN  NS  ns1.intranet.mdc.es.
    IN  NS  ns2.intranet.mdc.es.

; Los NS del subdominio (coinciden con el glue declarado en la zona padre)
ns1 IN A 192.0.2.155
ns2 IN A 192.0.2.156

; Hosts del subdominio
portal  IN  A     192.0.2.160
app     IN  A     192.0.2.161
db      IN  A     192.0.2.162

; Alias
www     IN  CNAME portal.intranet.mdc.es.
```

> **named.conf** (zona primaria del subdominio, en el/los NS de intranet):

```conf
zone "intranet.mdc.es" IN {
    type master;
    file "/var/named/db.intranet.mdc.es";
};
```

---

# 3) Zona secundaria de `mdc.es` (en `ns2.mdc.es`)

> **named.conf** en `ns2`:

```conf
zone "mdc.es" IN {
    type slave;
    masters { 192.0.2.53; };
    file "/var/named/slaves/db.mdc.es";
};
```


---

# 4) Zona inversa IPv4 para 192.0.2.0/24

**Fichero**: `/var/named/db.2.0.192.in-addr.arpa`

```zone
$TTL 3600
@   IN SOA  ns1.mdc.es. admin.mdc.es. (
        2025092601
        3600
        900
        1209600
        300 )

    IN  NS   ns1.mdc.es.
    IN  NS   ns2.mdc.es.

; PTR de los principales hosts
10   IN PTR  www.mdc.es.
20   IN PTR  mail.mdc.es.
53   IN PTR  ns1.mdc.es.
54   IN PTR  ns2.mdc.es.

; Subdominio intranet (PTR de sus NS y servicios)
155  IN PTR  ns1.intranet.mdc.es.
156  IN PTR  ns2.intranet.mdc.es.
160  IN PTR  portal.intranet.mdc.es.
161  IN PTR  app.intranet.mdc.es.
162  IN PTR  db.intranet.mdc.es.
```

> **named.conf**:

```conf
zone "2.0.192.in-addr.arpa" IN {
    type master;
    file "/var/named/db.2.0.192.in-addr.arpa";
    allow-transfer { 192.0.2.54; };
    also-notify { 192.0.2.54; };
};
```

---

# 5) Zona inversa IPv6 para 2001:db8:100::/48 (extracto)

**Fichero**: `/var/named/db.0.0.1.0.0.8.b.d.1.0.0.2.ip6.arpa`

> Regla: la zona es el **prefijo IPv6 en nibble-reverse**. Para `/48` se anclan los 12 nibbles iniciales.

```zone
$TTL 3600
@   IN SOA  ns1.mdc.es. admin.mdc.es. (
        2025092601 3600 900 1209600 300 )

    IN  NS   ns1.mdc.es.
    IN  NS   ns2.mdc.es.

; 2001:db8:0100::10  → PTR www.mdc.es.
; Nibbles (reverse): 0.1.0.0.0.8.b.d.1.0.0.2.ip6.arpa.
; Aquí solo muestro el último tramo por brevedad:

; ::10  → ... 0.0.0.0.0.0.0.0.0.0.0.1.0
0.1.0.0.0.0.0.0.0.0.0.0 IN PTR www.mdc.es.

; ::20  → ... 0.0.0.0.0.0.0.0.0.0.0.2.0
0.2.0.0.0.0.0.0.0.0.0.0 IN PTR mail.mdc.es.

; ::53 y ::54 (NS)
0.5.3.0.0.0.0.0.0.0.0.0 IN PTR ns1.mdc.es.
0.5.4.0.0.0.0.0.0.0.0.0 IN PTR ns2.mdc.es.
```

> **named.conf**:

```conf
zone "0.0.1.0.0.8.b.d.1.0.0.2.ip6.arpa" IN {
    type master;
    file "/var/named/db.0.0.1.0.0.8.b.d.1.0.0.2.ip6.arpa";
};
```

---

## Checklist rápido de coherencia

* Delegación `.es` → `mdc.es` con **NS** y **glue** (A/AAAA de `ns1/ns2.mdc.es`).
* `mdc.es`: **SOA**, **NS**, **A/AAAA** de NS, `www`, `mail`, **MX**, **TXT** (SPF).
* Delegación de `intranet.mdc.es` con **NS** y **glue** para `ns1/ns2.intranet.mdc.es`.
* `intranet.mdc.es`: **SOA**, **NS**, hosts internos y alias.
* Zonas inversas IPv4/IPv6 con **PTR** consistentes.
* `serial` incrementa cuando edites.
* `allow-transfer` solo hacia tus secundarios.

---
