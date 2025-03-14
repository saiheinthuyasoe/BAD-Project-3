c:\Users\saihe\Zai_Codes\Next.js\MMORPG WEB Game\
├── public/
│   ├── assets/
│   │   ├── models/
│   │   ├── textures/
│   │   ├── sounds/
│   │   └── images/
│   └── favicon.ico
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   └── auth/
│   │   │       └── [...nextauth]/
│   │   │           └── route.ts
│   │   ├── game/
│   │   │   └── page.tsx
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── game/
│   │   │   ├── Character.tsx
│   │   │   ├── GameScene.tsx
│   │   │   ├── Terrain.tsx
│   │   │   ├── Controls.tsx
│   │   │   └── World.tsx
│   │   └── ui/
│   │       ├── HUD.tsx
│   │       ├── Inventory.tsx
│   │       ├── ChatBox.tsx
│   │       ├── MainMenu.tsx
│   │       └── CharacterCreation.tsx
│   ├── hooks/
│   │   ├── useGameState.ts
│   │   ├── useCharacterControls.ts
│   │   ├── useMultiplayer.ts
│   │   └── useInventory.ts
│   ├── lib/
│   │   ├── game/
│   │   │   ├── physics.ts
│   │   │   ├── animation.ts
│   │   │   └── collision.ts
│   │   ├── multiplayer/
│   │   │   ├── socket.ts
│   │   │   └── sync.ts
│   │   └── db/
│   │       └── index.ts
│   ├── models/
│   │   ├── Character.ts
│   │   ├── Item.ts
│   │   ├── World.ts
│   │   └── Quest.ts
│   ├── store/
│   │   ├── gameSlice.ts
│   │   ├── playerSlice.ts
│   │   └── index.ts
│   ├── types/
│   │   ├── game.ts
│   │   ├── character.ts
│   │   └── api.ts
│   └── utils/
│       ├── math.ts
│       ├── helpers.ts
│       └── constants.ts
├── .env
├── .env.local
├── .eslintrc.json
├── .gitignore
├── next-env.d.ts
├── next.config.js
├── package.json
├── postcss.config.js
├── README.md
├── tailwind.config.js
└── tsconfig.json
