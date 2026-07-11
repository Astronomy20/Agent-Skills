---
name: update-mod
description: >
  Aggiorna una mod Minecraft (qualsiasi mod, qualsiasi modloader: NeoForge/Forge/Fabric/Quilt) a
  una nuova versione di Minecraft, creando prima un branch GitHub dedicato alla versione target
  (mai su main), pullandolo in locale, poi eseguendo l'intero aggiornamento: gradle.properties, e
  se cambia anche il modloader, un porting agentico best-effort del codice sorgente. Trigger:
  "/update-mod", "aggiorna a Minecraft X", "porta la mod a NeoForge/Forge/Fabric/Quilt X",
  "aggiorna la versione di Minecraft", "crea un branch per la versione X".
---

# Update Mod (Minecraft/NeoForge/Forge/Fabric/Quilt)

Command: `/update-mod <minecraft-version> [modloader]`

Skill **agnostica rispetto alla mod**: non assume nulla sul nome del progetto, sul modloader
"predefinito" o sul set di API usate — tutto viene rilevato a runtime dai file del progetto
corrente (`gradle.properties`, `build.gradle`/`build.gradle.kts`, `settings.gradle`, sorgenti).

- `<minecraft-version>` — obbligatoria, es. `1.21.3`. **Non** è `mod_version` (il SemVer della mod
  stessa) — quello si decide *dopo*, a porting finito e testato, con la skill `changelog-release`.
- `[modloader]` — opzionale, uno tra `neoforge`/`forge`/`fabric`/`quilt`. **Default: il modloader
  del branch corrente** (quello da cui si parte) — non cambiare loader a meno che l'utente non lo
  chieda esplicitamente. Nessuna versione di MC ha un loader "canonico": tutti e quattro supportano
  quasi ogni versione, quindi senza indicazione esplicita nessuno viene cambiato.

## Rilevare il progetto corrente (fai sempre questo per primo)

Prima di proporre qualunque valore, capisci con cosa hai a che fare — non assumerlo da un progetto
precedente:

1. **Nome/tipo di mod**: `settings.gradle`/`gradle.properties` (`mod_id`, `mod_name` o simili) —
   usa questi valori nei messaggi invece di un nome fisso.
2. **Modloader corrente**: cerca in `build.gradle`/`build.gradle.kts` i plugin applicati —
   `net.neoforged.moddev` → NeoForge, `net.minecraftforge.gradle` (o versioni più vecchie tipo
   ForgeGradle) → Forge, `fabric-loom` → Fabric, `org.quiltmc.loom` → Quilt. Non assumere un loader
   di default: se non è determinabile con certezza, chiedi.
3. **Versione MC corrente**: `minecraft_version` (o chiave equivalente) in `gradle.properties`.
4. **Mappings correnti**: Parchment (`parchment_*`), Yarn (`yarn_mappings`), Quilt Mappings, o MCP
   ufficiale (Forge) — la chiave usata dipende dal loader rilevato al punto 2.
5. **Dipendenze compat opzionali**: leggi `gradle.properties` per individuare quali integrazioni
   opzionali (es. JEI/REI/EMI, Patchouli, KubeJS, FTB Library/Quests, o qualsiasi altra libreria di
   compat) sono attualmente configurate — **non assumere un set fisso**: la lista varia da mod a
   mod. Aggiorna solo quelle chiavi che esistono già nel progetto.

## Branch model (condiviso con la skill `changelog-release`)

- **Mai lavorare o pushare su `main`.** Ogni versione MC supportata vive sul proprio branch.
- **Nome branch** = la versione MC. Due forme:
  - `1.21.1` — pinnato a quella patch esatta (`minecraft_version_range=[1.21.1]`).
  - `1.21.x` — traccia un intero minor tramite range aperto (`minecraft_version_range=[1.21,1.22)`).
  - Se non è ovvio quale delle due intenda l'utente, **chiedi** prima di creare il branch — non
    indovinare: pinnato è più prevedibile, range è più comodo per seguire patch future senza
    aggiornare manualmente.
- `mod_version` è indipendente per branch — questa skill non lo tocca mai (`changelog-release` lo
  gestisce, a porting concluso).
- **Il branch di default su GitHub segue sempre la versione MC più recente** tra i branch
  esistenti. Quando il nuovo branch rappresenta la versione MC più alta (confronto numerico, non
  lessicografico — vedi Fase 1), diventa il nuovo default. Se invece stai creando un branch per una
  versione MC **più vecchia** (es. per aggiungere/mantenere supporto legacy), il default **non
  cambia** — resta su quello più recente.

## Fase 0 — Controllo di sicurezza pre-branch (obbligatorio, mai saltare)

**Prima di qualsiasi altra cosa**, verifica lo stato del branch corrente:

1. `git status --short` deve essere **pulito**: nessuna modifica staged/unstaged, nessun untracked
   rilevante (ignora artefatti di build come `build/`). Se non è pulito → **fermati**, riporta cosa
   è non committato, chiedi come procedere. Mai creare un nuovo branch con working tree sporco.
2. Verifica che il branch corrente sia **in sync con `origin`**: `git status -sb` non deve mostrare
   `ahead`/`behind`. Se ci sono commit locali non pushati → **fermati** e chiedi se pusharli prima —
   mai lasciare lavoro "orfano" solo in locale mentre si cambia la linea di lavoro principale.
3. Verifica che il nome del branch corrente abbia senso come base (una versione MC tipo
   `1.21.1`/`1.21.x`, non un branch WIP/feature casuale) — se è ambiguo, chiedi all'utente da quale
   branch dovrebbe partire il nuovo lavoro, prima di procedere.

Passa alla Fase 1 solo quando tutti e tre i controlli sono superati.

## Fase 1 — Crea il branch e pullalo in locale

**Richiede conferma esplicita dell'utente prima del push** — creare un branch pubblico su GitHub è
un'azione visibile.

1. Determina il nome (vedi sopra, chiedi se ambiguo tra pinnato/range).
2. `git checkout -b <branch-name>` dal branch sorgente verificato in Fase 0.
3. Conferma con l'utente, poi `git push -u origin <branch-name>` — il checkout locale lo rende già
   "pullato" in locale, non serve un fetch/pull separato dopo il primo push.
4. Se il branch sorgente aveva un `CHANGELOG.md`, il nuovo branch lo eredita automaticamente dal
   checkout — **azzera `## [Unreleased]`** in cima (il nuovo branch inizia una propria cronologia
   dei rilasci, vedi `changelog-release`).
5. **Verifica se questo è il branch con la versione MC più recente**, e in caso affermativo
   impostalo come nuovo default su GitHub:
   ```
   git branch -r | grep -v HEAD | sed 's#origin/##' | grep -E '^[0-9]+\.[0-9]+(\.[0-9]+|\.x)?$' \
     | sed 's/\.x$/.9999/' | sort -t. -k1,1n -k2,2n -k3,3n
   ```
   Confronto **numerico** (major.minor.patch), non lessicografico — un branch `.x` conta come "patch
   più alta" di quel minor ai fini del confronto (`sed 's/\.x$/.9999/'` sopra è solo un trucco di
   ordinamento, non scriverlo mai letteralmente da nessuna parte). Se il branch appena creato risulta
   il più recente nella lista ordinata:
   1. Chiedi conferma esplicita (è un cambio di default branch, visibile a chiunque cloni/forka).
   2. `gh repo edit --default-branch <branch-name>`.
   3. Verifica: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` deve riportare il
      nuovo nome.
   Se non è il più recente (branch per supporto legacy di una versione MC più vecchia), **non
   toccare il default** — dillo esplicitamente nel riepilogo finale, così si legge come scelta
   intenzionale e non come dimenticanza.

## Fase 2 — Trova e conferma le versioni delle dipendenze

**Non scrivere numeri a memoria** — cercali, proponili, aspetta conferma prima di scrivere
`gradle.properties`.

1. WebFetch delle pagine ufficiali per la versione MC target, **in base al loader rilevato/target**:
   - NeoForge: `https://projects.neoforged.net/neoforged/neoforge` (versione raccomandata per quella
     MC).
   - Forge: `https://files.minecraftforge.net/net/minecraftforge/forge/` (indice file — trova la
     build raccomandata/più recente per la versione MC target) e i docs ufficiali MDK/getting-started
     linkati da lì per il setup corrente del plugin Gradle.
   - Fabric: `https://fabricmc.net/develop/` (pagina ufficiale getting-started/develop — versione
     corrente del plugin Loom + Fabric API) più l'API meta
     `https://meta.fabricmc.net/v2/versions/loader/<target-mc-version>` per la build esatta di
     Fabric Loader corrispondente a quella MC, e `https://meta.fabricmc.net/v2/versions/game` per
     confermare che la stringa di versione MC sia valida.
   - Quilt: `https://quiltmc.org/en/install/` o la documentazione ufficiale Quilt Loom/QSL corrente,
     più l'API meta equivalente (`https://meta.quiltmc.org/v3/versions/loader/<target-mc-version>`)
     per la build esatta di Quilt Loader.
   - Mappings: Parchment (`https://parchmentmc.org/docs/getting-started`, `parchment_minecraft_version`
     deve combaciare esattamente con `minecraft_version`) per NeoForge/Forge moderni; Yarn (dal
     meta Fabric, `https://meta.fabricmc.net/v2/versions/yarn/<target-mc-version>`) per Fabric; Quilt
     Mappings per Quilt; MCP ufficiale per Forge più datati che non usano Parchment.
2. Dipendenze compat opzionali: **solo quelle già rilevate come configurate nel progetto** (fase di
   rilevamento sopra) — non introdurne di nuove di tua iniziativa. Per ciascuna, verifica se esiste
   una build per la versione MC target; se non la trovi, **non inventare un numero** — dì
   all'utente che quell'integrazione non ha (ancora, o non avrà) supporto per quella versione, e
   chiedi se lasciarla non impostata (il che esclude il modulo compat corrispondente dalla build, se
   il progetto è strutturato per farlo condizionalmente).
3. Presenta la lista completa trovata (chiave → valore proposto → fonte) e **aspetta conferma**
   prima della Fase 3.

## Fase 3 — Aggiorna `gradle.properties`

Solo dopo la conferma della Fase 2. Le chiavi da aggiornare dipendono dal loader rilevato — **non
assumere nomi fissi**: verifica quelli realmente usati nel progetto (fase di rilevamento) e
aggiornali in modo coerente, tutti nello stesso commit (sono collegati — lasciarli disallineare
rompe la build). Esempi tipici per loader (verifica sempre contro le chiavi effettive del progetto):

```
# NeoForge/Forge moderni con Parchment
parchment_minecraft_version=<MC target>
parchment_mappings_version=<confermata in Fase 2>
minecraft_version=<MC target>
minecraft_version_range=[<MC target>]        # o [<minor>,<minor+1>) per un branch "x"
neo_version=<confermata in Fase 2>            # solo se loader = neoforge
forge_version=<confermata in Fase 2>          # solo se loader = forge

# Fabric
minecraft_version=<MC target>
yarn_mappings=<confermata in Fase 2>
loader_version=<confermata in Fase 2>
fabric_version=<confermata in Fase 2>

# Quilt
minecraft_version=<MC target>
quilt_mappings=<confermata in Fase 2>
quilt_loader_version=<confermata in Fase 2>
quilted_fabric_api_version=<confermata in Fase 2>
```

Più le chiavi compat confermate/lasciate non impostate in Fase 2. **Non toccare `mod_version`** —
non è compito di questa skill.

Se il loader **non** cambia, la Fase 3 è l'ultimo passo prima della Fase 5 (salta la Fase 4).

## Fase 4 — Porting del loader (solo se `[modloader]` differisce da quello corrente)

**Best-effort, agentico, non uno script deterministico.** Il test manuale in-game è previsto in
seguito — va bene così, l'obiettivo è portare tutto in modo ragionevole così da poter poi
rifinire in-game, non produrre codice garantito corretto al primo colpo.

### 4a — Toolchain (build.gradle / settings.gradle)

Usa quanto rilevato in "Rilevare il progetto corrente" per sapere il plugin/toolchain **attuale**
(es. `net.neoforged.moddev`, ForgeGradle, `fabric-loom`, `org.quiltmc.loom`) — non assumerlo.
Prima di toccare qualunque cosa:

1. WebFetch della documentazione/template ufficiale più recente del loader target (es. il template
   MDK di Forge, o il template Fabric/Quilt con Loom) — **non affidarti alla memoria** per le
   coordinate esatte dei plugin, cambiano da versione a versione.
2. Proponi il nuovo blocco plugin (`build.gradle`) e qualunque modifica a `settings.gradle`,
   incrociati con quanto trovato ufficialmente. Aspetta conferma prima di scrivere.
3. Aggiorna i nomi delle chiavi in `gradle.properties` se il loader target usa nomi diversi (vedi
   esempi in Fase 3) — verifica i nomi esatti dal template ufficiale, non inventarli.

### 4b — Rilevare la superficie API della mod (a runtime, non da una tabella fissa)

Non esiste una tabella fissa di package validi per "questa mod" — va costruita **al momento del
porting**, sul progetto reale:

1. Determina i package radice del loader corrente (es. `net.neoforged.*`/`net.minecraftforge.*` per
   NeoForge/Forge, `net.fabricmc.*` più `net.fabricmc.fabric.api.*` per Fabric, `org.quiltmc.*` per
   Quilt) e lancia `grep -rl "<package-radice>" src/main/java` (o `src/*/java` se ci sono source
   set multipli client/common) per ottenere l'elenco file **reale e aggiornato** da migrare — non
   fidarti di conteggi o elenchi visti in porting precedenti, il codice cambia nel tempo.
2. Raggruppa i risultati per concetto, non per singolo package — questi sono i concetti tipici da
   tradurre in qualunque porting fra loader Minecraft moderni (best-effort, verifica sempre contro
   la documentazione ufficiale del loader target durante il porting, non solo questa tabella):

| Concetto | NeoForge | Forge | Fabric | Quilt |
|---|---|---|---|---|
| Entry point della mod | `@Mod(MODID)` su una classe, costruttore `(IEventBus, ModContainer)` | `@Mod(MODID)`, costruttore basato su `FMLJavaModLoadingContext`/`IEventBus` a seconda della versione Forge | `implements ModInitializer { onInitialize() }`, dichiarato in `fabric.mod.json`, nessuna annotazione | `implements ModInitializer` (API Quilt o compat Fabric), dichiarato in `quilt.mod.json` |
| Event bus di gioco | `NeoForge.EVENT_BUS` + `@SubscribeEvent`/`@EventBusSubscriber` | `MinecraftForge.EVENT_BUS`, stesso pattern `@SubscribeEvent` | Nessun bus unico — registri di callback separati per tipo evento (`ServerLifecycleEvents`, ecc.), niente annotazioni | Simile a Fabric (spesso via Quilted Fabric API), callback per tipo evento |
| Marcatore client/dist | `net.neoforged.api.distmarker.{Dist,OnlyIn}` | `net.minecraftforge.api.distmarker.{Dist,OnlyIn}` (stesso pattern, package diverso) | Tipicamente diviso via source set `client`/`main`, non un'annotazione a runtime | Come Fabric, source set separati |
| Registry | `DeferredRegister.create(...)` + `RegisterEvent` | Stesso pattern `DeferredRegister` (eredità condivisa con NeoForge) | `Registry.register(BuiltInRegistries.X, id, instance)` diretto, nessun wrapper deferred | Simile a Fabric, `Registry.register` diretto |
| Capability (energia/fluido/item su un blocco) | `BlockCapability`/`ItemCapability` + `RegisterCapabilitiesEvent` | Sistema `Capability<T>` di Forge (`CapabilityManager`/`ICapabilityProvider`) — shape API sensibilmente diverso | Nessun sistema built-in — tipicamente `BlockApiLookup` di Fabric API | Quilted Fabric API o equivalente Quilt, pattern simile a Fabric |
| Networking | `PayloadRegistrar`/`IPayloadHandler` (`RegisterPayloadHandlersEvent`) | Dipende dalla versione Forge (canale `SimpleChannel` o una payload API più recente) | `ServerPlayNetworking`/`ClientPlayNetworking` + `CustomPayload` di Fabric API | Equivalente Quilt/Quilted Fabric API, pattern simile a Fabric |
| Config | `ModConfigSpec` | `ForgeConfigSpec` | Nessun sistema built-in (terze parti: Cloth Config, AutoConfig) | Quilt Config, o le stesse librerie terze di Fabric |

3. Fai la traduzione **file per file, package per package** (non un find-and-replace testuale alla
   cieca — ogni call site va guardato, la mappatura sopra è concettuale, non uno swap 1:1 letterale).
4. Segnala esplicitamente ogni punto in cui hai dovuto fare una scelta incerta (es. quale
   equivalente Fabric/Quilt usare per una capability complessa) con un commento
   `// TODO(loader-port):` nel codice, così è ricercabile in seguito.

## Fase 5 — Ciclo compila-e-correggi

1. Prova a compilare (usa Gradle se disponibile nella sessione; altrimenti chiedi all'utente di
   lanciarlo lui, o verifica se il progetto ha un workaround noto documentato altrove per compilare
   senza Gradle in sessione).
2. Per ogni errore: capisci se è (a) un cambio di API MC/loader tra versioni (fix diretto), o (b)
   una conseguenza del porting di Fase 4 (controlla la mappatura, correggi). Itera finché non
   compila, segnalando ogni punto dubbio invece di "zittire" l'errore con un cast/soppressione a
   caso.
3. Non è richiesto farlo compilare al 100% in autonomia se emergono reali ambiguità di design (es.
   un'API NeoForge/Forge senza equivalente diretto in Fabric/Quilt) — in quel caso, fermati e chiedi
   come procedere invece di inventare un workaround silenzioso.

## Fase 6 — Changelog e riepilogo finale

1. Aggiungi a `CHANGELOG.md` → `## [Unreleased]` → `### Changed`:
   `**Breaking:** update to Minecraft <X> (<loader> <Y>)` (o menziona il cambio di loader se
   applicabile) — quasi sempre breaking per chiunque abbia compilato contro la versione precedente.
2. Riporta un riepilogo all'utente: cosa è stato aggiornato, cosa compila, cosa resta con un marker
   `// TODO(loader-port):` da rivedere a mano, e che **il test in-game è obbligatorio** prima di
   considerare il porting concluso (non saltarlo per un cambio di loader — troppa superficie per
   fidarsi solo della compilazione).
3. **Non invocare `changelog-release` per taggare/rilasciare automaticamente** — quella skill
   decide la versione solo a porting testato e con conferma dell'utente, non subito dopo il porting
   meccanico.

## Cosa non fare

- ❌ Saltare la Fase 0 (controllo pulizia branch corrente) — mai creare un nuovo branch con working
  tree sporco/non pushato.
- ❌ Lavorare o pushare su `main`.
- ❌ Scrivere numeri di versione (loader/mappings/compat) senza prima cercarli e confermarli con
  l'utente.
- ❌ Cambiare loader senza che l'utente lo chieda esplicitamente (default: restare sul loader
  corrente).
- ❌ Assumere un loader "canonico" o un set fisso di dipendenze compat da una mod precedente —
  rileva sempre dal progetto corrente.
- ❌ Fare un find-and-replace testuale alla cieca per il porting del loader — va fatto sito per
  sito, concettualmente.
- ❌ Zittire errori di compilazione post-porting con cast/soppressioni invece di segnalarli.
- ❌ Taggare/rilasciare da questa skill — è compito di `changelog-release`, dopo test manuale.
- ❌ Toccare `mod_version` — non è un asse gestito da questa skill.
- ❌ Cambiare il branch di default su GitHub senza prima confrontare numericamente tutte le versioni
  MC dei branch esistenti — un confronto lessicografico sbaglierebbe (es. leggere "1.9" come "più
  recente" di "1.21" testualmente).
- ❌ Cambiare il branch di default per un branch di supporto legacy (una versione MC più vecchia) —
  il default segue sempre e solo la versione più recente, mai quella appena creata per definizione.
