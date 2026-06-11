# Cameron Fork Notes

## Repo Source

- Upstream repo: https://github.com/luyh7/milkonomy
- Local checkout: `D:\ClammerKonomy\milkonomy`
- License: MIT, copyright 2025 luyuhao
- Maintenance status: upstream README says the project is no longer maintained
- Current stack: Vue 3, Vite 6, TypeScript, Pinia, Vue Router, Element Plus, UnoCSS, Vitest
- Package manager: pnpm, with `pnpm-lock.yaml` lockfile version 9
- Tested package-manager command: `corepack pnpm@9.15.9`

### Remotes

- `origin`: https://github.com/thecameronboyer-beep/milkonomy.git
- `upstream`: https://github.com/luyh7/milkonomy.git

### Main Scripts

- `dev`: Vite private mode
- `dev:public`: Vite public mode
- `dev:private`: Vite private mode
- `build`: typecheck and build public mode
- `build:public`: typecheck and build public mode
- `build:private`: typecheck and build private mode
- `build:staging`: typecheck and build staging mode
- `preview`: Vite preview
- `test`: Vitest
- `lint`: ESLint with `--fix`
- `run-alchemy`: runs `scripts/alchemy-leaderboard/index.ts`

## Local Setup

- Install command: `corepack pnpm@9.15.9 install --frozen-lockfile`
- Dev command: `corepack pnpm@9.15.9 dev:public`
- Build command: `corepack pnpm@9.15.9 build`
- Preview command: `corepack pnpm@9.15.9 preview`

`corepack enable` failed locally with a Windows EPERM error while trying to create a global shim under `D:\Work App files\yarnpkg`. Using `corepack pnpm@9.15.9 ...` works and avoids changing package managers.

## Environment Notes

Environment files present:

- `.env`
- `.env.development`
- `.env.private`
- `.env.production`
- `.env.public`
- `.env.staging`

Variables present:

- `.env`: `VITE_APP_TITLE`, `VITE_ROUTER_HISTORY`
- `.env.development`: `VITE_PUBLIC_PATH`
- `.env.private`: `VITE_APP_TITLE`, `VITE_PUBLIC_PATH`, `VITE_BUILD_MODE`
- `.env.production`: `VITE_PUBLIC_PATH`
- `.env.public`: `VITE_APP_TITLE`, `VITE_PUBLIC_PATH`, `VITE_BUILD_MODE`
- `.env.staging`: `VITE_PUBLIC_PATH`

No sensitive-looking values were found in the env files during the setup pass. The GitHub workflows reference repository secrets such as `secrets.MILKONOMY` and `secrets.GITHUB_TOKEN`, but no secret values are stored in the repo.

## Codebase Map

- App entry: `src/main.ts`
- Root component: `src/App.vue`
- Router setup: `src/router/index.ts`
- Public routes: `src/router/routes/public.ts`
- Private routes: `src/router/routes/private.ts`
- Router config and guards: `src/router/config.ts`, `src/router/guard.ts`, `src/router/helper.ts`
- State setup: `src/pinia/index.ts`
- Main state stores: `src/pinia/stores/game.ts`, `src/pinia/stores/price.ts`, `src/pinia/stores/player.ts`, `src/pinia/stores/favorite.ts`, `src/pinia/stores/settings.ts`
- Main layout shell: `src/layouts/index.vue`
- Layout modes: `src/layouts/modes/LeftMode.vue`, `src/layouts/modes/LeftTopMode.vue`, `src/layouts/modes/TopMode.vue`
- Shared UI components: `src/common/components`
- Shared composables: `src/common/composables`
- Main page features: `src/pages/dashboard`, `src/pages/enhancer`, `src/pages/enhancest`, `src/pages/enhanposer`, `src/pages/manualchemy`, `src/pages/jungle`, `src/pages/junglest`, `src/pages/inherit`, `src/pages/decompose`, `src/pages/burial`, `src/pages/valhalla`
- Static game data: `public/data/data.json`
- Static market fallback data: `public/data/market.json`
- Game data fetch/update script: `scripts/fetch_game_data.py`
- Core calculation classes: `src/calculator/index.ts`, `src/calculator/gather.ts`, `src/calculator/manufacture.ts`, `src/calculator/alchemy.ts`, `src/calculator/enhance.ts`, `src/calculator/workflow.ts`, `src/calculator/utils.ts`
- Data/API adapter layer: `src/common/apis`
- i18n setup: `src/locales/index.ts`, `src/locales/lang/en.ts`, `src/locales/lang/zh-cn.ts`, `src/locales/lang/zh-tw.ts`
- Build config: `vite.config.ts`
- Deployment config: `.github/workflows/deploy.yml`, `.github/workflows/update-data.yml`, `.github/workflows/release.yml`
- Test setup: `vite.config.ts` Vitest config and `tests/**/*.test.ts`

## Milkonomy Areas To Customize Later

### Mobile UI

- Likely entry points: `src/layouts`, `src/common/assets/styles`, and page components under `src/pages`.
- Most relevant screens for a phone-first pass: `src/pages/dashboard/index.vue`, `src/pages/enhancer/index.vue`, `src/pages/enhanposer/index.vue`, `src/pages/jungle/index.vue`, and shared dashboard components under `src/pages/dashboard/components`.

### Strategy Model

- Current multi-step strategy logic lives mostly in `src/calculator/workflow.ts`.
- Existing generated strategy lists are assembled in `src/common/apis/leaderboard/index.ts`, `src/common/apis/manualchemy/index.ts`, `src/common/apis/jungle/index.ts`, and `src/common/apis/enhanposer/index.ts`.
- Existing saved single-calculator favorites use `src/pinia/stores/favorite.ts` and `src/common/apis/favorite/index.ts`.
- Cameron's later acquisition choices, connected strategies, and no-inventory option should be designed around this layer before UI work starts.

### Market Data

- Live market fetch is in `src/pinia/stores/game.ts`, currently using `https://www.milkywayidle.com/game_data/marketplace.json`.
- Cached game and market data are stored in localStorage by `src/pinia/stores/game.ts`.
- Manual price overrides are stored by `src/pinia/stores/price.ts` and exposed through `src/common/apis/price/index.ts`.
- Price display and edit UI is shared through `src/pages/dashboard/components/ManualPriceCard.vue`, `ActionPrice.vue`, and `PriceStatusSelect.vue`.

### Recipe And Static Data

- Main game data is in `public/data/data.json`.
- Item/action lookup helpers are in `src/common/apis/game/index.ts`.
- Static type declarations are under `types/game.d.ts` and `types/market.d.ts`.
- Game data refresh is handled by `scripts/fetch_game_data.py` and `.github/workflows/update-data.yml`.

### Profit Calculation

- Base profit formula is in `src/calculator/index.ts`.
- Gathering calculations: `src/calculator/gather.ts`
- Manufacturing calculations: `src/calculator/manufacture.ts`
- Alchemy/transmutation/decomposition/coinify calculations: `src/calculator/alchemy.ts`
- Enhancement calculations: `src/calculator/enhance.ts`
- Multi-step workflow calculations: `src/calculator/workflow.ts`
- Leaderboard assembly and filtering: `src/common/apis/leaderboard/index.ts` and `src/common/apis/utils.ts`

### Skill Pages

- General profit dashboard: `src/pages/dashboard/index.vue`
- Enhancement calculator page: `src/pages/enhancer/index.vue`
- Super enhancement page: `src/pages/enhancest/index.vue`
- Enhancement decomposition page: `src/pages/enhanposer/index.vue`
- Manualchemy page: `src/pages/manualchemy/index.vue`
- Jungle/enhance workflow pages: `src/pages/jungle`, `src/pages/junglest`, `src/pages/inherit`, `src/pages/decompose`

### Language And i18n

- i18n bootstrap: `src/locales/index.ts`
- English strings: `src/locales/lang/en.ts`
- Simplified Chinese strings: `src/locales/lang/zh-cn.ts`
- Traditional Chinese strings: `src/locales/lang/zh-tw.ts`
- Conversion helpers: `src/locales/convert` and `scripts/trans-convert`

### Public And Private Modes

- Modes are set through `.env.public`, `.env.private`, `.env.staging`, `.env.development`, and `.env.production`.
- `VITE_BUILD_MODE` is loaded in `vite.config.ts`.
- The route split lives in `src/router/routes/public.ts` and `src/router/routes/private.ts`.
- `vite.config.ts` contains a commented-out plugin intended to strip private routes/pages from public builds. It is currently commented out, so public builds still compile the private route modules.

## Setup Results

- Fork/clone result: forked and cloned with GitHub CLI into `D:\ClammerKonomy\milkonomy`
- Branch: `cameron/fork-prep`
- Install result: passed with `corepack pnpm@9.15.9 install --frozen-lockfile`
- Dev server result: passed with `corepack pnpm@9.15.9 dev:public`; HTTP probe returned 200 from `http://127.0.0.1:3333/`; server was stopped after verification
- Build result: passed with `corepack pnpm@9.15.9 build`
- Test result: passed with `corepack pnpm@9.15.9 test -- --run`; 3 test files, 14 tests
- Lint result: not run, because the repo script is `eslint . --fix` and would mutate files during a setup audit
- Commit hook result: normal `git commit` failed because `.husky/pre-commit` calls `tsx scripts/update-version.ts`, but `tsx` is not installed in this repo

## Blockers

- No runtime setup blocker remains.
- Local note: `pnpm` is not directly available on PATH in this environment, and `corepack enable` is blocked by a Windows EPERM issue. Use `corepack pnpm@9.15.9 ...` unless pnpm is installed or the Corepack shim issue is fixed.
- Commit hook blocker: `.husky/pre-commit` calls `tsx scripts/update-version.ts`, but `tsx` is not listed in `package.json`. Until fixed, normal commits will fail after build/typecheck and lint-staged pass.

## Recommended Thread 002

002 Milkonomy Codebase Map + Change Plan

Recommended scope:

- Build a deeper map of the current calculator and API flow from item data to table rows.
- Define the future strategy data model before changing UI.
- Decide where output-anchored strategies, connected strategy chains, acquisition choices, and the bottom totals bar should live.
- Keep this as a planning/design thread before implementing product behavior.
