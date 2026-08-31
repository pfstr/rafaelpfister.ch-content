---
title: "Leer y entender la factura de electricidad: una factura de EKZ partida por partida"
navTitle: "Entender la factura de electricidad"
description: "Energía, uso de la red, medición, gravámenes: qué figura realmente en una factura de electricidad suiza, quién fija cada uno de los precios y qué partidas pueden modificarse, con una factura de ejemplo interactiva basada en el modelo de EKZ."
date: "2026-08-20"
kategorie: "Electricidad y energía"
timeToRead: "9 min de lectura"
themen:
  - stromtarife-leg
hauptthema: "stromtarife-leg"
protokolle:
  - "strom"
related:
  - lokale-elektrizitaetsgemeinschaft-leg-erklaert
  - lohnt-sich-leg-beitritt
  - leg-preisrechner
translationId: "article-76c220e720fdffbe"
slug: "como-leer-y-entender-la-factura-de-electricidad-una-factura-de-ekz-partida-por-partida"
aiPrompt: "Ich füge dir gleich die Positionen meiner Schweizer Stromrechnung ein. Erkläre mir jede Position einzeln: was sie bedeutet, wer den Preis festlegt (Energielieferant, Netzbetreiber oder Bund/Gemeinde) und ob ich sie beeinflussen kann. Rechne zum Schluss aus, wie sich meine Gesamtkosten pro kWh zusammensetzen, und nenne die zwei grössten Hebel zum Sparen. Meine Rechnung:"
translationOf: stromrechnung-verstehen-ekz
translationSourceHash: d81a9bfcf0e980271b4b1f54234b918f4658a44fa002a3f7572dfd80df8ba9b1
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:55:36.408Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/como-leer-y-entender-la-factura-de-electricidad-una-factura-de-ekz-partida-por-partida
---

# Leer y entender la factura de electricidad: una factura de EKZ partida por partida

La factura de electricidad es uno de esos documentos que se pagan sin leerlos. Sin embargo, su estructura es transparente: cada partida tiene una finalidad clara, un emisor claro y una respuesta clara a la pregunta de si se puede cambiar algo en ella. Quien haya entendido una vez los cuatro bloques podrá leer cualquier factura de electricidad suiza, pues su estructura está establecida por ley y es la misma para todos los operadores de red.

Este artículo repasa partida por partida una factura de EKZ (Elektrizitätswerke des Kantons Zürich), nuestro operador de red. La factura de ejemplo interactiva que aparece a continuación sigue la estructura de nuestra propia factura trimestral; las cifras corresponden a un hogar modelo con un consumo de 1'800 kWh en el trimestre, calculado con las tarifas reales de EKZ de 2026.

## Quién participa realmente en la factura

En una factura de electricidad figuran tres emisores, aunque solo uno la envíe:

1. **El proveedor de energía** vende los kilovatios hora. En el suministro básico, es el proveedor local; los precios los revisan el Supervisor de Precios y la ElCom, respectivamente. Es el único bloque en el que se puede elegir producto.
2. **El operador de red** transporta la electricidad. La red es un monopolio regulado: no se puede cambiar, y la ElCom revisa las tarifas. No obstante, aquí existen tarifas opcionales y, desde 2026, el descuento LEG.
3. **La Confederación, el cantón y el municipio** añaden gravámenes: recargo de red, reserva de electricidad, tasas municipales. Ni el proveedor ni el operador de red pueden cambiar nada en ellos.

Con este esquema, se puede clasificar cada partida que aparece a continuación. Pase el cursor por las líneas de la factura de ejemplo (o tóquelas) para mostrar la explicación:

<div class="sr-embed">
<div class="sr-grid">
<div class="sr-paper" role="group" aria-label="Interaktive Beispielrechnung">
<div class="sr-head">
<div class="sr-brand">Musterwerk AG</div>
<div class="sr-meta">Su factura del 01.03.2026 al 31.05.2026<br>Hogar modelo, vivienda unifamiliar, tarifa de red EKZ Netz 400F, 1'800 kWh</div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Suministro de energía</div>
<div class="sr-row" tabindex="0" data-sr="energie-winter"><span class="sr-label">Tarifa de energía ene.–mar.</span><span class="sr-calc">600 kWh × 13.30 Rp.</span><span class="sr-amount">79.80</span></div>
<div class="sr-row" tabindex="0" data-sr="energie-sommer"><span class="sr-label">Tarifa de energía abr.–jun.</span><span class="sr-calc">1'200 kWh × 9.00 Rp.</span><span class="sr-amount">108.00</span></div>
<div class="sr-row" tabindex="0" data-sr="grundtarif"><span class="sr-label">Tarifa básica</span><span class="sr-calc">3 meses × CHF 3.00</span><span class="sr-amount">9.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Uso de la red</div>
<div class="sr-row" tabindex="0" data-sr="netz"><span class="sr-label">Red 400F</span><span class="sr-calc">1'800 kWh × 7.50 Rp.</span><span class="sr-amount">135.00</span></div>
<div class="sr-row" tabindex="0" data-sr="sdl"><span class="sr-label">Servicios de sistema (SDL)</span><span class="sr-calc">1'800 kWh × 0.27 Rp.</span><span class="sr-amount">4.86</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Medición</div>
<div class="sr-row" tabindex="0" data-sr="messung"><span class="sr-label">Tarifa de medición</span><span class="sr-calc">3 meses × CHF 5.00</span><span class="sr-amount">15.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Recargos y gravámenes</div>
<div class="sr-row" tabindex="0" data-sr="bundesabgaben"><span class="sr-label">Gravámenes federales</span><span class="sr-calc">1'800 kWh × 2.30 Rp.</span><span class="sr-amount">41.40</span></div>
<div class="sr-row" tabindex="0" data-sr="stromreserve"><span class="sr-label">Reserva de electricidad</span><span class="sr-calc">1'800 kWh × 0.41 Rp.</span><span class="sr-amount">7.38</span></div>
<div class="sr-row" tabindex="0" data-sr="solidarisiert"><span class="sr-label">Costes solidarios</span><span class="sr-calc">1'800 kWh × 0.05 Rp.</span><span class="sr-amount">0.90</span></div>
<div class="sr-row" tabindex="0" data-sr="effizienz"><span class="sr-label">Fomento de la eficiencia energética</span><span class="sr-calc">1'800 kWh × 0.16 Rp.</span><span class="sr-amount">2.88</span></div>
</div>
<div class="sr-block sr-sums">
<div class="sr-row sr-net" tabindex="0" data-sr="netto"><span class="sr-label">Importe neto (sin IVA)</span><span class="sr-calc"></span><span class="sr-amount">404.22</span></div>
<div class="sr-row" tabindex="0" data-sr="mwst"><span class="sr-label">8.1 % de IVA</span><span class="sr-calc"></span><span class="sr-amount">32.74</span></div>
<div class="sr-row sr-total" tabindex="0" data-sr="total"><span class="sr-label">Importe de la factura</span><span class="sr-calc"></span><span class="sr-amount">CHF 436.95</span></div>
</div>
</div>
<aside class="sr-panel" aria-live="polite">
<div class="sr-panel-inner" id="sr-panel-target">
<p class="sr-panel-hint">Pase el cursor por una partida o tóquela para ver qué hay detrás.</p>
</div>
</aside>
</div>
<div hidden id="sr-explanations">
<div data-exp="energie-winter"><strong>Energía, trimestre de invierno</strong><p>La electricidad propiamente dicha. Desde 2026, EKZ factura, en lugar de una tarifa de punta y de valle, una tarifa única por trimestre: 13.30 Rp./kWh en el semestre de invierno (de enero a marzo, de octubre a diciembre). Emisor: proveedor de energía. Es el único bloque en el que puede elegirse otro producto.</p></div>
<div data-exp="energie-sommer"><strong>Energía, trimestre de verano</strong><p>La misma energía, otro precio: 9.00 Rp./kWh de abril a septiembre. El verano es más barato porque entonces abunda la energía hidroeléctrica y solar. Los precios estacionales hacen visible lo que hace tiempo que rige en el mercado eléctrico: la electricidad tiene un valor distinto según la estación del año.</p></div>
<div data-exp="grundtarif"><strong>Tarifa básica de energía</strong><p>Importe mensual fijo del suministro de energía (CHF 3.00 al mes), independiente del consumo. Cubre la facturación y la distribución.</p></div>
<div data-exp="netz"><strong>Uso de la red, tarifa de consumo</strong><p>El transporte: construcción, operación y mantenimiento de la red eléctrica. Es un monopolio regulado; no se puede cambiar de operador de red. El precio depende del producto de red elegido: tarifa estándar 400ST 7.95 Rp./kWh, 400F con control favorable para la red 7.50 Rp./kWh, tarifa para bomba de calor 400WP 6.45 Rp./kWh (todos sin IVA). Precisamente en esta partida se aplica el descuento LEG del 20 o 40 por ciento.</p></div>
<div data-exp="sdl"><strong>Servicios de sistema</strong><p>La contribución a Swissgrid para la estabilidad de la red de transporte: mantenimiento de la frecuencia, potencia de regulación, energía reactiva. 0.27 Rp./kWh, similar en todos los operadores de red.</p></div>
<div data-exp="messung"><strong>Medición</strong><p>Operación del contador y puesta a disposición de los datos de medición, indicada como partida propia desde 2026 (anteriormente incluida en el uso de la red). CHF 5.00 al mes. Por cierto, el contador inteligente que se paga aquí es el requisito técnico para participar en una LEG.</p></div>
<div data-exp="bundesabgaben"><strong>Gravámenes federales (recargo de red)</strong><p>El recargo de red legal conforme al art. 35 de la Ley de Energía: 2.30 Rp./kWh para fomentar las energías renovables y la rehabilitación ecológica de la energía hidroeléctrica. Lo fija la Confederación y es igual para todos los consumidores finales.</p></div>
<div data-exp="stromreserve"><strong>Reserva de electricidad</strong><p>Tarifa de Swissgrid para financiar la reserva de invierno: reserva hidroeléctrica, centrales eléctricas de reserva, grupos electrógenos de emergencia. Una consecuencia de la crisis energética de 2022. 0.41 Rp./kWh.</p></div>
<div data-exp="solidarisiert"><strong>Costes solidarios</strong><p>Costes repartidos en toda Suiza para refuerzos de red (por ejemplo, para la conexión de instalaciones solares) y medidas de apoyo. La partida más pequeña: 0.05 Rp./kWh.</p></div>
<div data-exp="effizienz"><strong>Fomento de la eficiencia energética</strong><p>Gravamen cantonal o municipal para asesoramiento energético y programas de fomento, 0.16 Rp./kWh. Según el municipio, también pueden figurar aquí tasas de concesión adicionales.</p></div>
<div data-exp="netto"><strong>Importe neto</strong><p>Suma de todas las partidas antes del IVA. Para este hogar modelo: unos 22.5 Rp. por cada kWh consumido, de los que solo unos 10.4 Rp. corresponden realmente a la energía.</p></div>
<div data-exp="mwst"><strong>IVA</strong><p>8.1 por ciento sobre el importe neto, sobre todas las partidas incluidos los gravámenes estatales. Es decir: también se cobra IVA sobre los gravámenes.</p></div>
<div data-exp="total"><strong>Importe de la factura</strong><p>El importe final se redondea a 5 céntimos, por lo que difiere unos pocos céntimos de la suma exacta. EKZ indica por separado la diferencia de redondeo.</p></div>
</div>
</div>

<style>
</style>

<script>
</script>

## Los cuatro bloques en detalle

Para quienes prefieran leer texto corrido (y para los motores de búsqueda), aquí se explican las mismas partidas con más detalle.

### Suministro de energía: el único bloque con elección de producto

El suministro de energía es la electricidad propiamente dicha. En el suministro básico, en el que se encuentra la gran mayoría de los hogares, el proveedor local fija la tarifa cada año y la ElCom la revisa. En EKZ, el producto estándar se llama «EKZ Energie Erneuerbar» y cuesta en 2026 13.30 Rp./kWh en el semestre de invierno y 9.00 Rp./kWh en el de verano (sin IVA).

Es notable lo que ha desaparecido en 2026: la tarifa de punta y de valle. En lugar de «caro durante el día, barato por la noche», ahora se aplica una tarifa única que cambia cada trimestre. Por tanto, el consejo clásico de poner la lavadora por la noche ha quedado obsoleto desde el punto de vista tarifario; cuenta la estación, no la hora. Quien quiera mayor precisión puede cambiar a la tarifa opcional dinámica, en la que el precio sigue cada hora el precio bursátil.

A ello se añade una tarifa básica fija de CHF 3.00 al mes.

### Uso de la red: el monopolio regulado

El uso de la red paga la construcción, operación y mantenimiento de las líneas, estaciones transformadoras y subestaciones. No se puede cambiar de operador de red; como contrapartida, las tarifas están reguladas y la ElCom puede revisarlas.

Sin embargo, dentro del monopolio existen opciones que pueden merecer la pena. En 2026, EKZ ofrece tres productos de red para hogares:

| Producto de red | Tarifa de consumo (sin IVA) | Condición |
| --- | --- | --- |
| EKZ Netz 400ST (estándar) | 7.95 Rp./kWh | ninguna |
| EKZ Netz 400F | 7.50 Rp./kWh | EKZ puede controlar las cargas flexibles (calentador de agua, bomba de calor) de forma favorable para la red |
| EKZ Netz 400WP | 6.45 Rp./kWh | aplicaciones de calefacción con control |

Quien permita el control de su calentador de agua ahorra, por tanto, casi medio céntimo por kilovatio hora frente a la tarifa estándar. Y desde 2026 hay una segunda palanca en esta partida: quien se una a una comunidad eléctrica local (LEG) recibe un descuento legal del 20 o 40 por ciento sobre la tarifa de consumo del uso de la red para la electricidad comercializada localmente. Qué es una LEG se explica en el [artículo específico](/blog/lokale-elektrizitaetsgemeinschaft-leg-erklaert); y si sale a cuenta, [en el siguiente](/blog/lohnt-sich-leg-beitritt).

La partida «servicios de sistema» (0.27 Rp./kWh) se destina a Swissgrid para la estabilidad del sistema global: mantenimiento de frecuencia, energía de regulación, capacidad de arranque en negro.

### Medición: ahora visible

Desde 2026, EKZ indica los costes de medición por separado: CHF 5.00 al mes por el contador, la transmisión de datos y la puesta a disposición de los valores medidos. Antes estaban incluidos de forma invisible en el uso de la red. El contador inteligente que se paga aquí mide con precisión de quince minutos y es la base técnica de las novedades actuales del mercado eléctrico: tarifas dinámicas, facturación LEG y desplazamiento de carga.

### Recargos y gravámenes: el bloque estatal

Cuatro partidas sobre las que ni el proveedor ni el operador de red tienen influencia:

- **Gravámenes federales** (2.30 Rp./kWh): el recargo de red conforme a la Ley de Energía financia el fomento de las energías renovables y la rehabilitación de la energía hidroeléctrica.
- **Reserva de electricidad** (0.41 Rp./kWh): la prima de seguro del país contra situaciones de escasez de electricidad, introducida tras el invierno de 2022/23. Financia la reserva hidroeléctrica y las centrales eléctricas de reserva.
- **Costes solidarios** (0.05 Rp./kWh): refuerzos de red repartidos en toda Suiza, por ejemplo para conexiones de instalaciones solares.
- **Fomento de la eficiencia energética** (0.16 Rp./kWh): programas de fomento cantonal y municipal y asesoramiento energético. Dependiendo del lugar de residencia, se añaden tasas de concesión municipales.

Finalmente, al pie de la factura, el IVA: 8.1 por ciento sobre todo, también sobre los gravámenes.

## Lo que queda por kilovatio hora

Si se convierte la factura modelo a un único kilovatio hora (trimestre de verano, tarifa 400F), el resultado es aproximadamente el siguiente: 9.0 Rp. de energía, 7.8 Rp. de red y servicios de sistema, 2.9 Rp. de gravámenes, más la parte proporcional de la tarifa básica y de medición, así como el IVA. Resultado: de unos 21 a 22 céntimos por kWh, ni siquiera la mitad corresponde a la electricidad propiamente dicha. Quien debate sobre los precios de la electricidad debate a medias sobre la red y el Estado.

Precisamente por eso vale la pena fijarse en las partidas: la mayor palanca individual de un hogar es el propio consumo; después vienen la elección del producto de red, el producto energético y, ahora, la LEG. Para esta última hemos creado una [calculadora de precios](/tools/leg-rechner) que continúa el cálculo de este artículo.

*Todas las tarifas: EKZ 2026, sin IVA, fuente: recopilación de tarifas de EKZ 2026. Otras zonas de red tienen otros precios, pero la misma estructura de factura.*
