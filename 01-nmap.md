# Nmap: Chuleta de Escaneo
> Descubrimiento de hosts, escaneos TCP/UDP, detección de versiones, NSE, evasión básica y un flujo de trabajo probado para HTB/OSCP/eJPT.

Uso exclusivo en laboratorios y entornos con autorización explícita por escrito.

## Flujo recomendado

La idea es ir de lo rápido a lo profundo. No lances `-A` a ciegas contra 65535 puertos: pierdes tiempo y ruido.

1. Descubrir hosts vivos (si tienes un rango).
2. Barrido rápido de todos los puertos TCP.
3. Escaneo profundo solo sobre los puertos abiertos.
4. UDP en los puertos top (es lento, déjalo corriendo en segundo plano).
5. Scripts NSE dirigidos según el servicio.

```bash
# Paso 2: todos los puertos, rápido
sudo nmap -p- --min-rate 5000 -T4 -n -Pn -oA nmap/allports 10.10.10.5

# Extraer los puertos abiertos en formato 22,80,445
ports=$(grep -oP '\d+(?=/open)' nmap/allports.gnmap | paste -sd,)

# Paso 3: profundo solo en esos puertos
sudo nmap -p"$ports" -sC -sV -O -oA nmap/targeted 10.10.10.5
```

Cuándo usarlo: casi siempre como primer contacto con una máquina de HTB o del examen.

## Descubrimiento de hosts

| Flag | Qué hace | Cuándo |
|------|----------|--------|
| `-sn` | Ping scan, sin puertos | Mapear una subred |
| `-PR` | ARP (default en LAN) | Misma red local, muy fiable |
| `-PE` / `-PP` | ICMP echo / timestamp | Redes que bloquean solo echo |
| `-PS22,80,443` | SYN a puertos dados | Hosts que filtran ICMP |
| `-PA80` | ACK a puerto dado | Firewalls sin estado |
| `-Pn` | Asume que el host está vivo | Cuando el ping falla pero sabes que existe |
| `-n` | No resolver DNS | Más rápido |

```bash
# Subred completa, solo lista de vivos
sudo nmap -sn 10.10.10.0/24 -oG - | awk '/Up$/{print $2}' > vivos.txt

# Sin ICMP, usando TCP
sudo nmap -sn -PS22,80,443,445 -PA80 10.10.10.0/24

# Escanear desde lista
nmap -iL vivos.txt -p- --min-rate 3000 -oA nmap/red
```

Nota: sin root, `-sn` usa connect() en lugar de ARP y es menos fiable. En HTB casi siempre usarás `-Pn` porque el ICMP suele estar filtrado.

## Escaneos TCP y UDP

```bash
# SYN scan (default con root): rápido y sigiloso
sudo nmap -sS 10.10.10.5

# Connect scan (sin root, o a través de proxychains)
nmap -sT 10.10.10.5

# Top puertos / rangos específicos
nmap --top-ports 1000 10.10.10.5
nmap -p 1-1024,3306,8000-8100 10.10.10.5

# UDP: siempre acota los puertos
sudo nmap -sU --top-ports 25 -n 10.10.10.5
sudo nmap -sU -p 53,69,123,161,500 -sV 10.10.10.5

# TCP + UDP en una sola corrida
sudo nmap -sS -sU -p T:1-1000,U:53,161 10.10.10.5
```

Puertos UDP que vale la pena mirar siempre: 53 (DNS), 69 (TFTP), 123 (NTP), 161 (SNMP), 500 (IKE), 1900 (UPnP).

Velocidad: `--min-rate 5000` fuerza la tasa mínima de paquetes. En HTB funciona bien; en redes reales o inestables puede perder puertos. Si sospechas falsos negativos, baja a `--min-rate 1000` o usa `-T3`.

## Interpretación de estados

| Estado | Significado | Qué hacer |
|--------|-------------|-----------|
| `open` | Hay un servicio escuchando | Enumerar |
| `closed` | Responde RST, nadie escucha | Host vivo, puerto sin servicio |
| `filtered` | Sin respuesta o ICMP unreachable; firewall | Probar otras técnicas o reintentar |
| `unfiltered` | Alcanzable pero no se sabe si abierto (solo `-sA`) | Combinar con otro scan |
| `open\|filtered` | No se puede distinguir (típico en UDP, `-sF/-sN/-sX`) | Añadir `-sV` o probar con payload |
| `closed\|filtered` | Ambiguo (`-sI` idle scan) | Raro |

Pista rápida: si TODOS los puertos salen `filtered`, probablemente hay un firewall; si todos `closed`, el host está vivo y sin firewall sobre esos puertos. Mira la columna `REASON` con `--reason` (por ejemplo `syn-ack`, `reset`, `no-response`).

## Versiones, OS y scripts NSE

```bash
# Versión (intensidad 0-9, default 7)
nmap -sV --version-intensity 9 -p 80,443 10.10.10.5

# OS detection
sudo nmap -O --osscan-guess 10.10.10.5

# Combo clásico (scripts default + versión)
nmap -sC -sV -p 22,80 10.10.10.5

# Agresivo (incluye traceroute): ruidoso
sudo nmap -A -p 22,80 10.10.10.5

# Un script concreto o una categoría
nmap --script=http-title,http-headers -p 80 10.10.10.5
nmap --script "vuln and safe" -p 445 10.10.10.5
nmap --script-help smb-enum-shares
ls /usr/share/nmap/scripts | grep -i smb
```

NSE útiles por servicio:

```bash
# SMB
nmap --script smb-os-discovery,smb-enum-shares,smb-vuln-ms17-010 -p445 10.10.10.5
# HTTP
nmap --script http-enum,http-methods,http-robots.txt -p80 10.10.10.5
# FTP
nmap --script ftp-anon,ftp-syst -p21 10.10.10.5
# DNS
nmap --script dns-zone-transfer --script-args dns-zone-transfer.domain=dominio.htb -p53 10.10.10.5
# SNMP
nmap -sU --script snmp-info,snmp-processes -p161 10.10.10.5
# SMTP
nmap --script smtp-commands,smtp-enum-users -p25 10.10.10.5
# MySQL / MSSQL
nmap --script mysql-info,mysql-empty-password -p3306 10.10.10.5
nmap --script ms-sql-info,ms-sql-empty-password -p1433 10.10.10.5
# Fuerza bruta (usar con cuidado, solo si el alcance lo permite)
nmap --script ssh-brute --script-args userdb=users.txt,passdb=pass.txt -p22 10.10.10.5
```

Cuándo usarlo: `--script vuln` es útil como pista inicial, pero produce falsos positivos; confirma a mano antes de explotar.

## Evasión básica y ajustes de ruido

En labs casi nunca hace falta, pero conviene conocerlo (y para el examen: entender qué hace cada flag).

```bash
# Fragmentar paquetes
sudo nmap -f 10.10.10.5
sudo nmap --mtu 24 10.10.10.5

# Puerto origen conocido (a veces el firewall deja pasar el 53 o 80)
sudo nmap --source-port 53 10.10.10.5

# Señuelos (el tuyo se mezcla entre ellos)
sudo nmap -D RND:5 10.10.10.5

# Padding de datos y MAC falsa
sudo nmap --data-length 25 --spoof-mac 0 10.10.10.5

# Más lento y espaciado
nmap -T2 --scan-delay 1s --max-retries 1 10.10.10.5

# ACK scan para mapear reglas de firewall (no detecta abiertos)
sudo nmap -sA -p 1-1000 10.10.10.5
```

Plantilla de tiempos: `-T0` y `-T1` (IDS evasion, muy lento), `-T3` (default), `-T4` (recomendado en LAN/HTB), `-T5` (puede perder resultados).

## Salida, reanudar y utilidades

```bash
# Formatos: normal (-oN), grepable (-oG), XML (-oX), todos (-oA prefijo)
nmap -sC -sV -oA nmap/full 10.10.10.5

# Convertir XML a HTML
xsltproc nmap/full.xml -o nmap/full.html

# Reanudar un scan interrumpido (solo -oN o -oG)
nmap --resume nmap/full.nmap

# Ver progreso: pulsa 'v' (más verbose) o la barra espaciadora; o usa:
nmap --stats-every 10s -p- 10.10.10.5

# Por si se cae el resolver o hay dominios
echo "10.10.10.5 maquina.htb" | sudo tee -a /etc/hosts
```

Alternativa rápida para barrer puertos y luego pasar a nmap: `rustscan -a 10.10.10.5 -- -sC -sV`. Útil, pero aprende primero el flujo con nmap puro; en el OSCP y en entornos reales te conviene entender cada paso.

## Errores comunes

- Escanear solo los 1000 puertos por defecto y perderte un servicio en un puerto alto (8443, 50000, etc.). Haz siempre `-p-`.
- Olvidar UDP. SNMP, TFTP o IKE sin revisar es una fuente típica de punto de entrada.
- Correr `-sS`, `-sU` u `-O` sin `sudo`: nmap cae silenciosamente a `-sT` o falla.
- Usar `-T5` o `--min-rate` altísimo en redes lentas y confiar en el resultado "cerrado". Repite los puertos sospechosos.
- Dejar el DNS activo: sin `-n`, nmap se queda esperando resolución inversa.
- Creer que `filtered` es lo mismo que `closed`. Filtered significa que algo bloquea; puede haber un servicio detrás.
- Confiar ciegamente en `-sV`/`-O`: son heurísticas. Verifica banners con `nc` o `curl -i`.
- Ejecutar `--script vuln` y tratar cada salida como explotable. Valida manualmente.
- No guardar la salida con `-oA`. Cuando cambies de máquina querrás consultarla otra vez.
- Escanear con `-Pn` omitido y concluir "host caído" porque el ICMP está bloqueado.
