# build-libs-ghost-client1.0.0.jar
Wtf
Сделаю клиентский мод-«чит» под Fabric 1.21.11 — с прицелом на одиночку/личное использование. Сразу оговорюсь: пишу только клиентские утилиты (полёт, скорость, яркость, проход сквозь блоки). Килл-ауру, ESP и обход античитов — не пишу, это уже про других игроков на серверах.

Сначала сверил версии под 1.21.11 (это последняя **обфусцированная** версия MC, дальше 26.1 и Mojang mappings):

| Что | Версия |
|---|---|
| Minecraft | 1.21.11 |
| Yarn mappings | `1.21.11+build.4` |
| Fabric Loader | `0.18.1` |
| Fabric API | `0.140.0+1.21.11` |
| Loom | `1.14` |
| Java | 21 |

Важная деталь 1.21.9+: конструктор `KeyBinding` теперь принимает объект `KeyBinding.Category` вместо строки-категории — в коде ниже учтено (беру ванильную `MISC`, чтобы не морочиться с регистрацией своей категории).

---

## Структура проекта

```
ghostclient/
├─ build.gradle
├─ gradle.properties
├─ settings.gradle
├─ .github/workflows/build.yml     ← сборка jar без ПК
└─ src/main/
   ├─ java/com/yourname/ghostclient/GhostClient.java
   └─ resources/
      ├─ fabric.mod.json
      └─ assets/ghostclient/lang/
         ├─ en_us.json
         └─ ru_ru.json
```

## gradle.properties

```properties
org.gradle.jvmargs=-Xmx2G

# Fabric (https://fabricmc.net/develop)
minecraft_version=1.21.11
yarn_mappings=1.21.11+build.4
loader_version=0.18.1
fabric_version=0.140.0+1.21.11

# Mod
mod_version=1.0.0
maven_group=com.yourname
archives_base_name=ghostclient
```

## build.gradle

```groovy
plugins {
    id 'fabric-loom' version '1.14-SNAPSHOT'
}

version = project.mod_version
group   = project.maven_group
base { archivesName = project.archives_base_name }

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings  "net.fabricmc:yarn:${project.yarn_mappings}:v2"
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
}

processResources {
    inputs.property "version", project.version
    filesMatching("fabric.mod.json") { expand "version": project.version }
}

tasks.withType(JavaCompile).configureEach { it.options.release = 21 }

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

## settings.gradle

```groovy
pluginManagement {
    repositories {
        maven { name = 'Fabric'; url = 'https://maven.fabricmc.net/' }
        mavenCentral()
        gradlePluginPortal()
    }
}
```

## src/main/resources/fabric.mod.json

```json
{
  "schemaVersion": 1,
  "id": "ghostclient",
  "version": "${version}",
  "name": "GhostClient",
  "description": "Личный клиентский мод: Fly / Speed / Fullbright / NoClip / AutoSprint.",
  "authors": ["yourname"],
  "license": "MIT",
  "environment": "client",
  "entrypoints": {
    "client": ["com.yourname.ghostclient.GhostClient"]
  },
  "depends": {
    "fabricloader": ">=0.18.1",
    "fabric-api": "*",
    "minecraft": "~1.21.11",
    "java": ">=21"
  }
}
```

## GhostClient.java

```java
package com.yourname.ghostclient;

import net.fabricmc.api.ClientModInitializer;
import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientTickEvents;
import net.fabricmc.fabric.api.client.keybinding.v1.KeyBindingHelper;
import net.minecraft.client.MinecraftClient;
import net.minecraft.client.network.ClientPlayerEntity;
import net.minecraft.client.option.KeyBinding;
import net.minecraft.client.util.InputUtil;
import net.minecraft.entity.attribute.EntityAttributeInstance;
import net.minecraft.entity.attribute.EntityAttributes;
import net.minecraft.entity.player.PlayerAbilities;
import net.minecraft.text.Text;
import org.lwjgl.glfw.GLFW;

public class GhostClient implements ClientModInitializer {

    // ── бинды: H Fly · J Speed · K Fullbright · N NoClip · M AutoSprint
    private static KeyBinding keyFly, keySpeed, keyBright, keyGhost, keySprint;

    private static boolean fly, speed, bright, ghost, sprint;

    // сохранённые «ванильные» значения, чтобы честно вернуть всё назад
    private static double savedGamma = 1.0;
    private static double savedBaseSpeed = 0.1;
    private static boolean speedSaved = false;

    private static final float FLY_SPEED      = 0.15f; // обычный полёт
    private static final float FLY_SPEED_FAST = 0.35f; // полёт + Speed
    private static final double FAST_WALK     = 0.25;  // ваниль 0.1 → 2.5x

    @Override
    public void onInitializeClient() {
        keyFly    = reg("fly",    GLFW.GLFW_KEY_H);
        keySpeed  = reg("speed",  GLFW.GLFW_KEY_J);
        keyBright = reg("bright", GLFW.GLFW_KEY_K);
        keyGhost  = reg("ghost",  GLFW.GLFW_KEY_N);
        keySprint = reg("sprint", GLFW.GLFW_KEY_M);

        ClientTickEvents.END_CLIENT_TICK.register(GhostClient::onTick);
    }

    private static KeyBinding reg(String name, int glfwKey) {
        // 1.21.9+: категория — объект KeyBinding.Category, а не строка
        return KeyBindingHelper.registerKeyBinding(new KeyBinding(
                "key.ghostclient." + name,
                InputUtil.Type.KEYSYM,
                glfwKey,
                KeyBinding.Category.MISC));
    }

    private static void onTick(MinecraftClient client) {
        if (keyFly.wasPressed())    { fly    = !fly;    onFly(client, fly); }
        if (keySpeed.wasPressed())  { speed  = !speed;  onSpeed(client, speed); }
        if (keyBright.wasPressed()) { bright = !bright; onBright(client, bright); }
        if (keyGhost.wasPressed())  { ghost  = !ghost;  toast(client, "NoClip", ghost); }
        if (keySprint.wasPressed()) { sprint = !sprint; toast(client, "AutoSprint", sprint); }

        ClientPlayerEntity player = client.player;
        if (player == null) return;
        apply(client, player);
    }

    private static void apply(MinecraftClient client, ClientPlayerEntity player) {
        PlayerAbilities abilities = player.getAbilities();

        // ── FLY: держим полёт включённым, пока фича активна
        if (fly) {
            abilities.allowFlying = true;
            abilities.flying = true;
            abilities.setFlySpeed(speed ? FLY_SPEED_FAST : FLY_SPEED);
        }

        // ── SPEED: базовое значение атрибута MOVEMENT_SPEED
        if (speed) {
            EntityAttributeInstance attr =
                    player.getAttributeInstance(EntityAttributes.MOVEMENT_SPEED);
            if (attr != null) attr.setBaseValue(FAST_WALK);
        }

        // ── NOCLIP: проход сквозь блоки (в одиночке — летающая «камера»)
        player.noClip = ghost;

        // ── AUTOSPRINT
        if (sprint && player.isOnGround() && !player.isSneaking()
                && client.options.forwardKey.isPressed()) {
            player.setSprinting(true);
        }

        // ── FULLBRIGHT: максимальная ванильная гамма
        if (bright) client.options.getGamma().setValue(1.0);
    }

    private static void onFly(MinecraftClient client, boolean enabled) {
        ClientPlayerEntity p = client.player;
        if (p != null && !enabled) {
            PlayerAbilities ab = p.getAbilities();
            if (!p.isCreative() && !p.isSpectator()) { // креатив/спектейтор не ломаем
                ab.allowFlying = false;
                ab.flying = false;
            }
        }
        toast(client, "Fly", enabled);
    }

    private static void onSpeed(MinecraftClient client, boolean enabled) {
        ClientPlayerEntity p = client.player;
        if (p != null) {
            EntityAttributeInstance attr =
                    p.getAttributeInstance(EntityAttributes.MOVEMENT_SPEED);
            if (attr != null) {
                if (enabled) {
                    if (!speedSaved) {
                        savedBaseSpeed = attr.getBaseValue();
                        speedSaved = true;
                    }
                    attr.setBaseValue(FAST_WALK);
                } else if (speedSaved) {
                    attr.setBaseValue(savedBaseSpeed);
                    speedSaved = false;
                }
            }
        }
        toast(client, "Speed", enabled);
    }

    private static void onBright(MinecraftClient client, boolean enabled) {
        if (enabled) {
            savedGamma = client.options.getGamma().getValue();
            client.options.getGamma().setValue(1.0);
        } else {
            client.options.getGamma().setValue(savedGamma);
        }
        toast(client, "Fullbright", enabled);
    }

    /** Сообщение над хотбаром. Хочешь в чат — замени на p.sendMessage(Text.literal(msg)). */
    private static void toast(MinecraftClient client, String feature, boolean on) {
        client.inGameHud.setOverlayMessage(
                Text.literal(feature + ": " + (on ? "ВКЛ" : "ВЫКЛ")), false);
    }
}
```

## Языковые файлы

`assets/ghostclient/lang/en_us.json`:
```json
{
  "key.ghostclient.fly": "Fly",
  "key.ghostclient.speed": "Speed",
  "key.ghostclient.bright": "Fullbright",
  "key.ghostclient.ghost": "NoClip",
  "key.ghostclient.sprint": "AutoSprint"
}
```

`assets/ghostclient/lang/ru_ru.json`:
```json
{
  "key.ghostclient.fly": "Полёт",
  "key.ghostclient.speed": "Скорость",
  "key.ghostclient.bright": "Яркость",
  "key.ghostclient.ghost": "NoClip",
  "key.ghostclient.sprint": "Автоспринт"
}
```

## .github/workflows/build.yml (сборка jar без ПК)

```yaml
name: build
on: [push, workflow_dispatch]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - uses: gradle/actions/setup-gradle@v4
      - run: chmod +x ./gradlew
      - run: ./gradlew build --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: ghostclient-jar
          path: build/libs/*.jar
```

---

## Как это собрать и поставить

1. Сгенерируй заготовку на **fabricmc.net/develop/template** (mod name `ghostclient`, package `com.yourname.ghostclient`) — оттуда возьми `gradlew` и wrapper, они нужны для сборки.
2. Замени `gradle.properties` и `build.gradle` на мои, положи `fabric.mod.json`, `GhostClient.java`, lang-файлы и workflow.
3. Джарник:
   - с ПК: `./gradlew build` → `build/libs/ghostclient-1.0.0.jar` (файл **без** `-sources`);
   - без ПК (как раз твой случай): залей репозиторий на GitHub → вкладка Actions → Run workflow → внизу артефакт `ghostclient-jar`.
4. В PojavLauncher: профиль **Fabric 1.21.11** → в папку `mods` положи этот jar **и** Fabric API `0.140.0+1.21.11`. Запуск, потом Options → Controls → **Разное (Misc)** — там будут бинды `Fly / Speed / Fullbright / NoClip / AutoSprint`. Экранную клавиатуру в Pojav включи, чтобы нажимать H/J/K/N/M.

## Что стоит знать заранее

- **Fullbright тут «честный ванильный»**: гамма в 1.21 клампится до 1.0 (`SimpleOption` валидирует значение), поэтому это максимум яркости, а не пещерное «день». Настоящий fullbright делается миксином в `SimpleOption#setValue`/lightmap — скажи, докину файл.
- **NoFall осознанно не делал**: он требует подмены пакетов движения (`PlayerMoveC2SPacket`), иначе клиентским `fallDistance = 0` ничего не решается — урон считает сервер. В одиночке проще `/gamerule fallDamage false`.
- **Серверы**: Fly/NoClip/Speed — это то, от чего античиты кидают в кик/бан. Ставь на одиночку, на чужих серверах не стоит.
- 1.21.11 — последняя обфусцированная версия; дальше Yarn не будет, так что следующая ветка этого мода — уже на Mojang mappings.

Если хочешь, могу накидать ещё пару фич (X-Ray, Freecam, Step-up через `STEP_HEIGHT`, Fly с инерцией) или собрать простой внутриигровой экран с галочками вместо биндов.Готово — базовый клиент с пятью фичами и сборкой без ПК выше. Если что-то из API в 1.21.11 у тебя ругнётся при компиляции (например, сигнатура `setOverlayMessage` или категории биндов), скинь текст ошибки — подправлю точечно.
