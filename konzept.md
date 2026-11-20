# CAPI-Shapes: eigenes Init-Image statt Upload aus dem Service-Image (AE-25)

## Worum es geht

Jeder OLIVIA-Service lädt beim Start seine CAPI-Shapes hoch. Bisher passiert das in einem Init-Container, der das Service-Image selbst verwendet und ein Java-Skript aus einer ConfigMap in olivia-kustomize ausführt. Neu bekommt jeder Service ein kleines, eigenes Shapes-Image, das auf einem gemeinsamen Uploader-Image aufbaut. Das Service-Image braucht dafür nichts mehr und läuft auf einer JRE statt einem JDK.

## Heute

> Diagramm: **01-heute-vs-neu**

- Der Init-Container `capi-shape-upload` startet das **Service-Image** und führt per Shell-Skript `java ShapeUploader.java` aus.
- `ShapeUploader.java` liegt als ConfigMap `capi-shape-upload-class` in `olivia-kustomize/base/capi-shapes` und wird in jede Umgebung eingebunden.
- Die Shapes stecken im Service-Jar und werden beim Start daraus entpackt.

Probleme:

- `java Datei.java` braucht ein **JDK**. Darum muss jedes Service-Image ein JDK enthalten: grösser, mehr CVEs, unnötig zur Laufzeit.
- Upload-Code ohne Build, ohne Tests und ohne Versionierung, verteilt über kustomize.
- Shapes, Service und Upload-Logik sind eng gekoppelt.

## Neu

> Diagramm: **01-heute-vs-neu** (rechte Seite)

| Baustein | Was | Wo |
|---|---|---|
| `shape-uploader` | Generisches Base-Image: Token holen, alle Shape-Dateien unter `/shapes` hochladen. Java 25, getestet, versioniert | Repo `TOOL/shape-uploader` |
| `<service>-shapes` | Pro Service: `FROM shape-uploader` + `COPY capi-shapes/ /shapes/`. Enthält nur die Shapes | Service-Repo, `src/main/docker-shapes` |
| Service-Image | Basis `tkb-java-base-21-jre` bzw. `-25-jre`, ohne Shapes | Service-Repo |
| kustomize | `patch-shape-upload.yaml` setzt das Image des Init-Containers auf `<service>-shapes`. Der Tag steht im `images`-Eintrag | `olivia-kustomize/overlays/<env>/<service>` |

Die Shapes liegen im Service-Repo unter `src/main/capi-shapes` und kommen nicht mehr ins Jar.

## Build und Deployment

> Diagramm: **02-build-und-deployment**

1. Der Jenkins-Build baut wie bisher das Service-Image und **zusätzlich** das Shapes-Image (Gradle-Tasks `dockerBuildShapesImage` / `dockerPushShapesImage`). Beide bekommen **denselben Tag**.
2. `<service>-shapes` steht zusätzlich in `imageNames` (siehe Gradle). Jenkins prüft damit, ob alle Images in Artifactory sind.
3. Der Autodeploy ersetzt in `kustomization.yaml` **alle** `newTag`-Zeilen. Der Tag des Shapes-Images läuft damit automatisch mit, ohne Anpassung an den Jenkins-Skripten.
4. ArgoCD synchronisiert, der Pod startet zuerst den Init-Container mit dem Shapes-Image, danach den Service.

## Anpassungen im Gradle-Build

Pro Service-Projekt:

1. Shapes von `src/main/resources/capi-shapes` nach `src/main/capi-shapes` verschieben und das Kopieren ins Jar entfernen (z.B. den `bootJar`-Block mit `capi-shapes`).
2. `src/main/docker-shapes/Dockerfile` anlegen:
   ```dockerfile
   FROM tkb.artifactory.bitbeam.swisscom.com/tkb-docker-local/tool/ch.tkb.tool.shapeuploader.shape-uploader:<version>
   COPY capi-shapes/ /shapes/
   ```
3. Drei Tasks im Service-Modul registrieren:
   - `copyShapesDockerContext` (Sync): Dockerfile und Shapes nach `build/docker-shapes`
   - `dockerBuildShapesImage`: baut das Image
   - `dockerPushShapesImage`: pusht es
4. Das Shapes-Image mit dem Build pushen und `<service>-shapes` in `imageNames` eintragen.
5. Im Service-Dockerfile die Basis auf `tkb-java-base-21-jre` bzw. `tkb-java-base-25-jre` umstellen.

Der Name des Shapes-Images folgt dem Service-Image des Projekts, nur mit `-shapes` statt `-service`. Der Tag ist derselbe.

### Kotlin DSL und Groovy

Je nach Projekt ist der Build in Kotlin DSL (`*.gradle.kts`) oder Groovy (`*.gradle`) geschrieben. Die Tasks sind gleich, die Stellen unterscheiden sich leicht:

| | Kotlin DSL (OL-Projekte) | Groovy (`tkb-olivia-verwaltung-bff`) |
|---|---|---|
| `imageNames` | in **`gradle.properties`**:<br>`imageNames=produkte-service,…,produkte-shapes` | in **`settings.gradle`**:<br>`gradle.ext.imageNames='tkb-olivia-verwaltung-bff,tkb-olivia-verwaltung-bff-shapes'` |
| Image-Tag | `gradle.extra["imageTag"]` | `System.getenv('IMAGE_TAG')` |
| Image-Name | `"$artifactoryRepositoryUrl.$projectName-shapes"` | `"$artifactoryRepositoryUrl/$projectName-shapes"` |
| Task registrieren | `tasks.register<Sync>("copyShapesDockerContext") { … }` | `tasks.register('copyShapesDockerContext', Sync) { … }` |

Kotlin DSL (`service/build.gradle.kts`, z.B. produkte):

```kotlin
val shapesImage =
    "$artifactoryRepositoryUrl.$projectName-shapes:${gradle.extra["imageTag"]}"

val copyShapesDockerContext =
    tasks.register<Sync>("copyShapesDockerContext") {
        from("$projectDir/src/main/docker-shapes")
        from("$projectDir/src/main/capi-shapes") {
            into("capi-shapes")
        }
        into(layout.buildDirectory.dir("docker-shapes"))
    }

val dockerBuildShapesImage =
    tasks.register<DockerBuildImage>("dockerBuildShapesImage") {
        dependsOn(copyShapesDockerContext)
        inputDir.set(layout.buildDirectory.dir("docker-shapes"))
        network.set("host")
        images.add(shapesImage)
    }

val dockerPushShapesImage =
    tasks.register<DockerPushImage>("dockerPushShapesImage") {
        dependsOn(dockerBuildShapesImage)
        images.add(shapesImage)
    }
```

Groovy (`tkb-olivia-verwaltung-bff/build.gradle`):

```groovy
def shapesImage = "$artifactoryRepositoryUrl/$projectName-shapes:${System.getenv('IMAGE_TAG')}"

tasks.register('copyShapesDockerContext', Sync) {
    from("$projectDir/src/main/docker-shapes")
    from("$projectDir/src/main/capi-shapes") {
        into("capi-shapes")
    }
    into layout.buildDirectory.dir("docker-shapes")
}

tasks.register('dockerBuildShapesImage', DockerBuildImage) {
    dependsOn copyShapesDockerContext
    inputDir = layout.buildDirectory.dir("docker-shapes")
    network = "host"
    images.add(shapesImage)
}

tasks.register('dockerPushShapesImage', DockerPushImage) {
    dependsOn dockerBuildShapesImage
    images.add(shapesImage)
}
```

### Spring Boot und Quarkus

Wie das Shapes-Image in den Build eingehängt wird, hängt davon ab, wie das Service-Image gebaut wird:

| | Spring Boot (produkte, anlegen, kreditantrag, verwaltung-bff) | Quarkus (ebanking, lsv-cockpit, portal) |
|---|---|---|
| Service-Image | Docker-Plugin (`com.bmuschko.docker-remote-api`) | Quarkus-Extension `container-image-docker` |
| Shapes-Image pushen | `buildServiceDocker` hängt zusätzlich von `dockerPushShapesImage` ab | `build` hängt von `dockerPushShapesImage` ab, wie Frontend und Proxy |
| Plugin | schon vorhanden | im Service-Modul neu hinzugefügt: `alias(libs.plugins.docker.remote.api)` |
| Root-Build | unverändert | Der Docker-Hook in `build.gradle.kts` überspringt das Modul `service`, sonst würde er dort eigene Docker-Tasks anlegen |
| Service-Dockerfile | `src/main/docker/Dockerfile` | `src/main/docker/Dockerfile.jvm` |

Sonderfall portal: Jenkins baut den Service mit `-x :service:dockerPushShapesImage` und pusht das Shapes-Image in der Stage „Docker Images“ separat.

## Ablauf im Init-Container

> Diagramm: **03-ablauf-init-container**

1. Token beim Tokenmanager holen, im Cluster mit dem ServiceAccount-Token (Audience `tokenmanager`), lokal per Basic Auth.
2. Alle `*.yaml` unter `/shapes` lesen.
3. Jede Shape per `PUT <CAPI>/shapes/<entity>/<name>` hochladen.
4. Fehler beim Token, beim Lesen oder bei einem Upload: Exit-Code 1, **der Pod startet nicht** (`FAIL_ON_ERROR=true`, Standard). So läuft nie ein Service mit fehlenden Shapes.

Ressourcen des Init-Containers: 10m CPU, 128Mi Memory.

## Base-Images

> Diagramm: **04-image-hierarchie**

- `docker-images` baut neu getrennte JDK- und JRE-Images. Die JDK-Images behalten ihren Namen (`tkb-java-base-21`, `tkb-java-base-25`), damit noch nicht umgestellte Services weiter Updates bekommen. Neu dazu: `tkb-java-base-21-jre`, `tkb-java-base-25-jre`.
- Gepusht wird nur von `master`, auf Branches wird nur gebaut.
- Renovate erhöht pro Ordner die `VERSION`, die Services bekommen danach ihre Update-PRs.

## Lokal

- `docker-compose.yml` enthält den Service `capi-shape-upload` (Basic Auth auf DEV).
- Run-Konfiguration **Upload Shapes to DEV** in IntelliJ: Shapes ohne Deployment auf DEV hochladen.

## Was ändert sich für die Teams

- Neue oder geänderte Shapes: Datei unter `src/main/capi-shapes` ablegen, sonst nichts.
- Kein JDK mehr im Service-Image nötig.
- Ein fehlerhafter Shape-Upload stoppt den Pod sichtbar, statt später zur Laufzeit aufzufallen.

## Stand und Rollout

| Repo | Stand |
|---|---|
| `TOOL/shape-uploader` | auf develop, Image in Artifactory |
| `WEB/docker-images` | Branch `feature/jdk-jre-images` (JDK/JRE getrennt) |
| `TOOL/renovate` | Branch `feature/cve-prs` (neue Ordner, Default-Level Minor) |
| `OL/produkte`, `anlegen`, `kreditantrag`, `ebanking`, `lsv-cockpit`, `portal` | Branch `feature/AE-25-capi-shape-deployment` |
| `SVC/tkb-olivia-verwaltung-bff` | Branch `feature/AE-25-capi-shape-deployment`, Overlay in services-kustomize folgt |
| `TO/olivia-kustomize` | Branch `feature/AE-25-capi-shape-deployment`, dev-Overlays aller sechs Services |

Reihenfolge:

1. renovate und docker-images mergen.
2. Service-Branches bauen, damit für jeden Tag ein Shapes-Image existiert.
3. olivia-kustomize mergen: dev läuft mit dem neuen Init-Container.
4. Beim Promoten in die weiteren Umgebungen denselben Overlay-Patch mitnehmen.
5. Wenn alle Services in allen Umgebungen umgestellt sind: `base/capi-shapes` und die Komponente `capi-shape-upload` in olivia-kustomize entfernen.
