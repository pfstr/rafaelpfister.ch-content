---
title: "Downdetector bajo crítica: lo que la semana de caídas en EE. UU. revela sobre los reportes de incidencias"
navTitle: "Críticas a Downdetector"
description: "Entre el 13 y el 17 de noviembre de 2026, Downdetector registró en Estados Unidos más incidencias que en cualquier otro momento del último año. Las caídas de FAA, AWS, Chase, United y Apple fueron reales, pero tuvieron causas independientes y en su mayoría se resolvieron en cuestión de horas. Las cifras de Downdetector reforzaron la imagen de una crisis interconectada. Qué mide la plataforma, dónde radican las críticas y cómo pueden los administradores interpretar mejor las incidencias."
date: "2026-11-18"
kategorie: "Operaciones"
timeToRead: "7 min de lectura"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-bajo-critica-lo-que-la-semana-de-caidas-en-ee-uu-revela-sobre-los-reportes-de"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/es/blog/downdetector-bajo-critica-lo-que-la-semana-de-caidas-en-ee-uu-revela-sobre-los-reportes-de
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:45:23.383Z
translationReview: required
---

# Downdetector bajo crítica: lo que la semana de caídas en EE. UU. revela sobre los reportes de incidencias

Del 13 al 17 de noviembre de 2026 fallaron varios sistemas importantes en Estados Unidos: la administración de aviación FAA, con una prohibición nacional de despegues; partes de AWS en la región us-east-1; el banco JPMorgan Chase; la facturación de United Airlines y, el martes, el inicio de sesión de iCloud y otros servicios de Apple. Seguí la serie en un [minuto a minuto](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). Al concluir la semana, queda claro que las caídas tuvieron causas independientes, que no hay indicios de un ciberataque y que Europa solo se vio afectada de forma marginal. Sin embargo, en la percepción pública surgió la imagen de una Internet que falla simultáneamente por todas partes. Gran parte de esta imagen procede de los gráficos de Downdetector, citados en casi todos los informes de prensa. Desde el lunes, estos gráficos son objeto de amplias críticas.

## Qué mide Downdetector

Downdetector pertenece a Ookla y, desde el 17 de junio de 2026, por tanto a Accenture. La plataforma recopila reportes de problemas de usuarios que informan de una incidencia en la página de un servicio y, según su propia descripción, los complementa con señales de redes sociales y otras fuentes en línea. Para cada servicio, Downdetector calcula una línea de base: el promedio de reportes a la misma hora del día durante el último año. Si la cifra actual es significativamente superior, se considera una incidencia. Downdetector no publica cómo pondera las fuentes ni a partir de qué factor considera que un pico constituye una incidencia.

De ello se derivan tres características que a menudo se pasan por alto en la cobertura informativa:

1.  **Los reportes no son usuarios afectados.** Una cifra como «20.000 reportes» no dice nada sobre cuántos clientes realmente no tenían acceso. Cuenta a las personas que detectaron un problema y accedieron a Downdetector.

2.  **Los usuarios reportan el servicio que ven.** Si falla una red de pagos, un proveedor de nube o un servicio de identidad, los reportes aparecen en los bancos, aplicaciones y tiendas que dependen de ellos. La causa no se aprecia en el gráfico.

3.  **La línea de base no refleja semanas excepcionales.** Si los medios informan continuamente sobre caídas, más personas comprueban ante cada retraso si un servicio está afectado. Cada nuevo titular incrementa el número de reportes, incluso para servicios que funcionan con normalidad.

## Las cifras de la semana

Según Axios, Downdetector recibió casi 6 millones de reportes de problemas procedentes de Estados Unidos durante los siete días hasta el 16 de noviembre, el valor semanal más alto en un año. Sin embargo, el mismo análisis también muestra que, considerando 30 días, hubo tres grandes caídas, cuando lo habitual son cinco. Por tanto, la cifra récord de reportes contrasta con un número de grandes incidencias reales más bien inferior a la media.

Varios casos de la semana muestran hasta qué punto pueden diferir los reportes de la situación real:

- **Bancos el 16 de noviembre:** Los medios escribieron, citando a Downdetector, que los usuarios reportaban problemas en «prácticamente todos los grandes bancos estadounidenses», entre ellos Chase, Wells Fargo y Capital One. Chase afirmó que algunas transacciones tardaban más; los clientes seguían teniendo acceso a sus cuentas y saldos. Según The New York Times, la causa fue un error de software en hardware propio de JPMorgan Chase, que podría haber afectado a otros bancos mediante transacciones compartidas. Los demás bancos no confirmaron incidencias propias.

- **Gmail el 16 y 17 de noviembre:** Downdetector mostró más de 2.800 reportes, principalmente sobre correos electrónicos no entregados. Google explicó que no había indicios de una incidencia generalizada y el Workspace Status Dashboard permaneció sin ninguna entrada.

- **Charles Schwab el 17 de noviembre:** Según The Wall Street Journal, los reportes aumentaron hacia las 9:50 hora local y volvieron a descender media hora después. No existe ningún aviso de incidencia de Schwab al respecto.

- **Exchange Online el 12 de noviembre:** Para la misma incidencia, Reuters mencionó más de 97.000 reportes y una emisora local casi 119.000, ambos en momentos distintos. Según Downdetector, los reportes no aumentaron hasta unos 68 minutos después del inicio indicado por Microsoft en el Admin Center.

Las caídas de FAA, AWS, Chase, United y Apple fueron reales y confirmadas por los operadores. Lo exagerado fue vincularlas: de cinco incidencias independientes y un aumento general de los reportes surgió la narrativa de un ataque coordinado. La portavoz de la Casa Blanca dijo el 17 de noviembre que no había ninguna crisis de la que los estadounidenses debieran preocuparse. Liz Fong-Jones (Honeycomb) dijo a Axios que los incidentes tenían muy poco en común.

## Dónde se centra la crítica

Las críticas proceden de varias direcciones. Operadores como Google contradicen públicamente a Downdetector cuando este indica una incidencia que no observan en sus propios datos. Los medios especializados señalan que Downdetector cuenta reportes en lugar de usuarios afectados, muestra las incidencias con retraso y no registra en absoluto fallos en componentes que los clientes finales no ven. En debates entre desarrolladores, por ejemplo en Hacker News, también circula la hipótesis de que incluso abrir una página como «¿Está Chase caído?» cuenta como un reporte implícito y que, de ese modo, los picos se transfieren de un servicio a otros similares. Downdetector no ha confirmado esta hipótesis; solo está demostrado que la plataforma evalúa otras fuentes en línea además de los reportes activos y no revela su ponderación.

Los defensores de la plataforma replican que Downdetector a menudo muestra incidencias antes que las páginas de estado de los proveedores. Durante la caída de Apple del 17 de noviembre, la página de estado de Apple aún mostraba disponibilidad normal cuando los reportes ya aumentaban considerablemente. Como alerta temprana, el servicio es útil. No lo es como medida de la magnitud de una incidencia, y precisamente así se utilizó esa semana: titulares como «Miles reportan problemas» aparecían sin contexto junto a caídas confirmadas.

## Interpretar mejor las incidencias

Para los administradores que, ante una incidencia reportada, deben decidir si informan a los usuarios o inician medidas, se desprende un orden sencillo:

1.  **Comprobar la página de estado y el portal de administración del proveedor.** Para Microsoft 365, se trata de Service Health en el Admin Center, no de la página de estado pública. AWS, Google Cloud, Cloudflare y Apple mantienen sus propias páginas de estado con marcas de tiempo.

2.  **Recurrir a mediciones propias.** Una prueba sintética sencilla (inicio de sesión, envío de correo, consulta de una página) desde la propia red muestra si la propia organización está afectada. Mi [monitor de conectividad de M365](https://rafaelpfister.ch/liveticker) comprueba cada cinco minutos la disponibilidad de los puntos de conexión de Microsoft 365 desde Europa.

3.  **Leer Downdetector solo como indicio.** Un pico es un motivo para comprobar la situación. La cifra absoluta y los picos simultáneos en varios servicios similares dicen poco sobre la causa y la magnitud.

4.  **Tener en cuenta el origen regional.** La mayoría de las cifras de los informes de prensa se refieren al sitio estadounidense de Downdetector. Para los usuarios en Suiza, solo son relevantes si el proveedor también confirma la incidencia para las regiones europeas.

## El minuto a minuto

He cerrado el minuto a minuto sobre la serie de caídas con este artículo. Permanece en línea como crónica de la semana, con una conclusión final sobre las causas individuales y la indicación de que Europa no se vio afectada, salvo en los vuelos transatlánticos y la incidencia mundial de iCloud.

## Fuentes

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): casi 6 millones de reportes de Downdetector en siete días, tres grandes caídas en 30 días en lugar de las cinco habituales, valoración de Liz Fong-Jones.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): funcionamiento, línea de base por hora del día, adquisición por Accenture, cifras divergentes sobre la incidencia de Exchange Online del 12 de noviembre.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): reportes de Downdetector en varios bancos estadounidenses, declaración de Chase.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): cita a The New York Times y The Wall Street Journal sobre Apple, Charles Schwab y la causa en JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): reportes de Downdetector sobre Gmail y desmentido de Google.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): resumen de las explicaciones de los operadores.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): desarrollo de la caída de Apple, página de estado por detrás de los reportes.

8.  [Hacker News: Discusión sobre la metodología de Downdetector](https://news.ycombinator.com/item?id=50521819): hipótesis sobre reportes implícitos y transferencia entre servicios similares, no confirmadas por Downdetector.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): valoración de un experto en seguridad, sin indicios de relación entre los incidentes.
