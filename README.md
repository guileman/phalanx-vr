# Phalanx CIWS · VR

Versión para **Meta Quest 2** del simulador manual del cañón Phalanx CIWS ([proyecto Android original](https://github.com/guileman/phalanx)).
Corre en el navegador de la Quest con **WebXR** (Three.js), que por debajo usa OpenXR. No hay que instalar nada ni activar el modo desarrollador.

## Jugar
1. En la Quest, abrí el navegador y entrá a **https://guileman.github.io/phalanx-vr/**
2. Tocá **ENTER VR**.

Cada push a `main` se publica solo en GitHub Pages (workflow `.github/workflows/pages.yml`).

## Controles
| Acción | Quest 2 | PC (prueba sin visor) |
|---|---|---|
| Apuntar el cañón | Girar la cabeza | Mouse (clic para capturarlo) |
| FUEGO (mantener) | Gatillo de cualquier mando | Clic izquierdo o Espacio |
| RECARGA | A / X | R |
| AVIONES (prender/apagar) | B / Y | P |

## Qué cambia respecto de la versión Android
- **Fondo**: ya no hay cámara ni passthrough. Es un entorno virtual con suelo de pasto, cielo con sol, nubes y cerros lejanos.
- **Puntería**: el cañón sigue el giro de las gafas, y los caños convergen a 400 m sobre la mira.
- **Disparo**: gatillo de cualquiera de los dos mandos.
- **HUD colimado**: está fijo a la cabeza pero ubicado a 30 m, así se ve "en el infinito", como un HUD real, y no cansa la vista.
  - La escalera de cabeceo, el horizonte con N/E/S/O, los recuadros de los aviones y las retículas de adelanto están anclados al mundo en 3D.
- **Vibración**: en vez de la vibración del celular y el flash LED, vibra el mando que dispara.
  - Suave en el arranque, continua en la metralla y un golpe al soltar.
- **Sin sacudir la imagen** durante la metralla: en VR eso marea. Vibra el modelo del cañón y hay fogonazo con luz.
- **Sin detector de objetos**: no hay cámara que analizar, así que los blancos son los aviones.

## Se mantiene igual
- Máquina de estados del cañón: SAFE → SPIN_UP → FIRING → SPIN_DOWN. Incluye el freno con caída cuadrática si se suelta antes de disparar.
- 4000 disparos/min, cargador de 1550, recarga de 4 s y temperatura del cañón.
- Balística en arrays planos: 1100 m/s, gravedad, arrastre 0,12/s, vida 3,2 s y dispersión.
- Los 5 aviones a escala real, con las mismas pasadas de cruce y de ataque (45 %).
  - Impacto 3D por segmento contra esfera.
  - Humo cuando están dañados.
  - Al derribarlos: caída con giro, fuego y explosión al tocar el suelo.
- Sonido 100 % sintetizado: motor, metralla, descarga de aire, freno y explosiones retrasadas por la velocidad del sonido. Ahora corre en un AudioWorklet, con paneo estéreo.
- Explosiones y humo generados proceduralmente, anclados al mundo con radio en metros.

## Archivos
Todo el juego vive en un solo `index.html`. Three.js 0.169 se carga desde jsDelivr.
