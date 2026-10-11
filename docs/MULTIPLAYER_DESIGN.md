# OWCraft Multiplayer Design & Architecture

This document describes the multiplayer architecture for **OWCraft** (Minecraft running inside Outer Wilds, based on SkyCraft's passthrough framework). It covers the network topology, deterministic planetary coordinate mapping, custom C2S packet protocols, server-side movement handling, entity/avatar replication, and setup instructions for playing with friends.

---

## 1. System Overview

OWCraft splits responsibilities between two concurrently running game engines:

- **Outer Wilds (Host):** An OWML C# mod that manages the solar system physics, rotating/orbiting celestial bodies, collider harvesting, voxelization, lighting, and rendering of block meshes and UI overlays.
- **Minecraft (Guest):** A Minecraft (Fabric) instance that simulates block placement/breaking, player movement, physics, inventory, crafting, mobs, and chunk meshing.

```
       [Player 1: Host PC]                                [Player 2: Friend PC]
 ┌──────────────────────────────┐                   ┌──────────────────────────────┐
 │      Outer Wilds (OWML)      │                   │      Outer Wilds (OWML)      │
 │  - Planet surface collision  │                   │  - Planet surface collision  │
 │  - Sun lighting & shadows    │                   │  - Sun lighting & shadows    │
 │  - Unity block meshes        │                   │  - Unity block meshes        │
 └──────────────┬───────────────┘                   └──────────────┬───────────────┘
                │ IPC (Shared Memory)                              │ IPC (Shared Memory)
                │ Local\OWCraft_v1                                 │ Local\OWCraft_v1
 ┌──────────────┴───────────────┐                   ┌──────────────┴───────────────┐
 │   Minecraft Client (Player 1)│                   │   Minecraft Client (Player 2)│
 │  - Local physics & input     │                   │  - Local physics & input     │
 │  - Chunk meshing & export    │                   │  - Chunk meshing & export    │
 └──────────────┬───────────────┘                   └──────────────┬───────────────┘
                │                                                  │
                │  Minecraft TCP Protocol (Fabric Networking)      │
                │  Vanilla Packets + SkyNet Payloads               │
                └─────────────────────────┬────────────────────────┘
                                          │
                               ┌──────────┴──────────┐
                               │  Minecraft Server   │
                               │ (Integrated / LAN / │
                               │  Dedicated Server)  │
                               └─────────────────────┘
```

In singleplayer, Minecraft runs an internal `IntegratedServer` connected via local memory. In **multiplayer**, multiple Minecraft clients connect to a shared Minecraft server (hosted by one player via LAN/e4mc or on a dedicated server).

---

## 2. Deterministic Planetary Mapping (`AnchorGrid`)

Outer Wilds planets are rotating spheres with radial gravity ("down" points toward the center), whereas Minecraft assumes a flat world with a fixed downward Y axis.

OWCraft solves this using `AnchorGrid`, which projects the spherical surface of each celestial body onto an **equi-angular cube-sphere** divided into ~40 m tangent tiles. Each tile is assigned a dedicated, non-overlapping 512×512 block patch in the Minecraft world:

$$\text{McX}_0 = \text{Base} + (\text{face} \times \text{MaxTiles} + i) \times \text{Spacing}$$
$$\text{McZ}_0 = \text{Base} + (\text{planet.Slot} \times \text{MaxTiles} + j) \times \text{Spacing}$$
$$\text{Y}_0 = 192$$

Where:
- $\text{Base} = 16384$ blocks.
- $\text{Spacing} = 512$ blocks (ensuring tiles never enter each other's standard chunk render distance).
- $\text{planet.Slot}$ is a deterministic hash of the planet name or its `AstroObject.Name` enum.

### Why this enables zero-negotiation multiplayer:
Because `AnchorGrid` is purely mathematical and relies only on the planet's geometry and name:
1. When **Player 1** lands on Timber Hearth near the village campfire, their local Outer Wilds maps their position to tile `(TimberHearth, Face 0, Tile 2, 2)` $\rightarrow (\text{McX}_1, 192, \text{McZ}_1)$.
2. When **Player 2** lands at the same campfire, their local Outer Wilds calculates the **identical** coordinates $(\text{McX}_1, 192, \text{McZ}_1)$.
3. If Player 1 builds a cobblestone wall, Player 1's Minecraft sends block updates to the server.
4. Player 2's Minecraft receives the block updates, meshes the chunk section, and passes it to Player 2's Outer Wilds over IPC.
5. Player 2's `BlockRenderer` computes `AnchorForMc(McX, McZ)`, locates the corresponding anchor GameObject on Timber Hearth, and attaches the mesh.
6. Both players see the wall at the exact same physical location on the planet.

---

## 3. Network Protocol Extensions (`SkyNet`)

In vanilla Minecraft, a client cannot unilaterally reposition itself across large distances (e.g. 512-block tile reanchors or initial planet spawn). Doing so causes `ServerGamePacketListenerImpl` to trigger `"Player moved too quickly!"` or `"Player moved wrongly!"` and rubberband the client back to its old position.

Furthermore, `skycraft-owcraft` introduced airborne tile shifts (`shiftPlayer`) and entity transfers (`TileCarry`), which initially only operated on `minecraft.getSingleplayerServer()`.

To enable multiplayer, OWCraft extends the `SkyNet` payload registry with two custom C2S packets:

### A. `SkyNet.Teleport`
Used when entering Minecraft mode (F6) or landing on a planet:
- **Payload:** `(double x, double y, double z, float yaw, float pitch)`
- **Client Trigger:** When `SkyClient.requestTeleport` runs and `minecraft.getSingleplayerServer() == null`.
- **Server Action:**
  ```java
  ServerPlayNetworking.registerGlobalReceiver(Teleport.TYPE, (payload, context) -> {
      ServerPlayer player = context.player();
      context.server().execute(() -> {
          player.teleportTo(payload.x(), payload.y(), payload.z());
          player.setYRot(payload.yaw());
          player.setXRot(payload.pitch());
          player.resetFallDistance();
      });
  });
  ```
  Calling `player.teleportTo(...)` on the server authoritatively updates the server's tracking positions (`firstGoodX/Y/Z`), resets fall distance, and issues a standard teleport sequence acknowledgment to the client.

### B. `SkyNet.Shift`
Used during smooth airborne tile transitions (e.g. elytra gliding or falling across an anchor boundary):
- **Payload:** `(double dx, double dy, double dz, List<Float> rot)`
- **Client Trigger:** When crossing a tile boundary mid-air (`shiftPlayer`) on a multiplayer connection.
- **Server Action:**
  ```java
  ServerPlayNetworking.registerGlobalReceiver(Shift.TYPE, (payload, context) -> {
      ServerPlayer player = context.player();
      context.server().execute(() -> {
          Vec3 from = player.position();
          player.teleportTo(player.level(), payload.dx(), payload.dy(), payload.dz(), Relative.ALL, 0f, 0f, false);
          Vec3 to = player.position();
          float[] rot = new float[payload.rot().size()];
          for (int i = 0; i < rot.length; i++) rot[i] = payload.rot().get(i);
          TileCarry.carry(player.level(), from, to, rot);
      });
  });
  ```
  This performs a relative offset teleport on the server without zeroing velocity, and shifts any nearby mobs or dropped items to the new tile coordinates.

---

## 4. Server Authority & Anti-Cheat Handling

### Planet Surface Flight Kick
Because the Minecraft server (whether dedicated or integrated host) does not have access to the remote guest's local Outer Wilds shared memory, the guest appears to be walking on empty void air.

In vanilla Minecraft, floating or walking in air without block collision triggers a flight kick after 80 ticks:
$$\text{"Flying is not enabled on this server"}$$

**Resolution:**
The mod injects `MinecraftServerFlightMixin`:
```java
@Mixin(MinecraftServer.class)
public abstract class MinecraftServerFlightMixin {
    @Inject(method = "allowFlight", at = @At("HEAD"), cancellable = true)
    private void skycraft$allowFlight(CallbackInfoReturnable<Boolean> cir) {
        cir.setReturnValue(true);
    }
}
```
This globally enables flight permissions on servers running the mod, allowing clients to traverse arbitrary planet geometry without being kicked.

---

## 5. Avatar & Entity Replication

When multiple players are near each other on a planet:

1. **Remote Player Entities (`RemotePlayer`):**
   In Minecraft, connected friends appear as `RemotePlayer` entities.
2. **Posed Mesh Extraction (`AvatarExporter`):**
   `AvatarExporter` extracts all rendered entities in range (excluding the local player in 1st person), captures their active skins, textures, armor, and held items, and bundles them into `Proto.REN_SCENE`.
3. **Planet Projection (`SceneRenderer`):**
   `SceneRenderer` on the host side parses the `RenScene` packet, maps the origin to the planet's anchor frame using `AnchorForMc`, and renders the friend's animated Minecraft avatar standing on the planet surface.
4. **Third-Person View (F5):**
   When a player presses F5, their own avatar is sent via `Proto.REN_AVATAR` and rendered at their feet.

---

## 6. How to Play with Friends

There are three ways to play multiplayer OWCraft:

### Option A: Peer-to-Peer LAN / Internet (Recommended)
1. **Both players** install:
   - Outer Wilds + OWML + `Yaekai.OWCraft` mod.
   - Minecraft Java Edition 26.3 + Fabric Loader + Fabric API + `skycraft-owcraft.jar`.
2. **Host Player:**
   - Starts Outer Wilds and Minecraft.
   - In Minecraft, press **O** (or Escape) to open the menu.
   - Click **Open to LAN**, then click **Start LAN World**.
   - With the bundled `e4mc` mod, a link like `abc-def.e4mc.link` appears in Minecraft chat. Click it or copy it, and send it to your friend.
3. **Friend:**
   - Starts Outer Wilds and Minecraft.
   - In Minecraft, press **T** and type `/join abc-def.e4mc.link`.
   - Your Minecraft connects to your friend's world!
4. **Play together:**
   - Both players fly to the same planet (e.g. Timber Hearth).
   - Press **F6** to enter Minecraft mode.
   - You will see each other's Minecraft characters, place blocks together, and explore!
   - Type `/leave` anytime to return to your singleplayer void world.

### Option B: Dedicated Server
1. Set up a standard Fabric 26.3 dedicated server.
2. Drop `skycraft-owcraft.jar` and `fabric-api.jar` into the server's `mods/` directory.
3. In `server.properties`, set:
   ```properties
   level-type=skycraft:mirror
   allow-flight=true
   ```
4. In each player's `.minecraft/config/skycraft.properties`, specify:
   ```properties
   join=your.server.ip.address:25565
   ```
   Both clients will automatically connect to the dedicated server upon launch.

### Option C: Combined with Quantum Space Buddies (QSB)
To see each other's Outer Wilds spaceships and astronaut suits in addition to Minecraft blocks:
1. Install **Quantum Space Buddies (QSB)** from the Outer Wilds Mod Manager.
2. Join your friend's QSB session in Outer Wilds.
3. Join your friend's Minecraft session via Option A or Option B.
4. Result:
   - When in your spaceship or spacesuit: you see each other through QSB.
   - When in Minecraft mode (F6): you see each other's Minecraft player avatars and all placed voxel blocks!
