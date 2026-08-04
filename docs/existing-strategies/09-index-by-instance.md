# 09 — Index by instance

What already exists, so you can check before you build. Grep this file for a boss name.

Counts are registered action names / trigger names / multipliers, and total lines of code.
An empty multiplier column means the instance orders everything by relevance alone.

| Instance | Dir | Map | Strategy name | Actions | Triggers | Multipliers | LoC | Notes |
|---|---|---|---|---|---|---|---|---|
| Ruins of Ahn'Qiraj | `Aq20/` | 509 | `aq20` | 1 | 1 | 0 | 288 | near-empty stub |
| Temple of Ahn'Qiraj | `Aq40/` | 531 | `aq40` | 49 | 44 | 10 | 12415 | most heavily documented; `Scripts` hooks |
| Black Temple | `BT/` | 564 | `blacktemple` | 61 | 61 | 30 | 7835 | richest multiplier set |
| Blackwing Lair | `BWL/` | 469 | `bwl` | 32 | 33 | 9 | 2918 | |
| Eye of Eternity | `EoE/` | 616 | `wotlk-eoe` | 6 | 2 | 1 | 831 | drake vehicle |
| Gruul's Lair | `Gruul/` | 565 | `gruulslair` | 13 | 13 | 8 | 1666 | |
| Hyjal Summit | `Hyjal/` | 534 | `hyjal` | 31 | 31 | 13 | 3776 | `Scripts` hooks |
| Icecrown Citadel | `ICC/` | 631 | `icc` | 55 | 52 | 13 | 20796 | largest; `Scripts` hooks |
| Karazhan | `Kara/` | 532 | `karazhan` | 37 | 37 | 18 | 4334 | heaviest marker user |
| Magtheridon's Lair | `Mag/` | 544 | `magtheridon` | 12 | 12 | 4 | 1874 | `Scripts` hooks |
| Molten Core | `MC/` | 409 | `moltencore` | 16 | 16 | 3 | 871 | mostly resistance actions |
| Naxxramas | `Naxx/` | 533 | `naxx` | 27 | 26 | 11 | 3597 | **Heigan/Patchwerk/Gothik disabled** |
| Onyxia's Lair | `Ony/` | 249 | `onyxia` | 5 | 5 | 0 | 575 | |
| Obsidian Sanctum | `OS/` | 615 | `wotlk-os` | 6 | 7 | 1 | 757 | |
| Ruby Sanctum | `RS/` | 724 | `rs` | 27 | 28 | 12 | 6048 | heaviest server-side encounter manipulation |
| Serpentshrine Cavern | `SSC/` | 548 | `ssc` | 50 | 48 | 28 | 7083 | tainted-core item relay |
| Tempest Keep | `TK/` | 550 | `tempestkeep` | 44 | 40 | 17 | 5455 | **copies a core boss class** (see ch. 10) |
| Ulduar | `Uld/` | 603 | `ulduar` | 67 | 69 | **0** | 7457 | no multipliers; uses cheats/teleports instead |
| Vault of Archavon | `VoA/` | 624 | `voa` | 6 | 6 | 0 | 582 | |
| Zul'Aman | `ZA/` | 568 | `zulaman` | 28 | 28 | 18 | 2963 | |

Dungeons live in [src/Ai/Dungeon/](../../src/Ai/Dungeon/) (~9.2k LoC, same architecture):
AC, UK, UP, Nex, Occ, CoS, HoS, HoL, DTK, AN, GD, VH, OK, FoS, PoS, HoR, ToC.

---

## Cross-instance mechanic frequency

How many bosses already have a given verb. If your mechanic is high on this list, **there is
certainly a canonical implementation** — do not write a new one.

| Verb | Count | Canonical |
|---|---|---|
| `misdirect boss to main tank` | ~20 | `HighWarlordNajentusMisdirectBossToMainTankAction` |
| `<boss> fire/frost/nature/shadow resistance` | ~19 | `BossFireResistanceAction` (base library) |
| `tanks/main tank position boss` | ~18 | `HighWarlordNajentusTanksPositionBossAction` |
| `spread ranged [in circle/arc]` / `disperse ranged` | ~15 | `HighWarlordNajentusDisperseRangedAction` |
| `mark <thing>` | ~14 | `MarkTargetWithSkull` + `IsMechanicTrackerBot` |
| avoid/move away from `<hazard>` | ~30 | see [ch. 04](04-hazards-and-avoidance.md) |
| `assign dps priority` | ~10 | `FourHorsemenAttackInOrderAction` |
| `<x> tank position` (fixed spot) | ~12 | as above |
| `manage <x> timer` / `erase timers and trackers` | ~10 | `SupremusManagePhaseTimerAction` / `BlackTempleEraseTimersAndTrackersAction` |
| `choose target` | 12 | `Aq40CthunChooseTargetAction` |
| `run away from whirlwind` | 4 | `Aq40SarturaAvoidWhirlwindAction` |
| `fall from floor action` | 5 | `KologarnFallFromFloorAction` |
| `<x> cheat action` | 5 | see [ch. 06](06-server-side-hooks-and-cheats.md#k6-cheat-actions) |
| tank swap | 3 | `aq40 fankriss tank swap`, `al'ar swap tanks on boss`, `bwl ebonroc taunt` |

---

## Full action inventory by instance

Grep target. These are the registered names; each is a `creators["…"]` entry in the instance's
`<X>ActionContext.h` with a matching class in `<X>Actions.h`.

### Molten Core (`moltencore`)
`mc baron geddon fire resistance` · `mc core hound mark` · `mc garr fire resistance` ·
`mc gehennas shadow resistance` · `mc golemagg assist tank attack core rager` ·
`mc golemagg fire resistance` · `mc golemagg main tank attack golemagg` · `mc golemagg mark boss` ·
`mc lucifron shadow resistance` · `mc magmadar fire resistance` · `mc majordomo shadow resistance` ·
`mc move from baron geddon` · `mc move from group` · `mc ragnaros fire resistance` ·
`mc shazzrah move away` · `mc sulfuron harbinger fire resistance`

### Blackwing Lair (`bwl`)
`bwl attack hatcher` · `bwl attack pack anchor` · `bwl broodlord fire resistance` ·
`bwl broodlord position` · `bwl check onyxia scale cloak` · `bwl chromaggus position` ·
`bwl death talon wyrmguard ranged move away` · `bwl death talon wyrmguard tank move away` ·
`bwl drake position` · `bwl ebonroc fire resistance` · `bwl ebonroc taunt` ·
`bwl firemaw fire resistance` · `bwl flamegor fire resistance` · `bwl nefarian attack bone construct` ·
`bwl nefarian attack corrupted totem` · `bwl nefarian construct blink` · `bwl nefarian construct blizzard` ·
`bwl nefarian construct dps switch` · `bwl nefarian construct frost trap` ·
`bwl nefarian construct piercing howl` · `bwl nefarian fear ward` · `bwl nefarian phase one position` ·
`bwl nefarian position` · `bwl nefarian shackle construct` · `bwl razorgore avoid aoe` ·
`bwl razorgore fire resistance` · `bwl razorgore mark boss` · `bwl technician spread` ·
`bwl turn off suppression device` · `bwl use hourglass sand` · `bwl vaelastrasz fire resistance` ·
`bwl vaelastrasz move away`

### Onyxia's Lair (`onyxia`)
`ony avoid eggs move` · `ony kill whelps` · `ony move to safe zone` · `ony move to side` · `ony spread out`

### Ruins of Ahn'Qiraj (`aq20`)
`aq20 use crystal`

### Temple of Ahn'Qiraj (`aq40`)
**Shared:** `aq40 erase timers and trackers` · `aq40 manage resistance strategies`
**Trash:** `aq40 trash avoid dangerous aoe` · `aq40 trash burn mana drainer` · `aq40 trash choose target` ·
`aq40 trash clear mana drainer` · `aq40 trash drain mana drainer`
**Skeram:** `aq40 skeram acquire platform target` · `aq40 skeram control mind control` ·
`aq40 skeram focus real boss` · `aq40 skeram interrupt` · `aq40 skeram raid position` · `aq40 skeram tank images`
**Bug Trio:** `aq40 bug trio avoid poison cloud` · `aq40 bug trio choose target` · `aq40 bug trio interrupt heal`
**Sartura:** `aq40 sartura avoid whirlwind` · `aq40 sartura choose target`
**Fankriss:** `aq40 fankriss choose target` · `aq40 fankriss tank swap`
**Huhuran:** `aq40 huhuran choose target` · `aq40 huhuran poison spread`
**Twin Emperors:** `aq40 twin avoid hazard` · `aq40 twin avoid veklor` · `aq40 twin choose target` ·
`aq40 twin drag veknilash` · `aq40 twin healer anchor` · `aq40 twin pre swap` · `aq40 twin staging` ·
`aq40 twin tank` · `aq40 twin tank gap close` · `aq40 twin warlock tank`
**Ouro:** `aq40 ouro avoid sand blast` · `aq40 ouro avoid submerge` · `aq40 ouro avoid sweep` ·
`aq40 ouro choose target` · `aq40 ouro hold melee contact`
**Viscidus:** `aq40 viscidus choose target` · `aq40 viscidus shatter` · `aq40 viscidus use frost`
**C'Thun:** `aq40 cthun avoid dark glare` · `aq40 cthun choose target` · `aq40 cthun entry` ·
`aq40 cthun interrupt eye` · `aq40 cthun maintain spread` · `aq40 cthun phase2 add priority` ·
`aq40 cthun stomach dps` · `aq40 cthun stomach exit` · `aq40 cthun vulnerable burst`

### Karazhan (`karazhan`)
`attumen the huntsman manage dps timer` · `attumen the huntsman mark target` ·
`attumen the huntsman split bosses` · `attumen the huntsman stack behind` · `big bad wolf position boss` ·
`big bad wolf run away from boss` · `maiden of virtue move boss to healer` · `maiden of virtue position ranged` ·
`mana warp stun creature before warp breach` · `moroes main tank attack boss` · `moroes mark target` ·
`netherspite avoid beam and void zone` · `netherspite banish phase avoid void zone` ·
`netherspite block blue beam` · `netherspite block green beam` · `netherspite block red beam` ·
`netherspite manage timers and trackers` · `nightbane cast fear ward on main tank` ·
`nightbane control pet aggression` · `nightbane flight phase movement` ·
`nightbane ground phase position boss` · `nightbane ground phase rotate ranged positions` ·
`nightbane manage timers and trackers` · `prince malchezaar enfeebled avoid hazard` ·
`prince malchezaar main tank movement` · `prince malchezaar non tank avoid infernal` ·
`romulo and julianne mark target` · `shade of aran mark conjured elemental` ·
`shade of aran ranged maintain distance` · `shade of aran run away from arcane explosion` ·
`shade of aran stop moving during flame wreath` · `terestian illhoof mark target` ·
`the curator mark astral flare` · `the curator position boss` · `the curator spread ranged` ·
`wizard of oz mark target` · `wizard of oz scorch strawman`

### Gruul's Lair (`gruulslair`)
`gruul the dragonkiller shatter spread` · `gruul the dragonkiller spread ranged` ·
`gruul the dragonkiller tanks position boss` · `high king maulgar assign dps priority` ·
`high king maulgar banish fel stalker` · `high king maulgar first assist tank attack olm` ·
`high king maulgar mage tank attack krosh` · `high king maulgar main tank attack maulgar` ·
`high king maulgar misdirect ogres to tanks` · `high king maulgar moonkin tank attack kiggler` ·
`high king maulgar move away from blast nova danger` · `high king maulgar run away from whirlwind` ·
`high king maulgar second assist tank attack blindeye`

### Magtheridon's Lair (`magtheridon`)
`magtheridon assign dps priority` · `magtheridon erase timers and trackers` ·
`magtheridon first assist tank attack nw channeler` · `magtheridon main tank attack first three channelers` ·
`magtheridon main tank position boss` · `magtheridon manage timers and assignments` ·
`magtheridon misdirect hellfire channelers to main tank` · `magtheridon move out of debris` ·
`magtheridon second assist tank attack ne channeler` · `magtheridon spread ranged` ·
`magtheridon use manticron cube` · `magtheridon warlock cc burning abyssal`

### Serpentshrine Cavern (`ssc`)
**Shared:** `serpent shrine cavern erase timers and trackers` · `greyheart tidecaller mark water elemental totem` ·
`underbog colossus escape toxic pool`
**Hydross:** `hydross the unstable frost phase spread out` · `hydross the unstable manage timers` ·
`hydross the unstable misdirect boss to tank` · `hydross the unstable position frost tank` ·
`hydross the unstable position nature tank` · `hydross the unstable prioritize elemental adds` ·
`hydross the unstable stop dps upon phase change`
**The Lurker Below:** `the lurker below manage spout timer` · `the lurker below position main tank` ·
`the lurker below run around behind boss` · `the lurker below spread ranged in arc` ·
`the lurker below tanks pick up adds`
**Leotheras:** `leotheras the blind demon form tank attack boss` · `leotheras the blind destroy inner demon` ·
`leotheras the blind final phase assign dps priority` · `leotheras the blind manage dps wait timers` ·
`leotheras the blind melee dps run away from boss` · `leotheras the blind melee tanks don't attack demon form` ·
`leotheras the blind misdirect boss to demon form tank` · `leotheras the blind position ranged` ·
`leotheras the blind run away from whirlwind` · `leotheras the blind target spellbinders`
**Karathress:** `fathom-lord karathress assign dps priority` ·
`fathom-lord karathress first assist tank position caribdis` · `fathom-lord karathress main tank position boss` ·
`fathom-lord karathress manage dps timer` · `fathom-lord karathress misdirect bosses to tanks` ·
`fathom-lord karathress position caribdis tank healer` ·
`fathom-lord karathress second assist tank position sharkkis` ·
`fathom-lord karathress third assist tank position tidalvess`
**Morogrim:** `morogrim tidewalker misdirect boss to main tank` · `morogrim tidewalker move boss to tank position` ·
`morogrim tidewalker phase 2 reposition ranged`
**Lady Vashj:** `lady vashj assign phase 2 and phase 3 dps priority` · `lady vashj avoid toxic spores` ·
`lady vashj destroy tainted core` · `lady vashj loot tainted core` · `lady vashj main tank position boss` ·
`lady vashj misdirect boss to main tank` · `lady vashj misdirect strider to first assist tank` ·
`lady vashj pass the tainted core` · `lady vashj phase 1 spread ranged in arc` ·
`lady vashj set grounding totem in main tank group` · `lady vashj static charge move away from group` ·
`lady vashj tank attack and move away strider` · `lady vashj teleport to tainted elemental` ·
`lady vashj use free action abilities`

### Tempest Keep (`tempestkeep`)
**Al'ar:** `al'ar assist tanks pick up embers` · `al'ar avoid flame patches and dive bombs` ·
`al'ar boss tanks move between platforms` · `al'ar jump from platform` · `al'ar manage phase tracker` ·
`al'ar melee dps move between platforms` · `al'ar misdirect boss to main tank` · `al'ar move away from rebirth` ·
`al'ar ranged and ember tank move under platforms` · `al'ar ranged dps prioritize embers` ·
`al'ar return to room center` · `al'ar swap tanks on boss`
**Void Reaver:** `void reaver avoid arcane orb` · `void reaver erase trackers` · `void reaver spread ranged` ·
`void reaver tanks position boss` · `void reaver use aggro dump ability`
**Solarian:** `high astromancer solarian cast fear ward on main tank` ·
`high astromancer solarian move away from group` · `high astromancer solarian ranged leave space for melee` ·
`high astromancer solarian stack for aoe` · `high astromancer solarian target solarium priests`
**Kael'thas:** `kael'thas sunstrider assign advisor dps priority` ·
`kael'thas sunstrider assign legendary weapon dps priority` · `kael'thas sunstrider avoid flame strike` ·
`kael'thas sunstrider break mind control` · `kael'thas sunstrider break through shock barrier` ·
`kael'thas sunstrider cast fear ward on sanguinar tank` ·
`kael'thas sunstrider first assist tank position telonicus` ·
`kael'thas sunstrider handle advisor roles in phase 3` · `kael'thas sunstrider handle phoenixes and eggs` ·
`kael'thas sunstrider kite thaladred` · `kael'thas sunstrider loot legendary weapons` ·
`kael'thas sunstrider main tank position boss` · `kael'thas sunstrider main tank position sanguinar` ·
`kael'thas sunstrider manage advisor dps timer` · `kael'thas sunstrider misdirect advisors to tanks` ·
`kael'thas sunstrider move devastation away` · `kael'thas sunstrider reequip gear` ·
`kael'thas sunstrider spread and move away from capernian` · `kael'thas sunstrider spread out in midair` ·
`kael'thas sunstrider use legendary weapons` · `kael'thas sunstrider warlock tank position capernian`
**Trash:** `crimson hand centurion cast polymorph`

### Hyjal Summit (`hyjal`)
`hyjal summit erase trackers` · `rage winterchill main tank position boss` ·
`rage winterchill melee get out of death and decay` · `rage winterchill misdirect boss to main tank` ·
`rage winterchill spread ranged in circle` · `anetheron assign dps priority` ·
`anetheron bring infernal to infernal tank` · `anetheron first assist tank pick up infernals` ·
`anetheron main tank position boss` · `anetheron misdirect boss and infernals to tanks` ·
`anetheron spread ranged in circle` · `kaz'rogal assist tanks move in front of boss` ·
`kaz'rogal cast shadow protection spell` · `kaz'rogal low mana bot take defensive measures` ·
`kaz'rogal main tank position boss` · `kaz'rogal misdirect boss to main tank` ·
`kaz'rogal spread ranged in arc` · `azgalor disperse ranged` · `azgalor first assist tank position doomguard` ·
`azgalor main tank position boss` · `azgalor melee get out of fire and swap targets` ·
`azgalor misdirect boss to main tank` · `azgalor move to doomguard tank` ·
`azgalor ranged dps prioritize doomguards` · `azgalor wait at safe position` · `archimonde avoid doomfire` ·
`archimonde cast fear immunity spell` · `archimonde misdirect boss to main tank` ·
`archimonde move boss to initial position` · `archimonde remove doomfire dot` ·
`archimonde spread to avoid air burst`

### Black Temple (`blacktemple`)
**Shared:** `black temple erase timers and trackers`
**Naj'entus:** `high warlord naj'entus disperse ranged` · `high warlord naj'entus misdirect boss to main tank` ·
`high warlord naj'entus remove impaling spine` · `high warlord naj'entus tanks position boss` ·
`high warlord naj'entus throw impaling spine`
**Supremus:** `supremus disperse ranged` · `supremus kite boss` · `supremus manage phase timer` ·
`supremus misdirect boss to main tank` · `supremus move away from volcanos`
**Shade of Akama:** `shade of akama melee dps prioritize channelers`
**Teron Gorefiend:** `teron gorefiend avoid shadow of death` ·
`teron gorefiend control and destroy shadowy constructs` · `teron gorefiend misdirect boss to main tank` ·
`teron gorefiend move to corner to die` · `teron gorefiend position ranged on balcony` ·
`teron gorefiend tanks position boss`
**Gurtogg:** `gurtogg bloodboil manage phase timer` · `gurtogg bloodboil misdirect boss to main tank` ·
`gurtogg bloodboil ranged move away from enraged player` · `gurtogg bloodboil rotate ranged groups` ·
`gurtogg bloodboil tanks position boss`
**Reliquary of Souls:** `reliquary of souls adjust distance from suffering` ·
`reliquary of souls healers dps suffering` · `reliquary of souls misdirect boss to main tank` ·
`reliquary of souls spell reflect deaden` · `reliquary of souls spellsteal rune shield`
**Mother Shahraz:** `mother shahraz melee dps wait at safe position` ·
`mother shahraz misdirect boss to main tank` · `mother shahraz position ranged under pillar` ·
`mother shahraz run away to break fatal attraction` · `mother shahraz tanks position boss under pillar`
**Illidari Council:** `illidari council assign dps targets` · `illidari council command pets to attack gathios` ·
`illidari council disperse ranged` · `illidari council first assist tank focus malande` ·
`illidari council mage tank position zerevor` · `illidari council main tank position gathios` ·
`illidari council main tank reflect judgement of command` · `illidari council manage dps timer` ·
`illidari council misdirect bosses to tanks` · `illidari council position mage tank healer` ·
`illidari council second assist tank position darkshadow`
**Illidan:** `illidan stormrage assist tanks handle flames of azzinoth` ·
`illidan stormrage control pet aggression` · `illidan stormrage destroy hazards` ·
`illidan stormrage disperse ranged` · `illidan stormrage dps prioritize adds` ·
`illidan stormrage handle adds cheat` · `illidan stormrage isolate bot with parasite` ·
`illidan stormrage main tank reposition boss` · `illidan stormrage manage dps timer and rti` ·
`illidan stormrage melee go somewhere to not die` · `illidan stormrage misdirect to tank` ·
`illidan stormrage move away from landing point` · `illidan stormrage position above grate` ·
`illidan stormrage remove dark barrage` · `illidan stormrage set earthbind totem` ·
`illidan stormrage use shadow trap` · `illidan stormrage warlock tank handle demon boss`

### Zul'Aman (`zulaman`)
`akil'zon manage electrical storm timer` · `akil'zon misdirect boss to main tank` ·
`akil'zon move to eye of the storm` · `akil'zon spread ranged` · `akil'zon tanks position boss` ·
`amani'shi medicine man mark ward` · `halazzi assign dps priority` ·
`halazzi first assist tank attack spirit lynx` · `halazzi main tank position boss` ·
`halazzi misdirect boss to main tank` · `hex lord malacrass assign dps priority` ·
`hex lord malacrass casters stop attacking` · `hex lord malacrass misdirect boss to main tank` ·
`hex lord malacrass move away from freezing trap` · `hex lord malacrass run away from whirlwind` ·
`jan'alai avoid fire bombs` · `jan'alai mark amani'shi hatchers` · `jan'alai misdirect boss to main tank` ·
`jan'alai spread ranged in circle` · `jan'alai tanks position boss` · `nalorakk misdirect boss to main tank` ·
`nalorakk spread ranged` · `nalorakk tanks position boss` · `zul'jin avoid cyclones` ·
`zul'jin misdirect boss to main tank` · `zul'jin run away from whirlwind` · `zul'jin spread ranged` ·
`zul'jin tanks position boss`

### Naxxramas (`naxx`)
`anub'rekhan choose target` · `anub'rekhan position` · `four horsemen attack in order` ·
`four horsemen attract alternatively` · `gluth choose target` · `gluth position` · `gluth slowdown` ·
`grobbulus go behind the boss` · `grobbulus move away` · `grobbulus move center` ·
~~`heigan dance melee`~~ · ~~`heigan dance ranged`~~ *(registered but implementation and wiring commented out)* ·
`kel'thuzad choose target` · `kel'thuzad position` · `loatheb choose target` · `loatheb position` ·
~~`patchwerk ranged position`~~ *(wiring commented out)* · `razuvious target` ·
`razuvious use obedience crystal` · `rotate grobbulus` · `sapphiron flight position` ·
`sapphiron ground position` · `thaddius attack nearest pet` · `thaddius melee to place` ·
`thaddius move polarity` · `thaddius move to platform` · `thaddius ranged to place`

### Obsidian Sanctum (`wotlk-os`)
`avoid flame tsunami` · `avoid twilight fissure` · `enter twilight portal` · `exit twilight portal` ·
`sartharion attack priority` · `sartharion tank position`

### Eye of Eternity (`wotlk-eoe`)
`eoe drake attack` · `eoe fly drake` · `kill power spark` · `malygos position` · `malygos target` ·
`pull power spark`

### Vault of Archavon (`voa`)
`emalon fall from floor action` · `emalon lighting nova action` · `emalon mark boss action` ·
`emalon nature resistance action` · `emalon overcharge action` · `koralon fire resistance action`

### Ulduar (`ulduar`)
**Flame Leviathan:** `flame leviathan enter vehicle` · `flame leviathan vehicle`
**Razorscale:** `razorscale avoid devouring flames` · `razorscale avoid sentinel` ·
`razorscale avoid whirlwind` · `razorscale fire resistance action` · `razorscale fuse armor action` ·
`razorscale grounded` · `razorscale harpoon action` · `razorscale ignore flying alone`
**Ignis:** `ignis fire resistance action`
**Kologarn:** `kologarn crunch armor action` · `kologarn eyebeam action` · `kologarn fall from floor action` ·
`kologarn mark dps target action` · `kologarn nature resistance action` · `kologarn rti target action` ·
`kologarn rubble slowdown action`
**Auriaya:** `auriaya fall from floor action`
**Hodir:** `hodir biting cold jump` · `hodir frost resistance action` · `hodir move snowpacked icicle`
**Thorim:** `thorim arena positioning action` · `thorim frost resistance action` ·
`thorim gauntlet positioning action` · `thorim mark dps target action` · `thorim nature resistance action` ·
`thorim phase 2 positioning action` · `thorim unbalancing strike action`
**Freya:** `freya fire resistance action` · `freya mark dps target action` · `freya move away nature bomb` ·
`freya move to healing spore action` · `freya nature resistance action`
**Mimiron:** `mimiron aerial command unit action` · `mimiron cheat action` · `mimiron fire resistance action` ·
`mimiron p3wx2 laser barrage action` · `mimiron phase 1 positioning action` · `mimiron phase 4 mark dps action` ·
`mimiron rapid burst action` · `mimiron rocket strike action` · `mimiron shock blast action`
**Iron Assembly:** `iron assembly lightning tendrils action` · `iron assembly overload action` ·
`iron assembly rune of power action`
**Vezax:** `vezax cheat action` · `vezax mark of the faceless action` · `vezax shadow crash action` ·
`vezax shadow resistance action`
**Yogg-Saron:** `sara shadow resistance action` · `yogg-saron boss room movement cheat action` ·
`yogg-saron brain link action` · `yogg-saron death orb action` · `yogg-saron fall from floor action` ·
`yogg-saron guardian positioning action` · `yogg-saron illusion room action` ·
`yogg-saron lunatic gaze action` · `yogg-saron malady of the mind action` · `yogg-saron mark target action` ·
`yogg-saron move to enter portal action` · `yogg-saron move to exit portal action` ·
`yogg-saron ominous cloud cheat action` · `yogg-saron phase 3 positioning action` ·
`yogg-saron sanity action` · `yogg-saron shadow resistance action` · `yogg-saron use portal action`

### Icecrown Citadel (`icc`)
`icc adds dbs` · `icc adds lady deathwhisper` · `icc bpc ball of flame` · `icc bpc empowered vortex` ·
`icc bpc keleseth tank` · `icc bpc kinetic bomb` · `icc bpc main tank` · `icc bql group position` ·
`icc bql pact of darkfallen` · `icc bql vampiric bite` · `icc cannon fire` · `icc dark reckoning` ·
`icc dbs tank position` · `icc dogs tank position` · `icc festergut avoid malleable goo` ·
`icc festergut group position` · `icc festergut spore` · `icc gunship enter cannon` ·
`icc gunship rocket jump` · `icc gunship rocket pack setup` · `icc lich king adds` ·
`icc lich king necrotic plague` · `icc lich king shadow trap` · `icc lich king spirit bomb` ·
`icc lich king winter` · `icc lm tank position` · `icc putricide abomination` ·
`icc putricide avoid malleable goo` · `icc putricide gas cloud` · `icc putricide growing ooze puddle` ·
`icc putricide mutated plague` · `icc putricide volatile ooze` · `icc ranged position lady deathwhisper` ·
`icc rotface avoid vile gas` · `icc rotface group position` · `icc rotface move away from explosion` ·
`icc rotface tank position` · `icc rotting frost giant tank position` · `icc shade lady deathwhisper` ·
`icc sindragosa blistering cold` · `icc sindragosa chilled to the bone` · `icc sindragosa frost beacon` ·
`icc sindragosa frost bomb` · `icc sindragosa group position` · `icc sindragosa hot` ·
`icc sindragosa mystic buffet` · `icc sindragosa unchained magic` · `icc sister svalna` · `icc spike` ·
`icc valithria dream cloud` · `icc valithria group` · `icc valithria heal` · `icc valithria portal` ·
`icc valithria zombie kite` · `icc valkyre spear`

### Ruby Sanctum (`rs`)
`rs trash adds` · `rs trash assist tank` · `rs trash main tank` · `rs trash ranged` ·
`rs baltharus avoid front` · `rs baltharus brand` · `rs baltharus healer position` ·
`rs baltharus tank position` · `rs saviana avoid front` · `rs saviana conflagration` ·
`rs saviana melee spread` · `rs saviana tank position` · `rs zarithrian adds` · `rs zarithrian tank` ·
`rs halion adds` · `rs halion add tank` · `rs halion avoid cones` · `rs halion combustion` ·
`rs halion consumption` · `rs halion cutter` · `rs halion enter portal` · `rs halion heal consumption` ·
`rs halion meteor` · `rs halion p2 avoid cones` · `rs halion p2 tank position` · `rs halion start position` ·
`rs halion tank position`
