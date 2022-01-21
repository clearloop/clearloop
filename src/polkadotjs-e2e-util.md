# polkadotjs e2e util

| Repository             | Pull Request               |
|------------------------|----------------------------|
| [ChainSafe/PINT][PINT] | [ChainSafe/PINT#83][#83] |

designed an elegant e2e toolkit with polkadotjs in PINT.

## 0. config


```typescript
/**
 * js/e2e/index.ts
 */

// Tests
const TESTS = (api: ApiPromise): Extrinsic[] => [
    {
        pallet: "assetIndex",
        call: "addAsset",
        args: [
            42,
            1000000,
            api.createType("AssetAvailability" as any),
            1000000,
        ],
    },
];

// main
(async () => {
    await Runner.run(TESTS);
})();
```

[ChainSafe/PINT#83][#83]

## 1. runner

```typescript
/**
 * js/e2e/src/runner.ts
 */

export default class Runner implements Config {
    public api: ApiPromise;
    public pair: KeyringPair;
    public exs: Extrinsic[];
    
    /**
     * run E2E tests
     *
     * @param {Builder} exs - Extrinsic builder
     * @param {string} ws - "ws://0.0.0.0:9988" by default
     * @param {string} uri - "//Alice" by default
     * @returns {Promise<Runner>}
     */
    static async run(
        exs: Builder,
        ws: string = "ws://127.0.0.1:9988",
        uri: string = "//Alice"
    ): Promise<void> {
        console.log("bootstrap e2e tests...");
        console.log("establishing ws connections... (around 2 mins)");
        const ps = await launch("pipe");
        ps.stdout.on("data", async (chunk: Buffer) => {
            process.stdout.write(chunk.toString());
            if (chunk.includes(LAUNCH_COMPLETE)) {
                console.log("COMPLETE LAUNCH!");
                const runner = await Runner.build(exs, ws, uri);
                await runner.runTxs(ps);
            }
        });

        // Log errors
        ps.stderr.on("data", (chunk: Buffer) => console.log(chunk.toString()));

        // Kill all processes when exiting.
        process.on("exit", () => killAll(ps));

        // Handle ctrl+c to trigger `exit`.
        process.on("SIGINT", () => killAll(ps));
    }

    /**
     * Execute transactions
     *
     * @returns void
     */
    public async runTxs(ps: ChildProcess): Promise<void> {
        this.exs.forEach(async (ex: Extrinsic) => {
            console.log(`run extrinsic ${ex.pallet}.${ex.call}...`);
            console.log(`\t arguments: ${JSON.stringify(ex.args)}`);

            if (ex.block) await this.waitBlock(ex.block);
            
            // NOTE
            // 
            // here we use `this.api.tx[ex.pallet][ex.call](...ex.arg)` to extract 
            // transactions from our config
            await this.timeout(
                this.api.tx[ex.pallet]
                    [ex.call](...ex.args)
                    .signAndSend(this.pair, (res) => {
                        this.checkError(res);
                    }),
                ex.timeout
            ).catch((e) => {
                throw e;
            });
        });

        // exit
        console.log("COMPLETE TESTS!");
        ps.send && ps.send("exit");
        process.exit(0);
    }
    
    // ...
}
```

[ChainSafe/PINT#83][#83]


[PINT]: https://github.com/ChainSafe/PINT
[#83]: https://github.com/ChainSafe/PINT/pull/83

