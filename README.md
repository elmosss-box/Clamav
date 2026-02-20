<div align="center">
<h1>🛡️ Seguridad Perimetral: Postfix + ClamAV-Milter</h1>
<img src="https://img.shields.io/badge/Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white" alt="Debian Badge"/>
<img src="https://img.shields.io/badge/Postfix-005571?style=for-the-badge&logo=maildotru&logoColor=white" alt="Postfix Badge"/>
<img src="https://img.shields.io/badge/ClamAV-F13A31?style=for-the-badge&logo=c&logoColor=white" alt="ClamAV Badge"/>





<p><em>Integración de filtrado antivirus en tránsito sobre arquitectura Debian Linux.</em></p>
</div>

🖥️ 1. Prerrequisitos y Arquitectura de Hardware

El demonio de ClamAV (clamd) carga su inmensa base de datos de firmas directamente en la memoria RAM. Ejecutarlo en un entorno con recursos limitados provoca irremediablemente errores Out of Memory (OOM) y la caída del servicio.

⚙️ Acción de Infraestructura: Se auditaron y ampliaron los recursos del hipervisor (Proxmox), asignando 4 GiB de RAM a la máquina virtual para garantizar la estabilidad operativa del servidor.

<div align="center">
<img width="700" alt="Configuración de RAM en Proxmox" src="https://github.com/user-attachments/assets/53ccd0ec-fd25-4827-98fe-72b2747dc79e" />
</div>

📦 2. Instalación de la Pila de Seguridad

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
<img width="800" alt="Actualización de firmas" src="https://github.com/user-attachments/assets/730a0096-7a74-4d95-bd1d-e8db9eaf22c2" />
</div>

🔌 3. Configuración del Puente de Red (Milter)

Para evitar problemas de permisos de archivos y restricciones de entornos enjaulados (chroot), se configuró el Milter para escuchar a través de un socket TCP local en lugar del tradicional socket Unix.

Archivo modificado: /etc/clamav/clamav-milter.conf

<div align="center">
<img width="600" alt="Archivo conf parte 1" src="https://github.com/user-attachments/assets/148ffee2-b090-4da5-800a-86d6352d3601" />
</div>
```
MilterSocket inet:7357@127.0.0.1
```

<div align="center">
<img width="600" alt="Archivo conf parte 2" src="https://github.com/user-attachments/assets/07fc288d-c5ff-46a3-82b2-81c6e4be9300" />
</div>

✉️ 4. Integración en el MTA (Postfix)

Se modificó el núcleo del servidor de correo para desviar el tráfico entrante y saliente hacia el puerto del Milter, permitiendo la inspección de adjuntos.

Archivo modificado: /etc/postfix/main.cf
```
smtpd_milters = inet:127.0.0.1:7357
non_smtpd_milters = inet:127.0.0.1:7357
milter_default_action = accept
```

⚠️ Decisión de Diseño: Se configuró el parámetro milter_default_action = accept. En caso de que el demonio de ClamAV colapse, Postfix continuará procesando correos sin escanear. Esta política prioriza la Alta Disponibilidad del servicio de correo corporativo sobre el bloqueo preventivo.

<div align="center">
<img width="700" alt="Main.cf configuration" src="https://github.com/user-attachments/assets/a3b28882-56b9-4dc7-a8a0-ee9ef5bca72e" />
</div>

A continuación, se reiniciaron los servicios involucrados en el orden jerárquico correcto para aplicar los cambios en memoria:
```
systemctl restart clamav-daemon
systemctl restart clamav-milter
systemctl restart postfix
```

<div align="center">
<img width="500" alt="Reinicio de servicios" src="https://github.com/user-attachments/assets/0b8a52cf-0199-402b-bf5b-4a2505c2e90b" />
</div>

Se verificó empíricamente mediante el uso de sockets que el servicio estaba a la escucha en el puerto local definido (127.0.0.1:7357):
```
ss -tlnp | grep 7357
```

<div align="center">
<img width="800" alt="Verificación de puertos" src="https://github.com/user-attachments/assets/5297cc5e-47b8-4043-8894-a0c8c86ce2ee" />
</div>

🕵️‍♂️ 5. Auditoría y Pruebas de Intercepción (Prueba EICAR)

Para validar la solidez del entorno en producción simulada, se generó un archivo inofensivo con la firma de prueba estándar internacional EICAR.

<div align="center">
<img width="800" alt="Archivo EICAR" src="https://github.com/user-attachments/assets/7818a240-58b7-4b8a-8448-c3e1a8c498bc" />
</div>

Se procedió a simular una inyección de malware enviando el archivo como adjunto a través del cliente Thunderbird hacia el servidor Postfix.

<div align="center">
<img width="600" alt="Envío por Thunderbird" src="https://github.com/user-attachments/assets/7c0e44cc-0580-4be1-9b18-e09ce60e7434" />
</div>

📊 Gestión de Logs en Sistemas Modernos (systemd)

Durante la auditoría, se evidenció que las distribuciones modernas de Debian prescinden de archivos de texto plano tradicionales como /var/log/mail.log para centralizar la telemetría en el Journal binario de systemd.

Se interrogó a la base de datos del Journal con la siguiente instrucción para rastrear la actividad del milter:
```
journalctl | grep -iE 'clamav|milter|eicar' | tail -n 20
```

✅ Resultado empírico de la auditoría: El log confirmó una intercepción exitosa. El motor detectó la amenaza (Eicar-Signature FOUND) y ordenó a Postfix la retención inmediata del mensaje en la cola interna (quarantined by clamav-milter).

<div align="center">
<img width="800" alt="Log del Journal" src="https://github.com/user-attachments/assets/2e67d7ca-80eb-4ade-b7be-68b1f4a262f9" />
</div>

(Nota de retrocompatibilidad): Para mantener la compatibilidad con scripts de evaluación académica heredados, se instaló el demonio clásico de logs para forzar la escritura en /var/log/mail.log:
```
apt install rsyslog
```

<div align="center">
<img width="600" alt="Instalación de rsyslog" src="https://github.com/user-attachments/assets/78b8d82d-6d7d-416d-9423-35705c22603d" />
</div>
