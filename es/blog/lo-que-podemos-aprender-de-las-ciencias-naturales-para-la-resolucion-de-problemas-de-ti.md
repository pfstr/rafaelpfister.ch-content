---
title: "Lo que podemos aprender de las ciencias naturales para la resolución de problemas de TI"
navTitle: "Experimentos controlados"
description: "Falsabilidad, grupo de control, variables de confusión y sesgo de muestreo: el método con el que las ciencias naturales han trabajado durante siglos resuelve precisamente los problemas en los que la resolución de problemas de TI fracasa habitualmente, ilustrado con ejemplos del flujo de correo."
date: "2026-08-11"
kategorie: "SMTP / Flujo de correo"
timeToRead: "15 min de lectura"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - einliefernde-ip-adressen-aggregieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "lo-que-podemos-aprender-de-las-ciencias-naturales-para-la-resolucion-de-problemas-de-ti"
translationId: "article-098ed40e6d027b8b"
draft: false
translationOf: mailflow-fehlersuche-kontrollierte-experimente
translationSourceHash: 1759c709160723c161883fede66d48cd968bb6a587cb49c73781b3a39498cd66
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:30:17.634Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/lo-que-podemos-aprender-de-las-ciencias-naturales-para-la-resolucion-de-problemas-de-ti
---

# Lo que podemos aprender de las ciencias naturales para la resolución de problemas de TI

Un mensaje no llega. El protocolo proporciona un mensaje de error que sugiere inmediatamente una explicación. Usted comprueba esa explicación, encuentra indicios y, dos horas después, resulta que la explicación era errónea y los indicios, casualidad.

No es un error de principiante, sino la norma. Y llama la atención que nuestro sector rara vez disponga de un método para este problema, aunque existe uno desde hace siglos y funciona extraordinariamente bien. Las ciencias naturales tienen exactamente la misma tarea: inferir causas a partir de observaciones en sistemas que no pueden abarcarse por completo.

Este artículo aplica cinco principios básicos del método científico a la resolución de problemas en el flujo de correo. Los ejemplos proceden de la práctica, pero el enfoque no es específico del correo.

## Por qué la resolución de problemas de TI es sistemáticamente vulnerable

El flujo de correo es una cadena de sistemas que tienen cada uno su propia visión del mismo mensaje: la puerta de enlace, la capa de filtrado, el servidor de transporte local, el servicio en la nube y el buzón de destino. Cada mensaje está redactado desde la perspectiva de una sola capa.

A esto se suma que los textos de error son términos genéricos. La misma redacción suele describir situaciones completamente distintas, porque el sistema que rechaza solo conoce una clasificación aproximada. Los códigos de estado extendidos están concebidos precisamente para formar clases, no para nombrar casos individuales.

Un ejemplo: un servicio en la nube rechazó un mensaje indicando que el remitente no estaba autorizado para la entrega saliente. La misma redacción apareció en el mismo entorno en dos configuraciones fundamentalmente distintas. En una ocasión, un sistema intentaba entregar a un destinatario externo a través del servicio; era, por tanto, un verdadero intento de retransmisión hacia el exterior. En la otra, el destinatario era un buzón normal del servicio y se objetaba exclusivamente el dominio del remitente.

Quien toma el texto literalmente busca lo mismo en ambos casos. Y, como aparece la palabra «saliente», primero busca en el lugar equivocado.

## Principio 1: Una hipótesis debe excluir algo

Karl Popper enriqueció la filosofía de la ciencia con una idea de aplicación inmediata a la resolución de problemas: **una afirmación solo es útil si puede refutarse.** Una explicación que encaja con cualquier resultado de observación imaginable no explica nada.

Aplicado a este contexto: formule su sospecha de forma que contenga una **predicción** que pueda resultar falsa. No «hay algo mal con el dominio del remitente», sino «si envío el mismo mensaje con otro dominio de remitente por la misma ruta, llegará».

La segunda formulación tiene valor porque puede refutarse en cinco minutos. La primera puede alimentarse durante horas con indicios sin llegar a saber más.

Una buena prueba: antes del intento, pregúntese qué resultado **refutaría** su hipótesis. Si no se le ocurre ninguno, no tiene una hipótesis, sino una impresión.

## Principio 2: Una variable; todo lo demás igual

El núcleo del experimento es el control de las variables de confusión. En la práctica ocurre habitualmente lo contrario: se comparan dos casos disponibles por casualidad. Y casi siempre difieren simultáneamente en varias características.

De un caso real: los mensajes de `example-test.com` eran rechazados; los mensajes de `partner.example` llegaban. Los dos dominios diferían en al menos cuatro características: pertenencia a la organización, dónde se aloja el correo, si existe una directiva de autenticación estricta y la ruta de envío. De dos puntos de datos con cuatro diferencias no puede inferirse exactamente nada. Cualquiera de las cuatro explicaciones encaja.

Por eso, construya usted mismo la comparación. Mismo punto de entrega, mismo destinatario, misma ruta, misma hora y **exactamente una** característica modificada. Si sospecha del dominio del remitente, cambie solo ese elemento.

## Principio 3: Sin prueba de control, el resultado no vale nada

Esta es la parte que más apetece omitir y la más importante. En la investigación clínica, el grupo de control es algo evidente; en TI se suele prescindir de él y luego uno se sorprende de los resultados contradictorios.

**Su configuración de prueba debe reproducir primero el error.** Si no puede generar el caso de error con sus propios medios, un intento alternativo exitoso no dice nada. Quizá su mensaje de prueba funcione únicamente porque lo entrega en otro punto que el sistema original o porque una comprobación ni siquiera se aplica en su ruta.

Por eso, una prueba útil consta de al menos dos mensajes:

| | Propósito | Resultado esperado |
|---|---|---|
| Intento 1 | Control, reproduce el caso original | **debe fallar** |
| Intento 2 | Hipótesis, una variable modificada | debe tener éxito |

Si el intento 1 no falla, su configuración no es representativa. Entonces no ha aprendido nada sobre el caso original, sino solo sobre su configuración de prueba, y debe entregar el mensaje más cerca del origen.

## Un ejemplo desarrollado paso a paso

Volvamos al caso anterior, anonimizado. Los mensajes de un sistema no llegaban a destinatarios en la nube; otros mensajes dirigidos a los mismos destinatarios llegaban sin problemas. Tres intentos por la misma ruta, al mismo destinatario y con pocos minutos de diferencia:

| Intento | Dominio del remitente | Hipótesis que comprueba | Resultado |
|---|---|---|---|
| 1 (control) | `example-test.com` | La configuración es representativa | Rechazo, idéntico al original |
| 2 | `example.com`, dominio propio del destino | Se debe al dominio del remitente | Entregado |
| 3 | `other-test.com`, dominio externo de la misma organización | Se debe a la pertenencia a la organización | Entregado |

El intento 1 reprodujo el error, por lo que la configuración era válida. El intento 2 mostró que dependía del dominio del remitente y no del destinatario, el buzón, el enrutamiento o los permisos. El intento 3 fue el realmente elegante: comprobó específicamente la explicación alternativa más evidente y **la refutó**, pues `other-test.com` pertenecía a la misma organización y aun así pasó.

Tres mensajes, diez minutos y la causa quedó demostrada en vez de simplemente supuesta. Antes se habían invertido varias horas en intentos de explicación, ninguno de los cuales se sostuvo al final.

## Principio 4: Refutar es el verdadero avance

Una hipótesis refutada se siente como un retroceso. En realidad, es lo único que sabe con certeza. Las confirmaciones son débiles, pues una observación puede encajar con varias explicaciones. Una refutación limpia elimina toda una rama del espacio de búsqueda, y lo hace de forma permanente.

Precisamente aquí es donde el sesgo de confirmación actúa con más fuerza. Si tiene una sospecha, casi siempre encontrará algo que encaje con ella. En el análisis descrito anteriormente existía una correlación entre el rechazo y dónde alojaba su correo el dominio del remitente. Parecía convincente, pero se basaba en dos puntos de datos que diferían en varias características. El tercer intento la debilitó.

Por eso, anote las explicaciones refutadas junto con el motivo por el que se descartaron. No es otra cosa que un cuaderno de laboratorio. Tiene dos efectos: quien retome el caso más adelante no recorrerá los mismos callejones sin salida. Y usted mismo advertirá cuándo está pensando en círculos, porque una idea ya descartada vuelve con otro nombre.

En la documentación, los puntos refutados deben figurar explícitamente junto a los demostrados. Un informe que solo contiene la respuesta correcta oculta la mitad del trabajo e invita a repetirlo.

## Principio 5: Conozca su muestra

La fuente de error más sutil es el sesgo de muestreo, y en TI afecta sobre todo a las consultas que devuelven resultados paginados.

Consulta siete días de seguimiento de mensajes, filtra localmente por una característica y no obtiene ningún resultado. La conclusión obvia es que ese tráfico no existió. En realidad, solo ha filtrado la primera página, que, con un volumen elevado, cubre pocos minutos.

El resultado correcto es: no se encontró en el segmento analizado. No es: no existe. La diferencia es la misma que existe entre «en nuestro estudio no se puede demostrar ningún efecto» y «no hay ningún efecto».

Hay dos salidas que funcionan. Reduzca la ventana temporal hasta que una página la cubra por completo, algo reconocible por la ausencia del aviso de resultados adicionales. O recorra todas las páginas y analice después.

Y hay una tercera que a menudo se pasa por alto: para la pregunta de si algo **nunca** ocurre, una comprobación de configuración es superior a cualquier observación. Si un sistema no posee una ruta hacia un destino, no puede entregar allí, independientemente de cualquier ventana de observación. Esa es la diferencia entre un argumento empírico y uno estructural, y cuando puede disponer del estructural, utilícelo.

## La transferencia: vincular la carga de la prueba a la reversibilidad

Aquí termina la analogía con la ciencia y toma el relevo la perspectiva de ingeniería. La investigación busca la verdad; la operación busca una instalación funcional. De ello se deriva un criterio que la ciencia no conoce: **el esfuerzo necesario para demostrar algo depende de la reversibilidad de la intervención.**

Desactivar un conector es un comando, y revertirlo también. Para ello bastan indicios fundamentados, pues un error se corrige en un minuto y se detecta inmediatamente. Eliminar ese mismo conector no es reversible; para ello merece la pena aportar evidencia adicional mediante la configuración de la contraparte o un informe de uso del servidor.

Lo mismo se aplica a los cambios en las reglas. Puede introducir una fase de mera observación, que registra y no redirige nada, con una base de hechos limitada. No tiene consecuencias y obtiene precisamente los datos que faltan para el paso decisivo. Solo el cambio que puede retener mensajes exige pruebas sólidas.

Quien no aplique este criterio comete regularmente ambos errores a la vez: exige semanas de pruebas para un cambio que podría revertirse en segundos y activa sin protección algo que puede detener el tráfico de correo.

## Cuándo puede dejar de investigar

Hay un punto en el que seguir profundizando ya no aporta valor: cuando la solución está clara, pero el mecanismo sigue sin estarlo.

En el ejemplo anterior, tras tres intentos quedó demostrado que el dominio del remitente era el desencadenante, que todo lo demás funcionaba en la ruta de correo y que no había un problema más amplio. No quedó claro por qué el servicio en la nube decide exactamente así internamente. Para la corrección, eso no tenía importancia, pues recaía en la aplicación remitente.

Por tanto, separe conscientemente dos preguntas. ¿Qué debo cambiar para que funcione? ¿Y por qué se comporta así el sistema? Debe responder a la primera; la segunda puede dejarla en manos del fabricante. Un caso de soporte con tres intentos controlados, marcas de tiempo, identificadores de mensajes y un contraejemplo funcional es, de todos modos, muchísimo más valioso que una descripción del síntoma.

Por cierto, este es también el punto en el que ciencia y operación pueden separarse limpiamente. La ciencia no puede abandonar la pregunta sobre el mecanismo. La operación debe priorizarla.

## Resumen

Formule hipótesis que puedan fallar y pregúntese de antemano qué resultado las refutaría. Nunca compare dos casos disponibles por casualidad; construya la comparación con exactamente una variable modificada. Reproduzca el error en el intento de control antes de creer en el intento alternativo. Trate las refutaciones como avances y regístrelas por escrito. En cada consulta, compruebe si está viendo el conjunto completo o una muestra. Y ajuste la profundidad de la prueba exigida a la facilidad con la que pueda revertirse la intervención prevista.

Las consultas concretas se encuentran en [Analizar el flujo de correo de Exchange: seguimiento de mensajes, protocolos SMTP y conectores de recepción](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Quien prefiera generar los comandos haciendo clic en lugar de escribirlos encontrará todo en el [Generador de comandos](https://rafaelpfister.ch/tools/command-builder).

## Historial de cambios

- **25 de octubre de 2026:** enlaces de Fuentes actualizados a las nuevas direcciones de Mohr Siebeck y Microsoft Learn.

## Fuentes

1.  [Karl Popper: La lógica de la investigación científica](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161484100/): origen del principio de falsación, según el cual una afirmación solo es científica si sigue siendo refutable.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): explica por qué los códigos de estado extendidos son deliberadamente clases generales y permiten el mismo código para causas diferentes.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): tipos de eventos y campos, base para determinar el último paso de procesamiento.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): lógica de paginación del seguimiento de mensajes, que favorece errores de muestreo.
