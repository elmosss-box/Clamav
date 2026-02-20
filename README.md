# 📖 Parte I: Marco Teórico y Arquitectura de Seguridad

Como fase previa a la implementación técnica, es imperativo definir los componentes, el flujo de datos y las limitaciones del entorno de seguridad del servicio de correo.

## 1. Naturaleza y Limitaciones del Motor Antivirus

ClamAV es un motor antivirus de código abierto diseñado específicamente para detectar software malicioso (malware), troyanos, virus y otras amenazas basadas en firmas que suelen viajar como archivos adjuntos.

Sin embargo, el diseño arquitectónico de ClamAV presenta una limitación crítica: no es un sistema antispam ni antifraude. ClamAV es incapaz de detectar Phishing (suplantación de identidad), correos basura (Spam) o ataques de Spoofing. Por tanto, en un entorno de producción real, ClamAV es solo una capa de un modelo de defensa en profundidad.

## 2. El Paradigma de Integración: El rol del Milter

ClamAV no se integra nativamente con Postfix. Postfix es un Agente de Transferencia de Correo (MTA) cuya única responsabilidad es enrutar mensajes a través del protocolo SMTP; no está diseñado para abrir y destripar el contenido de los correos.

Para solventar esta brecha, se utiliza una pieza de middleware llamada Milter (Mail Filter), en este caso específico, clamav-milter. El flujo operativo es el siguiente:

Postfix recibe un correo de internet.

A través de las directivas smtpd_milters y non_smtpd_milters en el archivo main.cf, Postfix desvía el flujo de datos hacia el socket del clamav-milter.

El milter actúa como puente, entregando el contenido al demonio residente clamd para su análisis.

Si ClamAV dictamina que el archivo está limpio, el milter devuelve el control a Postfix para su entrega.

Interceptación: Si ClamAV detecta una firma maliciosa, el milter intercepta el flujo. Habitualmente ejecuta una acción de Cuarentena (milter-hold), lo que significa que Postfix congela indefinidamente el correo infectado en su cola de retención interna, evitando que llegue al buzón del usuario sin generar un rebote (bounce) que alerte al atacante.

## 3. [Investigación] Vector de Seguridad Integral en Correo

Para que este proyecto de servidor de correo sea considerado robusto frente a los estándares de ciberseguridad actuales, la implementación de ClamAV debe complementarse obligatoriamente con las siguientes tecnologías:

* Validación de Identidad (DNS): Implementación estricta de registros SPF (Sender Policy Framework), DKIM (DomainKeys Identified Mail) y DMARC. Esto evita que atacantes externos falsifiquen el dominio de nuestro servidor.

* Filtrado Heurístico (Anti-Spam): Despliegue de Rspamd o SpamAssassin para analizar la semántica del correo, cabeceras sospechosas y reputación de IPs emisoras.

* Cifrado en Tránsito: Configuración de certificados TLS/SSL en Postfix y Dovecot para evitar ataques Man-in-the-Middle (MitM) garantizando conexiones STARTTLS.

* Protección contra Fuerza Bruta: Integración de Fail2Ban leyendo los logs de Dovecot y Postfix para banear IPs que intenten adivinar contraseñas de los usuarios.

# 🛠️ Parte II: Implementación Técnica (Guía Práctica)

## 1. Prerrequisitos y Arquitectura de Hardware

El demonio de ClamAV (clamd) carga su inmensa base de datos de firmas directamente en la memoria RAM. Ejecutarlo en un entorno con recursos limitados provoca irremediablemente errores Out of Memory (OOM) y la caída del servicio.

⚙️ Acción de Infraestructura: Se auditaron y ampliaron los recursos del hipervisor (Proxmox), asignando 4 GiB de RAM a la máquina virtual para garantizar la estabilidad operativa del servidor.

<div align="center">
<img width="700" alt="Configuración de RAM en Proxmox" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/53ccd0ec-fd25-4827-98fe-72b2747dc79e" />
</div>

## 2. Instalación de la Pila de Seguridad

Se procedió a instalar el motor base, el demonio residente y el intermediario (milter) que servirá de puente de comunicación con el MTA (Postfix).  

```
apt update
apt install clamav clamav-daemon clamav-milter
```

Posteriormente, para garantizar la eficacia desde el minuto cero, se forzó una actualización manual de la base de datos de firmas de virus (CVD), deteniendo temporalmente el servicio de actualización automática para evitar bloqueos:

```
systemctl stop clamav-freshclam
freshclam
systemctl start clamav-freshclam
```

<div align="center">
<img width="800" alt="Actualización de firmas" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/730a0096-7a74-4d95-bd1d-e8db9eaf22c2" />
</div>

## 3. Configuración del Puente de Red (Milter)

Para evitar problemas de permisos de archivos y restricciones de entornos enjaulados (chroot), se configuró el Milter para escuchar a través de un socket TCP local en lugar del tradicional socket Unix.

Archivo modificado: /etc/clamav/clamav-milter.conf

<div align="center">
<img width="600" alt="Archivo conf parte 1" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/148ffee2-b090-4da5-800a-86d6352d3601" />
</div>

```
MilterSocket inet:7357@127.0.0.1
```

<div align="center">
<img width="600" alt="Archivo conf parte 2" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/07fc288d-c5ff-46a3-82b2-81c6e4be9300" />
</div>

## 4. Integración en el MTA (Postfix)

Se modificó el núcleo del servidor de correo para desviar el tráfico entrante y saliente hacia el puerto del Milter, permitiendo la inspección de adjuntos.

Archivo modificado: /etc/postfix/main.cf

```
smtpd_milters = inet:127.0.0.1:7357
non_smtpd_milters = inet:127.0.0.1:7357
milter_default_action = accept
```

⚠️ Decisión de Diseño: Se configuró el parámetro milter_default_action = accept. En caso de que el demonio de ClamAV colapse, Postfix continuará procesando correos sin escanear. Esta política prioriza la Alta Disponibilidad del servicio de correo corporativo sobre el bloqueo preventivo.

<div align="center">
<img width="700" alt="Main.cf configuration" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/a3b28882-56b9-4dc7-a8a0-ee9ef5bca72e" />
</div>

A continuación, se reiniciaron los servicios involucrados en el orden jerárquico correcto para aplicar los cambios en memoria:

```
systemctl restart clamav-daemon
systemctl restart clamav-milter
systemctl restart postfix
```

<div align="center">
<img width="500" alt="Reinicio de servicios" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/0b8a52cf-0199-402b-bf5b-4a2505c2e90b" />
</div>

Se verificó empíricamente mediante el uso de sockets que el servicio estaba a la escucha en el puerto local definido (127.0.0.1:7357):

```
ss -tlnp | grep 7357
```

<div align="center">
<img width="800" alt="Verificación de puertos" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/5297cc5e-47b8-4043-8894-a0c8c86ce2ee" />
</div>

## 5. Auditoría y Pruebas de Intercepción (Prueba EICAR)

Para validar la solidez del entorno en producción simulada, se generó un archivo inofensivo con la firma de prueba estándar internacional EICAR.

<div align="center">
<img width="800" alt="Archivo EICAR" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/7818a240-58b7-4b8a-8448-c3e1a8c498bc" />
</div>

Se procedió a simular una inyección de malware enviando el archivo como adjunto a través del cliente Thunderbird hacia el servidor Postfix.

<div align="center">
<img width="600" alt="Envío por Thunderbird" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/7c0e44cc-0580-4be1-9b18-e09ce60e7434" />
</div>

📊 Gestión de Logs en Sistemas Modernos (systemd)

Durante la auditoría, se evidenció que las distribuciones modernas de Debian prescinden de archivos de texto plano tradicionales como /var/log/mail.log para centralizar la telemetría en el Journal binario de systemd.

Se interrogó a la base de datos del Journal con la siguiente instrucción para rastrear la actividad del milter:

```
journalctl | grep -iE 'clamav|milter|eicar' | tail -n 20
```

✅ Resultado empírico de la auditoría: El log confirmó una intercepción exitosa. El motor detectó la amenaza (Eicar-Signature FOUND) y ordenó a Postfix la retención inmediata del mensaje en la cola interna (quarantined by clamav-milter).

<div align="center">
<img width="800" alt="Log del Journal" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/2e67d7ca-80eb-4ade-b7be-68b1f4a262f9" />
</div>

(Nota de retrocompatibilidad): Para mantener la compatibilidad con scripts de evaluación académica heredados, se instaló el demonio clásico de logs para forzar la escritura en /var/log/mail.log:

```
apt install rsyslog
```

<div align="center">
<img width="600" alt="Instalación de rsyslog" src="https://www.google.com/search?q=https://github.com/user-attachments/assets/78b8d82d-6d7d-416d-9423-35705c22603d" />
</div>
