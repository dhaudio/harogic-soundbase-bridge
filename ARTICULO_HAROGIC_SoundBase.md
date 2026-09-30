# HAROGIC SAN-60 y SoundBase: un puente para trabajar con espectro en directo

**Darío Hernández Martínez · RF Solutions**  
**Estado del proyecto: septiembre de 2026**

En coordinación RF necesito observar el espectro y trabajar con las frecuencias dentro de mi herramienta de planificación. Quería utilizar mi HAROGIC SAN-60 como fuente de datos para SoundBase, manteniendo el control del barrido y la lectura de la traza en directo.

Para conseguirlo he desarrollado, con asistencia de IA, un puente que comunica el analizador con SoundBase mediante una API HTTP. La configuración que he comprobado utiliza el HAROGIC conectado por USB a Ubuntu en Parallels y SoundBase Desktop ejecutándose en macOS.

También he preparado una adaptación para Windows de 64 bits en Boot Camp. Esta versión todavía necesita validación con el analizador físico y SoundBase en Windows.

## El problema que quería resolver

Mi objetivo era recibir barridos del SAN-60 directamente en SoundBase, sin exportar e importar un CSV cada vez que necesitara actualizar el espectro.

El puente ofrece a SoundBase una interfaz de configuración y lectura, mientras que el SDK HTRA de HAROGIC se encarga de comunicarse con el equipo. Esto permite separar dos responsabilidades: adquirir el espectro del analizador y presentar esos datos en el formato que consume SoundBase.

## Cómo funciona

SoundBase se conecta al servicio como **Spectrum Analyzer Bridge**. El servicio recibe las solicitudes de configuración, las traduce a llamadas del SDK HTRA y publica la última traza adquirida.

En el montaje probado:

1. El USB del SAN-60 está asignado a la máquina virtual Ubuntu.
2. El puente se ejecuta en Ubuntu y escucha en el puerto **8088**.
3. SoundBase en macOS se conecta a la dirección IP de Ubuntu.
4. Las solicitudes de barrido se ejecutan en el analizador y sus muestras se devuelven a SoundBase.

En mis pruebas, la dirección de la máquina virtual era `10.211.55.4`. Esa dirección pertenece a mi instalación y puede cambiar en otro ordenador.

La adaptación para Windows está preparada para ejecutar SoundBase y el puente en el mismo sistema. En ese caso, la dirección es `127.0.0.1` y el puerto sigue siendo `8088`.

## La API del puente

| Método | Ruta | Función |
|---|---|---|
| GET | `/info` | Identificación del analizador y capacidades anunciadas. |
| GET | `/configuration` | Consulta de la configuración actual. |
| POST | `/configuration` | Modificación del rango, RBW, VBW, paso y referencia. |
| POST | `/sweep/start` | Inicio de la adquisición continua. |
| POST | `/sweep/stop` | Solicitud de parada de la adquisición. |
| GET | `/trace` | Lectura de la última traza disponible. |

La traza incluye las frecuencias inicial y final, el paso entre muestras, el número de puntos, las amplitudes en dBm, la fecha de adquisición y un identificador de barrido: `sweepId`.

Ese identificador permite comprobar si llegan barridos nuevos. Que el servidor responda a `/info` confirma una parte de la conexión; para verificar la adquisición también hay que comprobar que `/trace` entrega datos y que `sweepId` avanza.

## Resolución: RBW y paso entre puntos

Una parte importante del desarrollo fue distinguir la **RBW** del **step size**.

La RBW es el ancho de banda de resolución utilizado en el análisis. El paso describe la separación en frecuencia entre las muestras de la traza publicada. Son parámetros distintos: configurar una RBW de 10 kHz no implica recibir un punto cada 10 kHz.

En una primera prueba, con un rango solicitado de **470 a 900 MHz**, obtuve **3.988 puntos** y un paso real de aproximadamente **107,88 kHz**, con RBW y VBW configuradas a 10 kHz.

Después reduje el rango solicitado a **470–800 MHz** y aumenté la petición de puntos hasta un máximo de **16.000**. En una adquisición comprobada obtuve:

| Parámetro | Resultado |
|---|---:|
| Frecuencia inicial de la traza | 469.982.649,10 Hz |
| Frecuencia final de la traza | 800.013.421,20 Hz |
| Puntos recibidos | 15.995 |
| Paso real entre muestras | 20.634,66 Hz |
| RBW y VBW configuradas | 10.000 Hz |

La traza no coincide exactamente con los extremos solicitados porque el SDK devuelve una rejilla de frecuencias propia. El puente publica esos valores reales.

Este resultado describe una adquisición concreta, no un número fijo de puntos garantizado para todos los ajustes. SoundBase puede modificar la configuración, y el SDK determina la traza resultante.

## Conservar las muestras del analizador

El puente utiliza el barrido completo del modo **SWP** y la función de recorte de espectro del SDK para obtener el tramo solicitado. Publica las muestras resultantes sin interpolar ni reducir su número.

Como el formato de la traza utiliza frecuencia inicial, paso y número de puntos, el puente comprueba que el eje sea aproximadamente uniforme. Si no lo es, rechaza la traza en lugar de asignarle una rejilla de frecuencias inventada.

En la implementación se sustituyen las amplitudes no finitas por **−160 dBm**, conservando su posición. Ese valor es una representación de datos inválidos, no una medición del suelo de ruido del SAN-60.

La versión utilizada no incorpora un promedio adicional en el puente: el modo publicado es `clear-write`.

## Activar y desactivar sin usar la terminal

Para facilitar el uso preparé un control de escritorio que permite activar el servicio, desactivarlo y consultar su registro.

En Linux ya he comprobado el funcionamiento del control. Para Windows preparé una ventana con botones **Activar**, **Desactivar** y **Ver registro**.

Activar el servicio no significa que el equipo esté barriendo: la adquisición comienza cuando SoundBase solicita el inicio. Al terminar, desactivar el puente permite liberar el analizador para utilizarlo con otro programa.

## Instalación y estado de las versiones

| Entorno | Estado |
|---|---|
| Ubuntu ARM64 en Parallels + SoundBase en macOS | Conexión, configuración y recepción de barridos comprobadas con el SAN-60. |
| Control de escritorio en Linux | Instalado y comprobado. |
| Windows x64 en Boot Camp + SoundBase en el mismo Windows | Paquete preparado; validación física pendiente. |

El paquete de Windows incluye el SDK HTRA **0.55.100**, el código del puente, el control y las instrucciones. Su instalador necesita Internet para preparar Python y las dependencias. SoundBase y el controlador USB oficial se instalan por separado.

Las comprobaciones realizadas para Windows incluyen la sintaxis del código, las estructuras y posiciones de los campos del SDK y pruebas de la lógica con un analizador simulado. No equivalen a ejecutar las DLL, el instalador y el SAN-60 en Windows.

## Qué queda por comprobar

El siguiente paso es validar la adaptación de Windows con el equipo real y medir la estabilidad durante sesiones largas. También quiero registrar los tiempos de barrido para distintas combinaciones de rango, RBW y puntos.

En macOS observé que SoundBase podía quedarse en blanco tras un periodo sin interacción. La causa sigue pendiente de diagnóstico; no doy por resuelto ese comportamiento con el instalador ni lo atribuyo todavía al puente.

## Compartir el proyecto

Este proyecto nace de una necesidad práctica en mi trabajo como coordinador RF: utilizar el SAN-60 dentro del flujo de planificación de SoundBase y comprobar qué datos llegan realmente a la aplicación.

Si pruebas el puente, un informe útil incluye el sistema operativo, el modelo y firmware del analizador, la versión del SDK, la configuración del barrido, el número de puntos recibido y el registro del error si lo hay.

**Proyecto independiente de RF Solutions. HAROGIC y SoundBase son productos de sus respectivos titulares; este puente no constituye una integración oficial de sus fabricantes.**
