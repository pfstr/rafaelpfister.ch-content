---
title: "Brave Web Discovery Project: por qué las partes largas de palabras en las URL excluyen páginas"
navTitle: "Brave: partes largas de URL"
description: "Brave descarta en el Web Discovery Project cualquier URL cuya ruta contenga una parte de palabra de más de 18 caracteres. Los compuestos alemanes como Verschlüsselungsgateway quedan afectados. Como la búsqueda web de Claude se basa en el índice de Brave, esto también afecta a la visibilidad en Claude. La regla en el código fuente, otras heurísticas, el papel de la URL canónica, un script para comprobar el propio sitemap, una medición con 825 términos en 24 idiomas y una valoración de por qué la regla perjudica a los idiomas con palabras compuestas."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min de lectura"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-por-que-las-partes-largas-de-palabras-en-las-url-excluyen-paginas"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/es/blog/brave-web-discovery-project-por-que-las-partes-largas-de-palabras-en-las-url-excluyen-paginas
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T08:59:09.615Z
translationReview: required
---

# Brave Web Discovery Project: por qué las partes largas de palabras en las URL excluyen páginas

La búsqueda web de Claude ofrece sus resultados desde el índice de Brave Search. Anthropic incluye a Brave Search como subencargado desde marzo de 2025, y las citas en las respuestas de Claude coinciden en gran medida con los resultados de Brave. Por tanto, quien quiera ser citado en respuestas de Claude debe figurar en el índice de Brave. Los envíos mediante Google Search Console, Bing Webmaster Tools o IndexNow no llegan directamente a Brave.

Brave alimenta su índice a partir de dos fuentes: un rastreador propio y el Web Discovery Project (WDP). El WDP es una función opcional del navegador Brave que comunica de forma anonimizada a Brave las páginas visitadas. Antes de que el navegador comunique una página, comprueba la URL mediante una serie de heurísticas. Una de ellas afecta especialmente a los sitios web en alemán: si la ruta contiene una parte de palabra de más de 18 caracteres, la página nunca se comunica.

## Lo que comunica el Web Discovery Project

El WDP recopila dos tipos de datos: consultas de búsqueda en Google, Bing, Brave y algunos otros motores de búsqueda junto con la lista de resultados, y visitas a páginas con URL, título, tiempo de permanencia e interacción. No hay ID de usuario; cada comunicación se envía a Brave individualmente, cifrada y distribuida aleatoriamente en el tiempo.

Para que no se comuniquen contenidos privados, el navegador clasifica una página como privada cuando

- una segunda solicitud sin cookies devuelve una página claramente distinta (inicio de sesión, personalización),
- el dominio apunta a una dirección IP privada,
- la página lleva `noindex`,
- la URL parece un enlace secreto (URL de capacidad, por ejemplo, un enlace compartido con token).

La última comprobación la realiza la función `dropLongURL`. El navegador almacena permanentemente en un filtro Bloom una URL clasificada como privada y no vuelve a comprobarla.

## La regla: ninguna parte de palabra de más de 18 caracteres

`dropLongURL` divide la ruta en barras, puntos, guiones bajos, espacios, guiones, dos puntos, signos más y puntos y comas, y compara cada parte con el límite `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` está establecido en el código fuente como `18`. `return true` significa: la URL se considera sospechosa. Se comprueba la URL decodificada: el navegador codifica porcentualmente los caracteres no ASCII (`%C3%BC`), y el WDP los convierte antes de la comprobación mediante `cleanCurrentUrl` (`decodeURIComponent`) de vuelta. Se cuentan caracteres de JavaScript, es decir, unidades de código UTF-16: una `ü` o un carácter chino cuenta una sola vez; un carácter fuera del plano básico de Unicode o un acento añadido como carácter independiente cuenta dos veces. La regla se aplica en ambos modos de comprobación, el normal y el estricto. El propio Brave describe las heurísticas en el README como conservadoras: muchas páginas públicas se clasifican erróneamente como posible enlace secreto, algo que se acepta para el propósito del WDP.

El guion separa, una palabra escrita junta no. Por tanto, lo decisivo es la longitud de la palabra individual más larga en el slug, no la longitud de todo el slug:

| Slug | Parte más larga | Resultado |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | se comunica |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | se descarta |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | se descarta |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | se comunica |

## Por qué los slugs alemanes se ven especialmente afectados

Los términos técnicos ingleses se escriben separados; los alemanes, unidos. A esto se suma la transcripción de las diéresis: `ü` se convierte en `ue`, por lo que cada palabra con diéresis se alarga. `Zertifikatserneuerung` tiene 21 caracteres; `Verschlüsselungsgateway` tiene 24 en transcripción ASCII. En las lenguas escandinavas ocurre algo similar: el sueco y el noruego también escriben juntos los compuestos.

En este sitio web, según un análisis del sitemap, están afectadas 24 de 750 URL: tres artículos alemanes, uno inglés y 20 traducciones suecas o noruegas. El caso inglés procede del campo de encabezado `MessageDirectionality`, que se incorporó como palabra al slug.

## Cuándo ayuda la URL canónica

Si la URL solicitada falla en `dropLongURL`, el navegador comprueba la URL canónica de la página. Si esta supera la comprobación, comunica la página bajo la URL canónica. Esto cubre el caso típico de que se acceda a una página con parámetros de seguimiento, pero tenga una URL canónica limpia.

No ayuda cuando hay una parte de palabra demasiado larga en el slug: por regla general, la URL canónica es idéntica a la URL solicitada y falla por la misma regla. Si ambas fallan, el navegador descarta la página.

Que se aplique el modo de comprobación estricto también depende de la URL canónica: solo se relaja cuando la canónica difiere de la URL solicitada y es más corta. En una página que se indica a sí misma como canónica, se aplica el modo estricto.

## Otras heurísticas en dropLongURL

Además de la longitud de las partes de palabra, `dropLongURL` descarta una URL, entre otros casos, cuando tiene

- una cadena de consulta de más de 30 caracteres o más de cuatro parámetros (en modo estricto, desde 23 caracteres o dos parámetros),
- una secuencia de más de 12 dígitos en la ruta o la cadena de consulta (en modo estricto, más de 8); antes se eliminan los caracteres especiales, de modo que una ruta de fecha como `/2026/09/25/` cuenta como un número de ocho dígitos,
- partes de palabra que un clasificador de Markov clasifica como hash,
- segmentos de ruta como `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` o `/token`,
- una dirección de correo electrónico en la URL.

Para una URL de artículo normal sin parámetros, normalmente solo es relevante la longitud de las palabras.

## Contexto: quórum y rastreador

La regla solo afecta a la vía a través del WDP. Dos aspectos limitan su importancia:

- **Quórum:** Brave solo puede descifrar una comunicación de página cuando más de un determinado número de usuarios ha comunicado la misma URL desde redes distintas en un plazo de 30 días. El README no indica el umbral. Por ello, el WDP apenas es relevante para páginas con pocas visitas desde Brave.
- **Rastreador:** El rastreador de Brave funciona independientemente del WDP. No se presenta con un agente de usuario propio y sigue las reglas de robots.txt para Googlebot. Una URL con una parte de palabra larga puede llegar al índice a través del rastreador de todos modos.

Así, una parte de palabra larga priva a una página de una de las dos vías hacia el índice de Brave; sigue siendo accesible a través del rastreador.

## Comprobar las propias URL

Los siguientes comandos leen un sitemap y muestran cada URL cuya ruta contiene una parte de más de 18 caracteres. En el caso de un índice de sitemaps, compruebe los sitemaps individuales (`sitemap-0.xml` etc.) uno tras otro.

En Linux o macOS:

```bash
curl -s https://example.com/sitemap-0.xml \
  | grep -oE '<loc>[^<]+' \
  | sed -E 's#<loc>https?://[^/]*##' \
  | awk -F'[/._ :+;-]' '{
      for (i = 1; i <= NF; i++)
        if (length($i) > 18) { print length($i), $i, $0; break }
    }'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `curl -s` | Descarga el sitemap sin indicador de progreso. |
| `grep -oE '<loc>[^<]+'` | Muestra solo las entradas `<loc>` (`-o`), con expresiones regulares extendidas (`-E`). |
| `sed -E 's#…##'` | Elimina `<loc>`, el esquema y el nombre de host; queda la ruta. |
| `awk -F'[/._ :+;-]'` | Divide la ruta usando los mismos separadores que `dropLongURL`. |
| `length($i) > 18` | Muestra longitud, parte de palabra y ruta en cuanto una parte supera el límite. |

</details>

En Windows con PowerShell:

```powershell
$sitemap = Invoke-RestMethod -Uri 'https://example.com/sitemap-0.xml'
foreach ($loc in $sitemap.urlset.url.loc) {
    $pfad = [uri]::UnescapeDataString(([uri]$loc).AbsolutePath)
    $zuLang = $pfad -split '[/._ :+;-]' |
        Where-Object { $_.Length -gt 18 }
    if ($zuLang) {
        '{0}  ->  {1}' -f ($zuLang -join ', '), $pfad
    }
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Invoke-RestMethod -Uri` | Descarga el sitemap y lo devuelve directamente como objeto XML. |
| `$sitemap.urlset.url.loc` | Lee todas las entradas `<loc>` del XML. |
| `([uri]$loc).AbsolutePath` | Elimina el esquema y el nombre de host y devuelve la ruta. |
| `[uri]::UnescapeDataString()` | Decodifica la codificación porcentual, tal como hace el WDP antes de la comprobación. |
| `-split '[/._ :+;-]'` | Divide la ruta usando los mismos separadores que `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Conserva solo las partes de palabra de más de 18 caracteres. |

</details>

Ambas variantes comprueban solo la longitud de las palabras, no las demás heurísticas. La variante de Bash cuenta la forma codificada porcentualmente y, por tanto, solo proporciona valores correctos para slugs puramente ASCII; para slugs con diéresis u otras escrituras, use la variante de PowerShell.

## Adaptar slugs

Para artículos nuevos, la regla puede cumplirse al definir el slug: separar los compuestos del slug con guiones (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) y acortar las denominaciones tomadas de términos técnicos.

En las URL existentes, conviene sopesar un cambio. Cada modificación del slug requiere una redirección 301 de la URL antigua a la nueva, un sitemap actualizado y enlaces internos adaptados. En páginas que ya tienen buen posicionamiento o enlaces externos, el cambio aporta poco: de todos modos están en el índice mediante el rastreador. Resulta útil sobre todo para páginas recientes que aún no figuran en ningún índice.

## Medición: ¿en qué medida se ven afectados distintos idiomas?

Se puede medir si la regla afecta de forma distinta a los idiomas pasando los mismos términos en varios idiomas por el código original de Brave. Los scripts, los datos y los resultados individuales están en el repositorio público [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); la medición puede reproducirse allí con cinco comandos.

**Diseño:**

- **Código:** repositorio de Brave `web-discovery-project`, commit `58b1b53` del 9 de septiembre de 2026. Las funciones `cleanCurrentUrl` y `dropLongURL` se ejecutan sin cambios en Node.js; solo se han sustituido las conexiones con el navegador y el almacenamiento. Los propios casos de prueba de Brave para `dropLongURL` producen así los resultados esperados.
- **Términos:** la lista «Vital Articles Level 3» de la Wikipedia inglesa, alrededor de 1000 artículos centrales. Mediante los enlaces interlingüísticos de Wikipedia se recuperaron los títulos de los mismos artículos en otros 23 idiomas. Hay 825 términos en los 24 idiomas. Por término, por tanto, solo cambia el idioma.
- **URL:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, formadas como en el navegador (codificadas porcentualmente), luego pasadas como en el WDP primero por `cleanCurrentUrl` y después por `dropLongURL` en modo normal y estricto.
- **Causa:** cada URL descartada se comprobó una segunda vez, con la regla de longitud desactivada. Si entonces se comunica, el descarte se debe a la regla de longitud.
- **Estadística:** intervalo de confianza del 95 % según Wilson; comparación con el inglés mediante los términos emparejados (prueba exacta de McNemar).

El núcleo del análisis:

```javascript
for (const begriff of begriffe) {
  const kodiert = new URL(wikiUrl(sprache, begriff)).href;
  const url = WDP.cleanCurrentUrl(kodiert);
  const verworfen = WDP.dropLongURL(url);
  WDP.rel_part_len = Infinity;   // Längenregel aus
  const ohneLaenge = WDP.dropLongURL(url);
  WDP.rel_part_len = 18;         // Originalwert
  if (verworfen && !ohneLaenge) durchLaengenregel++;
}
```

**Resultado** (modo normal, 825 términos por idioma):

| Idioma | Descartadas | IC del 95 % | de ellas por regla de longitud | p frente al inglés |
|---|---|---|---|---|
| Inglés | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Alemán | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Húngaro | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Neerlandés | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finés | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Sueco | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Noruego | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Danés, francés, español, portugués, turco, polaco | 1 cada uno (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Ruso, ucraniano, japonés | 1 cada uno (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italiano, griego, árabe, hebreo, persa, hindi, chino, coreano | 0 (0,0 %) | 0,0–0,5 % | – | – |

Se descartaron, por ejemplo, `Schwangerschaftsabbruch`, `Empfängnisverhütung` y `Ingenieurwissenschaften` (alemán), `Terhességmegszakítás` (húngaro), `Milieuverontreiniging` (neerlandés) y `Tietojenkäsittelytiede` (finés). Los casos individuales en los demás idiomas se refieren casi todos al mismo término: ácido desoxirribonucleico en la lengua respectiva. En ningún caso se comunicó un término en la lengua local y se descartó en inglés.

**Análisis:**

- Los idiomas con escrituras no latinas no se ven perjudicados. Brave decodifica la URL antes de la comprobación, por lo que un carácter cirílico o chino cuenta como un carácter.
- Los idiomas que escriben juntos los compuestos tienen una desventaja pequeña, pero sistemática. Para el alemán, con p = 0,002, es significativa incluso si se considera que se compararon simultáneamente 23 idiomas (umbral de Bonferroni: 0,0022). El húngaro queda apenas por encima.
- La proporción medida se aplica a títulos de Wikipedia, que normalmente constan de una o dos palabras. Los slugs de blogs contienen más términos técnicos; en este sitio web están afectadas 3 de 66 URL de artículos en alemán (4,5 %).

**Limitaciones:** Se midió la regla de comprobación, no la inclusión real en el índice de Brave. Es probable que la propia Wikipedia esté en la lista de permitidos de Brave y no se vea afectada en la práctica; la medición muestra cómo trata la regla a un sitio web sin permiso con tales URL. El commit público no tiene por qué corresponder a la versión distribuida en el navegador. La medición no refleja la vía alternativa mediante la URL canónica.

## Opinión: un límite de longitud que perjudica a los idiomas con palabras compuestas

*Esta sección refleja la valoración del autor. Los hechos correspondientes se encuentran en las secciones anteriores.*

El límite de 18 caracteres cuenta caracteres y, por tanto, afecta a los idiomas que escriben unidos los términos. La medición muestra el efecto: con términos idénticos, la regla descarta el 1,2 % de las URL alemanas y el 0 % de las inglesas, y todos esos descartes alemanes se deben a la regla de longitud. El slug inglés `data-protection-regulation` supera la comprobación; `datenschutzgrundverordnung` no. El efecto es pequeño, pero afecta siempre a los mismos idiomas: alemán, húngaro, neerlandés, finés y las lenguas escandinavas. Desde mi punto de vista, esto supone una desventaja para estos idiomas, aunque no sea intencionada.

### Quién asume los costes

Brave escribe en el README que las clasificaciones erróneas no son un gran problema para su propio propósito. Para Brave es cierto: una página descartada le cuesta a Brave una comunicación. Para los sitios web afectados, supone una desventaja sistemática que afecta sobre todo a los textos especializados, precisamente porque los términos técnicos forman compuestos largos. Como la búsqueda web de Claude se basa en el índice de Brave, estas páginas pierden una vía de acceso al índice del que cita Claude.

Además, existe una lista de permitidos en el servidor: los patrones de URL que Brave incluye allí (`allowlisted`) omiten la comprobación. No está documentado públicamente qué patrones son. Los sitios especializados individuales no pueden influir en ello.

### Qué respalda la regla

El propósito es legítimo. Los enlaces compartidos con token, por ejemplo para documentos compartidos, son un riesgo real, y un enlace así en el índice de búsqueda sería un grave incidente de privacidad. Un límite de longitud estricto es sencillo, rápido y difícil de eludir. Tampoco puede sustituirse sin más por la detección de hashes: en una prueba adicional con 2000 tokens aleatorios de letras minúsculas de 22 caracteres, `isHash` de Brave clasificó solo el 58 % como hash; con tokens de letras minúsculas y dígitos, el 94 %. La función reconoció correctamente que todos los compuestos probados, como `schwangerschaftsabbruch` o `datenschutzgrundverordnung`, no eran hashes. Por tanto, para enlaces secretos formados solo por letras minúsculas, el límite de longitud es la protección real. Además, el rastreador sigue llegando a las páginas afectadas y la proporción medida es pequeña.

### Qué podría cambiar Brave

- **Distinguir palabras de tokens:** Una simple elevación del límite para palabras compuestas solo por letras permitiría el paso de enlaces secretos compuestos por letras minúsculas, según la prueba adicional. Podría añadirse una comprobación de plausibilidad solo para partes de palabra de entre 19 y unos 30 caracteres, por ejemplo mediante la secuencia de vocales y consonantes o un pequeño modelo lingüístico entrenado con palabras de los idiomas afectados. Brave tendría que comprobar si esto es suficientemente fiable.
- **Documentar la regla:** La página de ayuda sobre el rastreador menciona el WDP, pero no las heurísticas. Bastaría una indicación para los operadores de sitios web, de modo que puedan adaptar sus slugs.

He comunicado a Brave la medición y este conflicto de objetivos como [Issue #498](https://github.com/brave/web-discovery-project/issues/498) en el repositorio del WDP. Hasta entonces, solo queda adaptar el sitio web: separar con guiones los compuestos del slug. Que este trabajo recaiga en los operadores de determinadas áreas lingüísticas y no en el procedimiento es el núcleo de mi crítica.

*Corrección del 25/09/2026: una versión anterior propuso elevar de forma general el límite para palabras compuestas solo por letras; la prueba adicional de detección de hashes muestra que esto debilitaría la protección. Una versión anterior de esta sección afirmaba que las escrituras no latinas y los diacríticos se veían especialmente perjudicados por la codificación porcentual, basándose en una medición con URL codificadas porcentualmente. Una comprobación independiente mostró que Brave decodifica las URL con `cleanCurrentUrl` antes de la comprobación. La afirmación era incorrecta y se ha eliminado; la medición anterior utiliza el proceso correcto.*

## Fuentes

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): descripción del WDP por Brave, con tipos de mensajes, segunda solicitud sin cookies, heurísticas de URL de capacidad y quórum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): código fuente con `dropLongURL`, `calculateStrictness` y los límites `rel_part_len: 18` y `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): página de ayuda oficial sobre el rastreador, la ausencia de un agente de usuario propio y el papel del WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): incorporación de Brave Search a la lista de subencargados de Anthropic en marzo de 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): informe con más indicios, como el parámetro `BraveSearchParams` en la búsqueda web de Claude.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): decodificación de la URL antes de la comprobación; el comentario en onLocationChange (línea 1724) describe la URL decodificada como representación interna del WDP.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): lista de términos de la medición; los títulos en los demás idiomas proceden de los enlaces interlingüísticos de la API de Wikipedia.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): scripts de medición, conjunto de datos y resultados individuales de la medición lingüística de este artículo, incluida la prueba adicional de detección de hashes (`hash-test.mjs`) y la primera versión marcada como errónea.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): comunicación a Brave con los resultados de la medición y el conflicto de objetivos entre el límite de longitud y la detección de hashes.
