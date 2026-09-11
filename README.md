# hc-actor — Human Computing の cell を「計画」するだけの、実行しないアクター境界

**名乗り**: `hc` は Human Computing（スキマバイト + マイクロタスク）プラットフォームの
略である。ただし**この repo にプラットフォームは入っていない**。ここに在るのは、
そのアクターの **純粋な `.cljc` 計画境界 1 本だけ** —— `hc.murakumo`
（`src/hc/murakumo.cljk`）。

**この repo にシフトを作るコードも、タスクを配るコードも、労働基準法を検査する
コードも無い。** ネットワークに触る関数も、DB に触る関数も、Matrix に投稿する関数も
1 つも無い。`hc.murakumo` は入力を受けて **effect の *記述*** を返す純関数の集まりで、
その記述を誰かが実行するかどうかは、この repo の外側の話である。依存は
`clojure.string` だけ。

## この repo に在るもの（`git ls-files`、2026-09-06 実測）

| path | 何か |
|---|---|
| `src/hc/murakumo.cljk` | 唯一の実装。下記の計画境界 |
| `test/hc/murakumo_test.cljk` | その契約テスト（9 tests / 239 assertions） |
| `deps.edn` | `:test`（cognitect test-runner）/ `:lint`（clj-kondo） |
| `actor-manifest.jsonld` | アクター identity + pipeline + governance の宣言 |
| `.well-known/did.json` | 公開 DID 文書 |
| `storage-profile.edn` | ストレージ profile の宣言 |
| `CLAUDE.md` | **この repo ではなく上流デプロイの説明**（下記） |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Rider v3.1 |
| `README.md` | この文書 |
| `docs/operator-quickstart.md` | 実際に踏んだ手順 |

これに `.gitignore` と `.nojekyll` を足して、ファイルは全部で 12 個である。

## この repo に**無い**もの —— CLAUDE.md の読み方

`CLAUDE.md`（11,932 バイト）は SvelteKit の SuperApp・XRPC gateway・Matrix
protocol 連携・Arrow テーブル 8 本・労働基準法の自動検査・13 locale の契約書・
OEM サービスプロバイダ登録パイプラインを説明している。**それらの実装は 1 行もここに
無い。** 参照されている path を引くと次のようになる（2026-09-06 実測）:

```
wasm/etzhayyim-wasm-hc-hc0mp7ng/svelte                              MISSING
wasm/etzhayyim-wasm-hc-hc0mp7ng/svelte/src/lib/legal/contracts.ts   MISSING
90-docs/rules/compliance/per-did-kyumei-shinka-autonomy.md          MISSING
90-docs/platform/260403-live-data-kyumei-shinka-consolidated.md     MISSING
package.json                                                        MISSING
```

したがって CLAUDE.md の「Build & Deploy」節はここでは踏めない:

```bash
cd wasm/etzhayyim-wasm-hc-hc0mp7ng/svelte   # ← このディレクトリが無い
pnpm install && pnpm build
etzhayyim build                              # ← `etzhayyim` は PATH に無い
etzhayyim deploy --smoke-url https://hc0mp7ng.etzhayyim.com/health
```

`etzhayyim` はこのマシンにコマンドとして存在せず、smoke URL のホスト
`hc0mp7ng.etzhayyim.com` は **NXDOMAIN**（`dig +short` が何も返さない）。

CLAUDE.md が記述しているのは上流モノレポ（`etzhayyimcojp/20-actors` から
2026-05-21 に移設、`NOTICE` 参照）に在るデプロイであって、この repo の中身では
ない。**ここで踏める手順は [`docs/operator-quickstart.md`](docs/operator-quickstart.md)
が正本。**

## `hc.murakumo` が答えること

manifest 由来の **cell**（17 個）ごとに「いま effect を出してよいか」を判定し、
出してよければ MST への put-record effect を組み立てる。

### cell は manifest から導かれている

17 という数はマジックナンバーではなく、`actor-manifest.jsonld` の宣言の総数である
（2026-09-06 実測）:

| 出所 | 数 | cell |
|---|---|---|
| `pipelines[].trigger.type = "xrpc"` の nsid | 6 | `:listhc` `:searchhc` `:createhc` `:createspapplication` `:summarizehc` `:get` |
| `triggers.subscribeRepos.collections` | 5 | `:task` `:spapplication` `:spverification` `:spaudit` `:manufacturer` |
| `requiredCollections` | 2 | `:shinkaevolution` `:shinkaknowledge` |
| `requiredLoops` | 4 | `:shinka` `:koji` `:kyumei` `:domain-knowledge` |
| **合計** | **17** | |

`pipelines[].trigger.type = "cron"` の 2 本（日報 `0 9 * * *` と coverage
`0 */6 * * *`）からは cell が導かれていない。cell key は NSID の**最後のドット
区切りだけ**を小文字にして作られるので、`com.etzhayyim.apps.hc.coverage.get` は
`:get` という一般的すぎる key になっている。

各 cell はちょうど 1 collection、phase は全て `:event`、murakumo node は全て
`reuben`。

### gate は 7 本、deny-by-default

全 cell 共通で 7 本（`:council-charter-attestation`
`:no-platform-held-key-baseline` `:no-probing-baseline`
`:murakumo-only-inference-baseline` `:did-primary-baseline`
`:append-only-gate-baseline` `:kotoba-only-substrate-baseline`）。

attestation が 1 つも無いとき、17 cell すべてが `:status :blocked` を返し、
**生成される effect は合計 0 件**。gate が 1 本でも欠けていれば `:effects []` で、
部分実行は起きない。

```
(cell-plan :task {})                                ;; => :blocked, effects 0
(cell-plan :task {:attestations <7 gates のうち 6>})  ;; => :blocked, effects 0
(cell-plan :task {:attestations <7 gates> ...})     ;; => :ready,   effects 1
(all-cell-plans {})                                 ;; => 全 17 cell 合計 effects 0
(all-cell-plans {:attestations <7 gates> ...})      ;; => 全 17 cell 合計 effects 17
```

この「欠けたら止まる」性質はテストが両方向で押さえている
（`cell-plan-blocks-when-gates-missing` / `cell-plan-ready-when-gates-satisfied`）。
テストは cell 名を直書きせず `cell-specs` を走査するので、manifest が cell を
足しても契約は成り立ち続ける。自分の手で両方向を出す手順は quickstart の
「4.」に置いた。

## ⚠ 計画される collection は、manifest が宣言する NSID と一致しない

**2026-09-06 実測。既知の未修正の食い違いであって、この README が導入したものでは
ない。**

`collection` 関数は `(str "com.etzhayyim.hc." name)` を返すが、manifest が実際に
宣言している NSID は `com.etzhayyim.apps.hc.*` である。ずれは 2 つ:

| | 計画される collection | manifest の宣言 |
|---|---|---|
| 名前空間 | `com.etzhayyim.hc.task` | `com.etzhayyim.apps.hc.task` |
| 大文字小文字 | `com.etzhayyim.hc.spapplication` | `com.etzhayyim.apps.hc.spApplication` |

`apps.` セグメントが落ち、camelCase が平坦化されている。つまり **gate が全部
揃った場合に書かれる先は、manifest が subscribe している collection ではない。**

これは hc 固有ではなく scaffold 生成器の性質である（同じ生成器から出た
`cloud-itonami/akuma` も `com.etzhayyim.akuma.` を返す）。**ここでは記録するに
とどめ、直していない** —— 直すと生成される effect の中身が変わるので、どちらを
正本にするかの決定と、同じ scaffold を共有する repo 群の扱いが要る（この workspace の
`orgs/cloud-itonami/` checkout だけで 78 repo が同型の `src/<name>/murakumo.cljc` を
持つ。2026-09-06 実測。checkout されていない repo は数えていないので、これは下限）。

## Identity —— DID が 3 つあり、コードが名乗るものは解決しない

2026-09-06 に実際に引いた結果:

| DID | 引いた URL | 結果 |
|---|---|---|
| `did:web:etzhayyim.com:actor:hc` | `https://etzhayyim.com/actor/hc/did.json` | **200**、`id` が `.well-known/did.json` と一致 |
| `did:web:hc.etzhayyim.com` | `https://hc.etzhayyim.com/.well-known/did.json` | **NXDOMAIN**（ホストが存在しない） |
| `did:web:etzhayyim.github.io:com-etzhayyim-hc` | `https://etzhayyim.github.io/com-etzhayyim-hc/did.json` | 404 |

`.well-known/did.json` が公開しているのは 1 行目（解決する方）。

⚠ **一方 `src/hc/murakumo.cljk` の `actor-did` は 2 行目**
（`did:web:hc.etzhayyim.com`）**を持っており、そのホストには DNS レコードが無い。**
`actor-manifest.jsonld` の `@id` と CLAUDE.md の URL も同じ 2 行目を使っている。
つまり **生成される全 effect の `:actor` は、解決しない DID を名乗る。**

さらに、解決する方の DID 文書も、この repo の `.well-known/did.json` と**同一では
ない**。`id` は一致するが、配信されている側は `alsoKnownAs` が空で
`serviceEndpoint` が `https://pds.aozora.app`、この repo の写しは `alsoKnownAs` を
4 件持ち `serviceEndpoint` が `https://pds.etzhayyim.com` を指す。

これらは既知の未修正の食い違いである。**ここでは記録するにとどめ、直していない**
—— 直すと effect の中身が変わるので、identity の正本をどれにするかの決定と、
`actor-manifest.jsonld` の追随が要る。検証コマンドは quickstart の「5.」に置いた。

## 使う

[`docs/operator-quickstart.md`](docs/operator-quickstart.md) —— fresh clone から
test / lint / nbb 実行 / DID 検証まで、**実際に踏んだコマンドだけ**を載せてある。

## Status

`actor-manifest.jsonld` は `standardStatus: required` を宣言し、`heartbeatRequired`
と `domainKnowledgeRequired` を真にしているが、それらを満たす実行系はこの repo に
無い。この repo 単体で検証できるのは、上の計画境界が gate を閉じることと、cell の
集合が manifest の宣言と一致することだけである。

## License

Apache License 2.0 + etzhayyim Charter Compliance Rider v3.1（`NOTICE` 参照）。
