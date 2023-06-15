# Idealista Challenge - Víctor González Martínez


## DISCLAIMER:

- Si se trata de una pull request tan grande, seguro que se ha hecho un análisis técnico más profundo y hay consensuada una arquitectura, modelo, etc.
- Seguramente el primer paso para llegar a entender cómo mejorar el microservicio sería hacer una llamada. Hay mucho por revisar y comentar, y sería todo más fluido por llamada. ¡La comunicación es super importante!
- Siguiendo por ese punto, puede haber cosas que a mí personalmente no me encajen, pero por falta de contexto tengan sentido. Siempre dispuesto a escuchar otros puntos de vista.
- Este código lo he revisado a fondo. En realidad, en una revisión de código real, depositaría más confianza en el desarrollador y seguro que no encontraría tantas cosas.

### Arquitectura:
Voy a dar por hecho que en el equipo se sigue este modelo de arquitectura hexagonal, el cual creo que está bien estructurado (he investigado).

## TESTS:
Solo existe un test: `AdsServiceImplTest`. Estaría bien que `AdsController` también tenga pruebas unitarias, e incluso quizás algunas de integración.
- `AdsServiceImplTest#23` -> El nombre de la variable debería ser "adsServiceImpl" o similar.
- `AdsServiceImplTest#calculateScoresTest` -> El test no hace ninguna aserción. Debería, como mínimo, comprobar que los cálculos de las puntuaciones son los esperados.
- Tampoco se están testeando los métodos `AdsServiceImpl#findQualityAds` y `AdsServiceImpl#findPublicAds`.
- Testear las consultas a la base de datos no tiene sentido actualmente porque no hay implementación real.

## CONFIGURACIONES:
`application.yml` está vacío. No sé cuántas configuraciones pueden faltar. Depende de dónde despliegues el microservicio (por ejemplo, AWS). Pero como entiendo que el microservicio debe terminar en producción, aquí faltan configuraciones seguramente.
- `pom.xml` -> Sabiendo que es un proyecto nuevo, aprovecharía para utilizar una versión más reciente de Java y de Spring Boot.

## INFRASTRUCTURE:
- **Persistence:**
    - `AdVO` y `PictureVO` -> Estos Value Objects deberían ser inmutables, con todos los campos `final` y sin setters.
    - `AdVO` tiene muchos parámetros en el constructor y es posible que en el futuro crezca más. Sería bueno aplicar el patrón Builder.
    - `InMemoryPersistence` -> Entiendo que es una implementación que no saldrá a producción, así que tampoco haría foco en mejorar esta parte. Lo que sí veo útil son los métodos de mapeo de dominio a persistencia y viceversa. Esos métodos los extraería a un Adapter/Mapper para que todas las implementaciones de persistencia puedan aprovecharlos.
- **API:**
    - `AdsController#calculateScores()` -> Falta contexto, pero en este mismo proyecto me imagino un CRUD para los anuncios (`Ad`), y que sea en el momento de dar de alta o modificar un anuncio donde ya se calcule la puntuación. Me imagino este método utilizándose solo una vez para añadir un score a los anuncios que existan antes de la implementación de esta

lógica.
- `AdsController` -> Añadir un manejo de errores, quizás con `@ExceptionHandler` o alguna librería interna si existe.
- `AdsController#qualityListing()` -> En realidad, este sería el endpoint para que el encargado de calidad vea los anuncios irrelevantes, así que lo renombraría a `getIrrelevantAds()` o `findIrrelevantAds()` con el path "/ads/irrelevant".
- `AdsController#publicListing()` -> Renombrar a `getPublicAds()` o `findPublicAds()`.
- `QualityAd` y `PublicAd` -> Para evitar repetir código, quizás se pueda crear una clase base llamada `BaseAd` con los campos comunes, y que estas dos clases la extiendan. De hecho, `PublicAd` podría ser un `BaseAd`. También podrían ser inmutables estas clases, ya que solo se van a crear instancias.

## DOMAIN:
Quizás crearía un paquete `model` y otro `repository`. Pero todavía hay pocas clases...
- `Constants` -> En lugar de nombrar las constantes con números, podríamos nombrar cada puntuación con su significado. Por ejemplo: `Constants.ZERO -> Constants.INITIAL_SCORE = 0; Constants.TWENTY -> Constants.SCORE_HD_PHOTO = 20;`. De esta manera, si el día de mañana se modifican las puntuaciones, será mucho más mantenible.
- `Typology` -> Suena demasiado genérico, quizás `HousingType` o `PropertyType`... (No estoy seguro, se aceptan sugerencias).

## APPLICATION:
- `AdsService` -> (Opcional) En este caso no veo necesaria la interfaz. Entiendo que para casos de uso / lógica no hay posibilidad de tener varias implementaciones.
- `AdsServiceImpl#findPublicAds()` -> Similar a lo de infraestructura: Separar la parte de mapeo y llevarla a un adapter/mapper para cumplir con el principio de responsabilidad única (Single Responsibility). El mapeo se puede hacer sin usar setters para mantener `PublicAd` y `QualityAd` como objetos inmutables.
- `AdsServiceImpl#findQualityAds()` -> `findIrrelevantAds()` -> Similar a lo de infraestructura: Separar la parte de mapeo y llevarla a un adapter/mapper para cumplir con el principio de responsabilidad única (Single Responsibility).
- `AdsServiceImpl#21` -> Por defecto, se ordena ascendente, así que los primeros anuncios serían los menos relevantes. Habría que usar `reverseOrder()`.
- `AdsServiceImpl#calculateScore()` -> Siguiendo el enfoque DDD, toda la lógica del cálculo de la puntuación de un anuncio podría incluirse dentro de la clase `Ad`. De esta manera, el propio anuncio sería responsable de calcular su puntuación. Creo que sería más fácil de entender y mantener.
- `AdsServiceImpl#calculateScore()` -> Trataría de separar en métodos privados las diferentes secciones de puntuación. Por ejemplo: `calculateScoreByPhotos()`, `calculateScoreByDescription()`, etc.
- `AdsServiceImpl#84` -> Crear el `Optional` ahí mismo para comprobar si es nulo no tiene mucho sentido. O bien se retira el `Optional` y se usa un `if (description != null)` clásico, o se actualiza laclase `Ad` para que el campo `description` sea un `Optional<String>`.
- `AdsServiceImpl#86` -> Si mantenemos el `Optional`, se pueden fusionar este `if` y el de la `#89`.
- `AdsServiceImpl#93` -> Podríamos almacenar en una variable la cantidad de palabras para evitar llamar constantemente al método `size()`. Además, cambiaría `wds` a `words`.
- `AdsServiceImpl#99` -> Con un `else if` nos evitamos que los flujos que cumplan la condición anterior intenten volver a entrar por la condición de aquí.
- `AdsServiceImpl#104` -> Se pueden fusionar las dos condiciones. Además, según el cliente, en el caso de los chalets solo se añaden los 20 puntos si tiene más de 50 palabras: `if (Typology.CHALET.equals(ad.getTypology()) && words.size() > Constants.FIFTY)`.
- `AdsServiceImpl#110` -> Quizás habría que hacer un `.toLowerCase()` para evitar problemas con mayúsculas y minúsculas.
- `AdsServiceImpl#110` -> Se podrían guardar los strings en un array llamado "keywords" o similar, y recorrer el array para aplicarles el `+5`.
- `AdsServiceImpl#119` -> Deberían sumarse 40 puntos al score, no igualarlo a 40.
- `AdsServiceImpl#124` -> Desde esta línea hasta la `#130`, se puede resumir con este código (que quizás es más ligero y óptimo): `ad.setScore(Math.max(Constants.ZERO, Math.min(Constants.ONE_HUNDRED, ad.getScore())));` (me suena porque en Kotlin hace poco usé el `coerceIn()`).

## DUDAS:
- Quizás, en la capa de aplicación, se podría dividir el servicio en casos de uso.
- `Ad#hashCode()` -> Quizás solo sea necesario el campo `id`... lo mismo para `Picture`.
- Faltaría una implementación real con una base de datos, ya sea relacional o no relacional.
- "Que el anuncio tenga un texto descriptivo suma 5 puntos" -> Necesitaría más detalles para saber qué considerar texto descriptivo y qué no. Actualmente, la aplicación suma esos 5 puntos simplemente si la descripción no está vacía. No sé si es lo que busca el cliente.

Si tienes alguna otra pregunta o necesitas más ayuda, estaré encantado de asistirte.
