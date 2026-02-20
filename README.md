Markdown
# Implementación de Seguridad Perimetral: Postfix + ClamAV-Milter en Debian

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

Posteriormente, se forzó una actualización manual de la base de datos de firmas de virus (CVD) deteniendo temporalmente el servicio de actualización automática:

Bash
systemctl stop clamav-freshclam
freshclam
systemctl start clamav-freshclam
(Captura de la actualización de firmas exitosa)

3. Configuración del Puente de Red (Milter)
Para evitar problemas de permisos de archivos (chroot), se configuró el Milter para escuchar a través de un socket TCP local en lugar de un socket Unix.

Archivo modificado: /etc/clamav/clamav-milter.conf

Plaintext
MilterSocket inet:7357@127.0.0.1
(Captura de la modificación del archivo clamav-milter.conf)

4. Integración en el MTA (Postfix)
Se modificó el núcleo del servidor de correo para desviar el tráfico entrante y saliente hacia el puerto del Milter.

Archivo modificado: /etc/postfix/main.cf

Plaintext
smtpd_milters = inet:127.0.0.1:7357
non_smtpd_milters = inet:127.0.0.1:7357
milter_default_action = accept
Decisión de Diseño: Se configuró milter_default_action = accept. En caso de que el demonio de ClamAV colapse, Postfix continuará procesando correos sin escanear, priorizando la disponibilidad del servicio sobre el bloqueo preventivo.

(Captura de la configuración de main.cf)

A continuación, se reiniciaron los servicios para aplicar los cambios:

Bash
systemctl restart clamav-daemon
systemctl restart clamav-milter
systemctl restart postfix
(Captura del reinicio de servicios)

Se verificó mediante el uso de sockets que el servicio estaba a la escucha en el puerto correcto (127.0.0.1:7357):

Bash
ss -tlnp | grep 7357
(Captura del servicio escuchando en el puerto local y la IP asignada 10.0.0.50)

5. Auditoría y Pruebas de Intercepción (Prueba EICAR)
Para validar el entorno, se generó un archivo inofensivo con la firma de prueba estándar internacional EICAR.
(Captura de la creación del archivo pruebavirus.txt)

Se procedió a enviar el archivo como adjunto a través del cliente Thunderbird.
(Captura del envío del correo malicioso simulado)

Gestión de Logs en Sistemas Modernos (systemd)
Durante la auditoría, se evidenció que las distribuciones modernas de Debian prescinden de archivos de texto plano tradicionales como /var/log/mail.log para centralizar la telemetría en el Journal binario.

Se interrogó al Journal con la siguiente instrucción para confirmar la detección:

Bash
journalctl | grep -iE 'clamav|milter|eicar' | tail -n 20
Resultado de la auditoría: El log confirmó una intercepción exitosa. El motor detectó la amenaza (Eicar-Signature FOUND) y ordenó a Postfix la retención del mensaje (quarantined by clamav-milter).
(Captura del log confirmando la detección y cuarentena del virus)

(Opcional): Para mantener la compatibilidad con scripts de evaluación heredados, se instaló el demonio clásico de logs:

Bash
apt install rsyslog
(Captura de la instalación del paquete rsyslog)
