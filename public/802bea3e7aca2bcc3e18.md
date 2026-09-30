---
title: Microsoftの「Flint」はグラフの中間言語だった — Obsidianプラグインにして分かった設計と3つの罠
tags:
  - Obsidian
  - TypeScript
  - ECharts
  - データ可視化
  - Microsoft
private: false
updated_at: '2026-08-21T22:12:30+09:00'
id: 802bea3e7aca2bcc3e18
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
Microsoft が [flint-chart](https://github.com/microsoft/flint-chart) という OSS を出しています。README の一行目にはこう書いてあります。

> Flint is a visualization intermediate language enabling AI agents to convert simple, editable chart specifications into polished visualizations.

**中間言語**。グラフのライブラリではなく、コンパイラです。ここを読み飛ばすと設計を間違えます。実際に間違えかけました。

これを Obsidian のコードブロックに埋め込むプラグインを作ったので、その過程で分かったことを書きます。作ったものは [obsidian-flint-chart](https://github.com/yut0takagi/obsidian-flint-chart) です。

なお本記事の計測は `flint-chart@0.5.1` 時点のものです。まだプレ 1.0 で、npm 上の最終更新は 2026-08-14。API は動く可能性が高いので、バージョンは見てください。

## Flint が解こうとしている問題

Vega-Lite や ECharts のような宣言的な文法は、データの**型**と視覚的な**マッピング**が素直に対応しているときはよく働きます。崩れるのは、格納形式と意味がズレたときです。Flint の README はこの例を挙げています。

- 整数 `202001` は「量」ではなく **年月**
- 温度や比率のような非加算的な指標を積み上げてしまう
- 発散する値（増減率など）を連続的なカラーランプに載せてしまう

熟練者は長いスペックを書けばこれを直せます。ただそのスペックは、フィールドを差し替えたりヒートマップの向きを変えたりチャート種別を変えたりした途端に壊れます。低レベルなパラメータが互いに依存しているからです。

Flint は「この列が何を意味するか」を第一級の入力にして、そこからスケール・軸・集約・書式・レイアウト・配色を導出します。入力はこの形です。

```ts
import { assembleECharts } from 'flint-chart/echarts';

const option = assembleECharts({
  data: { values: rows },
  semantic_types: {          // 各列が何を意味するか
    quarter: 'Quarter',
    revenue: 'Price',
    region:  'Category',
  },
  chart_spec: {              // どんなグラフが欲しいか
    chartType: 'Bar Chart',
    encodings: {
      x: { field: 'quarter' },
      y: { field: 'revenue' },
      color: { field: 'region' },
    },
  },
});
```

意味型は 44 種あり（`flint-chart/core` の `SemanticTypes` から実測）、`Price` `Quantity` `Percentage` `PercentageChange` `YearMonth` `YearQuarter` `Rank` `Temperature` `Latitude` あたりまで揃っています。`semantic_types` はデータセットにつき一度書けば使い回せて、探索中に変えるのは `chart_spec` だけになる、というのが設計意図です。

軸のスケール、目盛りの刻み、桁区切り、凡例の位置、カテゴリの色。これらを一切書いていないのに出てくるのは、意味から導出しているからです。

## 罠1: Flint は描画しない

`assembleECharts()` の戻り値は ECharts の `option` オブジェクトです。DOM も canvas も触りません。同じ入力から Vega-Lite / ECharts / Chart.js / Plotly / Excel のネイティブスペックを吐く、という設計になっています。

```ts
const vlSpec  = assembleVegaLite(input);   // Vega-Lite の JSON
const ecSpec  = assembleECharts(input);    // ECharts の option
const cjsSpec = assembleChartjs(input);    // Chart.js の config
```

つまり `npm install flint-chart` だけでは何も表示されません。描画ライブラリは自分で同梱します。`peerDependencies` に `vega` `vega-lite` `echarts` `chart.js` `plotly.js` が並んでいるのはそのためで、使う 1 つを入れれば足ります。

Obsidian プラグインは 1 ファイル（`main.js`）に全部バンドルして配布するので、ここでどれを選ぶかが最初の分岐になりました。

## バックエンドの選択を実測で決める

対応チャート数と、esbuild でバンドルしたサイズを測りました。

| バックエンド | チャート種別 | 特徴 |
| --- | --- | --- |
| Vega-Lite | 36 | Flint のリファレンス実装。軸・スケール・faceting の再現度が最も高い |
| **ECharts** | **37** | sunburst / treemap / sankey / gauge / graph / tree / parallel / calendar heatmap を持つ |
| Chart.js | 22 | 軽量。パラメータ面を意図的に小さくしている |

```
flint-chart/echarts 単体                224KB
+ echarts 全体                          1.3MB
+ echarts/core 絞り込み（4種のみ登録）    796KB
```

`flint-chart` は npm の unpackedSize が 42.8MB あります。全バックエンド・全テーマ・テストデータ・Python のソースプレビューまで同梱しているからです。ただしサブパス export（`flint-chart/echarts`）が効いていて、tree-shaking 後は 224KB に落ちます。42MB を見て諦めなくて大丈夫でした。

`echarts/core` で使うチャートだけ `use()` する絞り込みは、500KB しか削れないのに 37 種のうち 31 種が使えなくなります。この交換レートは悪いので、ECharts 全体を入れる 1.3MB を採りました。参考に、Obsidian のコミュニティストアにある obsidian-chartsview-plugin は `main.js` が 3MB を超えています。

ECharts を選んだ決め手は、Vega-Lite 側にない構造系（sankey / treemap / sunburst / gauge / calendar heatmap）です。Markdown にグラフを埋め込むプラグインでサンキーが書ける、というのはあまりない気がしました。

## 罠2: `theme_spec` は Vega-Lite 専用

Flint には `theme_spec` があって、`economist` のようなプリセット名を渡すか、独自のデザインシステムを JSON で書けます。

```json
{ "theme_spec": "economist" }
```

ダークモード対応をこれでやるつもりで theme 仕様のドキュメントを開いたら、冒頭に引用ブロックで一行ありました。

> ThemeSpec currently affects Vega-Lite output. Other backend assemblers ignore it.

**ECharts では丸ごと無視されます。** プリセットも独自テーマも効きません。ドキュメントに書いてあるので仕様どおりです。ただ `theme_spec` が `ChartAssemblyInput` の第一級フィールドとして並んでいる以上、バックエンドによって効かないとは思いにくい。テーマ設計を始める前に気づけたのは運が良かっただけで、実装してから気づいたら作り直しでした。

さらに困ったことに、Flint が返す ECharts option は軸ラベルの色を**設定しません**。

```js
xAxis: {
  type: 'category',
  data: ['Q1', 'Q2'],
  name: 'quarter',
  axisLabel: { rotate: 0, fontSize: 12 },   // color がない
  axisTick:  { show: true, alignWithLabel: true },
  axisLine:  { show: true },
}
```

色が未設定だと ECharts の既定色（濃いグレー）が効きます。ライトテーマなら問題ありませんが、ダークテーマの背景では読めません。加えてタイトル類は `graphic` の text 要素として描かれ、`style.fill` が `#333` でハードコードされています。

結局、生成された option を後処理する方式にしました。Obsidian は CSS 変数でテーマ色を公開しているので、それを読みます。

```ts
const style = getComputedStyle(element);
const palette = {
  text:   style.getPropertyValue('--text-normal').trim(),
  muted:  style.getPropertyValue('--text-muted').trim(),
  border: style.getPropertyValue('--background-modifier-border').trim(),
  surface: style.getPropertyValue('--background-secondary').trim(),
};
```

`theme-dark` クラスを見てライト/ダークで分岐する書き方もありますが、CSS 変数を直接読むほうが結果的に良かった。ユーザーが入れているコミュニティテーマにも自動で追従します。プリセットが使えない代償として悪くない落ち着き先でした。

系列の色（`option.color` と `series[].itemStyle.color`）には触りません。あれは Flint が意味から決めたデータの色なので、テーマ側が上書きするものではない。

## 罠3: ユニットテストが自分の誤解をそのまま検証していた

これが一番書きたかった話です。

後処理で各コンポーネントの文字色を塗る関数を書き、テストも書きました。radar チャートの分はこうです。

```ts
// 実装
radar: {
  name: { ...radar.name, textStyle: { color: palette.text } },
  ...
}

// テスト
expect(option.radar.name.textStyle.color).toBe('#dcddde');
```

テストは通ります。当然です、実装とテストを同じ思い込みで書いているので。

しばらく後に、jsdom 上で実際の ECharts に食わせて SVG を出す統合テストを足しました。そこで初めてこれが出ました。

```
[ECharts] DEPRECATED: textStyle hierarchy in name has been removed since 4.0.
[ECharts] `[object Object]` is invalid id or name. Must be a string or number.
```

2 つ間違えていました。`name.textStyle` の階層は ECharts 4 で廃止済み。そして ECharts のコンポーネントにおける `name` は**そのコンポーネントの識別子**です。文字列か数値でなければならない。そこにオブジェクトを書いたので、ECharts は radar 全体を拒否していました。radar チャートは何も描かれていなかったわけです。

正解は `axisName` でした。

```ts
radar: {
  axisName: { ...radar.axisName, color: palette.text },
}
```

Flint が返す radar の option を見に行ったら、ちゃんと `axisName` を使っていました。私が勝手に `name` だと思っただけです。

教訓は「統合テストを書け」ではなく、もう少し具体的だと思っています。**外部ライブラリのオプション構造に対する自分の記憶は、ユニットテストでは検証できない。** テストと実装が同じ人の同じ思い込みから出ているので、two-way の検証になっていない。答え合わせの相手は本物のライブラリしかいない。

同じ統合テストで、もう 2 つ拾いました。

**CSV の区切り文字を誤検出していた。** 「全行で列数が揃う候補を選ぶ」というスコアリングにしていたら、`a,b,c\n1,2` でタブ区切りが勝ちました。タブで切ると全行が 1 列で、綺麗に揃うからです。列数が 1 以下の候補は区切り文字ではない、という条件を足して直しました。

**Markdown 表のセル分割が `[[Note|Alias]]` を割っていた。** パイプでセルを切っているので、wikilink のエイリアス区切りが巻き込まれます。`[[ ]]` の内側とインラインコードの内側では切らないようにしました。

最終的に、Flint が公開している 37 種すべてを実際の ECharts で描画してアサートするテーブル駆動のテストにしました。

```ts
for (const chartType of Object.keys(ENCODINGS)) {
  it(chartType, async () => {
    const svg = await paint(chartType);
    expect(svg).toContain('<svg');
    expect(svg.length).toBeGreaterThan(600);
  });
}
```

チャート種別ごとに受け付けるチャネルが違うので（`Pie Chart` は `x`/`y` を取らず `size`/`color`、`Gauge Chart` は `size`/`column` だけ）、この表を書く作業そのものが仕様の理解になりました。

## Obsidian 側で何を作ったか

コードブロックに書けるようにしました。中身は YAML です。

````
```flint
chartType: Bar Chart
title: 四半期売上
x: quarter
y: revenue
color: region
data:
  - { quarter: Q1, revenue: 1200, region: 関東 }
  - { quarter: Q2, revenue: 1580, region: 関東 }
  - { quarter: Q1, revenue: 800,  region: 関西 }
  - { quarter: Q2, revenue: 1120, region: 関西 }
```
````

YAML パーサは Obsidian が `parseYaml` を export しているので、バンドルサイズは増えません。YAML は JSON の上位集合なので、JSON をそのまま貼っても通ります。

`chart_spec` キーがある場合は Flint の `ChartAssemblyInput` としてそのまま読みます。Flint には MCP サーバ（`flint-chart-mcp`）があるので、AI が生成したスペックを無加工で貼れるようにしておきたかった。

### 意味型は省略できる

`semantic_types` を書かないと、列名と値から推論します。値の形が確定的なもの（`2026-08-21` → `Date`、`2026-08` → `YearMonth`、`2026Q3` → `YearQuarter`）を先に判定し、次に列名のヒント（`revenue` → `Price`、`orders` → `Count`、`share` → `Percentage`）を見て、最後に型でフォールバックします。日本語の列名（`売上` `件数` `構成比` `地域`）も引きます。

金額・件数系のヒントは**数値列にだけ**当てています。そうしないと `price_band: "high"` のような文字列列が `Price` になってしまう。

`chartType` も省略できます。Flint に `ecRecommendChartTypes()` があるので、データを渡して推薦させています。

### ノートの表がそのままグラフになる

Obsidian でしか出せない価値はここだと思っています。

````
```flint
chartType: Bar Chart
x: Region
y: Revenue
source: "[[売上メモ#Q3]]"
```
````

指定した見出しの下の Markdown 表を行に変換します。セルからは太字・斜体・インラインコード・wikilink・Markdown リンクを剥がし、`1,580` と `42%` は数値として読みます。ファイルの変更を購読しているので、表を編集するとグラフが追随します。

表は手で編集する対象のまま置いておいて、グラフはそれを見る。データの二重管理が発生しないのが気に入っています。`.csv` `.tsv` `.json` を `source:` に指定するのも同じ経路です。

### 空になるより、理由が出るほうがいい

チャート種別が受け付けないチャネルを書いたときは、黙って空のグラフを出すのではなく行番号付きで理由を出すようにしました。

```
Note  "Gauge Chart" ignores the `x` channel. It accepts: size, column.  line 3
```

この一覧は Flint の `ecAllTemplateDefs` から実行時に引いています。ハードコードしていないので、`flint-chart` を上げたら勝手に追随します。チャート種別名も同じところから取っているので、`sankey` と書けば `Sankey Diagram` に解決されるし、綴り間違いには近い候補を出せます。

## 使ってみる

[コミュニティストア](https://community.obsidian.md/plugins/flint-chart)からインストールしてください
書けるチャートは 37 種です。sankey / treemap / sunburst / tree / network graph / parallel coordinates / calendar heatmap / gantt / candlestick / bullet / gauge / radar / rose / waterfall / streamgraph / bump / slope / ECDF あたりまで含みます。

## おわりに

Flint で面白いのは、可視化を「意味の宣言」と「意図の宣言」に分けたことだと思います。AI エージェントに冗長なネイティブスペックを吐かせるより、コンパクトな中間表現を吐かせるほうが安いし、小さな編集で壊れにくい。README の動機はそこにあります。

一方で、v0.5.1 という段階なりの穴もあります。`theme_spec` がバックエンドによって効かないことは、フィールドの並びからは読み取れません。この手のものを組み込むときは、ドキュメントを読むのと同じくらい、生成物を実際にライブラリに食わせて何が起きるか見たほうが早い。自分のユニットテストが自分の思い込みを承認していただけだった、という経験を踏んだので書いておきます。

- Flint: https://github.com/microsoft/flint-chart
- 作ったプラグイン: https://github.com/yut0takagi/obsidian-flint-chart
- Obsidian Plugin: https://community.obsidian.md/account/plugins/flint-chart
