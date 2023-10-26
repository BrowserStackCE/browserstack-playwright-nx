<div style="display: flex; gap: 1rem; align-items: center;">
<h1>BrowserStack SDK + Playwright + NX</h1>
<img src="https://www.browserstack.com/blog/content/images/2019/06/Browserstack-logo-white-02-4.png" width="50" height="50" style="object-fit: contain;"  />
<img src="https://seeklogo.com/images/P/playwright-logo-22FA8B9E63-seeklogo.com.png" width="50" height="50" style="object-fit: contain;" />
<img src="https://seeklogo.com/images/N/nx-logo-8EB5A23B44-seeklogo.com.png" width="50" height="50" style="object-fit: contain;" />
</div>

This repository contains an example of how to integrate BrowserStack SDK with playwright project inside a monorepo created using [nx.dev](https://nx.dev)

# Getting Started

- Inside of your project folder create a new folder named executors
- Create a new folder named playwright
- Add two files [playwright.ts](/packages/playwright/src/executors/playwright/playwright.ts) & [schema.json](/packages/playwright/src/executors/playwright/schema.json)

playwright.ts
```ts 
import { execSync, fork,exec } from 'child_process';
import {
  ExecutorContext,
  getPackageManagerCommand,
  names,
  output,
  workspaceRoot,
} from '@nx/devkit';

export interface PlaywrightExecutorSchema {
  /*
   * if 'projects' is configured then that name needs to be provided instead of
   * all, chromium, firefox, webkit
   **/
  browser?: 'all' | 'chromium' | 'firefox' | 'webkit' | string;
  config?: string;
  debug?: boolean;
  forbidOnly?: boolean;
  fullyParallel?: boolean;
  grep?: string;
  globalTimeout?: number;
  grepInvert?: string;
  headed?: boolean;
  ignoreSnapshots?: boolean;
  workers?: string;
  list?: boolean;
  maxFailures?: number | boolean;
  noDeps?: boolean;
  output?: string;
  passWithNoTests?: boolean;
  project?: string[];
  quiet?: boolean;
  repeatEach?: number;
  reporter?:
    | 'list'
    | 'line'
    | 'dot'
    | 'json'
    | 'junit'
    | 'null'
    | 'github'
    | 'html'
    | 'blob';
  retries?: number;
  shard?: string;
  timeout?: number;
  trace?:
    | 'on'
    | 'off'
    | 'on-first-retry'
    | 'on-all-retries'
    | 'retain-on-failure';
  updateSnapshots?: boolean;
  ui?: boolean;
  uiHost?: string;
  uiPort?: string;
  skipInstall?: boolean;
  browserstackConfig?:string
}

export async function playwrightExecutor(
  options: PlaywrightExecutorSchema,
  context: ExecutorContext
) {
  const projectRoot =
    context.projectGraph?.nodes?.[context?.projectName]?.data?.root;

  if (!projectRoot) {
    throw new Error(
      `Unable to find the Project Root for ${context.projectName}. Is it set in the project.json?`
    );
  }

  if (!options.skipInstall) {
    output.log({
      title: 'Ensuring Playwright is installed.',
      bodyLines: ['use --skipInstall to skip installation.'],
    });
    const pmc = getPackageManagerCommand();
    execSync(`${pmc.exec} playwright install`, {
      cwd: workspaceRoot,
      stdio: 'inherit',
    });
  }

  const args = createArgs(options);
  const p = runPlaywright(args, context.root,options.browserstackConfig);
  p.stdout.on('data', (message) => {
    process.stdout.write(message);
  });
  p.stderr.on('data', (message) => {
    process.stderr.write(message);
  });

  return new Promise<{ success: boolean }>((resolve) => {
    p.on('close', (code) => {
      resolve({ success: code === 0 });
    });
  });
}

function createArgs(
  opts: PlaywrightExecutorSchema,
  exclude: string[] = ['skipInstall','browserstackConfig']
): string[] {
  const args: string[] = [];

  for (const key in opts) {
    if (exclude.includes(key)) continue;

    const value = opts[key];
    // NOTE: playwright doesn't accept pascalCase args, only kebab-case
    const arg = names(key).fileName;

    if (Array.isArray(value)) {
      args.push(`--${arg}=${value.map((v) => v.trim()).join(',')}`);
    } else if (typeof value === 'boolean') {
      // NOTE: playwright don't accept --arg=false, instead just don't pass the arg.
      if (value) {
        args.push(`--${arg}`);
      }
    } else {
      args.push(`--${arg}=${value}`);
    }
  }

  return args;
}

function runPlaywright(args: string[], cwd: string,browserStackConfig?:string) {
  const env:any = {}
  if(browserStackConfig){
    env['BROWSERSTACK_CONFIG_FILE'] = browserStackConfig
  }
  try {
    return exec(['browserstack-node-sdk playwright test',...args].join(' '),{
        cwd,
        env:{
          ...process.env,
          ...env
        }
    })
  } catch (e) {
    console.error(e);
    throw new Error('Unable to run playwright. Is @playwright/test installed?');
  }
}

export default playwrightExecutor;
```

schema.json

```json
{
    "$schema": "http://json-schema.org/schema",
    "version": 2,
    "title": "Playwright executor",
    "description": "Run Playwright tests.",
    "type": "object",
    "properties": {
      "browser": {
        "type": "string",
        "description": "Browser to use for tests, one of 'all', 'chromium', 'firefox' or 'webkit'. If a playwright config is provided/discovered then the browserName value is expected from the configured 'projects'",
        "x-priority": "important"
      },
      "config": {
        "type": "string",
        "description": "Configuration file, or a test directory with optional",
        "x-completion-type": "file",
        "x-completion-glob": "playwright?(*)@(.js|.cjs|.mjs|.ts|.cts|.mtx)",
        "x-priority": "important"
      },
      "browserstackConfig":{
        "type": "string",
        "description": "Browserstack SDK Config file"
      },
      "debug": {
        "type": "boolean",
        "description": "Run tests with Playwright Inspector. Shortcut for 'PWDEBUG=1' environment variable and '--timeout=0',--max-failures=1 --headed --workers=1' options"
      },
      "forbidOnly": {
        "type": "boolean",
        "description": "Fail if test.only is called"
      },
      "fullyParallel": {
        "type": "boolean",
        "description": "Run all tests in parallel"
      },
      "grep": {
        "alias": "g",
        "type": "string",
        "description": "Only run tests matching this regular expression"
      },
      "globalTimeout": {
        "type": "number",
        "description": "Maximum time this test suite can run in milliseconds"
      },
      "grepInvert": {
        "alias": "gv",
        "type": "string",
        "description": "Only run tests that do not match this regular expression"
      },
      "headed": {
        "type": "boolean",
        "description": "Run tests in headed browsers",
        "x-priority": "important"
      },
      "ignoreSnapshots": {
        "type": "boolean",
        "description": "Ignore screenshot and snapshot expectations"
      },
      "workers": {
        "alias": "j",
        "type": "string",
        "description": "Number of concurrent workers or percentage of logical CPU cores, use 1 to run in a single worker"
      },
      "list": {
        "type": "boolean",
        "description": "Collect all the tests and report them, but do not run"
      },
      "maxFailures": {
        "alias": "x",
        "oneOf": [{ "type": "number" }, { "type": "boolean" }],
        "description": "Stop after the first N failures"
      },
      "noDeps": {
        "type": "boolean",
        "description": "Do not run project dependencies"
      },
      "output": {
        "type": "string",
        "description": "Folder for output artifacts"
      },
      "passWithNoTests": {
        "type": "boolean",
        "description": "Makes test run succeed even if no tests were found",
        "default": true
      },
      "project": {
        "description": "Only run tests from the specified list of projects",
        "type": "array",
        "items": {
          "type": "string"
        }
      },
      "quiet": {
        "alias": "q",
        "type": "boolean",
        "description": "Suppress stdio"
      },
      "repeatEach": {
        "type": "number",
        "description": "Run each test N times"
      },
      "reporter": {
        "type": "string",
        "enum": [
          "list",
          "line",
          "dot",
          "json",
          "junit",
          "null",
          "github",
          "html",
          "blob"
        ],
        "description": "Reporter to use, comma-separated, can be 'list', 'line', 'dot', 'json', 'junit', 'null', 'github', 'html', 'blob'. To configure reporter options, use the playwright configuration."
      },
      "retries": {
        "type": "number",
        "description": "Maximum retry count for flaky tests, zero for no retries"
      },
      "shard": {
        "type": "string",
        "description": "Shard tests and execute only the selected shard, specify in the form 'current/all', 1-based, for example '3/5'"
      },
      "timeout": {
        "type": "number",
        "description": "Specify test timeout threshold in milliseconds, zero for unlimited"
      },
      "trace": {
        "type": "string",
        "enum": [
          "on",
          "off",
          "on-first-retry",
          "on-all-retries",
          "retain-on-failure"
        ],
        "description": "Force tracing mode, can be 'on', 'off', 'on-first-retry', 'on-all-retries', 'retain-on-failure'"
      },
      "updateSnapshots": {
        "alias": "u",
        "type": "boolean",
        "description": "Update snapshots with actual results. Snapshots will be created if missing."
      },
      "ui": {
        "type": "boolean",
        "description": "Run tests in interactive UI mode"
      },
      "uiHost": {
        "type": "string",
        "description": "Host to serve UI on; specifying this option opens UI in a browser tab"
      },
      "uiPort": {
        "type": "string",
        "description": "Port to serve UI on, 0 for any free port; specifying this option opens UI in a browser tab"
      },
      "skipInstall": {
        "type": "boolean",
        "description": "Skip running playwright install before running playwright tests. This is to ensure that playwright browsers are installed before running tests.",
        "default": false
      }
    },
    "required": []
  }
```


- create a new [executors.json](/executors.json) in the root of your monorepo
- Add following line to your [package.json](/package.json)
```json
{
...
"executors": "./executors.json"
...
}
```
- Create a new file inside of your project dir and name it [browserstack.yml](/packages/playwright/browserstack.yml). For more details on BrowserStack SDK config file please refer [here](https://www.browserstack.com/docs/automate/selenium/sdk-config-generator)
```yml

buildName: "Your Build Name"
projectName: "Your Project Name"
testObservability: true
browserstackAutomation: true  # Set this to false if you are executing tests locally.

```
- Inside of every project where you want to use playwright with browserstack. Add a new target
```json

"e2e:bstack": {
  "executor": "./:playwright",
  ...other config
  "options": {
    "config": "/path/to/playwright/config",
    "browserstackConfig":"/path/to/browserstack/config"
  }
},

```

- Run your test
```shell
# Use your own target name here
npx nx run project:target
# eg npx nx run playwright:e2e:bstack
```