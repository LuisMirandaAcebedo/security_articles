# Vulnerabilidades en Balizas V16 Conectadas: Cuando la Seguridad Vial se Vuelve Insegura

## Introducción: El Dispositivo que Debía Salvarnos

Desde 2026, las balizas V16 conectadas sustituyen obligatoriamente a los triángulos de emergencia en las carreteras españolas. Estos dispositivos IoT se comunican con el sistema DGT 3.0 para alertar de accidentes en tiempo real. Una idea simplemente decente sobre el papel: tecnología moderna al servicio de la seguridad vial.

El problema es que la implementación de seguridad de al menos uno de estos dispositivos es... digamos que *optimista*. Tanto, que cualquiera con un portátil y conocimientos básicos podría convertir tu baliza de emergencia en algo bastante menos útil que un pisapapeles luminoso.

En este artículo vamos a analizar las vulnerabilidades encontradas en la **baliza Help Flash IoT**, un dispositivo que según Vodafone ha vendido más de 250.000 unidades en España. 
Aviso: no vas a necesitar ser un hacker  para entender por qué esto es preocupante.

<p align="center" width="100%">
    <img width="50%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251202112301.png">
</p>


A pesar de que los ejemplos de fallos de seguridad se centran en este modelo examinado, los problemas de seguridad encontrados en la parte de comunicaciones parecen ser **comunes a todos los dispositivos**, pues otros modelos también examinados tienen el mismo funcionamiento y los mismos problemas.

Grosso modo, el funcionamiento de estos dispositivos se basa en el siguiente esquema:
- La baliza, una vez encendida por el usuario, intenta obtener coordenadas GPS y se conecta por NB-IoT al operador de telefonía.
- Cada cierto tiempo envía un mensaje a un servidor del propio fabricante donde indica parámetros como la hora de la incidencia, coordenadas GPS, identificadores del dispositivo, etc.
- El servidor del fabricante y la baliza no transmiten datos a través de internet, si no que el operador les da conectividad por medio de lo que se conoce como APN privado. Esto es una red privada y aislada.
- El servidor del fabricante procesa los mensajes que recibe de sus balizas y envía, ahora sí, a través de internet, las alertas a los servidores de la DGT3.0
- DGT3.0 distribuye las alertas a aplicaciones de navegación (Google Maps, Waze, ...), a los paneles luminosos de las autopistas/autovías, y a otras aplicaciones.

<p align="center" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251203123335.png">
</p>

## Primera Vulnerabilidad: Comunicaciones al Descubierto

### El Problema: Hablando en Voz Alta con la Cara Tapada

Imagina que estás en una emergencia en la carretera. Activas tu baliza V16 y esta envía tu ubicación exacta, el identificador de tu dispositivo, y toda la información de red a los servidores de la DGT. Genial, ¿verdad? El problema es que **lo hace todo en texto plano**, sin ningún tipo de cifrado.

<p align="center" width="100%">
    <img width="50%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251203113720.png">
</p>

Es como si en lugar de llamar discretamente a emergencias, gritaras tu dirección exacta por un megáfono para que todo el vecindario se entere. Pero en este caso, "el vecindario" es cualquiera que pueda estar monitorizando el tráfico de red.

Además de enviar los datos en claro, hay otro problema, la **conexión no está autenticada**. Esto significa que el receptor no puede verificar que el mensaje llega de quién se supone que tiene que llegar.

<p align="left" width="100%">
    <img width="80%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251126203844.png">
</p>



Sin este tipo de mecanismos de seguridad, se puede suplantar a la baliza, pues el servidor no puede saber si es la baliza auténtica u otro dispositivo que se está haciendo pasar por ella.

### ¿Qué Datos se Exponen?

Del análisis del puerto serie del dispositivo (sí, tiene un puerto de depuración accesible sin contraseña, pero eso es otra historia), podemos ver claramente:

- **Coordenadas GPS exactas**: Porque saber dónde estás en una emergencia es información sensible
- **IMEI del dispositivo**: Tu identificador único
- **Parámetros de red**: Cell ID, intensidad de señal, operador
- **Timestamp**: Cuándo activaste la baliza
- etc...

<p align="left" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251126204723.png">
</p>


Me llama la atención la **ausencia de mecanismos de integridad del mensaje**, algo que asegure al receptor del mensaje que éste no ha sido manipulado o afectado por errores en la transmisión. 

### ¿Por Qué Esto es un Problema?

1. **Privacidad inexistente**: Cualquiera en posición de observar el tráfico de datos puede rastrear tu ubicación en tiempo real durante la emergencia, conocer el IMEI de tu dispositivo, cuándo lo has activado... 

2. **Suplantación facilitada**: Como el IMEI va en claro y es el único "secreto" del dispositivo, cualquiera que lo intercepte puede potencialmente hacerse pasar por tu baliza. En realidad ni siquiera esto es necesario, pues en el modelo analizado el IMEI se encuentra impreso en la parte inferior del dispositivo.

3. **Integridad cero**: No hay forma de verificar que los datos no han sido modificados en tránsito. Potencialmente se podría alterar tu ubicación reportada si alguien intercepta el mensaje y lo modifica.

El operador de turno te dirá que todo esto está muy bien, pero que la baliza no se conecta directamente a internet, si no a una red privada sobre la que tienen completo control y, por lo tanto, nadie puede ni interceptar ni modificar ni conectarse a esta red para hacer maldades. En teoría, esto es así. En teoría.

### La Excusa del APN Privado

La seguridad del sistema parece basarse en el concepto de "seguridad por oscuridad": usan un APN privado de Vodafone, así que nadie puede acceder, ¿verdad? 
Bueno, no exactamente. Los parámetros de conexión también están expuestos en el puerto serie:

```
AT+QBAND=3,20,8,28
AT+COPS=0
AT+QCGDEFCONT="IP",""
```

Esto significa que con acceso físico a un dispositivo y extrayendo la eSIM (o sin hacer ésto y utilizando el dispositivo modificado como placa de desarrollo o como módem conectado a un PC), un atacante podría conectarse a esa red privada y hacer que todos estos problemas de seguridad sí  importen. La "muralla" de seguridad resulta ser más bien una valla de plástico.

### El Ataque de la Estación Base Falsa: Fake eNodeB

Pero hay un vector de ataque aún más interesante: **estaciones base LTE falsas (fake eNodeB)**. Estos dispositivos permiten a un atacante hacerse pasar por una torre de telefonía móvil legítima y que los dispositivos se conecten a ella en lugar de a la del operador. ¿Y qué tiene esto que ver con nuestra baliza?

El módulo celular (Quectel BC65) es exclusivo de NB-IoT/LTE-M, sin capacidad 2G. Levantar una BTS (estación base) 2G es relativamente sencillo y es un ataque clásico a dispositivos móviles, lo utilizan a diario ciberdelincuentes para ataques de phishing SMS o para espionaje, aunque en este caso no sería posible porque el módem del dispositivo no tiene fallback a 2G (que es el mecanismo que se utiliza para realizar ataques a dispositivos actuales, mediante un downgrade attack), pero esto no impide el ataque, simplemente lo hace más... moderno.

<p align="center" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251202170607.png">
</p>

**Cómo funciona el ataque:**

1. **Desplegar fake eNodeB**: Con una SDR (Software Defined Radio) tipo BladeRF, USRP o LimeSDR (~500-1000€) y software como srsRAN u OpenAirInterface, se monta una estación base LTE falsa.

2. **Jamming selectivo**: El atacante hace jamming de las frecuencias NB-IoT legítimas en las bandas 3, 20, 8 y 28 (las que usa el dispositivo según los logs analizados), esto es, las bloquea para que la señal de la estación base del operador no sea recibida por la baliza.

3. **Captura del dispositivo**: La fake eNodeB se anuncia como la torre con mejor señal. El dispositivo se conecta automáticamente mediante el comando `AT+COPS=0` (selección automática de operador, se conecta al mejor disponible en cada momento).

4. **Dos modalidades de ataque:**

   **Modalidad A - Ataque simple (DoS/Eavesdropping):**
   - El dispositivo se conecta a la fake eNodeB
   - La estación falsa no tiene conexión con la red real
   - Las balizas envían datos al vacío (denegación de servicio)
   - El atacante puede interceptar y leer todo el tráfico (que va en claro por UDP, sin ningún tipo de cifrado)
   - Útil para: espionaje, rastreo de ubicaciones, impedir que lleguen alertas y, de paso, hacerse con los IMEI válidos de las balizas que se conecten a él y emplearlos posteriormente para poder construir tramas válidas y emitir falsas alertas.

   **Modalidad B - Man-in-the-Middle completo:**
   - El atacante necesita una **eSIM extraída de otro dispositivo** comprometido
   - La fake eNodeB tiene conectividad real al APN privado de Vodafone
   - El atacante puede:
     - Interceptar todo el tráfico
     - Modificar datos en tránsito (coordenadas GPS falsas, parámetros de red)
     - Inyectar tramas arbitrarias
     - Reenviar selectivamente el tráfico para no levantar sospechas
   - Es transparente para el sistema: el servidor recibe datos, pero modificados

<p align="center" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251201174836.png">
</p>

**El "cifrado" de LTE no protege nada aquí**: Aunque LTE tiene cifrado en la capa de transporte, el payload de la aplicación, el mensaje en sí (las tramas UDP) **va completamente en claro**. 

**Lo mejor (o peor) de todo**: No necesitas ni siquiera comprometer el dispositivo físicamente. Una fake eNodeB portátil en una furgoneta puede capturar todas las balizas en un radio de varios cientos de metros.

**Herramientas necesarias:**
- BladeRF, USRP B210 o LimeSDR: 500-1000€
- PC con Linux (o Raspberry Pi 4 para versión portátil): 50-200€
- Software libre: srsRAN, OpenAirInterface (gratis)
- Opcional para MitM completo: eSIM extraída de dispositivo comprometido
- Conocimientos: Documentación técnica + algo de paciencia (más complejo que 2G, pero totalmente factible)

Sí, por menos de 1500€ y con software libre, alguien puede interceptar y manipular las comunicaciones "seguras" de estas balizas. El presupuesto de una semana de vacaciones de hotel para hacer algo que podría afectar a miles de dispositivos.

### El Problema del Protocolo Propietario

Dado que no se usa un protocolo estándar con cifrado end-to-end (como MQTT sobre TLS, CoAP con DTLS, o HTTP/2), toda la "seguridad" recae en:

1. La oscuridad del APN privado (ya sabemos cómo va eso)
2. La "seguridad" de la red del operador
3. La esperanza de que nadie se moleste en atacarlo (spoiler: me molesté)

Es un castillo de naipes, y cada capa es igual de frágil que la anterior. Un soplido y...

### Tranquilidad...
Pero bueno, para llevar a cabo los ataques más peligrosos y sofisticados que pueden explotar estas vulnerabilidades es necesario destripar el dispositivo y acceder a su Hardware.
O no...

## Segunda Vulnerabilidad: Actualizaciones OTA Sin Control

### El Botón Mágico de 8 Segundos

Aquí es donde la cosa se pone realmente interesante. El dispositivo tiene un sistema de actualización Over-The-Air (OTA) vía WiFi no documentado. Es decir, el fabricante ha dejado un mecanismo para poder actualizar el software interno de las balizas a través de Wifi, pero ésto no figura en ningún manual o documentación oficial. Para activarlo, solo necesitas mantener pulsado el botón de encendido durante unos 8 segundos.

No hace falta PIN. No hace falta contraseña. No hace falta confirmación en una app móvil. Solo acceso al botón. Ocho segundos.

<p align="center" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251126203617.png">
</p>

### Credenciales Compartidas: Porque la Seguridad es Cosa de Todos

Cuando el dispositivo entra en modo OTA, busca automáticamente una red WiFi específica:

- **SSID**: `HF-UpdateAP-5JvqFV`
- **Contraseña**: `HF-UpdateAP-5JvqFV` (sí, idéntica al SSID, porque recordar dos cosas diferentes es complicado)

Lo mejor de todo: estas credenciales están **hardcodeadas en el firmware** y no son únicas por dispositivo. Tras probar con dos unidades diferentes compradas con meses de separación, confirmé que comparten las mismas credenciales. Es decir, **los 250.000+ dispositivos vendidos probablemente usan la misma contraseña WiFi** para su mecanismo de actualización.

Es como si todos los coches del mundo usaran la misma llave. Muy útil para los ladrones, supongo.

### HTTP: El Protocolo de los Años 90 Vuelve con Fuerza

Una vez conectado al WiFi, el dispositivo descarga las actualizaciones usando... HTTP. No HTTPS, no. HTTP plano, sin cifrado, en el puerto 8080.

```
CHECK UPDATES URL : http://-5JvqFV:8080/settings_v18.json
```

Para los que no estéis familiarizados con la diferencia: HTTP es como enviar postales (todo el mundo puede leerlas e incluso manipular su contenido, pues el mensaje está "a la vista"), HTTPS es como enviar cartas en sobres cerrados y lacrados. Aquí estamos enviando el firmware del dispositivo en postales, de forma que se puede leer y modificar el contenido sin protección.

### Sin Autenticación del Servidor

El dispositivo **no verifica**:
- Certificados SSL/TLS (porque no usa HTTPS)
- Firma digital del servidor
- Identidad del servidor

Básicamente, acepta actualizaciones de cualquiera que responda en el hostname correcto. Es como si cualquiera pudiese  enviarte actualizaciones de la app de tu banco sin tener que demostrar que realmente es tu banco y que tú no te dieses ni cuenta.

### DNS Spoofing: Porque ¿Quién Necesita Verificación?

Cuando tecleas en tu navegador www.google.es para hacer una búsqueda, tu ordenador tiene que consultar qué dirección IP corresponde con ese dominio google.es y así poder enviar la petición a la dirección IP correcta. Esto se hace mediante DNS (Domain Name System) y existen una serie de medidas de seguridad para que sólo ciertos servidores puedan enviarte información veraz sobre la dirección a la que tienes que conectarte cuando intentas acceder, por ejemplo, a una web.

El hostname del servidor de actualizaciones que busca la baliza se construye a partir del SSID de la red WiFi. Y se resuelve mediante DNS... sin DNSSEC. Esto permite ataques de DNS spoofing triviales: te haces pasar por un servidor DNS, le dices al dispositivo que el servidor de actualizaciones está donde tú quieras, tu ordenador, por ejemplo, y a él le vale.


## Prueba de Concepto: Cómo Hackear una Baliza en 60 Segundos

He desarrollado una prueba de concepto completa que demuestra lo fácil que es comprometer uno de estos dispositivos del modelo analizado. Aquí el proceso:

### Preparación (una sola vez)

1. **Crear punto de acceso WiFi malicioso**
   - SSID: `HF-UpdateAP-5JvqFV`
   - Contraseña: la misma (ya sabemos cuál)

2. **Configurar servidor DHCP**
   - Para asignar dirección IP al dispositivo

3. **Montar servidor DNS wildcard**
   - Para resolver el hostname del servidor de actualizaciones a nuestra IP. Esto es, engañar a la baliza para que se fíe de la dirección en la que le digo que tiene que descargarse la actualización

4. **Preparar servidor HTTP malicioso**
   - En el puerto 8080
   - Con un archivo JSON de configuración falso
   - Y un firmware modificado

<p align="left" width="100%">
    <img width="60%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251126205035.png">
</p>


### Ejecución del Ataque

1. Activar el hotspot malicioso
2. Mantener pulsado el botón del dispositivo 8 segundos
3. El dispositivo entra en modo OTA automáticamente
4. Se conecta a nuestro WiFi falso
5. Descarga nuestro archivo JSON malicioso con un link al nuevo firmware
6. Descarga nuestro firmware modificado
7. Se reinicia con el firmware comprometido

**Tiempo total: 30-60 segundos**

Para los frikis, aquí podéis ver el log de mi servidor de actualizaciones falso que utilicé para las pruebas.

<p align="left" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251202154325.png">
</p>

Y aquí el log del dispositivo cuando la actualización remota se inicia de forma correcta y comienza a descargarse el firmware malicioso que preparé.

<p align="left" width="100%">
    <img width="100%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251129135450.png">
</p>

### Resultado

Una vez instalado el firmware malicioso, el control del dispositivo es total. Podríamos:

- Enviar ubicaciones falsas al servidor DGT 3.0
- Acceder al APN privado del operador
- Generar falsas alarmas masivas
- Potencialmente interceptar comunicaciones
- Convertir el dispositivo en un ladrillo inútil
- Convertir la baliza en un dispositivo aparentemente homologado, que no cumple la homologación
- ...

<p align="left" width="100%">
    <img width="80%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251126203211.png">
</p>

## Efecto Cascada: Cuando Todo se Desmorona

Incialmente envié un informe a INCIBE con toda la información sobre las vulnerabilidades en cuestión de comunicaciones, el cual rechazó asignar CVE porque estas vulnerabilidades "requieren acceso físico a la electrónica del dispositivo". Técnicamente correcto: necesitas abrir el dispositivo, conectar un cable serie, extraer la eSIM...

Para quién no esté familiarizado, CVE es una lista de vulnerabilidades de ciberseguridad divulgadas públicamente con un identificador único, una descripción y al menos una referencia.

Pero aquí está la gracia: **una vez que comprometes el dispositivo mediante OTA, ya no necesitas acceso físico a nada más**. El firmware malicioso puede:

1. Conectarse al APN privado
2. Enviar tramas arbitrarias al servidor
3. Suplantar otros dispositivos
4. Lanzar ataques DoS contra la infraestructura

Es decir, las vulnerabilidades "no explotables" se vuelven perfectamente explotables gracias a las vulnerabilidades OTA. Una bonita cadena de ataques donde el acceso físico momentáneo (8 segundos de pulsar un botón) escala a compromiso total y permanente.

## Escenarios de Explotación Realistas

### Escenario 1: El Taller Malicioso

Llevas tu coche al taller. Mientras están "revisando los niveles", comprometen tu baliza. Ahora pueden rastrear tus movimientos (mientras dure la batería, claro), generar falsas alarmas, o simplemente convertir tu baliza en un pisapapeles.

### Escenario 2: Ataque Masivo

Un atacante despliega varios puntos de acceso falsos en zonas de alto tráfico (centros comerciales, gasolineras, áreas de servicio). Con las credenciales compartidas, cualquier dispositivo que entre en modo OTA en esas zonas se compromete automáticamente. Escala exponencialmente.

### Escenario 3: La Furgoneta con Antena

El más cinematográfico pero perfectamente factible: un atacante en una furgoneta con una fake BTS tipo eNodeB recorre la ciudad o se posiciona en una autopista.

<p align="center" width="100%">
    <img width="50%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251202165449.png">
</p>

**Versión simple (DoS):** Intercepta las comunicaciones de todas las balizas cercanas, leyendo ubicaciones en tiempo real. Las balizas creen que están enviando alertas, pero **los datos se pierden** en el vacío. Útil para impedir que lleguen emergencias reales de personas específicas o en zonas específicas.

**Versión avanzada (MitM completo):** Con una eSIM extraída de un dispositivo comprometido, la fake eNodeB tiene acceso al APN privado. Ahora puede no solo interceptar, sino **modificar** activamente los datos: alterar ubicaciones, inyectar falsas alarmas, o incluso hacer que una emergencia real parezca estar en otro lugar.


## El Aspecto Irónico: (IN)Seguridad en un Dispositivo de Seguridad

Lo verdaderamente irónico es que estamos hablando de un **dispositivo de seguridad crítico**. Su única función es salvar vidas en emergencias. Y tiene una seguridad informática que haría llorar a un estudiante de primero de ciberseguridad.

No estamos hablando de hackear una bombilla inteligente o un robot aspirador. Estamos hablando de un dispositivo que:

- Es **obligatorio** por ley
- Se integra con infraestructura crítica (DGT 3.0)
- Gestiona situaciones de vida o muerte
- Ha sido **homologado** y aprobado oficialmente

## Recomendaciones: Cómo Arreglar Este Desastre

### Críticas (Hacerlo YA)

1. **Implementar HTTPS con pinning de certificados**: Usa TLS 1.3 para todas las comunicaciones OTA. No es 1995.

2. **Firmar digitalmente el firmware**: Cada actualización debe estar firmada criptográficamente. El dispositivo debe verificar la firma antes de instalar nada.

3. **Autenticación mutua**: El dispositivo verifica el servidor, el servidor verifica el dispositivo. Con certificados de verdad.

4. **Credenciales únicas por dispositivo**: Cada baliza debe tener su propio SSID y contraseña únicos, generados aleatoriamente en fábrica.

5. **Autenticación de usuario para OTA**: PIN, confirmación por app móvil, dos factores... lo que sea, pero algo más que 8 segundos de botón.

### Importantes (Hacerlo Pronto)

6. **Actualizar ESP-IDF**: El firmware usa ESP-IDF v4.3.2 de 2021. Hay versión 5.x con mejoras de seguridad importantes.

7. **Activar Secure Boot y Flash Encryption**: El ESP32-C3 tiene estas características. Úsalas.

8. **Cifrar comunicaciones celulares**: DTLS sobre UDP, o cambiar a TCP/TLS. Pero cifrar, por favor.

9. **Proteger el puerto de depuración**: Al menos pon una contraseña o deshabilítalo en producción.

### Recomendables (Para Dormir Más Tranquilos)

10. **Implementar detección de anomalías en el servidor**: Si un dispositivo empieza a comportarse raro, el servidor debería notarlo.

11. **Rate limiting y anti-replay**: Para evitar ataques de repetición y DoS.

12. **Monitorización y alertas**: Si alguien está comprometiendo dispositivos masivamente, deberíais enteraros.

## La Respuesta de INCIBE: Cuando "Físico" No Es Tan Físico

INCIBE declinó asignar CVEs, argumentando que las vulnerabilidades requieren "acceso físico". Su criterio:

> "Physical theft, damage, or destruction are not by themselves cybersecurity Vulnerabilities."

Es comprensible, todos sabemos que un ciberdelincuente o ciberterrorista ve una placa de circuito y entra en pánico, ni por asomo se les ocurriría meter mano al hardware de un dispositivo para poder llevar a cabo un ataque.

<p align="center" width="100%">
    <img width="50%" src="https://github.com/LuisMirandaAcebedo/security_articles-/blob/main/help_flash_iot/pictures/Pasted%20image%2020251202165708.png">
</p>


La respuesta de INCIBE técnicamente es correcta para las vulnerabilidades de comunicaciones, estemos o no de acuerdo con la explotabilidad de las mismas. El problema es que las vulnerabilidades OTA **convierten el acceso físico momentáneo (pulsar un botón) en compromiso remoto permanente**. Y eso, bajo mi criterio, sí es una vulnerabilidad de ciberseguridad en toda regla.

He enviado también la solicitud a MITRE para asignación de CVE, pues ellos cuentan con antecedentes de vulnerabilidades tenidas en cuenta a pesar de requerir acceso físico a botones y hacer un uso no esperado de ellos. 
Veremos qué opinan ellos sobre si 8 segundos de pulsar un botón cuenta como "acceso físico significativo" o no.

*Nota: Sin ánimo de criticar  a INCIBE, que hace un trabajo importante, pero quizás las autoridades en materia de Ciberseguridad deberían reconsiderar la definición de "físico" en la era del IoT.*

## Impacto Real: Más Allá de la Teoría

### Cuantificando el Riesgo

- **Dispositivos afectados**: 250.000+ solo de este modelo
- **Complejidad del ataque**: Baja (script de Python + Raspberry Pi)
- **Coste del ataque**: <100€ en hardware
- **Tiempo de compromiso**: <60 segundos
- **Conocimientos necesarios**: Intermedios
- **Escalabilidad**: Alta (credenciales compartidas)

### Consecuencias Potenciales

1. **Seguridad vial comprometida**: Balizas que no funcionan en emergencias reales
2. **Privacidad violada**: Potencial rastreo de ubicaciones en tiempo real
3. **Falsas alarmas masivas**: Saturación de servicios de emergencia
4. **Pérdida de confianza**: En tecnologías de seguridad vial conectadas


## Conclusión: IoT Significa "Insecurity of Things"

Este caso es un ejemplo perfecto de cómo NO hacer seguridad en IoT:

✗ Comunicaciones sin cifrar  
✗ Sin autenticación de servidor  
✗ Sin mecanismo de integridad en los mensajes
✗ Credenciales hardcodeadas compartidas  
✗ HTTP en lugar de HTTPS  
✗ Sin firma digital de firmware  
✗ Puerto de depuración abierto  
✗ Activación de OTA sin autenticación  
✗ Firmware desactualizado  
✗ Secure Boot deshabilitado  

Es casi como si en la fase de diseño alguien hubiese aparecido con una checklist de malas prácticas y alguien hubiese dicho "sí, queremos todas estas vulnerabilidades en nuestro dispositivo, por favor. Póngame una de cada".

Lo más preocupante es que este es un dispositivo **obligatorio por ley**, **homologado oficialmente**, e **integrado en infraestructura crítica nacional**. Si esto es lo que pasa los controles de calidad y seguridad, ¿qué podemos esperar del resto del ecosistema IoT de seguridad?

## Mensaje Final

A los fabricantes: La seguridad no es opcional, especialmente en dispositivos críticos para la seguridad. HTTPS existe. Las firmas digitales existen. La autenticación existe. Úsenlas.

A los reguladores: Necesitamos estándares de seguridad IoT más estrictos y auditorías reales antes de homologar dispositivos críticos.

A los usuarios: Si tenéis una de estas balizas (o cualquier dispositivo IoT), sed conscientes de que la seguridad puede ser... optimista. No pongáis vuestra vida en manos de la tecnología sin entender sus limitaciones.


---

## Sobre Esta Investigación

Esta investigación se realizó de forma completamente legal y ética:
- Sobre dispositivos de mi propiedad
- Sin acceder a sistemas de terceros
- Sin interceptar comunicaciones ajenas
- Siguiendo protocolos de divulgación responsable
- Con el objetivo de mejorar la seguridad del ecosistema IoT

**Investigador**: Luis Miranda Acebedo  
**Contacto**: luis.miranda.acebedo@gmail.com  

---

*Nota: Algunos detalles técnicos específicos y el código completo del PoC se han omitido para evitar la explotación masiva de estas vulnerabilidades. 
