# Operator quickstart

**ここに載っているコマンドは、2026-09-06 に実際に実行したものだけである。** 出力は
貼った時点の実測値。踏めなかった手順は載せていない（AGENTS.md にある
`pnpm build` / `etzhayyim build` / `etzhayyim deploy` は、その対象ファイルが
この repo に無いので**ここでは踏めない**。理由は [README](../README.md) の
「この repo に**無い**もの」を参照）。

所要は 5 分弱。JVM を起こすのは 2. と 3. だけで、4.〜6. は nbb（Node）で走る。

## 0. 前提

| 道具 | 用途 | 確認 |
|---|---|---|
| `git` | 1. clone | `git --version` |
| `clojure` | 2. test / 3. lint | `clojure --version` |
| `nbb` | 4. gate 確認 / 6. cell 照合（JVM 不要） | `nbb --version` |
| `curl` / `dig` | 5. DID 検証 | 標準 |

`clojure` と `nbb` は独立している。**片方しか無くても 1〜6 のうち踏める分は踏める。**

## 1. clone

```bash
git clone git@github.com:cloud-itonami/hc-actor.git
cd hc-actor
```

この workspace の west checkout から使う場合は
`orgs/cloud-itonami/hc-actor`（**remote 名は `origin` ではなく `cloud-itonami`**
—— `git fetch origin` は `fatal: 'origin' does not appear to be a git repository`
で exit 128 になる）。

## 2. テストを走らせる

```bash
kbb -M:test
```

```
Running tests in #{"test"}

Testing hc.murakumo-test

Ran 9 tests containing 239 assertions.
0 failures, 0 errors.
```

初回は依存（cognitect test-runner）の解決で 1〜2 分かかる。2 回目以降は数秒。

## 3. lint

```bash
kbb -M:lint
```

```
src/hc/murakumo.cljk:173:14: warning: unused binding input
linting took <N>ms, errors: 0, warnings: 1
```

（所要 ms は貼っていない —— このマシンでは並行セッションの負荷で大きく振れる。
見るのは `errors: 0, warnings: 1` の方。）

**warning 1 件は既知で、exit 0 である**（`:lint` alias は `--fail-level error`）。
`records-for` が `:as input` を束縛して使っていない。ここを直すのは lint の仕事で
あってこの quickstart の仕事ではないが、**exit 1 になったら warning ではなく
error が増えている**ので、その差分を見ること。

## 4. gate が閉まることを自分で確かめる（JVM 不要）

この repo の主張は「attestation が揃わなければ effect を 1 つも出さない」である。
**それを信じずに、その場で両方向を出す。**

```bash
kbb --backend sci --classpath src -e '
(ns probe (:require [hc.murakumo :as m]))
(let [all (into {} (map (fn [g] [g true]) m/common-gates))
      one-short (dissoc all (first m/common-gates))]
  (println "gates required:" (count m/common-gates))
  (println "no attestation   ->" (:status (m/cell-plan :task {}))
           "effects:" (count (:effects (m/cell-plan :task {}))))
  (println "one gate missing ->" (:status (m/cell-plan :task {:attestations one-short}))
           "effects:" (count (:effects (m/cell-plan :task {:attestations one-short}))))
  (println "all 7 attested   ->" (:status (m/cell-plan :task {:attestations all :request-id "req-1"}))
           "effects:" (count (:effects (m/cell-plan :task {:attestations all :request-id "req-1"}))))
  (println "fleet-wide, zero attestation -> total effects:"
           (reduce + (map #(count (:effects %)) (vals (m/all-cell-plans {})))))
  (println "fleet-wide, all attested     -> total effects:"
           (reduce + (map #(count (:effects %))
                          (vals (m/all-cell-plans {:attestations all :request-id "req-1"}))))))'
```

```
gates required: 7
no attestation   -> :blocked effects: 0
one gate missing -> :blocked effects: 0
all 7 attested   -> :ready effects: 1
fleet-wide, zero attestation -> total effects: 0
fleet-wide, all attested     -> total effects: 17
```

**最後の 2 行を両方見ること。** `:blocked` と 0 件しか出ない実行は、gate が閉まって
いる証拠ではなく、単に何も動いていない証拠かもしれない（classpath を間違えて別の
名前空間を読んでいても 0 件は出る）。**17 が出てから 0 が出ることに意味がある。**
7 本目を 1 本抜いただけで止まることが、この境界の主張そのもの。

## 5. DID を検証する

`.well-known/did.json` が公開している DID と、コード（`actor-did`）が名乗る DID は
**別物**（[README](../README.md) の Identity 節）。両方をその場で引く:

```bash
for u in "https://etzhayyim.com/actor/hc/did.json" \
         "https://etzhayyim.github.io/com-etzhayyim-hc/did.json"; do
  printf '%-58s %s\n' "$u" "$(curl -sS -o /dev/null -w '%{http_code}' --max-time 20 "$u")"
done
dig +short hc.etzhayyim.com
dig +short hc0mp7ng.etzhayyim.com
```

```
https://etzhayyim.com/actor/hc/did.json                    200
https://etzhayyim.github.io/com-etzhayyim-hc/did.json      404
                                     <- hc.etzhayyim.com は NXDOMAIN（空行）
                                     <- hc0mp7ng.etzhayyim.com も NXDOMAIN（空行）
```

`dig` が**何も返さない**のが現時点の期待値である（レコードが無い）。ここに IP が
出るようになったら、`src/hc/murakumo.cljk` の `actor-did` と AGENTS.md の API URL が
指す先が実在し始めたということなので、README の Identity 節を測り直すこと。

## 6. cell が manifest と一致していることを確かめる（JVM 不要）

README は「17 cell は `actor-manifest.jsonld` の宣言から導かれている」と書いている。
**その対応をコードと manifest の両側から数え直す。**

```bash
kbb --backend sci --classpath src -e '
(ns v (:require ["fs" :as fs] [hc.murakumo :as m]))
(def j (js->clj (js/JSON.parse (fs/readFileSync "actor-manifest.jsonld" "utf8"))))
(defn last-seg [s] (last (.split s ".")))
(def xrpc (->> (get j "pipelines") (map #(get % "trigger"))
               (filter #(= "xrpc" (get % "type"))) (map #(last-seg (get % "nsid")))))
(def subs (map last-seg (get-in j ["triggers" "subscribeRepos" "collections"])))
(def reqc (map last-seg (get j "requiredCollections")))
(def reql (get j "requiredLoops"))
(def declared (set (map #(keyword (.toLowerCase %)) (concat xrpc subs reqc reql))))
(println "manifest declares:" (count declared))
(println "cell-specs has   :" (count m/cell-specs))
(println "match            :" (= declared (set (keys m/cell-specs))))
(println "only in manifest :" (pr-str (sort (remove (set (keys m/cell-specs)) declared))))
(println "only in code     :" (pr-str (sort (remove declared (keys m/cell-specs)))))'
```

```
manifest declares: 17
cell-specs has   : 17
match            : true
only in manifest : ()
only in code     : ()
```

**`match: true` と、両方の "only in" が空であることを見ること。** 数が一致するだけ
では足りない（別々の 17 個でも 17 = 17 になる）。

この照合が false になったら、manifest が動いたのに scaffold が再生成されていないか、
その逆である。

## 踏めないもの（なぜ載っていないか）

| AGENTS.md のコマンド | ここで踏めない理由 |
|---|---|
| `cd wasm/etzhayyim-wasm-hc-hc0mp7ng/svelte` | `wasm/` がこの repo に無い |
| `pnpm install && pnpm build` | `package.json` がこの repo に無い |
| `etzhayyim build` | `etzhayyim` が PATH に無い |
| `etzhayyim deploy --smoke-url https://hc0mp7ng.etzhayyim.com/health` | 同上 + smoke URL のホストが NXDOMAIN |

これらは上流モノレポのデプロイ手順である。**この repo を clone しただけの
オペレータには実行できない**ので、ここには書かない。
