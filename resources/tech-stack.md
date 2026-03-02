# Tech Stack

| Category | Technology | Version | Rationale |
|----------|-----------|---------|-----------|
| Engine | Unity | 2022 LTS | Industry standard for 2D, team familiarity |
| Renderer | 2D URP | Latest | Better 2D lighting, performance |
| Language | C# | 10+ | Unity native |
| Input | Unity Input System | 1.x | Multi-device, rebindable, buffer support |
| Physics | Rigidbody2D | Built-in | Knockback, launch, wall bounce |
| Animation | Animator + Overrides | Built-in | Shared state machine, per-character overrides |
| Tweening | DOTween | 2.x | Hitstop, screen shake, easing |
| Data | ScriptableObjects | Built-in | Inspector-friendly, hot-reload, no singletons |
| Serialization | JsonUtility / Newtonsoft | Built-in / 13.x | Save/load to persistentDataPath |
| Co-op (TBD) | Fishnet or Mirror | Latest | Rollback netcode for fighting precision |
| AI Coordination | AgentPilot | Local | Context-optimized task execution |
| Version Control | Git | 2.x | Standard, branch-per-pillar strategy |
