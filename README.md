Markdown
# Postfix + ClamAV-Milter en Debian

Este repositorio documenta la integración de un motor antivirus (ClamAV) en un servidor de correo existente (Postfix/Dovecot) desplegado sobre Debian Linux, utilizando la arquitectura de filtrado de correo (Milter).

## 1. Prerrequisitos y Arquitectura de Hardware
El demonio de ClamAV (`clamd`) carga su base de datos de firmas directamente en memoria. Ejecutarlo en un entorno con recursos limitados provoca errores *Out of Memory* (OOM).
* **Acción:** Se auditaron y ampliaron los recursos del hipervisor (Proxmox), asignando **4 GiB de RAM** a la máquina virtual para garantizar la estabilidad operativa.

## 2. Instalación de la Pila de Seguridad
Se instalaron el motor base, el demonio residente y el intermediario (milter) que servirá de puente con el MTA (Postfix).
```bash
apt update
apt install clamav clamav-daemon clamav-milter
(Captura de la instalación de los paquetes)
```
<img width="847" height="248" alt="image" src="https://github.com/user-attachments/assets/53ccd0ec-fd25-4827-98fe-72b2747dc79e" />

Posteriormente, se forzó una actualización manual de la base de datos de firmas de virus (CVD) deteniendo temporalmente el servicio de actualización automática:

```Bash
systemctl stop clamav-freshclam
freshclam
systemctl start clamav-freshclam
```
<img width="1020" height="489" alt="image" src="https://github.com/user-attachments/assets/730a0096-7a74-4d95-bd1d-e8db9eaf22c2" />

## 3. Configuración del Puente de Red (Milter)
Para evitar problemas de permisos de archivos (chroot), se configuró el Milter para escuchar a través de un socket TCP local en lugar de un socket Unix.

Archivo modificado: /etc/clamav/clamav-milter.conf

<img width="724" height="186" alt="image" src="https://github.com/user-attachments/assets/148ffee2-b090-4da5-800a-86d6352d3601" />

```Plaintext
MilterSocket inet:7357@127.0.0.1
```
<img width="710" height="237" alt="image" src="https://github.com/user-attachments/assets/07fc288d-c5ff-46a3-82b2-81c6e4be9300" />

## 4. Integración en el MTA (Postfix)
Se modificó el núcleo del servidor de correo para desviar el tráfico entrante y saliente hacia el puerto del Milter.

Archivo modificado: /etc/postfix/main.cf

```Plaintext
smtpd_milters = inet:127.0.0.1:7357
non_smtpd_milters = inet:127.0.0.1:7357
milter_default_action = accept
```
Decisión de Diseño: Se configuró milter_default_action = accept. En caso de que el demonio de ClamAV colapse, Postfix continuará procesando correos sin escanear, priorizando la disponibilidad del servicio sobre el bloqueo preventivo.

<img width="864" height="171" alt="image" src="https://github.com/user-attachments/assets/a3b28882-56b9-4dc7-a8a0-ee9ef5bca72e" />

A continuación, se reiniciaron los servicios para aplicar los cambios:

```Bash
systemctl restart clamav-daemon
systemctl restart clamav-milter
systemctl restart postfix
```
<img width="514" height="151" alt="image" src="https://github.com/user-attachments/assets/0b8a52cf-0199-402b-bf5b-4a2505c2e90b" />

Se verificó mediante el uso de sockets que el servicio estaba a la escucha en el puerto correcto (127.0.0.1:7357):

```Bash
ss -tlnp | grep 7357
```
<img width="1255" height="238" alt="image" src="https://github.com/user-attachments/assets/5297cc5e-47b8-4043-8894-a0c8c86ce2ee" />

## 5. Auditoría y Pruebas de Intercepción (Prueba EICAR)
Para validar el entorno, se generó un archivo inofensivo con la firma de prueba estándar internacional EICAR.
<img width="989" height="276" alt="image" src="https://github.com/user-attachments/assets/7818a240-58b7-4b8a-8448-c3e1a8c498bc" />

Se procedió a enviar el archivo como adjunto a través del cliente Thunderbird.
<img width="868" height="751" alt="image" src="https://github.com/user-attachments/assets/7c0e44cc-0580-4be1-9b18-e09ce60e7434" />

Gestión de Logs en Sistemas Modernos (systemd)
Durante la auditoría, se evidenció que las distribuciones modernas de Debian prescinden de archivos de texto plano tradicionales como /var/log/mail.log para centralizar la telemetría en el Journal binario.

Se interrogó al Journal con la siguiente instrucción para confirmar la detección:

```Bash
journalctl | grep -iE 'clamav|milter|eicar' | tail -n 20
Resultado de la auditoría: El log confirmó una intercepción exitosa. El motor detectó la amenaza (Eicar-Signature FOUND) y ordenó a Postfix la retención del mensaje (quarantined by clamav-milter).
```
<img width="1046" height="539" alt="image" src="https://github.com/user-attachments/assets/2e67d7ca-80eb-4ade-b7be-68b1f4a262f9" />

Para mantener la compatibilidad con scripts de evaluación heredados, se instaló el demonio clásico de logs:

```Bash
apt install rsyslog
```
<img width="708" height="418" alt="image" src="https://github.com/user-attachments/assets/78b8d82d-6d7d-416d-9423-35705c22603d" />
