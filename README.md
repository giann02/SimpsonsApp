# SimpsonsApp — Errores encontrados

**Alumno:** Panigatti Gianluca  
**Errores encontrados:** 12

---

## Errores que impiden compilar el app

### Error 1 — Bloque `init` fuera de la clase
**Archivo:** `domain/model/Episode.kt` — líneas 13–15

**Código:**
```kotlin
data class Episode(
    val id: Int,
    ...
)

init {    // está afuera de la clase
    return Episode; //NO BORRAR
}
```

El bloque `init` en Kotlin solo va adentro de una clase, no afuera. Como está puesto acá, no compila. Encima `return Episode;` tampoco es código Kotlin válido: `init` no puede tener `return` y los puntos y coma no se usan.

**Fix:** borrar las líneas 13 a 15 enteras.

---

### Error 2 — El `override` no coincide con lo que declara la interfaz
**Archivo:** `data/repository/EpisodeRepositoryImpl.kt` — línea 23

**Código:**
```kotlin
// La interfaz tiene: fun get_episodes()
// La implementación tiene:
override fun getEpisodes(): Flow<PagingData<Episode>> {
```

`getEpisodes()` no existe en la interfaz, así que el `override` no encuentra nada que sobrescribir. Falla en compilación.

**Fix:** cambiar el nombre en la interfaz a `getEpisodes()` (ver Error 5) y dejar la implementación como está.

---

### Error 3 — Falta el import de `SimpsonsApi`
**Archivo:** `data/repository/EpisodeRepositoryImpl.kt` — sección de imports

**Código:**
```kotlin
import com.example.simpsonsapp.data.remote.EpisodeRemoteMediator
// falta: import com.example.simpsonsapp.data.remote.SimpsonsApi

class EpisodeRepositoryImpl @Inject constructor(
    private val simpsonsApi: SimpsonsApi,   // referencia sin resolver
```

`SimpsonsApi` está en `data.remote` y este archivo está en `data.repository`. Son paquetes distintos, así que sin el import el compilador no la encuentra.

**Fix:** agregar:
```kotlin
import com.example.simpsonsapp.data.remote.SimpsonsApi
```

---

## Error que crashea en runtime

### Error 4 — Falta el `baseUrl` en Retrofit
**Archivo:** `di/DataModule.kt` — líneas 34–38

**Código:**
```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()   // falta el .baseUrl() antes de acá
    .create(SimpsonsApi::class.java)
```

Retrofit lanza `IllegalStateException: Base URL required` si no se le pone una URL base antes del `.build()`. La app explota al arrancar cuando Hilt trata de construir la dependencia.

**Fix:**
```kotlin
return Retrofit.Builder()
    .client(client)
    .baseUrl("https://thesimpsonsapi.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

---

## Error de comportamiento incorrecto

### Error 5 — `LazyRow` en vez de `LazyColumn`
**Archivo:** `main/MainScreen.kt` — línea 119

**Código:**
```kotlin
LazyRow(
    state = listState,
    modifier = Modifier.fillMaxSize()
) {
```

`LazyRow` hace scroll horizontal. Por lo general una lista de episodios se espera que scrollee verticalmente, por lo que lo más apropiado sería usar `LazyColumn`.

**Fix:**
```kotlin
LazyColumn(
    state = listState,
    modifier = Modifier.fillMaxSize()
) {
```

---

## Error de configuración

### Error 6 — JDK hardcodeado en `gradle.properties`
**Archivo:** `gradle.properties` — línea 32

**Código:**
```properties
org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home
```

Esta ruta es específica de la máquina donde se armó el proyecto. En cualquier otra computadora esa ruta no existe y Gradle tira el error:
```
Value '...' given for org.gradle.java.home Gradle property is invalid
```

**Fix:** borrar la línea 32. Cada dev configura su JDK desde Android Studio (File > Project Structure > SDK Location) o Gradle lo detecta solo.

---

## Errores de práctica incorrecta

### Error 7 — `class` en vez de `data class` en `EpisodeEntity`
**Archivo:** `data/local/entity/EpisodeEntity.kt` — línea 7

**Código:**
```kotlin
class EpisodeEntity(
    @PrimaryKey val id: Int,
    ...
)
```

Las entidades de Room que se usan con Paging 3 tienen que ser `data class`. Sin eso no se generan `equals()` ni `hashCode()`, y el sistema de diff de Paging no sabe qué cambió en la lista.

**Fix:**
```kotlin
data class EpisodeEntity(
    @PrimaryKey val id: Int,
    ...
)
```

---

### Error 8 — `class` en vez de `data class` en `RemoteKeyEntity`
**Archivo:** `data/local/entity/RemoteKeyEntity.kt` — línea 7

Mismo problema que el Error 7. Las entidades de Room necesitan `data class`.

**Fix:**
```kotlin
data class RemoteKeyEntity(
    @PrimaryKey val episodeId: Int,
    ...
)
```

---

### Error 9 — Método con nombre en snake_case en la interfaz
**Archivo:** `domain/repository/EpisodeRepository.kt` — línea 8

**Código:**
```kotlin
fun get_episodes(): Flow<PagingData<Episode>>
```

En Kotlin los métodos van en camelCase, no en snake_case. Además este nombre no coincide con el que usa la implementación, lo que genera el Error 2.

**Fix:**
```kotlin
fun getEpisodes(): Flow<PagingData<Episode>>
```

---

### Error 10 — URL completa dentro del `@GET`
**Archivo:** `data/remote/EpisodeRemoteMediator.kt` — línea 106

**Código:**
```kotlin
@GET("https://thesimpsonsapi.com/api/episodes")
suspend fun getEpisodes(@Query("page") page: Int): EpisodesResponse
```

Con Retrofit la idea es poner la URL base una sola vez en el builder, y en los `@GET` solo la ruta relativa. Poner la URL entera en la anotación rompe esa separación.

**Fix:**
```kotlin
@GET("api/episodes")
suspend fun getEpisodes(@Query("page") page: Int): EpisodesResponse
```

---

## Observaciones adicionales

### Error 11 — Imports con wildcard que no sirven para nada
**Archivo:** `AppNavigation.kt` — líneas 9–10

**Código:**
```kotlin
import androidx.navigation.*    // redundante, las clases necesarias ya están importadas arriba
import androidx.compose.*       // este paquete no tiene clases directas, importa nada
```

El primero es redundante porque todas las clases de `androidx.navigation` que se usan ya estaban importadas de forma explícita. El segundo no importa nada útil porque `androidx.compose` no es un paquete hoja, todas las clases están en subpaquetes.

**Fix:** eliminar esas dos líneas.

---

### Error 12 — Parámetro incorrecto al llamar `MainScreen` en el test
**Archivo:** `androidTest/java/com/example/simpsonsapp/ui/main/MainScreenTest.kt` — línea 18

**Código:**
```kotlin
private val FAKE_DATA = listOf("Sample1", "Sample2", "Sample3")

@Before
fun setup() {
    composeTestRule.setContent { MainScreen(FAKE_DATA) }
}
```

`MainScreen` tiene esta firma:
```kotlin
fun MainScreen(
    onNavigateToDetail: (Int) -> Unit,
    viewModel: MainViewModel = hiltViewModel()
)
```

El primer parámetro tiene que ser una función `(Int) -> Unit`, no un `List<String>`. No compila el módulo de tests instrumentados.

**Fix:**
```kotlin
composeTestRule.setContent {
    MainScreen(onNavigateToDetail = { })
}
```
