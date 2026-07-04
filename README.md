# Fix Report - v14.593.1-Null's Royale

The project was thoroughly inspected by running a strict TypeScript check (`tsc --noEmit`) on all `src/` files. All errors that would have prevented the server from running with `npm start` were detected and fixed (because `ts-node` in this project is configured with `"transpileOnly": false`, meaning it refuses to run if any Type error is found in any file).

## Fixed Errors

### 1) Global Name Collision Between Files (The Most Important Root Cause)
Most package files (Messages/Commands) are written in the old CommonJS style (`require` / `module.exports`) without any ES Module type `import`/`export`. This causes TypeScript to treat each of these files as a **global Script** instead of an **independent Module**, so all files share the same global Scope.

**Result:** The file `Commands/Client/BuyShopChest.ts` defines a variable named `ChestContentMessage`, and at the same time, the file `Messages/Server/ChestContentMessage.ts` defines a class with the same name `ChestContentMessage` — a direct conflict that completely stops compilation:
```
error TS2451: Cannot redeclare block-scoped variable 'ChestContentMessage'.
```

**Solution:** `export {};` was added at the beginning of each of these files (70 files) to force them to be independent Modules with their own scope, without changing any programming logic.

### 2) Incorrect Placement of `// @ts-nocheck` Directive
Some files contain `// @ts-nocheck` to disable strict checking in them, but this directive only works if it is the **first effective line** in the file. After adding `export {};` at the top (Fix #1), `@ts-nocheck` became on the second line and was automatically disabled, revealing dozens of "implicit any" errors in files that were previously working normally.

**Solution:** The affected files (65 files) were reordered so that `// @ts-nocheck` always remains on the first line.

### 3) Incorrect Path for `fingerprint.json` in `LoginMessage.ts`
The code was reading the file via a relative path to the current working directory (`process.cwd()`):
```ts
path.resolve("Gamefiles/fingerprint.json")
```
This only works if the server is run directly from within the project folder, and any execution from a different path would cause an unhandled exception (`ENOENT`) during login, immediately disconnecting the client — a silent "login failure."

**Solution:** The path is now always built from `ctx.rootPath` (the actual project root), with try/catch handling that prints a clear error message instead of a silent crash.

### 4) Logical Error in Discord Bot (Optional, Disabled by Default)
The code was using the `discordConfig.tokenEnv` value (which is the *name* of an environment variable) directly as if it were the token itself, instead of reading `process.env[tokenEnv]`. This has been corrected. It does not affect server operation because the Discord feature is disabled by default in `config.json` (`"discord": { "enabled": false }`).

### 5) Missing TypeScript Type Package
The file `TcpGameServer.ts` imports `figlet` using `import figlet from "figlet"`, and this package does not come with its own types. `@types/figlet` was not present in `devDependencies`. It has been added so that `npm install` + strict checking succeed.

## Result
`tsc --noEmit` was run on the entire project after all these fixes:
```
0 errors
```
The project now compiles successfully 100%, and therefore `npm start` (which uses `ts-node` with strict checking) will run without any interruptions due to Type errors.

## How to Run
```bash
npm install
npm start
```
## Developer
Anonymous

The server will run on port `9330` (modifiable from `config.json`), and the Content Patch Server on port `9331`.

> Note: According to the original `README.md`, this project is an "old archive not ready for production." All compilation and logical errors found have been fixed, but this does not mean it is 100% identical to the official server's behavior — some features (like alliances) are still partial as explained in the README itself.
