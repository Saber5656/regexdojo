# regexdojo 設計書 (v1)

- Repository: `github.com/Saber5656/regexdojo`
- Tagline: "Competitive regex puzzle and code-golf playground."
- License: MIT（前提）/ 完全クライアントサイド・クラウド送信なし / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存サービスとの位置づけ

regexdojo は、正規表現の code golf を「道場」として遊べる完全クライアントサイドの Web ゲームである。
お題（マッチすべき文字列リスト / マッチしてはいけない文字列リスト）に対して**できるだけ短い正規表現**を書き、
文字数でスコアを競う。xkcd #1313 で広まった regex golf の遊びを、
「OSS としてお題を PR で増やせる」「Wordle 方式のネタバレなし共有文字列で非同期に競える」「解説付きで学べる」
形に再構築する。サーバは持たず、判定も進捗もすべてブラウザ内で完結する。

**既存サービスとの差別化（車輪の再発明にならない位置づけ）**

| 既存 | 形態 | regexdojo との違い |
|---|---|---|
| Regex Golf (alf.nu) | 元祖 regex golf。xkcd #1313 の元ネタ | ソース非公開・お題追加の口なし・結果共有機能なし。regexdojo は **OSS でお題を PR でき、共有文字列で非同期対戦できる** |
| Regex Crossword (regexcrossword.com) | クロスワード型学習パズル | ジャンルが違う。「与えられた regex に合う文字を埋める」穴埋め型。regexdojo は「regex を書く」golf 型 |
| jimbly/regex-crossword (OSS) | 六角クロスワードの OSS 実装 | 同じくクロスワード型。**golf 型の OSS ゲーム**という空白を regexdojo が埋める |
| regex101 / RegExr | テスター・学習ツール（RegExr は OSS） | ゲームではない。競技・スコア・共有の仕掛けがない。判定技術（Worker 実行）は RegExr の実装を先例として参考にする |

つまり「**お題がコミュニティで育つ、共有で煽り合える regex golf**」が存在理由。
リアルタイム対戦サーバで勝負せず、Wordle が証明した**コピペ共有文字列による非同期対戦**と
**PR でお題が増える OSS 構造**で差別化する。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | ソロ道場（code golf スコアリング） | お題を解く → パターン文字数がスコア。par と比較 |
| 入れる | 同梱お題 20〜30 問（難易度 belt 制） | White → Yellow → Brown → Black。JSON でリポジトリ同梱 |
| 入れる | Web Worker 判定 + タイムアウト terminate | ReDoS（壊滅的バックトラック）で UI を固めない。§4・§5 |
| 入れる | リアルタイム ✅/❌ フィードバック | 1 文字打つごとに全テストケースを再判定。体験の核 |
| 入れる | Wordle 方式の結果共有文字列 | ネタバレなし（パターン非含有）。クリップボードコピーのみ |
| 入れる | クリア後の解説表示 | 想定解の考え方を 2〜4 文で。教育的差別化 |
| 入れる | お題フォーマット公開 + CI 検証 + CONTRIBUTING | **PR でお題を追加できる**OSS 成長戦略の中核。§5 |
| 入れる | 進捗・ベストスコアの localStorage 保存 | 端末内のみ |
| 入れる | JavaScript 方言の明示 | UI フッターと README に「Flavor: JavaScript — not PCRE」 |
| 入れない | リアルタイムオンライン対戦 | サーバ・WebSocket・マッチメイキングの運用が必要で、**クラウド送信なし原則と個人 OSS の維持コストに反する**。非同期の共有文字列対戦で「競技性」は成立する |
| 入れない | 同一画面 2 人交互対戦 | v2。ターン管理と相手の解の隠蔽 UI が必要で「早く出す」に反する。ソロ判定エンジンをそのまま流用できるため v2 で薄く追加可能 |
| 入れない | フラグ入力（/i /g /s 等） | v1.x。ルールとスコア比較を単純に保つ。全お題フラグなし前提で設計 |
| 入れない | デイリーチャレンジ | v2。date-seed でサーバレス実装は可能だが、お題 30 問では 1 ヶ月で枯渇する。お題が PR で増えてから |
| 入れない | ヒント機能・マッチ可視化 | v2。regex101 的な解析表示はスコープ肥大。解説で代替 |
| 入れない | PWA / オフライン対応 | v1.x。8bitme と違い「ローカル完結の証明」がコア価値ではない。通常のブラウザキャッシュで十分 |
| 入れない | UI 多言語化 | 開発者向けのため英語 UI のみ。お題・解説も英語 |
| 入れない | アカウント・ランキングサーバ | クラウド送信なし原則に反する。順位は共有文字列の貼り合いに委ねる |

---

## 3. 対応プラットフォームと優先順位

| 優先度 | 対象 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | デスクトップブラウザ: Chrome / Edge / Firefox / Safari 各最新 2 メジャー | 対応 | 正規表現を「書く」体験は物理キーボードが前提。一次ターゲット |
| 2 (v1) | モバイルブラウザ: iOS Safari 16+ / Android Chrome | 対応（閲覧・プレイ可能） | SNS に貼られた共有文字列からの着地先として重要。1 カラムレスポンシブ + 入力欄 16px でプレイ可能にする。**記号入力の快適化（専用キーパッド等）は v2** |
| - | IE / 旧 Android WebView | 非対応と明記 | サポート表明しない |

必要 API は Web Worker / Clipboard API / localStorage のみで、いずれも対象ブラウザで安定。
JS RegExp の ES2018+ 機能（lookbehind、named groups 等）も対象ブラウザすべてで利用できるため、お題での使用を許容する。

---

## 4. 技術選定

### UI スタック比較

| 候補 | バンドル | 開発コスト | 判定 |
|---|---|---|---|
| **TypeScript + Preact + Vite（採用）** | Preact ~4KB。初回ロードが軽い | React 互換 API。状態管理ライブラリ不要の規模 | ✅ |
| React + Vite | ~45KB | 標準的だが本規模には過剰 | ❌ |
| Vanilla TS | 0KB | ✅/❌ のライブ更新・画面遷移・クリア演出の状態管理を手作りすることになり、かえって高コスト | ❌ |
| Svelte | 小 | 良いが、同作者の 8bitme（Preact + Vite）とスタックを揃えた方が保守が楽 | ❌ |

### 判定エンジン比較（本プロダクト最大の技術論点）

| 候補 | 方言 | ReDoS 耐性 | 判定 |
|---|---|---|---|
| **ブラウザ組込み RegExp を Web Worker 内で実行 + タイムアウトで `worker.terminate()`（採用）** | 本物の JavaScript | 暴走時は worker ごと殺して respawn。メインスレッドは無傷 | ✅ |
| メインスレッドで RegExp を直接実行 | 同上 | ❌ `(a+)+$` 型の壊滅的バックトラックでタブが固まる。Chromium は自主的に打ち切らない | ❌ |
| WASM 製 linear-time エンジン（rust regex / RE2 系） | ❌ backreference・lookaround 非対応で JS 方言と乖離。golf の解が変わってしまう | ◎ 原理的に安全 | ❌ |
| サーバサイド判定 API | 任意 | ○ | ❌ クラウド送信なし原則に反する |

この「**Worker 内で実行し、応答がなければ terminate**」方式は OSS の RegExr が本番採用している実績がある
（`gskinner/regexr` の `BrowserSolver.js` が `setTimeout` → `worker.terminate()` を実装。参考資料参照）。
ブラウザ側の保護は当てにできない（Firefox には内部タイムアウトの報告がある一方、Chromium は回り続ける）ため、
**アプリ側で一律のタイムアウトを持つ**。メインスレッドで裸の RegExp を回すコードは書かない（レビュー原則）。

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript | 判定結果・お題スキーマを型で固定。コントリビュートも受けやすい |
| UI | Preact + 素の CSS | 上表のとおり。ルーティングは hash ベース（`#/p/12`）で Pages のサーバ設定不要 |
| ビルド | Vite | 静的出力。GitHub Pages 向けが容易 |
| 判定 | JS `RegExp` を専用 Web Worker で実行。タイムアウト（既定 300ms、#1 spike で確定）で terminate → respawn | 上表のとおり。RegExr 先例あり |
| お題データ | JSON（`puzzles/*.json`、ビルド時 bundle） | PR 差分が読みやすい。スキーマは §5 |
| テスト | Vitest（判定・スコア純関数 + 全お題の整合性検証） | お題検証は CI の必須ジョブ（§5） |
| CI/CD | GitHub Actions → GitHub Pages | お題 PR のマージだけで新お題が配信される |
| ランタイム依存 | Preact のみ | 依存最小を売りにする |

### 正規表現方言の扱い（仕様として明記）

- 判定は**ブラウザの JavaScript (ECMAScript) RegExp** で行う。`\A` `\z` やアトミックグループ、所有量指定子など **PCRE 固有機能は使えない**
- UI フッターに常時 `Flavor: JavaScript (ECMAScript) — not PCRE` を表示し、README にも差異の注意書きと MDN へのリンクを載せる
- お題の想定解は方言差の出にくい構文を基本とする（lookbehind 等 ES2018+ は対象ブラウザ全対応のため使用可）

---

## 5. アーキテクチャ

サーバなし。GitHub Pages が配る静的ファイルだけで完結する。

```
┌────────────────────────────────────────────────────────────┐
│ ブラウザ                                                    │
│                                                            │
│  ┌─────────────────────┐          ┌──────────────────────┐ │
│  │ UI (Preact)          │postMessage│ Judge Worker         │ │
│  │  PuzzleView          │{pattern, │  (Web Worker)        │ │
│  │  ├ パターン入力       │ cases}   │                      │ │
│  │  ├ ✅/❌ ライブ表示   │─────────▶│ 1. new RegExp(p)     │ │
│  │  ├ 文字数 / par      │          │    構文エラーは即返送  │ │
│  │  └ 解説(クリア後)     │◀─────────│ 2. match[] を test() │ │
│  │                      │ 判定結果  │ 3. nonMatch[] を test│ │
│  │  タイムアウト監視      │──kill───▶│    ※応答が無ければ    │ │
│  │  (terminate+respawn) │          │      殺される         │ │
│  └──────┬──────────────┘          └──────────────────────┘ │
│         │                                                  │
│  ┌──────▼───────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ puzzles.json │  │ share.ts      │  │ progress.ts      │  │
│  │ 同梱 20–30 問 │  │ 共有文字列生成 │  │ localStorage     │  │
│  │ (ビルド時取込) │  │ → Clipboard   │  │ クリア/ベスト記録 │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 処理フロー（1 キーストローク）

1. 入力を 150ms debounce し、`{pattern, match[], nonMatch[]}` を Judge Worker へ postMessage（判定中に来た入力は最新 1 件だけ保持）
2. Worker: `new RegExp(pattern)`（フラグなし）。`SyntaxError` なら即エラー応答
3. Worker: match / nonMatch の各行に `regex.test(line)` を実行し、真偽配列を返す
4. UI: ✅/❌ を更新。**全 match が true かつ全 nonMatch が false** でクリア。スコア = パターンの文字数（UTF-16 code unit 長）
5. タイムアウト（既定 300ms）を過ぎたら `worker.terminate()` → 新 Worker を生成し、`Too slow — possible catastrophic backtracking` を表示（入力は保持）
6. クリア時: par と比較して表示、localStorage のベスト更新、共有文字列生成を有効化

### お題データフォーマット（公開仕様。PR でお題を追加する際の契約）

```jsonc
// puzzles/white/001-first-steps.json
{
  "id": 1,                      // 通し番号。共有文字列の "#001" に使う
  "slug": "first-steps",
  "title": "First steps",
  "belt": "white",              // "white" | "yellow" | "brown" | "black"
  "match":    ["cat", "cart", "coat"],  // 全行で test() が true になること
  "nonMatch": ["dog", "dot", "door"],   // 全行で test() が false になること（1 件以上必須）
  "par": 6,                     // 想定解の文字数。referenceSolution と CI で一致検証
  "referenceSolution": "c.a?rt|coat",   // 想定解。OSS のため公開だが UI には出さない
  "explanation": "English, 2-4 sentences. Shown only after clearing.",
  "author": "Saber5656"         // 貢献者の GitHub ユーザー名
}
```

ルール仕様:

- 判定は `test()`（部分一致）。「全体一致」を要求したいお題は nonMatch 側の文字列設計で縛る
- フラグは入力不可（`new RegExp(pattern)` 固定）。スコアはパターン文字数のみ
- 制約: パターン ≤ 256 文字、テスト行 ≤ 64 文字、match / nonMatch 各 1〜10 行

### お題の CI 検証（OSS 成長戦略の中核）

| 検査 | 内容 |
|---|---|
| 解の実在 | `referenceSolution` が全 match に true / 全 nonMatch に false |
| par の整合 | `referenceSolution.length === par`（par 詐称の防止） |
| スキーマ | 必須フィールド・belt の enum・行長・件数制限 |
| 自明解の防止 | nonMatch 1 件以上必須（`.*` で全クリアできるお題を拒否）・match / nonMatch に重複行がない |

これにより「お題 = JSON 1 ファイルの PR。CI が解けること・par が正しいことを機械検証し、
マージすれば自動デプロイで配信される」という**コントリビュートの導線**が成立する。
なお referenceSolution と解説はリポジトリ上で見えるため、ネタバレを避けたい人向けに
README に「`puzzles/` を覗くとネタバレ」と一言添える（OSS としては割り切り）。

---

## 6. UI/UX

### 主要画面（1 画面 + お題一覧）

```
┌──────────────────────────────────────────────┐
│ regexdojo 🥋        #012 / 30   🟡 Yellow    │← ヘッダ。一覧へのリンク
│ Write the shortest regex that matches all ✅ │← ルールはこの 1 行だけ
│ and none of the ❌.                           │
├──────────────────────────────────────────────┤
│ Match all of these     │ Reject all of these │
│ ✅ cat                 │ ✅ dog              │
│ ✅ cart                │ ❌ dot ← 通ってしまって│
│ ✅ coat                │ ✅ door    いる行は赤 │
├──────────────────────────────────────────────┤
│  /  c.a?rt|coat  /        11 chars   par 11 │← 等幅・大きめ。文字数は常時更新
│  (⚠ Invalid regex … / ⏱ Too slow … を状況で) │
├──────────────────────────────────────────────┤
│ 🎉 Cleared — even par!   [Copy result 📋]    │← クリア時のみ出現
│ ▼ Explanation（クリア後にだけ展開できる）        │
│ [◀ Prev]                          [Next ▶]  │
└──────────────────────────────────────────────┘
```

お題一覧: belt 別の 4 セクションに、クリア済みチェックとベスト文字数（par 差付き）を並べる。
hash ルーティング `#/p/12` で個別お題へ直リンクできる（共有文字列の URL に使用）。

### スコア表示

- 入力欄の横に**現在の文字数を常時表示**（打つたびに増減が見える = golf の緊張感）
- クリア時は par 比較を golf 表記で: `2 under par!` / `even par` / `+3`
- ベスト記録は localStorage に保存し、一覧とお題画面に表示。更新時は `New best!`

### 共有文字列（Wordle 方式・ネタバレなし）

```
regexdojo #012 🟡
📏 11 chars (par 11, E)
https://saber5656.github.io/regexdojo/#/p/12
```

- **パターン本体は絶対に含めない**（ネタバレ防止。文字数と par 差だけで煽れる）
- belt は色付き絵文字（⚪🟡🟤⚫）でひと目で難度が伝わる
- 生成はローカルの文字列組み立てのみ。コピーはユーザーの `Copy result` クリック時だけ（§7）

### 初回起動体験（勝負は最初の 30 秒）

1. URL を開くと**チュートリアルなしで #001（White belt の入門問題）が即座に開く**。モーダルもツアーもなし
2. ルール説明はヘッダ下の 1 行だけ。#001 は「`a` の 1 文字で解ける」ような、regex を知らなくても英字で解ける難度にする
3. 1 文字打つごとに両カラムの ✅/❌ が即時に反転する——この**手応えのループ**が最初の 10 秒で伝わることを最優先
4. クリアで帯色の演出 + `Copy result` が点灯し、共有への導線を作る
5. `Next ▶` で次のお題へ。腕に覚えがあれば一覧から Black belt へ直行できる

### エッジケース

| ケース | 挙動 |
|---|---|
| 構文エラー | 入力欄下に `Invalid regex: <message>` を 1 行表示。✅/❌ は直前の状態のままグレーアウト |
| タイムアウト（ReDoS 疑い） | `Too slow — possible catastrophic backtracking` を表示し worker respawn。入力は消さない |
| 空入力 | 判定しない（初期状態に戻す） |
| 256 文字超 | それ以上入力できず、文字数カウンタを赤表示 |
| `.*` 等のズル | お題データ側で防ぐ（nonMatch 必須。§5 の CI 検証） |
| クリップボード拒否 | 共有文字列をその場に選択可能なテキストで表示するフォールバック |
| localStorage 不可（プライベートモード等） | 記録なしで動作継続。「Progress won't be saved」を小さく表示 |
| モバイル幅 | 2 カラムを縦積みに。入力欄は font-size 16px（iOS の自動ズーム防止） |

---

## 7. プライバシー

### 原則

1. **外部にデータを送る通信経路をそもそも作らない**（バックエンド・API・外部 SaaS なし）
2. **スコア共有はユーザーのコピー操作のみ**。アプリが SNS へ投稿したり、どこかへ送信したりする機能は持たない
3. 設定で OFF ではなく、**能力として不可能**にする

### 担保策

| 層 | 施策 |
|---|---|
| アーキテクチャ | 判定・スコア・共有文字列生成・進捗保存のすべてがブラウザ内で完結。入力した正規表現がネットワークに乗る経路が存在しない |
| CSP | `default-src 'self'` を meta タグで宣言。外部への fetch はブラウザレベルでブロックされる |
| 共有 | `Copy result` クリック時に Clipboard API へ書き込むだけ。**どこに貼るかは完全にユーザーの手**。URL に解やスコアを埋め込む機能も持たない |
| 計測ゼロ | アナリティクス・トラッカー・外部フォント・CDN を一切使わない（CSP とも整合） |
| 保存 | 進捗・ベストスコアは localStorage（端末内のみ）。消したければブラウザのサイトデータ削除で完全に消える |
| OSS | 全コード公開 + GitHub Actions でビルドされるため、配信物とソースの対応を検証できる |

README にも Privacy 節として「no backend, no analytics, sharing is just your clipboard」を明記する。

---

## 8. 配布方法

| 項目 | v1 の方針 | 理由 |
|---|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/regexdojo/`） | 無料・独自サーバなし。1 クリックで遊べる URL を README 最上部に置く |
| デプロイ | GitHub Actions: main への push → `vite build` → Pages deploy | 手作業ゼロ。**お題 PR のマージ = 新お題の配信** |
| バージョニング | タグ + GitHub Releases（CHANGELOG） | 静的アプリのため「更新 = 再訪で最新」 |
| 独自ドメイン | なし | 任意・後回し。Pages の URL で十分 |
| ストア/パッケージ | なし | Web 1 本。インストール障壁ゼロが売り |
| ライセンス | MIT（コード・お題データとも） | お題も PR で受けるため、データのライセンスも README に明記 |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: 道場の帯 + 正規表現が飛び交う横長 PNG>

# regexdojo 🥋
> Competitive regex puzzle & code-golf playground — 100% in your browser.

[license バッジ] [deploy (Pages CI) バッジ] [PRs welcome バッジ]   ← 3 個まで

<デモ GIF: タイプするたび ✅/❌ が反転 → クリア → Copy result までの 10 秒ループ>

**👉 Play now: https://saber5656.github.io/regexdojo/** — no install, no sign-up, no server.

## How to play
- Write the shortest regex that matches all ✅ strings and none of the ❌ strings.
- Your score is the pattern length. Beat the par.
- 20+ puzzles from White belt (beginner) to Black belt (boss).

## Share your result
- Wordle 方式のネタバレなし共有文字列の実例ブロック（§6 と同じもの）
- "Paste it in your team chat and start a golf war." の 1 文

## Regex flavor
- Judged with JavaScript (ECMAScript) RegExp — not PCRE の明記
- 差異の代表例 1〜2 個と MDN リンク

## Add your own puzzle
- puzzles/*.json 1 ファイルの PR で追加できること、CI が解と par を検証すること
- CONTRIBUTING.md へのリンク。「puzzles/ を覗くとネタバレ」の注意

## Privacy
- No backend, no analytics, no tracking. Your regex never leaves the page.
- Sharing = copying text to your clipboard. That's it.

## License
MIT (code and puzzle data)
```

ポイント: デモ GIF と Play now リンクがファーストビューに収まること。
バッジは license / deploy / PRs welcome の 3 つまで。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| P0 | Worker terminate 方式の実効性 | `(a+)+$` + 長い `aaa...b` 型の入力で、タイムアウト → `worker.terminate()` → respawn がメインスレッド無傷で成立するか。RegExr の先例はあるが自分の構成で確認する | 使い捨てプロトタイプ（Issue #1） |
| P0 | リアルタイム判定のレイテンシ | keystroke → Worker 往復 → 表示更新が体感リアルタイム（目標 < 50ms）か。terminate → respawn 直後の連続入力で判定が詰まらないか（必要なら warm standby worker） | 同プロトタイプで実測し、タイムアウト値・debounce 値を確定 |
| P1 | お題の品質と par の妥当性 | golf として面白いお題が 20〜30 問作れるか。par が緩すぎ/きつすぎないか。CI は機械検証のみで面白さは保証しない | referenceSolution 付きで執筆し、序盤 5 問を regex 非熟練者に触ってもらう |
| P1 | 共有文字列の互換性 | X / Discord / Slack に貼った際に絵文字・改行が崩れないか | 実端末で貼り付け確認（Issue #5 の受け入れ条件） |
| P1 | JS 方言明示の伝わり方 | PCRE 慣れのユーザーが「動くはずの構文が動かない」と感じる混乱 | フッター常時表示 + README 注記。お題の想定解は方言差の出にくい構文を基本にする |
| P2 | モバイルでの記号入力体験 | ソフトキーボードで `\` `^` `$` が打ちにくく離脱要因になる | v1 はレイアウト崩れなしまでを保証し、専用キーパッドは v2 判断 |
| P2 | localStorage の消失 | プライベートモードやサイトデータ削除で進捗が消える | 消えても遊べる設計（進捗は付加情報）+ 保存不可時の注意表示 |
| P3 | 名称の衝突 | "regexdojo" と類似の既存サービス・商標の簡易確認 | リリース前に検索 |

**最重要リスク**: P0 の 2 件。判定エンジンの安全性と応答性はこのプロダクトの土台であり、
ここが崩れると方式変更（WASM エンジン採用 = 方言変更）になりスコープが壊れる。
**本実装前に Issue #1 の spike で必ず検証する。**

---

## 11. v1 Issue 分割案（9 個）

- **#1 `Spike: prove worker-based regex judge survives catastrophic backtracking`** — ラベル: `spike`, `design`
  Web Worker 内で `new RegExp` + `test()` を実行し、`(a+)+$` 型の ReDoS パターンをタイムアウトで `terminate()` → respawn する方式の使い捨てプロトタイプを作り、P0 リスク 2 件（安全性・レイテンシ）を検証する。
  受け入れ条件: ReDoS 入力でメインスレッドが固まらず respawn 後も判定が継続すること、通常パターンの往復が体感リアルタイム（< 50ms 目標）であること、タイムアウト値・debounce 値・respawn 方式の結論が Issue コメントに記録されていること。

- **#2 `Define puzzle JSON schema and validation script`** — ラベル: `design`
  §5 のお題フォーマット（match / nonMatch / par / referenceSolution / explanation / belt）を TypeScript 型 + スキーマとして確定し、全お題を機械検証するスクリプトを実装する。「PR でお題を追加できる OSS」の土台。
  受け入れ条件: referenceSolution が全 match に true・全 nonMatch に false・長さ = par であることを `npm run validate:puzzles` で検証できること、スキーマ違反が読めるエラーで報告されること、サンプル 3 問が通ること。

- **#3 `Implement judge worker and golf scoring engine`** — ラベル: `enhancement`
  #1 の結論に基づき、判定 Worker（構文エラー即応答・ケース別 test()・タイムアウト terminate + respawn）とスコア計算（文字数・par 比較・クリア判定）を本実装する。judge は UI 非依存の純関数 + Worker ラッパとして分離する。
  受け入れ条件: 判定リクエストの直列化と最新入力のみ判定が機能すること、構文エラー / タイムアウト / 判定結果の 3 状態が型で区別されること、Vitest の単体テストが通ること。

- **#4 `Build dojo screen with live pass/fail feedback`** — ラベル: `enhancement`, `ux`
  メイン画面（match / nonMatch 2 カラムの ✅/❌ ライブ更新・等幅パターン入力・文字数 / par 表示・クリア演出・クリア後の解説展開）を §6 のレイアウトと文言どおりに実装する。
  受け入れ条件: 1 文字入力ごとに全テストケースの表示が更新されること、構文エラーとタイムアウトが §6 の仕様どおり表示されること、モバイル幅で 1 カラムに折り返し入力欄が 16px 以上であること。

- **#5 `Add Wordle-style spoiler-free result sharing`** — ラベル: `enhancement`, `ux`
  クリア時に「問題番号・belt 絵文字・文字数・par 差・お題 URL」だけの共有文字列を生成し、`Copy result` でクリップボードにコピーする。パターン本体は絶対に含めない。
  受け入れ条件: 共有文字列に正規表現本体が含まれないこと、Clipboard API 拒否時に選択可能テキスト表示へフォールバックすること、X / Discord / Slack への貼り付けで崩れないことを目視確認済みであること。

- **#6 `Add puzzle list, belt tiers, and local progress`** — ラベル: `enhancement`
  belt（White → Yellow → Brown → Black）別のお題一覧、クリア済み表示、ベストスコアの localStorage 永続化、hash ルーティング（`#/p/12` 直リンク）を実装する。
  受け入れ条件: 一覧から任意のお題を開けてクリア状況とベスト文字数が再訪後も保持されること、共有 URL から個別お題が直接開けること、localStorage 不可の環境でも保存なしで動作すること。

- **#7 `Author 20-30 puzzles with pars and explanations`** — ラベル: `design`
  White〜Black belt で計 20〜30 問（match / nonMatch / par / referenceSolution / 英語解説）を執筆する。#2 の検証を全問通し、#001 は regex 非熟練者がチュートリアルなしで解ける難度にする。
  受け入れ条件: 20 問以上が validate:puzzles を通過していること、各 belt に最低 4 問あり #001 が 1〜3 文字で解ける入門問題であること、全問に 2〜4 文の英語解説が付いていること。

- **#8 `Set up GitHub Pages deploy and puzzle-validating CI`** — ラベル: `infra`
  GitHub Actions で PR 時に typecheck / test / validate:puzzles を実行し、main への push で Vite build → GitHub Pages へ自動デプロイする。お題 PR のマージだけで新お題が配信される状態にする。
  受け入れ条件: お題 JSON のみ変更する PR で検証が走り不正なお題が fail すること、マージ後に手作業なしで本番 URL が更新されること、CSP メタタグが配信 HTML に含まれること。

- **#9 `Write README and CONTRIBUTING with puzzle PR guide`** — ラベル: `docs`
  §9 の構成で英語 README（バナー・デモ GIF・Play now 導線・JavaScript flavor 注記・Privacy 節）を作成し、お題追加 PR の手順（フォーマット・par の決め方・CI・ネタバレ注意）を CONTRIBUTING.md にまとめる。
  受け入れ条件: デモ GIF と Play now リンクがファーストビューに収まること、方言が JavaScript であることと PCRE との差異が明記されていること、CONTRIBUTING の手順どおりに新規お題 PR が出せることをセルフレビューで確認済みであること。

推奨着手順: #1 → #2 → (#3, #7 並行) → #4 → (#5, #6 並行) → (#8, #9 並行)。

---

## 参考資料（設計時の調査ソース）

- regex golf の原型と文化: [Regex Golf (alf.nu)](https://alf.nu/RegexGolf) / [xkcd #1313 "Regex Golf"](https://xkcd.com/1313/) / [HN スレッド (2013)](https://news.ycombinator.com/item?id=6941231)
- クロスワード型の既存: [Regex Crossword](https://regexcrossword.com/) / [jimbly/regex-crossword（OSS 六角変種）](https://jimbly.github.io/regex-crossword/)
- テスター系ツールと OSS 実装: [regex101](https://regex101.com/) / [gskinner/regexr（OSS。Worker + terminate の先例）](https://github.com/gskinner/regexr) / [SunneV/OpenRegex（ReDoS 保護を掲げる自己ホスト型テスター）](https://github.com/SunneV/OpenRegex) / [slevithan/awesome-regex（ツール一覧）](https://github.com/slevithan/awesome-regex)
- 壊滅的バックトラック / ReDoS: [Catastrophic backtracking (javascript.info)](https://javascript.info/regexp-catastrophic-backtracking) / [Sonar — Vulnerable regular expressions in JavaScript](https://www.sonarsource.com/blog/vulnerable-regular-expressions-javascript/) / [Snyk — ReDoS and catastrophic backtracking](https://snyk.io/blog/redos-and-catastrophic-backtracking/) / [Node.js での ReDoS 緩和とブラウザ挙動差](https://www.josephkirwin.com/2016/03/12/nodejs_redos_mitigation/)
- JS 正規表現の方言仕様: [MDN — Regular expressions guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions)

---

## Changelog

- 2026-07-05: 初版
