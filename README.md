# EcoLens

App de reciclaje para Android e iOS que reconoce materiales con la cámara y convierte cada escaneo en puntos, niveles y recompensas.

EcoLens es mi Trabajo de Fin de Grado (DAM). Está escrita en Kotlin Multiplatform con Compose Multiplatform, comparte la lógica de negocio y casi toda la interfaz entre Android e iOS, y usa Supabase como backend.

## Por qué lo hice

La idea no salió de "quiero hacer una app de reciclaje" sino de un problema muy concreto: casi todo el mundo sabe que *debería* reciclar, pero en la práctica falla en dos puntos.

Primero, no siempre sabe en qué contenedor va cada cosa — un brik, una caja de pizza con grasa, un vidrio de perfume — y duda o tira mal por pereza de comprobarlo.

Segundo, aunque lo haga bien, no recibe ningún tipo de respuesta: reciclar es un acto invisible, sin feedback, y eso hace que cueste mantenerlo como hábito.

Podía haber hecho una app puramente informativa (un buscador de "qué va en qué contenedor"), pero ese formato ya existe y no resuelve el problema de fondo, que es de comportamiento, no de información.

Por eso planteé EcoLens en dos capas: una capa de reconocimiento que te dice al momento, apuntando con la cámara, dónde va el objeto; y una capa de gamificación (puntos, niveles, rachas, recompensas) que da ese feedback que falta y convierte cada escaneo en algo que se nota.

Elegí este enfoque porque además era el que mejor encajaba con un TFG de DAM orientado a movilidad: me permitía tocar cámara, IA on-device, un backend real con autenticación, y una app multiplataforma de verdad, no una maqueta de pantallas bonitas.

## Qué hace

- **Escaneo de materiales**: apuntas con la cámara y la app identifica si lo que ves es Vidrio, Papel y Cartón, Envases u Orgánico, y te dice el color de contenedor correspondiente en tiempo real, antes incluso de pulsar el botón de captura.
- **Puntos, niveles y rachas**: cada escaneo válido da puntos. Los puntos de por vida determinan el nivel; hay una racha diaria que se rompe si un día no escaneas nada, y una misión diaria (escanear un número de objetos) que da un bonus extra la primera vez que se completa.
- **Eco-Dex**: una galería de los materiales que ya has escaneado alguna vez, a modo de colección — ves de un vistazo qué categorías te faltan.
- **Tienda de recompensas**: canjeas puntos por cupones. Cada canje genera un código QR único que queda marcado como "activo" hasta que se valida, y la validación usa una condición atómica sobre ese estado para que el mismo cupón no se pueda usar dos veces.
- **Eco-Retorno (SDDR)**: una simulación del futuro sistema de depósito, devolución y retorno de envases en España — escaneas o lees un código y se acumula un saldo simulado por cada envase devuelto.
- **Mapa de puntos de reciclaje**: puntos limpios de Madrid (datos abiertos del ayuntamiento, con caché de 24h) filtrables por tipo y por cercanía, con Google Maps en Android y MapKit en iOS.
- **Historial y ranking**: registro de todos tus escaneos y una clasificación entre usuarios por puntos.
- **Asistente de dudas**: un chat con respuestas por palabras clave (a qué contenedor va X, ideas de reutilización) para las preguntas típicas que no necesitan cámara. Es un sistema de reglas simple, no un modelo conversacional.

## Decisiones técnicas

**Kotlin Multiplatform en vez de dos apps nativas o Flutter/React Native.** Con KMP escribo modelos, repositorios, lógica de puntos y la mayoría de las pantallas una sola vez en `commonMain`, y solo bajo a código específico (`androidMain`/`iosMain`) donde de verdad hace falta: cámara, mapas y el modelo de IA.

La parte específica de cada plataforma se conecta mediante `expect/actual` (por ejemplo `PlatformCameraView`). El coste es que hay dos toolchains que mantener (Gradle y Xcode), y que algunas piezas de Android, como el modelo TFLite embebido, no tienen equivalente directo en iOS — es un trade-off que asumí a cambio de no duplicar toda la lógica de negocio.

**Modelo TFLite propio en vez de depender solo de una API externa.** El reconocimiento de las cuatro categorías principales (Vidrio, Papel y Cartón, Envases, Orgánico) corre con un modelo TFLite entrenado por mí y empaquetado dentro de la app vía ML Kit Custom Model, así que funciona sin conexión y sin coste por petición.

Para lo que ese modelo no reconoce bien, hay una cadena de fallbacks: primero un labeler genérico de ML Kit con reglas propias para filtrar ruido de escena, después la API de Gemini como respaldo con conexión, y por último un backend HTTP propio y OCR sobre el texto del envase. Es más trabajo que llamar a una sola API, pero significa que la función principal de la app no depende de tener internet ni de la cuota de un servicio externo. El límite honesto es que ese modelo local solo existe en Android por ahora; en iOS el reconocimiento pasa directamente por los fallbacks con conexión.

**Supabase con Row Level Security en vez de un backend propio.** Uso Postgrest, Auth y Storage de Supabase con políticas RLS activadas en las tablas de usuarios, historial y cupones, para que cada fila solo la pueda leer o modificar su dueño a nivel de base de datos, no solo a nivel de código de la app.

Para un proyecto en solitario esto me ahorra escribir y mantener un backend completo, y de paso me obligó a pensar el modelo de permisos desde la base de datos en vez de confiar solo en la lógica del cliente, que es más fácil de saltarse.

**Offline-first con sincronización diferida.** Los puntos, escaneos y rachas se guardan primero en local con Multiplatform Settings y se sincronizan a Supabase con un debounce de dos segundos, así que la app responde al instante y no dispara una petición de red por cada punto ganado.

El historial de escaneos mantiene además una cola de pendientes que se reintenta al recuperar conexión. La contrapartida es tener que vigilar que el estado local y el remoto no diverjan, sobre todo al iniciar sesión en un dispositivo nuevo con datos ya guardados en la nube.

**Sistema de niveles simple basado en umbrales.** En vez de una fórmula continua de experiencia, los niveles usan umbrales fijos para los primeros tramos y un cálculo lineal a partir de ahí. Es menos elegante que una curva matemática cerrada, pero es mucho más fácil de depurar y de explicar en una memoria de TFG, y en la práctica el usuario no nota la diferencia.

## Stack

- **Lenguaje / UI**: Kotlin 2.1, Compose Multiplatform 1.7 (Material 3)
- **Backend**: Supabase (Postgrest, Auth, Storage, Realtime) con Row Level Security
- **Red**: Ktor Client
- **IA / visión**: modelo TFLite propio vía ML Kit Custom Model (Android), ML Kit Image Labeling y Barcode Scanning, API de Gemini como fallback con conexión
- **Cámara y mapas nativos**: CameraX + Google Maps (Android), AVFoundation + MapKit (iOS)
- **Persistencia local**: Multiplatform Settings
- **Serialización**: kotlinx.serialization, kotlinx.datetime

## Cómo ejecutarlo

Requisitos: Android Studio (Ladybug o más reciente) con SDK 24+, y para iOS, Xcode con un Mac.

1. Clona el repositorio:
   ```
   git clone https://github.com/nexabuildev/EcoLens-KMP.git
   ```
2. Copia `local.properties.example` a `local.properties` y rellena tus propias credenciales:
   ```
   sdk.dir=/ruta/a/tu/Android/sdk
   MAPS_API_KEY=tu_clave_de_google_maps
   SUPABASE_URL=https://tu-proyecto.supabase.co
   SUPABASE_KEY=tu_clave_anon_de_supabase
   ML_BACKEND_URL=https://tu-backend.example.com/
   ```
   `local.properties` no se sube al repositorio; Gradle genera a partir de él un `EcoLensSecrets.kt` que usa la app.
3. En tu proyecto de Supabase crea las tablas `usuarios`, `historial_escaneos`, `cupones_tienda`, `cupones_canjeados` y `notificaciones`, y activa Row Level Security con una política de "solo el dueño de la fila puede leer/escribir".
4. Android: `./gradlew :composeApp:assembleDebug`
5. iOS: abre `iosApp/iosApp.xcodeproj` en Xcode y ejecuta en simulador o dispositivo.

---

Proyecto de Fin de Grado (DAM) de Rubén Simón.
