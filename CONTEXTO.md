Te paso el contexto completo de un juego de realidad virtual para Meta Quest 2 que ya está desarrollado. Leelo entero antes de responder: quiero que me ayudes a seguir desarrollándolo respetando todo lo que ya existe.

# PROYECTO: "Phalanx CIWS VR" — simulador manual del cañón Phalanx en realidad virtual

Repositorio: https://github.com/guileman/phalanx-vr (rama main)
Se publica en GitHub Pages: https://guileman.github.io/phalanx-vr/

Cada push a main dispara el workflow `.github/workflows/pages.yml`, que publica el repo entero en Pages con:
- actions/checkout@v4
- configure-pages@v5
- upload-pages-artifact@v3
- deploy-pages@v4

En la Pages del repo, la fuente tiene que ser "GitHub Actions". El repo tiene `.nojekyll`.

Es la versión VR de un juego Android existente (https://github.com/guileman/phalanx, Kotlin, realidad aumentada con la cámara del celular). La lógica del juego es la misma; cambian el fondo, la puntería y los controles.

## Concepto
El jugador está parado sobre una plataforma en medio de un entorno virtual y opera a mano un cañón Phalanx CIWS.
- El cañón apunta hacia donde mira la cabeza (el giro de las gafas) y dispara con el gatillo de cualquiera de los dos mandos.
- Encima se ve un HUD verde fósforo estilo avión de combate.
- Todo existe en coordenadas del MUNDO: proyectiles, aviones, explosiones y humo.

## Stack técnico
- Un único archivo `index.html` con todo el juego: HTML, CSS y un `<script type="module">`. No hay build ni bundler.
- Three.js 0.169.0 cargado desde jsDelivr con importmap ("three" y "three/addons/").
  - Solo se usa el addon `webxr/VRButton.js`.
- WebXR (sesión immersive-vr, que por debajo corre sobre OpenXR en el navegador de la Quest).
  - Espacio de referencia 'local-floor'.
  - Foveación fija nivel 1.
  - Requiere HTTPS.
- Sin assets externos: ni imágenes ni audio. Las texturas se generan con canvas 2D en tiempo de ejecución y todo el sonido se sintetiza en un AudioWorklet.
- Funciona también en PC sin visor para probar:
  - el mouse con pointer lock controla yaw/pitch de la cámara a 1,7 m de altura;
  - clic izquierdo o Espacio dispara, R recarga, P activa los aviones.

## Sistema de coordenadas
- three.js: X = Este, Y = Arriba, −Z = Norte. Metros. Origen = el jugador (pies en y=0, cabeza a ~1,6–1,7 m).
- Rumbo: 0° = Norte (−Z), 90° = Este (+X). Helper `dirAz(az)` = (sin az, 0, −cos az).
- `readHead()` saca cada frame de `camera.matrixWorld` lo siguiente:
  - pos, quaternion e inversa;
  - los vectores fwd/up/right;
  - rumbo (hdg), inclinación (pitch) y alabeo (roll) en grados;
  - el vector horizontal hacia adelante (hf) y hacia la derecha (hr).
- En VR, al principio de cada frame se llama `renderer.xr.updateCamera(camera)` para usar la pose de ESTE frame (si no, el cañón y el HUD quedan un frame atrasados y tiemblan al girar).

## Entorno
- Cielo: esfera de 15 km con shader propio.
  - Degradé del horizonte (0,75/0,84/0,92) al cenit (0,20/0,40/0,74).
  - Disco de sol y halo en la dirección SUN.
- Niebla lineal del color del horizonte, de 1500 a 13000 m.
- Luces: HemisphereLight más DirectionalLight desde el sol.
- Suelo: plano de 30 km con textura de pasto procedural de 256 px, repetida 400×, con anisotropía máxima.
- Plataforma gris de 4,5 m de radio bajo el jugador, con anillo amarillo.
- 16 cerros lejanos: conos low-poly con flatShading, a 4,5–9 km.
- 40 nubes: sprites con textura procedural, a 700–1600 m de altura y 1,5–9 km de distancia. Derivan hacia el este a 4 m/s.

## Cañón (máquina de estados, fuente única de verdad: objeto `gun`, función `gunUpdate`)
- SAFE → SPIN_UP (1 s, el rotor acelera) → FIRING (4000 disparos/min con acumulador) → SPIN_DOWN.
- Las balas salen RECIÉN en FIRING. Los disparos de un mismo frame se reparten en el tiempo (timeAhead).
- Al soltar, el SPIN_DOWN depende de cuándo:
  - Durante FIRING: SPIN_DOWN normal de 0,5 s con caída lineal.
  - Durante SPIN_UP: SPIN_DOWN "aborted" de 0,3 s con caída cuadrática.
  - En ambos casos se guardan `aborted` y `spinAtRelease`.
- Si se vuelve a apretar en SPIN_DOWN, reacelera desde el giro actual.
- Cargador de 1550 tiros. RECARGA de 4 s, solo en SAFE.
- Temperatura del cañón:
  - sube 0,0009 por disparo y se enfría 0,07/s cuando no dispara;
  - al llegar a 1 se sobrecalienta y no dispara hasta bajar de 0,6.
- Cadencia medida con los disparos de la última ventana de 1 s.

## Modelo 3D del cañón y puntería
- Grupo `gun3d` con recámara (caja), camisa (cilindro) y 6 caños que giran (`spinG`, velocidad visual = spin × 38 rad/s), más 2 abrazaderas.
- Posición: cabeza + 0,3 m a la derecha + 0,45 m abajo (en el marco horizontal), sin alabeo.
- Orientación: `lookAt` hacia un punto a 400 m sobre la mira. Los caños convergen ahí, y las balas salen de la boca (`muzzle`, a 1,85 m de la base) en esa dirección.
- Mientras dispara: el modelo tiembla ±6 mm, hay un fogonazo (sprite aditivo con tamaño y rotación al azar) y una PointLight naranja parpadeante en la boca.
- La imagen NO se sacude ni se desenfoca, porque en VR marea.

## Balística (arrays planos Float32Array, capacidad 600, coordenadas del mundo)
- Velocidad de boca 1100 m/s, gravedad 9,81, arrastre exponencial 0,12/s, vida 3,2 s.
- Dispersión gaussiana de 1,4 mrad por eje (~2,8 mrad).
- Se guarda la posición anterior (ox/oy/oz) para detectar cruces. Las balas se borran con swap-remove.
- Al tocar el suelo (y ≤ 0) la bala desaparece y levanta una bocanada de polvo: como máximo una cada 50 ms y solo a menos de 2,5 km.

## Trazadoras
- Dos InstancedMesh de cajas con blending aditivo: un núcleo y un brillo (3,2× más ancho, 28 % de intensidad).
- Cada trazadora va orientada según la velocidad y mide velocidad × 0,014 s (~15 m) de largo.
- Ancho = max(0,05, distancia × 0,0016) m, así el tamaño angular se mantiene parejo.
- Color de blanco-amarillo cerca a naranja a 1500 m. Se desvanece en los últimos 0,6 s de vida.

## Aviones (B/Y o tecla P los prende y apaga)
- Con los aviones prendidos y la cabeza a más de 10° sobre el horizonte, aparece uno cada 10 s (el primero a los 2 s). Como máximo 4 vivos.
- Cinco modelos a escala real, armados con geometría:
  - fuselaje: LatheGeometry de 8 lados;
  - alas, estabilizadores y derivas: cuadriláteros planos de doble cara;
  - cabina oscura en los cazas y 4 motores en el C-130;
  - MeshLambertMaterial con flatShading y niebla.
  - Modelo local: +Z = nariz, +Y = arriba.

| Modelo | Largo | Envergadura | Velocidad | Impactos |
|---|---|---|---|---|
| F-16 | 15 m | 9,5 m | 240 m/s | 4 |
| Mirage III | 15 m | 8,2 m | 230 m/s | 4 (ala delta, sin estabilizador horizontal) |
| A-4 Skyhawk | 12,2 m | 8,4 m | 200 m/s | 4 |
| MiG-29 | 17,3 m | 11,4 m | 250 m/s | 5 (doble deriva) |
| C-130 | 29,8 m | 40,4 m | 140 m/s | 10 (ala alta, 4 motores) |

- Pasadas:
  a) Cruce (55 %):
     - aparece a 26–40° del rumbo de la cabeza, a una distancia de 900–1900 m (C-130: 400–1300 m) y una elevación de 12–28° (mínimo 120 m de altura);
     - vuela nivelado hacia el otro lado y se acerca un poco.
  b) Ataque (45 %, ATTACK_CHANCE):
     - aparece de frente (±15°) a 2000–3200 m y 11–40° de elevación;
     - baja en picada hacia un punto a 15–105 m de costado y 70–200 m de altura;
     - al pasar ese punto, o si baja de 70 m, tira hacia arriba a 3,5 g hasta 35° de trepada y se va.
- Mientras un avión en picada está a menos de 2,5 km, el HUD parpadea "¡AMENAZA ENTRANTE! modelo distancia" y su etiqueta lleva "▼ATQ".
- Impacto 3D: el segmento de la bala en el frame, relativo al avión (posición anterior y actual), contra una esfera de radio 0,42 × max(largo, envergadura).
  - Cada impacto genera una explosión chica en el punto de impacto. Sonido como máximo uno cada 140 ms.
- Dañado: estela de humo cada 60 ms, más grande cuanto más dañado.
- Derribado:
  - explosión grande y 4 explosiones secundarias en el primer segundo;
  - gira sobre su eje (1,5–3,2 rad/s) y cae con gravedad, con fuego y humo denso cada 50 ms;
  - al tocar el suelo hay una explosión final y 7 columnas de humo.
- Se elimina si se aleja a más de 7,5 km o si pasaron 120 s.
- Superposiciones 3D por avión (sprites sin depthTest, tamaño angular constante):
  - recuadro de "brackets": verde, o ámbar con rombo si está bloqueado;
  - etiqueta con "MODELO Axx", distancia, nudos y vida (■), que se actualiza 5 veces por segundo;
  - retícula de adelanto: intercepción iterativa de 3 pasadas, con tiempo de vuelo según el arrastre (t = −ln(1 − k·s/v0)/k) y compensación de caída ½·g·t².
- Bloqueo: el avión más cercano a la mira, dentro de 20°.

## Efectos (sprites anclados al mundo, radio en metros)
- Hojas de sprites generadas con canvas al iniciar:
  - explosión: 4×4 cuadros de 128 px (fogonazo blanco → fuego naranja → humo oscuro);
  - humo: 2×2 bocanadas grises.
- Pools `FXPool`: 160 explosiones y 260 humos. Cada sprite clona la textura (comparte la imagen) y anima offset/repeat. Si se llena, se recicla el más viejo.
- El humo sube, crece hasta 3,2×, aparece con fundido de entrada y se desvanece. También sirve, teñido, para el polvo del suelo y el humo negro.

## Sonido (AudioWorklet 'phalanx' creado desde un Blob, salida estéreo con soft-clip tanh)
1. Motor: tono de 55 a 1205 Hz según el giro, con 3 armónicos y ruido de escobillas (pasa-banda en 3,2 kHz).
2. Metralla a 66,7 Hz (4000/min), generada dentro del worklet mientras firing = true.
   - Por disparo: crack agudo, cuerpo grave filtrado y thump de 70 Hz.
   - Más un traqueteo continuo con pasa-banda en 1350 Hz.
3. Al soltar después de disparar: descarga de aire de 0,5 s, con ruido pasa-banda que barre de 2600 a 250 Hz.
4. Al soltar antes de disparar: golpe de freno de 95 Hz más un chirrido pasa-banda que sigue la frecuencia del motor (0,3 s).
5. Explosiones con 8 voces:
   - retumbo filtrado, thump grave y crack;
   - se atenúan con la distancia y el filtro se oscurece a lo lejos;
   - se retrasan distancia/343 s (máximo 2,2 s);
   - paneo estéreo según la dirección respecto de la cabeza.
- Mensajes del hilo principal al worklet:
  - {t:'s', spin, firing}, cada frame;
  - {t:'rel', aborted, spin};
  - {t:'boom', d, s, p};
  - {t:'vol', v}.
- El AudioContext arranca con el primer gesto del usuario (pointerdown, keydown o inicio de la sesión XR).

## HUD
Colores: verde #39FF6A, ámbar #FFB020 para avisos, rojo #FF3B30 para alertas. Fuente monoespaciada en negrita con sombra.
- HUD colimado: un canvas de 1024×576 en un plano hijo de la cámara a 30 m (56° de ancho), sin depthTest, que se redibuja cada 2 frames. Al estar lejos se ve "en el infinito" y no cansa la vista.
- Cinta de rumbo arriba con caja de lectura "000°" y N/E/S/O.
- Escala de alabeo en arco abajo.
- Mira central con anillo de progreso del giro del rotor. El color cambia con el estado: verde LISTO, ámbar ACELERANDO, DETENIENDO o RECARGA, rojo FUEGO.
- Panel izquierdo: INCLIN, RUMBO, ALABEO, ESTADO, GIRO %, CADENCIA /min, DISPAROS, CARGADOR con barra y TEMP CAÑÓN con barra.
- Panel derecho: BLANCOS, IMPACTOS, EN VUELO, AVIONES (OFF o N ACT), DERRIBOS, más los datos del bloqueo (ID, modelo, distancia, tiempo de vuelo).
- Flechas rojas en el borde con la distancia para los aviones fuera de vista.
- Mensajes centrales:
  - sobrecalentado;
  - recargando %;
  - sin munición;
  - "CONTACTO AÉREO: modelo";
  - "APUNTE AL CIELO (>10°) PARA CONTACTOS";
  - "PRÓXIMO CONTACTO Ns".
- Anclado al mundo en 3D (radio 30 m alrededor de la cabeza, sin depthTest):
  - escalera de cabeceo conformal cada 5°: las negativas punteadas, números cada 10°, sigue el rumbo y muestra solo los peldaños a ±26° de la inclinación actual;
  - círculo del horizonte con marcas cada 10° y N/E/S/O.

## Controles
- Cabeza: apunta el cañón.
- Gatillo de cualquier mando (gamepad.buttons[0]): FUEGO, mantener apretado.
- A / X (buttons[4]): RECARGA. B / Y (buttons[5]): AVIONES, como interruptor. Se detecta el flanco por mano.
- Háptica (hapticActuators[0].pulse) en el mando que dispara:
  - suave y creciente en SPIN_UP;
  - 0,55 continua en FIRING;
  - golpe de 0,9 durante 70 ms al soltar.
- Volumen: el del visor.

## Qué se sacó respecto de la versión Android
- La cámara y el passthrough, porque ahora hay un entorno virtual.
- El detector de objetos MediaPipe y su seguimiento.
- El flash LED y la vibración del celular, reemplazados por la háptica del mando.
- La sacudida y el desenfoque de la imagen.

## Convenciones a respetar
- Todo lo que deba quedar fijo al entorno va en coordenadas del mundo (metros, origen = el jugador).
- Nada de assets externos: texturas procedurales con canvas, y todo sonido nuevo se agrega al worklet.
- Comentarios y textos de la interfaz en español rioplatense.
- Que siga siendo un solo `index.html` sin build, y sin dependencias nuevas salvo que sea necesario.
- Cuidar el rendimiento de la Quest 2 (72 Hz): pocos draw calls, sin sacudir la cámara y sin texturas enormes que se suban cada frame.

## Lo que necesito ahora
[ESCRIBÍ ACÁ LO QUE QUERÉS PEDIRLE]
