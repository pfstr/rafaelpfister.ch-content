---
title: "Lo que podemos aprender de las ciencias naturales para la resolución de problemas de TI"
navTitle: "Experimentos controlados"
description: "Falsabilidad, grupo de control, variables de confusión y sesgo de muestreo: el método con el que las ciencias naturales llevan siglos trabajando resuelve precisamente los problemas en los que la resolución de problemas de TI fracasa regularmente. Con ejemplos prácticos del flujo de correo."
date: "2026-08-11"
kategorie: "SMTP / Flujo de correo"
timeToRead: "15 min de lectura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
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
url: https://rafaelpfister.ch/es/blog/lo-que-podemos-aprender-de-las-ciencias-naturales-para-la-resolucion-de-problemas-de-ti
translationSourceHash: d2466d0e63e5b08052fe7a47766ec2500b94c84097bfcfe91f8f6348cd6d1cc2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:20:36.414Z
translationReview: automatic
---

# Lo que podemos aprender de las ciencias naturales para la resolución de problemas de TI

Un mensaje no llega. El registro proporciona un mensaje de error que sugiere inmediatamente una explicación. Comprueba esa explicación, encuentra indicios y, dos horas después, resulta que la explicación era errónea y los indicios, pura casualidad.

No es un error de principiante, sino la norma. Y resulta notable que nuestro sector rara vez disponga de un método para este problema, aunque existe uno desde hace siglos y funciona extraordinariamente bien. Las ciencias naturales tienen exactamente la misma tarea: inferir causas a partir de observaciones, en sistemas que no se pueden abarcar por completo.

Este artículo aplica cinco principios del método científico a la resolución de problemas en el flujo de correo. Los ejemplos proceden de la práctica, pero el enfoque no es específico del correo.

## Por qué la resolución de problemas de TI es sistemáticamente vulnerable

El flujo de correo es una cadena de sistemas, cada uno con su propia visión del mismo mensaje: la puerta de enlace, la capa de filtrado, el servidor de transporte local, el servicio en la nube y el buzón de destino. Cada mensaje se redacta desde la perspectiva de una sola capa.

Además, los textos de error son términos genéricos. La misma redacción describe a menudo situaciones completamente distintas porque el sistema que rechaza solo conoce una clasificación aproximada. Los códigos de estado mejorados están diseñados precisamente para formar categorías, no para nombrar casos individuales.

Un ejemplo: un servicio en la nube rechazó un mensaje indicando que el remitente no estaba autorizado para la entrega saliente. La misma redacción apareció en el mismo entorno en dos situaciones fundamentalmente distintas. En una, un sistema intentaba entregar a un destinatario externo a través del servicio, es decir, un auténtico intento de retransmisión hacia fuera. En la otra, el destinatario era un buzón normal del servicio y solo se objetaba el dominio del remitente.

Quien tome el texto literalmente buscará lo mismo en ambos casos. Y como aparece la palabra «saliente», primero buscará en el extremo equivocado.

## Principio 1: Una hipótesis debe prohibir algo

Karl Popper enriqueció la filosofía de la ciencia con una idea que resulta inmediatamente práctica para la resolución de problemas: **Una afirmación solo es útil si es refutable.** Una explicación que encaja con cualquier resultado de observación imaginable no explica nada.

Aplicado a este contexto: formule su suposición de modo que incluya una **predicción** que pueda ser falsa. No «hay algún problema con el dominio del remitente», sino «si envío el mismo mensaje con otro dominio de remitente por la misma ruta, llegará».

La segunda formulación vale algo porque se puede destruir en cinco minutos. La primera se puede alimentar con indicios durante horas sin llegar a saber más.

Una buena prueba: antes del intento, pregúntese qué resultado **refutaría** su hipótesis. Si no se le ocurre ninguno, no tiene una hipótesis, sino una impresión.

## Principio 2: Una variable; por lo demás, todo igual

El núcleo del experimento es el control de las variables de confusión. En la práctica, ocurre regularmente lo contrario: se comparan dos casos disponibles por casualidad. Y casi siempre difieren simultáneamente en varias características.

De un caso real: los mensajes de `example-test.com` fueron rechazados; los mensajes de `partner.example` llegaban. Los dos dominios diferían en al menos cuatro características: su pertenencia a la organización, dónde se aloja el correo, si existe una política de autenticación estricta y la ruta de envío. De dos puntos de datos con cuatro diferencias no se puede concluir exactamente nada. Cualquiera de las cuatro explicaciones encaja.

Por tanto, construya usted mismo la comparación. Mismo punto de envío, mismo destinatario, misma ruta, mismo momento y **exactamente una** característica modificada. Si sospecha del dominio del remitente, cambie solo ese aspecto.

## Principio 3: Sin un experimento de control, el resultado no vale nada

Esta es la parte que uno preferiría omitir, y es la más importante. En la investigación clínica, el grupo de control es algo evidente; en TI normalmente se prescinde de él y luego uno se sorprende de los resultados contradictorios.

**Su configuración de prueba debe reproducir primero el error.** Si no puede generar el caso de error con sus propios medios, un contraensayo exitoso no dice nada. Quizá su mensaje de prueba solo funciona porque lo envía desde otro punto distinto al sistema original, o porque una comprobación ni siquiera se aplica en su ruta.

Por ello, una prueba útil consta de al menos dos mensajes:

| | Propósito | Expectativa |
|---|---|---|
| Intento 1 | Control, replica el caso original | **debe fallar** |
| Intento 2 | Hipótesis, una variable modificada | debe funcionar |

Si el intento 1 no falla, su configuración no es representativa. Entonces no ha aprendido nada sobre el caso original, sino solo sobre su configuración de prueba, y debe enviar más cerca del origen.

## Un ejemplo práctico completo

Volvamos al caso anterior, anonimizado. Los mensajes de un sistema no llegaban a destinatarios en la nube; otros mensajes dirigidos a los mismos destinatarios llegaban sin problemas. Tres intentos por la misma ruta, al mismo destinatario, con pocos minutos de diferencia:

| Intento | Dominio del remitente | Hipótesis que comprueba | Resultado |
|---|---|---|---|
| 1 (control) | `example-test.com` | La configuración es representativa | Rechazo, idéntico al original |
| 2 | `example.com`, dominio propio del destino | El problema está en el dominio del remitente | entregado |
| 3 | `other-test.com`, dominio externo de la misma organización | El problema está en la pertenencia a la organización | entregado |

El intento 1 reprodujo el error, por lo que la configuración era válida. El intento 2 mostró que dependía del dominio del remitente, y no del destinatario, el buzón, el enrutamiento o los permisos. El intento 3 fue el realmente elegante: comprobó de forma específica la explicación alternativa más evidente y **la refutó**, pues `other-test.com` pertenecía a la misma organización y, aun así, pasó.

Tres mensajes, diez minutos, y la causa quedó demostrada en lugar de meramente supuesta. Antes se habían invertido varias horas en intentos de explicación, de los que al final ninguno se sostuvo.

## Principio 4: Refutar es el verdadero avance

Una hipótesis refutada se siente como un retroceso. En realidad, es lo único que sabe con certeza. Las confirmaciones son débiles, pues una observación puede encajar con varias explicaciones. Una refutación limpia elimina toda una rama del espacio de búsqueda, y lo hace de forma permanente.

Aquí es precisamente donde el sesgo de confirmación tiene mayor efecto. Cuando tiene una sospecha, casi siempre encuentra algo que encaja con ella. En el análisis descrito anteriormente, existía una correlación entre el rechazo y el lugar donde el dominio del remitente alojaba su correo. Parecía convincente, pero se basaba en dos puntos de datos que diferían en varias características. El tercer intento la desmontó.

Por ello, anote las explicaciones refutadas junto con el motivo por el que fueron descartadas. No es más que un cuaderno de laboratorio. Tiene dos efectos: quien asuma el caso más tarde no recorrerá los mismos callejones sin salida. Y usted mismo notará cuando piense en círculos, porque una idea ya descartada vuelve con otro nombre.

En la documentación, los puntos refutados deben figurar explícitamente junto a los demostrados. Un informe que solo contiene la respuesta correcta oculta la mitad del trabajo e invita a repetirlo.

## Principio 5: Conozca su muestra

La fuente de error más sutil es el sesgo de muestreo, y en TI afecta sobre todo a las consultas que devuelven resultados paginados.

Consulta siete días de seguimiento de mensajes, filtra localmente por una característica y no obtiene ningún resultado. La conclusión parece evidente: ese tráfico no existió. En realidad, solo filtró la primera página, y con un volumen alto esta cubre apenas unos minutos.

El resultado correcto es: no se encontró en el fragmento. No es: no existe. La diferencia es la misma que entre «en nuestro estudio no se puede demostrar ningún efecto» y «no existe ningún efecto».

Hay dos soluciones. Reduzca la ventana temporal hasta que una página la cubra por completo, lo que se reconoce porque deja de aparecer el aviso de más resultados. O recorra todas las páginas y evalúe después.

Y una tercera, que a menudo se pasa por alto: para la cuestión de si algo **nunca** ocurre, una comprobación de configuración es superior a cualquier observación. Si un sistema no tiene ninguna ruta a un destino, no puede entregar allí, independientemente de cualquier ventana de observación. Esa es la diferencia entre un argumento empírico y uno estructural; cuando pueda disponer del estructural, úselo.

## La transferencia: vincular la carga de la prueba a la reversibilidad

Aquí termina la analogía con la ciencia y toma el relevo la perspectiva de ingeniería. La investigación busca la verdad; la operación busca una instalación funcional. De ello se deriva un criterio que la ciencia no conoce: **El esfuerzo de demostración depende de la reversibilidad de la intervención.**

Desactivar un conector es un comando, y revertirlo también. Para ello bastan indicios fundamentados, pues un error se corrige en un minuto y se detecta de inmediato. Eliminar ese mismo conector no es reversible; para ello merece la pena una comprobación adicional mediante la configuración de la contraparte o un informe de uso del servidor.

Lo mismo se aplica a los cambios de reglas. Puede introducir una fase puramente de observación, que registra y no redirige nada, con una base factual limitada. No tiene consecuencias y obtiene precisamente los datos que faltan para el paso decisivo. Solo el cambio que puede retener mensajes exige pruebas sólidas.

Quien no aplique este criterio comete regularmente ambos errores a la vez: exige semanas de pruebas para un cambio que podría revertirse en segundos y activa sin respaldo algo que puede detener el tráfico de correo.

## Cuándo puede dejar de investigar

Hay un punto en el que seguir profundizando ya no aporta valor: cuando la solución está clara, pero el mecanismo sigue sin estarlo.

En el ejemplo anterior, tras tres intentos quedó demostrado que el dominio del remitente era el desencadenante, que todo lo demás en la ruta de correo funcionaba y que no había un problema más amplio. Por qué el servicio en la nube decide internamente de ese modo quedó abierto. Para la corrección era irrelevante, porque correspondía a la aplicación remitente.

Por ello, separe deliberadamente dos preguntas. ¿Qué debo cambiar para que funcione? ¿Y por qué se comporta así el sistema? Debe responder la primera; la segunda puede remitirla al fabricante. Un caso de soporte con tres intentos controlados, marcas de tiempo, identificadores de mensajes y un contraejemplo funcional es, de todos modos, muchísimo más valioso que una descripción del síntoma.

Por cierto, este es también el punto en el que ciencia y operación pueden separarse limpiamente. La ciencia no puede abandonar la pregunta sobre el mecanismo. La operación debe priorizarla.

## La versión breve

Formule hipótesis que puedan fallar y pregúntese de antemano qué resultado las refutaría. Nunca compare dos casos disponibles por casualidad; construya la comparación con exactamente una variable modificada. Reproduzca el error en el experimento de control antes de creer en el contraensayo. Considere las refutaciones como avances y regístrelas por escrito. En cada consulta, compruebe si está viendo el conjunto completo o una muestra. Y ajuste la profundidad de prueba exigida a la facilidad con que pueda revertirse la intervención prevista.

Las consultas concretas se encuentran en [Analizar el flujo de correo de Exchange: seguimiento de mensajes, registros SMTP y conectores de recepción](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Quien prefiera seleccionar los comandos en lugar de escribirlos, los encontrará en el [Generador de comandos](https://rafaelpfister.ch/tools/command-builder).

## Fuentes

1.  [Karl Popper: La lógica de la investigación](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): origen del principio de falsación, según el cual una afirmación solo es científica si sigue siendo refutable.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): explica por qué los códigos de estado mejorados son deliberadamente categorías generales y permiten el mismo código para causas distintas.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): tipos de eventos y campos, base para determinar el último paso de procesamiento.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): lógica de paginación del seguimiento de mensajes, que favorece errores de muestreo.
