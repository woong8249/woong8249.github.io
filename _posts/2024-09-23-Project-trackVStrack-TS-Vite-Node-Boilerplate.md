---
title: "Project | trackVStrack | vite-node-ts boilerplate"
categories: [Projects, tackVStrack]
tags: [ts-vite-node,boilerplate,eslint,vite,typescript]
image: ts-logo-256.png
---

## Summary
프레임 워크없이 node-ts 위한 개발환경 설정(boiler plate)

- vite
  - vite-node
  - vitest
  - vite-plugin-eslint 

- eslint (linting & formatting)
  - flat config
  - typescript-eslint
  - eslint-config-airbnb-base
  - vscode extension eslint

- winston
  - winston-console-format

### Why vite-node instead of ts-node?

물론 `Vite`는 프론트엔드 전용개발툴이다.(속도가 빠른게 최장점인)

하지만 `TypeScript` 컴파일을 지원하는 덕분에 `vite-node` 패키지를 사용하여 백엔드에서도 활용할 수 있다.

또한, Vite의 생태계는 이미 매우 성숙하여 다양한 플러그인들이 풍부하게 지원되며
결정적으로 풀스텍으로 확장해가며 프론트엔드, 백앤드, 테스트환경까지 Vite 기반으로 통일함으로써 코드베이스를 더욱 일관성 있게 관리할수 있음에 가장 큰 장점으로 느꼈다..



## Tree

```bash
.
├── eslint.config.js
├── eslint4vite.js
├── importless-airbnb-base.cjs
├── index.ts
├── node_modules
├── package.json
├── pnpm-lock.yaml
├── src
│   ├── config
│   │   └── config.ts
│   └── logger
│       └── winston.ts
├── tsconfig.json
└── vite.config.ts
```

## `Package.json`

```json
{
  "name": "ts-crawling",
  "version": "1.0.0",
  "description": "migrate crawling js => ts",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "lint": "eslint .",
    "start": "run(){ NODE_OPTIONS=--experimental-import-meta-resolve vite-node ${1:-index.ts}; }; tsc --noEmit && run",
    "dev": "run(){ NODE_OPTIONS=--experimental-import-meta-resolve vite-node --watch ${1:-index.ts}; }; tsc --noEmit && run"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "winston": "^3.14.2",
    "winston-console-format": "^1.0.8"
  },
  "devDependencies": {
    "@tsconfig/esm": "^1.0.5",
    "@tsconfig/node22": "^22.0.0",
    "@tsconfig/strictest": "^2.0.5",
    "@types/node": "^22.5.5",
    "@typescript-eslint/eslint-plugin": "^8.6.0",
    "eslint": "^9.10.0",
    "eslint-config-airbnb-base": "^15.0.0",
    "typescript": "^5.6.2",
    "typescript-eslint": "^8.6.0",
    "vite": "^5.4.6",
    "vite-node": "^2.1.1",
    "vite-plugin-eslint": "^1.8.1",
    "vitest": "^2.1.1"
  }
}
```

## `eslint`

린팅은 eslint로 포맷팅은 preetier로 하기 싫었다.

vscode eslint extension으로 포맷팅또한 eslint를 이용하도록 했음

- flat config 이용

```jsx
//eslint.config.js
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
import { FlatCompat } from '@eslint/eslintrc';
import tsPlugin from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';

const filename = fileURLToPath(import.meta.url);
const dirName = dirname(filename);
const compat = new FlatCompat();
const defaultConfig = [
  {
    files: ['**/*.ts'],
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        project: true,
      },
    },
    plugins: {
      '@typescript-eslint': tsPlugin,
      ts: tsPlugin,
    },
    rules: {
      ...tsPlugin.configs.base.rules,
      ...tsPlugin.configs['eslint-recommended'].rules,
      ...tsPlugin.configs['strict-type-checked'].rules,
    },
    settings: {
      'import/resolver': {
        typescript: true,
      },
    },
  },
];

const customConfig = [
  ...compat.extends(join(dirName, 'importless-airbnb-base.cjs')),
  {
    languageOptions: {
      parserOptions: { // for import.meta.url
      // Eslint doesn't supply ecmaVersion in `parser.js` `context.parserOptions`
      // This is required to avoid ecmaVersion < 2015 error or 'import' / 'export' error
        ecmaVersion: 'latest',
        sourceType: 'module',
      },
    },

    rules: {
      'no-shadow': 'off',
      'no-underscore-dangle': ['error', { allow: ['__filename', '__dirname'] }],
    },
  },
];

const config = [...defaultConfig, ...customConfig];

export default config;

```

- airbnb rule 적용
    - ...compat.extends(join(dirName, 'importless-airbnb-base.cjs')),
    
    모듈 시스템 문제와 import관련룰이 말썽이여 뺌
    

```jsx
//importless-airbnb-base.cjs
const airbnbBase = require('eslint-config-airbnb-base');

module.exports = {
  extends: airbnbBase.extends.filter((x) => !x.endsWith('imports.js')),
};

```

## tsconfig.json

- `"resolvePackageJsonExports": false,`
TypeScript는 package.json의 exports 필드를 무시하고, 패키지 내부 파일에 대한 직접적인 접근을 허용합니다.

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "types": [
      "node",
    ],
    "baseUrl": "./",
    "resolvePackageJsonExports": false,
  },
  "extends": [
    "@tsconfig/strictest/tsconfig",
    "@tsconfig/node22/tsconfig",
    "@tsconfig/esm/tsconfig"
  ],
  "include": [
    "src/**/*.ts",
    "*.ts",
  ]
}
```

## Vite

- `vite-plugin-eslint` 추가
- eslint4vitePath ⇒ eslint flat config를 사용하기위해 작성
    - 플러그인자체가 eslint flat config를 지원하지않음

```jsx
//eslint4vite.js
import { resolve } from 'path';

import eslint from 'eslint/use-at-your-own-risk';

const overrideConfigFile = resolve('eslint.config.js');

export class ESLint extends eslint.FlatESLint {
  constructor(params) {
    Object.assign(params, { overrideConfigFile });
    super(params);
  }
}

```

```jsx
//vite.config.ts

import { defineConfig, loadEnv, type Plugin } from 'vite';
import eslint from 'vite-plugin-eslint';
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const eslint4vitePath = join(__dirname, 'eslint4vite.js');
const plugins = [
  { // default settings on build (i.e. fail on error)
    ...eslint({ eslintPath: eslint4vitePath }),
    apply: 'build',
  } as Plugin,
  { // do not fail on serve (i.e. local development)
    ...eslint({
      eslintPath: eslint4vitePath,
      failOnWarning: false,
      failOnError: false,
      emitError: true,
      emitWarning: true,
    }),
    apply: 'serve',
    enforce: 'post',
  } as Plugin,
];

// https://vitejs.dev/config/
export default defineConfig(({ mode }) => {
  process.env = { ...process.env, ...loadEnv(mode, process.cwd()) };
  return {
    plugins,
  };
});

```

## winston

- `winston-console-format` 라이브러리 적용

```jsx
import { createLogger, format, transports } from 'winston';
import { consoleFormat } from 'winston-console-format';

import config from '../config/config';

const { logLevel } = config.app;

const winLogger = createLogger({
  level: logLevel,
  format: format.combine(
    format.errors({ stack: true }),
    format.splat(),
  ),
  transports: [
    new transports.Console({
      handleExceptions: true,
      format: format.combine(
        format.colorize({ all: true }),
        format.padLevels(),
        consoleFormat({
          showMeta: true,
          metaStrip: ['timestamp', 'service'],
          inspectOptions: {
            depth: Infinity,
            colors: true,
            maxArrayLength: Infinity,
            breakLength: 120,
            compact: Infinity,
          },
        }),
      ),
    }),
  ],
});

export default winLogger;

```