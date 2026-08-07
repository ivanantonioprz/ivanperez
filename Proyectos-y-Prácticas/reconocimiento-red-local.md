# Reconocimiento y Fingerprinting de Dispositivos en Red Local

**Autor:** Ivan
**Entorno:** Kali Linux (VM, VirtualBox) sobre red doméstica propia
**Alcance:** Mi propia red local — todos los dispositivos analizados me pertenecen o están bajo mi autorización directa.

---

## 1. Objetivo

Practicar la metodología de reconocimiento de red aplicada a un entorno realista: identificar todos los dispositivos conectados a mi red doméstica, entender qué información exponen pasivamente (sin necesidad de escanearlos activamente), y correlacionar múltiples fuentes de datos para llegar a una identificación positiva de cada uno.

Este ejercicio simula la primera fase de cualquier pentest interno: **reconocimiento y mapeo de activos**, antes de pasar a cualquier etapa de explotación.

---

## 2. Metodología

Se dividió el trabajo en reconocimiento **pasivo** (observar tráfico sin generar ninguno propio) y **activo** (enviar paquetes propios para obtener respuestas, como hace `nmap`). En un pentest real siempre se prioriza pasivo primero, ya que no deja rastro y reduce el riesgo de alertar al objetivo.

| Fase | Tipo | Herramienta |
|---|---|---|
| Configuración del entorno | — | VirtualBox networking |
| Escaneo de puertos y servicios | Activo | `nmap` |
| Resolución IP↔MAC | Pasivo/Activo | ARP, `ip neigh` |
| Descubrimiento de hostnames | Pasivo | mDNS (Wireshark) |
| Identificación de fabricante | Pasivo | OUI lookup (nmap/MAC vendor) |
| Confirmación de servicio | Activo | `curl` a APIs locales (Cast) |
| Captura de tráfico | Pasivo | Wireshark / tcpdump |

---

## 3. Preparación del entorno: troubleshooting de red en VM

Antes de poder escanear nada útil, hubo que resolver un problema de configuración de la VM.

**Síntoma inicial:** `nmap` contra la propia IP (tanto pública como local) devolvía `1000 filtered tcp ports (no-response)`, y `curl ifconfig.me` no devolvía nada usable (resultó ser una IPv6, resuelto forzando `-4`).

**Diagnóstico:** la VM de Kali estaba en modo **NAT** dentro de VirtualBox. En este modo, VirtualBox crea una subred aislada solo para la VM (típicamente `10.0.2.x`) y bloquea por default cualquier conexión entrante — incluso el propio tráfico de "auto-escaneo" quedaba filtrado por esa capa.

**Solución:** cambiar el adaptador de red de la VM de **NAT** a **Bridged (Adaptador puente)**. En este modo la VM obtiene una IP dentro del mismo rango que la red física (la del router), comportándose como un dispositivo más de la LAN — sin la capa de NAT interna de VirtualBox de por medio.

Tras el cambio, `hostname -I` confirmó una IP en el rango correcto de la red doméstica, y los escaneos empezaron a dar resultados reales.

**Lección:** un resultado "vacío" en una herramienta de recon no siempre significa ausencia de objetivos — primero hay que descartar que el propio entorno de red esté mal configurado.

---

## 4. Fase 1 — Escaneo con Nmap

Con el entorno corregido, se corrió un escaneo estándar contra la propia máquina y contra otros hosts de la red:

```bash
sudo nmap -sV -sC -O <ip> -oN scan.txt
```

- `-sV` → fingerprinting de versión exacta de cada servicio (clave para buscar CVEs conocidos después).
- `-sC` → scripts de reconocimiento por default de nmap.
- `-O` → detección de sistema operativo vía fingerprinting del stack TCP/IP.
- `-oN` → guarda el resultado en texto plano como evidencia.

### Hallazgo A — Host Windows

Un escaneo contra un host de la red arrojó el siguiente patrón de puertos, firma clásica de una máquina Windows:

```
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5357/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

- **135/tcp (RPC):** comunicación interna entre servicios de Windows.
- **139/tcp (NetBIOS Session Service):** compartición de archivos, versión legacy.
- **445/tcp (SMB / microsoft-ds):** el puerto más sensible del grupo — protocolo de compartición de archivos de Windows, y vector histórico de **EternalBlue/WannaCry** (2017). Ameritaría seguimiento con `smb-os-discovery` y `smb-vuln*` para precisar versión y exposición.
- **5357/tcp (WSDAPI):** permite que el equipo aparezca en el "Descubrimiento de red" de otros hosts Windows.

### Hallazgo B — Dispositivos IoT vía vendor lookup

El campo `MAC Address` del output de nmap incluye automáticamente el fabricante (vía base de datos OUI local, sin necesidad de internet):

- Un dispositivo resolvió a **Amazon Technologies Inc.** → correspondía a un Amazon Fire Stick.
- Otro dispositivo resolvió a **ZTE** → correspondía a un decodificador Android TV con el servicio Xview+ (Megacable), confirmado en la fase siguiente.

---

## 5. Fase 2 — ARP: resolución IP↔MAC

ARP (Address Resolution Protocol) traduce direcciones IP a direcciones MAC dentro de la red local. Es necesario porque las tarjetas de red no entienden IPs, solo direcciones físicas.

**Funcionamiento:**
1. Un host manda un **ARP Request** en broadcast: *"¿quién tiene la IP X?"*
2. El dueño de esa IP responde con un **ARP Reply** directo: *"soy yo, mi MAC es tal"*.
3. Ambos hosts guardan la relación en su caché ARP local.

**Regla de lectura de paquetes:** en un Reply, el campo **Sender IP/MAC** contiene siempre la información confirmada y confiable del dispositivo que respondió. El campo **Target** corresponde a quien hizo la pregunta original — no a la fuente de la información.

**Relevancia de seguridad:** ARP no tiene ningún mecanismo de autenticación. Cualquier dispositivo puede enviar una respuesta falsa (*ARP spoofing*) y el resto de la red le va a creer ciegamente, lo cual es la base de ataques man-in-the-middle. Es una debilidad de diseño heredada de los años 80, mantenida por compatibilidad.

```bash
ip neigh        # tabla ARP actual
arp -a          # alternativa clásica
```

---

## 6. Fase 3 — mDNS y fingerprinting de dispositivos

Capturando tráfico con Wireshark (`sudo wireshark`, filtro `mdns` o `udp.port == 5353`) se observó que varios dispositivos de la red **anuncian su propio hostname `.local` de forma espontánea**, sin que nadie les pregunte — esto es reconocimiento 100% pasivo: no se envió ni un solo paquete propio para obtener esta información.

### Caso 1 — Amazon Fire Stick

- **mDNS** reveló un hostname asociado a Amazon.
- **Vendor lookup (nmap/MAC)** confirmó `Amazon Technologies Inc.`
- Dos fuentes independientes correlacionadas → identificación positiva.

### Caso 2 — Decodificador Megacable (Xview+)

- **nmap** encontró el servicio en el puerto 8009/tcp, identificado por su firma de nmap como *"Ninja Sphere Chromecast driver"* — un nombre heredado de un proyecto de domótica open-source que en su día sirvió de referencia para la firma de detección del protocolo **castv2** (Google Cast), no indica marca real del dispositivo.
- El patrón de puertos (8008 HTTP, 8009 castv2/TLS, 8443 HTTPS) confirmó Google Cast integrado.
- Se consultó la API local de configuración:
  ```bash
  curl http://<ip>:8008/setup/eureka_info
  ```
- El campo `name` del JSON devolvió **"Xview+"** — el servicio OTT de Megacable sobre Android TV, coherente con el fabricante **ZTE** detectado por MAC (proveedor común de hardware de marca blanca para operadores de cable en Latinoamérica).

**Conclusión del caso:** de una IP sin identificar a una identificación completa (fabricante, protocolo, nombre exacto del servicio) cruzando 3 fuentes pasivas/semi-pasivas: mDNS + vendor MAC + API local del propio protocolo Cast — sin necesidad de acceso físico ni credenciales.

---

## 7. Fase 4 — Tráfico cifrado: qué se ve y qué no

Se probó capturar tráfico de un dispositivo propio reproduciendo contenido de streaming (YouTube), para ilustrar los límites del sniffing en una red con switches (no hubs) y tráfico HTTPS.

**Hallazgos:**
- En una red doméstica normal, el AP/switch **solo entrega a cada dispositivo los paquetes dirigidos a él** — no hay forma pasiva de ver tráfico ajeno sin técnicas activas como ARP spoofing (fuera de alcance de este ejercicio por motivos éticos, incluso dentro de la propia red familiar).
- Aunque se capturara el tráfico, **HTTPS cifra el contenido** — no se puede ver qué video específico se está reproduciendo.
- Sí queda visible el **SNI (Server Name Indication)** del handshake TLS — el dominio de destino viaja en texto plano aunque el resto esté cifrado, porque el servidor lo necesita para saber qué certificado presentar:
  ```
  tls.handshake.extensions_server_name
  ```
  Esto explica por qué existen mecanismos como DNS-over-HTTPS y Encrypted SNI (ECH): incluso con HTTPS, el destino de la conexión sigue siendo observable por un tercero en la red.

---

## 8. Limitación encontrada: modo monitor en VM

Se intentó poner la interfaz WiFi en modo monitor (`airmon-ng start wlan0`) para practicar captura de tráfico 802.11 a nivel de radio.

**Resultado:** la VM solo expone `eth0` y `lo`, sin extensiones inalámbricas (`no wireless extensions`). Causa: en modo Bridged, VirtualBox virtualiza el adaptador físico (WiFi o cable) como una interfaz genérica tipo Ethernet — la VM nunca tiene acceso directo al hardware WiFi real, por lo que no existe una tarjeta física que pueda ponerse en modo monitor.

**Solución fuera de alcance de este ejercicio:** requeriría un adaptador WiFi USB externo con soporte de modo monitor/inyección (chipsets Atheros AR9271 o Realtek RTL8812AU son los más recomendados), pasado por USB passthrough a la VM.

**Nota conceptual:** el modo monitor en sí es una técnica **pasiva** — la tarjeta solo escucha el espectro sin transmitir ni asociarse a ninguna red, por lo que no requiere ningún tipo de ocultamiento. Deja de ser pasivo recién cuando se combina con técnicas activas como *deauth attacks*, que sí inyectan tráfico y son detectables por sistemas WIDS/WIPS.

---

## 9. Hallazgos y recomendaciones

| Hallazgo | Riesgo | Mitigación sugerida |
|---|---|---|
| Dispositivos IoT (Fire Stick, decodificador) exponen hostname y modelo vía mDNS sin autenticación | Bajo-medio: permite fingerprinting pasivo de la red por cualquier dispositivo conectado | Segmentar IoT en una VLAN/red de invitados separada de los equipos principales |
| Host Windows con SMB (445/tcp) expuesto en la LAN | Medio-alto: vector histórico de exploits críticos (EternalBlue) | Confirmar parches al día, restringir SMB solo a los hosts que realmente lo necesiten, deshabilitar SMBv1 si sigue activo |
| ARP sin autenticación en toda la red | Medio: base de ataques MITM | Fuera del control del usuario final en la mayoría de routers domésticos; mitigable con ARP inspection en equipos gestionados |
| IP pública sin puertos expuestos (`filtered`, no-response) | N/A — resultado positivo | Confirma que el NAT/firewall del router está funcionando correctamente por default |

---

## 10. Aprendizajes clave

- La diferencia entre `closed` (el host respondió que no hay nada) y `filtered` (nadie respondió, firewall silencioso) cambia completamente la interpretación de un escaneo.
- El entorno de laboratorio (configuración de red de la VM) es en sí mismo una fuente común de resultados falsos negativos — vale la pena descartarlo antes de asumir que "no hay nada que encontrar".
- Correlacionar múltiples fuentes pasivas (mDNS + MAC vendor + protocolo específico) da una identificación mucho más sólida que confiar en una sola señal.
- El cifrado (HTTPS/TLS) protege el contenido, pero no necesariamente los metadatos de conexión (SNI) — importante para entender qué nivel de privacidad realmente ofrece.

---

*Todas las pruebas de este writeup se realizaron exclusivamente contra dispositivos propios dentro de mi red doméstica.*
