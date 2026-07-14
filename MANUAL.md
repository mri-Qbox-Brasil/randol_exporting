# randol_exporting — Manual

Trabalho de exportação de veículos com progressão por XP e tiers: o jogador recebe um carro com bomba e tem tempo limitado para entregá-lo no ponto de drop-off antes que ele exploda.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Tiers e progressão](#tiers-e-progressão)
5. [Fluxo da missão](#fluxo-da-missão)
6. [Banco de dados](#banco-de-dados)
7. [Integrações](#integrações)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` **ou** `es_extended` | Sim | Bridge automático em `bridge/` |
| `ox_lib` | Sim | Callbacks, locale, points, zones, textUI, contexto, scaleform. Precisa ter `GetActivePlayers()` no `init.lua` (versão recente) |
| `oxmysql` | Sim | Persistência de XP na tabela `export_xp` |
| `qb-target` | Não | Só se `UseTarget = true`. Com `false`, a interação com o NPC usa `lib.zones` + tecla **E** |
| Script de combustível | Não | Só se `Fuel.enable = true` |

---

## Instalação

1. Copie a pasta `randol_exporting` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_exporting
   ```
3. Não é preciso importar SQL: a tabela `export_xp` é criada automaticamente no start do recurso.
4. Não há itens de inventário a cadastrar.

---

## Configuração

### `shared.lua` (compartilhado)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Debug` | bool | Sim | Desenha as zonas (`lib.zones`) e imprime no console do servidor os dados da missão gerada |
| `PedModel` | hash | Sim | Modelo do NPC que dá o trabalho (padrão `csb_isldj_03`) |
| `PedCoords` | `vec4` | Sim | Posição e heading do NPC. O NPC é criado a 30 metros de distância |
| `VehicleSpawn` | `vec4` | Sim | Onde o veículo da missão é criado. Se houver qualquer veículo num raio de 3.0, o trabalho é recusado |
| `BlipInfo.String` | string | Sim | Nome do blip do ponto de entrega |
| `BlipInfo.Sprite` | número | Sim | Sprite do blip (padrão 430) |
| `BlipInfo.Scale` | número | Sim | Escala do blip |
| `BlipInfo.Colour` | número | Sim | Cor do blip |
| `BlipInfo.Alpha` | número | Sim | Opacidade do blip |
| `BlipInfo.Radius_Alpha` | número | Sim | Opacidade do blip de raio (150 metros) |
| `BlipInfo.Radius_Colour` | número | Sim | Cor do blip de raio |
| `Fuel.enable` | bool | Sim | `true` chama `exports[Fuel.script]:SetFuel(veh, 100.0)`; `false` usa o statebag `Entity(veh).state.fuel` |
| `Fuel.script` | string | Sim | Nome do recurso de combustível (padrão `LegacyFuel`) |
| `UseTarget` | bool | Sim | `true` usa `qb-target` no NPC; `false` usa zona + tecla **E** |

### `sv_config.lua` (servidor)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `DeleteVehicleTimer` | número (minutos) | Sim | Tempo até o veículo destruído ser removido do mundo após a falha (padrão `2`) |
| `Vehicles` | tabela por tier (`S`, `A`, `B`, `C`, `D`) | Sim | Definição de cada tier — ver campos abaixo |
| `Vehicles.<tier>.threshold` | número | Sim | XP mínimo para o jogador alcançar o tier |
| `Vehicles.<tier>.timer` | número (segundos) | Sim | Tempo para entregar o veículo daquele tier |
| `Vehicles.<tier>.payout` | `{min, max}` | Sim | Faixa de pagamento sorteada |
| `Vehicles.<tier>.list` | lista de strings | Sim | Modelos de veículo possíveis no tier |
| `Vehicles.<tier>.locations` | lista de `vec3` | Sim | Pontos de entrega do tier. Um é sorteado por missão |
| `Vehicles.<tier>.xp` | `{min, max}` | Sim | Faixa de XP ganho ao concluir |
| `MissionRewards` | função `(Player, payout)` | Sim | Hook de recompensa. Por padrão chama `AddMoney(Player, 'cash', payout)`. Adicione aqui itens ou outras recompensas |

---

## Tiers e progressão

O tier do jogador é derivado do XP acumulado, comparando com o `threshold` de cada tier (padrão: `D` = 0, `C` = 1000, `B` = 5500, `A` = 7000, `S` = 10000).

O tier do **veículo sorteado** não é necessariamente o do jogador — há chance de cair um tier abaixo:

| Tier do jogador | Chance de cair um tier abaixo |
|---|---|
| `S` | 80% (vira `A`) |
| `A` | 75% (vira `B`) |
| `B` | 67% (vira `C`) |
| `C` | 50% (vira `D`) |
| `D` | sempre `D` |

Interagindo com o NPC, o contexto mostra XP, tier atual, missões concluídas e missões falhadas.

---

## Fluxo da missão

1. **Solicitar trabalho** no NPC. Se já houver missão ativa ou um veículo bloqueando o spawn, a solicitação é recusada.
2. O servidor sorteia tier, modelo, pagamento, XP e ponto de entrega, cria o veículo e warpa o jogador para dentro dele. O carro recebe placa `EXP#####`, upgrades de performance (motor 3, transmissão 2, suspensão 3, blindagem 4, freios 2, turbo), cores aleatórias e combustível cheio.
3. Um cronômetro aparece na tela. Nos últimos 60 segundos ele fica vermelho; nos últimos 10 segundos toca um som de contagem.
4. **A missão falha** se o tempo acabar ou se o jogador se afastar mais de **85 metros** do veículo. Nos dois casos o carro explode, o contador de falhas sobe e o veículo é deletado após `DeleteVehicleTimer` minutos.
5. No ponto de entrega (zona de 8x8x8), a tecla **E** inicia uma progressbar de 2 segundos. O servidor confere se o jogador está a menos de 10 metros do ponto e se o veículo é o da missão, paga, credita XP e deleta o carro.

---

## Banco de dados

Criada automaticamente no start do recurso:

```sql
CREATE TABLE IF NOT EXISTS `export_xp` (
  `cid` VARCHAR(255) NOT NULL,
  `xp` int(11) DEFAULT 0,
  `completed` int(11) DEFAULT 0,
  `failed` int(11) DEFAULT 0,
  PRIMARY KEY (`cid`)
);
```

Cada personagem ganha uma linha no primeiro login. O recurso também recarrega o cache de todos os jogadores online 2 segundos após um restart ao vivo.

---

## Integrações

### Script de combustível

Com `Fuel.enable = true`, o veículo da missão recebe tanque cheio via `exports[Fuel.script]:SetFuel`. Com `false`, usa o statebag `Entity(veh).state.fuel = 100` (compatível com `ox_fuel`).

### qb-target / ox_target

Com `UseTarget = true`, a interação com o NPC é registrada em `exports['qb-target']:AddTargetEntity`. O `ox_target` também aceita essa chamada por compatibilidade. Com `UseTarget = false`, o recurso não depende de nenhum target.

### Chaves do veículo (QB)

No bridge QB, `handleVehicleKeys` dispara `qb-vehiclekeys:server:AcquireVehicleKeys` com a placa. No bridge ESX a função é um stub — implemente ali se o seu servidor usa chaves.

---

## Localização

As strings são traduzidas via `ox_lib` locale. Os arquivos ficam em `locales/`:

- `en.json` — inglês
- `pt-br.json` — português do Brasil

O locale ativo é definido pela convar no `server.cfg`:

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
randol_exporting/
├── bridge/
│   ├── client/
│   │   ├── esx.lua        — notificação, login/logout e chaves (stub) no ESX
│   │   └── qb.lua         — notificação, login/logout e qb-vehiclekeys no QB
│   └── server/
│       ├── esx.lua        — GetPlayer, identificador e AddMoney no ESX
│       └── qb.lua         — GetPlayer, citizenid e AddMoney no QB
├── cl_exports.lua         — NPC, contexto de reputação, cronômetro, zona de entrega, scaleforms
├── sv_exports.lua         — sorteio de tier/veículo, spawn, XP, sucesso/falha, tabela export_xp
├── shared.lua             — config compartilhada (NPC, blip, combustível, target)
├── sv_config.lua          — config do servidor (tiers, veículos, pagamentos, recompensas)
├── locales/
│   ├── en.json
│   └── pt-br.json
└── fxmanifest.lua
```
