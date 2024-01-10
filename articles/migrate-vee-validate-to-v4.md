---
title: "VeeValidateをV4へアップグレード ～Fieldのカスタムコンポーネント～"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "vee-validate", "nuxt.js"]
published: true
---
今回Nuxt2からNuxt3への移行作業に携わる機会がありました。VueのバリデーションライブラリであるVeeValidateも、移行に伴って大きく仕様変更がなされています。

この記事ではバージョンアップによる移行の方針や注意事項の共有ができればと思っています。

*※記事中のコードは本記事用に記述したものですので、実際のコードとは異なります。*

## VeeValidate V3との違い
### 単体のバリデーションから打って変わり、フォームのバリデーションへ
あくまで個人の感想ですが、V3は他のフォームライブラリに比べると独特な記法でした。

ReactでポピュラーなフォームライブラリであるFormikでは、htmlの `form`に対応する `Form`コンポーネントをルートとして `<Field/>`コンポーネントが束ねられます。バリデーションスキーマはルートである `Form`で定義する感じです。VeeValidate V4においても概ね同じような構造となります。

対するV3では、フォームというひとまとまりのバリデーションというよりは、一つの `input`に対してのバリデーションを定義(`ValidationProvider`)し、ついでにフォームとしてまとめあげる機能(`ValidationObserver`)もあるという印象です。

これは複数ページにわたって必要事項を記入していく申込サイトやフォームと呼ぶほどでもない入力欄のバリデーションといったユースケースに向いていますよね。

Vue3でコンポジションAPIが導入されたこともV4のコンセプトに影響を与えたのだろうと思いますが、別物感は否めないですね・・・。

### 各種バリデーションライブラリを使用できるようになった
V3までのルールの記法に加えて、yupやzodなどサードパーティーのバリデーションライブラリが利用できるようになっています。したがって、V4からVeeValidateを使い始める場合は使い勝手がいいと思います。

V4でも従来の記法はサポートされていますので、移行に際してはGlobal Validatorの登録手順が変わった以外は大きな修正は不要です。

### 他にもいろいろ
`<ErrorMessage/>`コンポーネントが追加されました。これはFormikと同様に、対象の`<Field/>`を指定しておくことでエラーメッセージを表示してくれます。今回の記事では扱いませんが、手軽にフォームを作成できて便利ですね。

また、`<FieldArray/>`コンポーネントも追加されました。これは複数の`<Field/>`をまとめて管理できるコンテナのようなもので、Todoリストなど可変であってもバリデーションでき、リスト内のアイテムの移動、入れ替えや追加・削除もできます。

## 移行の方針
移行作業をするにあたって、機能の開発と並行して取り組む必要があったため、定期的に変更をNuxt3移行ブランチに取り込まなければなりませんでした。

したがって、V4の機能を最大限活用するというよりは、なるべくコードの変更量を最小限にして移行できるようにしています。

## 移行作業と注意点
まずは関連パッケージのインストールを行います。
```bash
npm i -D vee-validate @vee-validate/i18n @vee-validate/rules
```
### プラグイン定義
プロジェクトではNuxtを使用していますので、以下のコードはすべて`/plugins/veeValidate.ts`の以下関数内に記述しています。
```typescript
export default defineNuxtPlugin((nuxtApp) => {
  // ここに記述
})
```
#### 組み込みルール
extendをdefineRuleに書き換えればOKです。
```typescript
// before
import * as rules from 'vee-validate/dist/rules';
import { extend } from "vee-validate";

Object.keys(rules).forEach(rule => {
  extend(rule, rules[rule]);
});

// after
import { defineRule } from 'vee-validate';
import * as AllRules from '@vee-validate/rules';

Object.keys(AllRules).forEach(rule => {
  defineRule(rule, AllRules[rule]);
});
```

#### カスタムルール
組み込みルールと基本は同じですが、一部記述方法が変わっていますので、書き直す必要があります。
- バリデーションルールは関数のみになりました。
- フィールド名はコンテクストオブジェクトから取得するようになりました。
- params(Cross-Field Validationなどで使われる、 `:rules="is_not:foo"`の `foo`の部分)がオブジェクトではなく配列になりました。

[Global Validators (logaretm.com)](https://vee-validate.logaretm.com/v4/guide/global-validators/)

extendに関数を渡していた場合は簡単：
```typescript
// before
extend("phoneNumber", (value, params) => {
  return /^(050|070|080|090)\d{8}$/.test(value) ? true : `正しい携帯電話番号を入力してください。`;
})
// after
defineRule("phoneNumber", (value, params) => {
  return /^(050|070|080|090)\d{8}$/.test(value) ? true : `正しい携帯電話番号を入力してください。`;
})
```
extendにオブジェクトを渡していた場合：
```typescript
// before
extend("password-confirm", {
  params: ["password"],
  validate: (value, { password }) => {
    if(value !== password){
      return "{_field_}は同じ値を入力してください。"
    }
    return true;
  },
})
// after
defineRule<string, { password: string }>("passwordConfirm", (value, [ password ], ctx) => {
  if(value !== params[0]){
    return `${ctx.field}は同じ値を入力してください。`
  }
  return true;
})
```
`params`の型もオブジェクトから配列に変更されていますので、注意が必要です。詳しい`rules`の記法は[公式ドキュメント](https://vee-validate.logaretm.com/v4/guide/global-validators#cross-field-validation)で確認してください。

`ctx`は以下の型となっています。
```typescript
interface FieldValidationMetaInfo {
  field: string;
  name: string;
  label?: string;
  value: unknown;
  form: Record<string, unknown>;
  rule?: {
    name: string;
    params?: Record<string, unknown> | unknown[];
  };
}
```
上記の`passwordConfirm`の場合、`ctx`は以下の値が入ります。`form`はForm全体の値を保持しています。`field`と`label`の違いは分かりませんが、V3から移行する場合は`field`を見ておけばよさそうです。
```json
{
  "field": "パスワード確認",
  "name": "passwordConfirm",
  "label": "パスワード確認",
  "value": "pass0123",
  "form": {
    "password": "pass0123",
    "passwordConfirm": "pass4567"
  },
  "rule": {
    "name": "passwordConfirm",
    "params": [
      "pass0123"
    ]
  }
}
```
#### グローバルコンポーネントの登録
この記事では移行負荷を下げることを目的としていますので、コンポーネント名の差異を吸収するようにしています。どちらにしろ名称は一括置換ですぐに終わりますのでご自由に。ValidationProviderは後述しますが、カスタムコンポーネントを作成して内部でFieldを作るようにしています。
```typescript
import { Form, ErrorMessage } from "vee-validate";
import { ValidationProvider } from "~/components/ValidationProvider"

declare module '@vue/runtime-core' {
  export interface GlobalComponents {
    ValidationObserver: typeof Form
    ValidationProvider: typeof ValidationProvider
    ValidationErrorMessage: typeof ErrorMessage
  }
}

export default defineNuxtPlugin( (nuxtApp) => {
  nuxtApp.vueApp.component('ValidationObserver', Form);
  nuxtApp.vueApp.component('ValidationProvider', ValidationProvider);
  nuxtApp.vueApp.component('ValidationErrorMessage', ErrorMessage);
  // ...
```

#### ローカライズ
ローカライズは[この記事](https://tech.andpad.co.jp/entry/2022/12/05/100000#%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%81%AE%E4%BF%AE%E6%AD%A3)をそのまま利用します。

### Form(ValidationObserver)
VeeValidate V4では、コンポーネントで記述できる`<Form>`と、Vue3で登場したコンポーザブルで記述できる`useForm()`のどちらかを使うことができます。内部的には、`<Form>`も`useForm()`を使っています。 V3の`ValidationObserver`から比較するといくつか変更はありますが、いずれも大きな労力とはならないと思います。

V3でのValidationObserverは以下のようなものです：
```vue
<ValidationObserver slim v-slot="{ invalid }" />
<button :disabled="invalid">申し込む</button>
```

#### HTMLの`<form>`タグの抑止
V4の`<Form>`では、名前の通り`<form>`を描画するようになりました。V3ではデフォルトでは`<span>`タグで、`tag="form"`のようにタグを指定していました。V4では`as`プロパティを使って描画するHTMLタグを指定できます。`as=""`と指定することで、タグを描画しないようにもできます。これはV3では`slim`プロパティに該当します。

今回のプロジェクトでは複数ページにわたる申込サイトであるため、あまり好ましい変更ではありません。したがって、以下のように書き換えます。
```vue
<ValidationObserver as="" v-slot="{ invalid }" />
<button :disabled="invalid">申し込む</button>
```

#### v-slotの変更
v-slotも変更があります。`dirty`や`valid`といったフォームのメタ情報は、`meta`オブジェクトの子へ変更されました。

```vue
<ValidationObserver as="" v-slot="{ meta: { valid } }" />
<button :disabled="!valid">申し込む</button>
```

また、`invalid`は削除されました。代替は以下になります：
```vue
<ValidationObserver as="" v-slot="{ meta: { valid, validated, dirty } }" />
<span v-show="!valid && validated">フォームを入力してください。</span>
<span v-show="!valid && dirty">フォームを入力してください。</span>
```
`dirty`は値が変更されると`true`になります。申し込みが複数ページに渡るなど、フィールドに初期値がセットされる場合では`dirty`が`false`になるため、`validated`を見た方がいいです。

##### `fields`プロパティの代替
V3での`fields`オブジェクトは[公式ドキュメント](https://vee-validate.logaretm.com/v3/api/validation-observer.html#scoped-slot-props)には存在しませんが、実際にはスコープドスロットとして各フィールドの情報を取得することができていました。

V3での`fields`オブジェクトの型は以下です：
```typescript
// fields: Record<string, ObserverField>;
interface ObserverField {
  id: string;
  name: string;
  failedRules: Record<string, string>;
  pristine: boolean;
  dirty: boolean;
  touched: boolean;
  untouched: boolean;
  valid: boolean;
  invalid: boolean;
  pending: boolean;
  validated: boolean;
  changed: boolean;
  passed: boolean;
  failed: boolean;
}
```

これはフォームの無効な項目をページ下部に一覧表示する際や、記入したフィールドの数に応じてシークバーを進めたりするのに便利でしたが、V4ではこれに値する機能はなくなってしまいました。

今回は代替案として、ValidationProviderでField作成時にVuexストアにメタ情報を保存するような方策を取りました。詳細は後述します。
#### ValidationObserverのref
ValidationObserverのrefも上記の変更を受けていますので注意が必要です。

validやdirtyなどフォームのmeta情報やフォームのリセット等は、V4より追加された[コンポーザブル](https://vee-validate.logaretm.com/v4/api/composition-helpers/)を利用できます。
```typescript
import { useIsFormValid } from 'vee-validate';
const isValid = useIsFormValid();
isValid.value; // true or false
```
注意点として、上記のコンポーザブルは`ValidationObserver`を呼び出しているコンポーネント、およびそれより先祖のコンポーネントでは呼び出せません。その場合は`ValidationObserver`の`ref`からアクセスするしかなさそうです。

### Field(ValidationProvider)
`<Field/>`でもコンポーネントと`useField()`コンポーザブルのどちらかを利用できます。
#### 公式のFieldコンポーネントを使うパターン
まずは公式で用意されている`<Field/>`をそのまま使うケースについて説明します。

V3
https://vee-validate.logaretm.com/v3/api/validation-provider.html#scoped-slot-props
V4
https://vee-validate.logaretm.com/v4/api/field/

主な変更は以下です：
- rulesの指定が不要な場合、nullの代わりに空文字を指定する必要があります。
- ValidationObserverと同様、`slim`, `tag`は`as`へ変更になりました。
- vidはnameに変更されました。
- nameはlabelに変更されました。
- **modeは廃止されました。**

特にmodeことインタラクションモードがV4では廃止されたことが大きな変更となります。この挙動を再現する手法は[ドキュメント](https://vee-validate.logaretm.com/v4/examples/dynamic-validation-triggers/)に例がありますが、`<Field/>`では再現ができません。必ず`useField()`を使う必要があります。

次のセクションでは上のドキュメントを拡張したカスタムコンポーネントを作成していきます。

#### useFieldを用いたカスタムコンポーネント
動作例やコードについてはCodeSandboxを用意しました。
https://codesandbox.io/p/github/yuyuyukie/vee-validate-v4-sample/master?layout=%257B%2522sidebarPanel%2522%253A%2522EXPLORER%2522%252C%2522rootPanelGroup%2522%253A%257B%2522direction%2522%253A%2522horizontal%2522%252C%2522contentType%2522%253A%2522UNKNOWN%2522%252C%2522type%2522%253A%2522PANEL_GROUP%2522%252C%2522id%2522%253A%2522ROOT_LAYOUT%2522%252C%2522panels%2522%253A%255B%257B%2522type%2522%253A%2522PANEL_GROUP%2522%252C%2522contentType%2522%253A%2522UNKNOWN%2522%252C%2522direction%2522%253A%2522vertical%2522%252C%2522id%2522%253A%2522clr7tdria00063j818l2o1bir%2522%252C%2522sizes%2522%253A%255B70%252C30%255D%252C%2522panels%2522%253A%255B%257B%2522type%2522%253A%2522PANEL_GROUP%2522%252C%2522contentType%2522%253A%2522EDITOR%2522%252C%2522direction%2522%253A%2522horizontal%2522%252C%2522id%2522%253A%2522EDITOR%2522%252C%2522panels%2522%253A%255B%257B%2522type%2522%253A%2522PANEL%2522%252C%2522contentType%2522%253A%2522EDITOR%2522%252C%2522id%2522%253A%2522clr7tdria00023j8109rswxwn%2522%257D%255D%257D%252C%257B%2522type%2522%253A%2522PANEL_GROUP%2522%252C%2522contentType%2522%253A%2522SHELLS%2522%252C%2522direction%2522%253A%2522horizontal%2522%252C%2522id%2522%253A%2522SHELLS%2522%252C%2522panels%2522%253A%255B%257B%2522type%2522%253A%2522PANEL%2522%252C%2522contentType%2522%253A%2522SHELLS%2522%252C%2522id%2522%253A%2522clr7tdria00043j81php75fv4%2522%257D%255D%252C%2522sizes%2522%253A%255B100%255D%257D%255D%257D%252C%257B%2522type%2522%253A%2522PANEL_GROUP%2522%252C%2522contentType%2522%253A%2522DEVTOOLS%2522%252C%2522direction%2522%253A%2522vertical%2522%252C%2522id%2522%253A%2522DEVTOOLS%2522%252C%2522panels%2522%253A%255B%257B%2522type%2522%253A%2522PANEL%2522%252C%2522contentType%2522%253A%2522DEVTOOLS%2522%252C%2522id%2522%253A%2522clr7tdria00053j818ca856vz%2522%257D%255D%252C%2522sizes%2522%253A%255B100%255D%257D%255D%252C%2522sizes%2522%253A%255B50%252C50%255D%257D%252C%2522tabbedPanels%2522%253A%257B%2522clr7tdria00023j8109rswxwn%2522%253A%257B%2522id%2522%253A%2522clr7tdria00023j8109rswxwn%2522%252C%2522activeTabId%2522%253A%2522clr7tkdpy00t83j81zvncqzj5%2522%252C%2522tabs%2522%253A%255B%257B%2522id%2522%253A%2522clr7tdria00013j81dwy603dt%2522%252C%2522mode%2522%253A%2522permanent%2522%252C%2522type%2522%253A%2522FILE%2522%252C%2522filepath%2522%253A%2522%252FREADME.md%2522%257D%252C%257B%2522type%2522%253A%2522FILE%2522%252C%2522filepath%2522%253A%2522%252F.codesandbox%252Ftasks.json%2522%252C%2522id%2522%253A%2522clr7tkdpy00t83j81zvncqzj5%2522%252C%2522mode%2522%253A%2522permanent%2522%257D%255D%257D%252C%2522clr7tdria00053j818ca856vz%2522%253A%257B%2522id%2522%253A%2522clr7tdria00053j818ca856vz%2522%252C%2522activeTabId%2522%253A%2522clr7tk4j400qk3j81uplfw9ok%2522%252C%2522tabs%2522%253A%255B%257B%2522type%2522%253A%2522TASK_PORT%2522%252C%2522taskId%2522%253A%2522dev%2522%252C%2522port%2522%253A3000%252C%2522id%2522%253A%2522clr7tk4j400qk3j81uplfw9ok%2522%252C%2522mode%2522%253A%2522permanent%2522%252C%2522path%2522%253A%2522%252F%2522%257D%255D%257D%252C%2522clr7tdria00043j81php75fv4%2522%253A%257B%2522id%2522%253A%2522clr7tdria00043j81php75fv4%2522%252C%2522activeTabId%2522%253A%2522clr7tdtpz00573j81amrzj8un%2522%252C%2522tabs%2522%253A%255B%257B%2522id%2522%253A%2522clr7tdria00033j81vbi1n82m%2522%252C%2522mode%2522%253A%2522permanent%2522%252C%2522type%2522%253A%2522TERMINAL%2522%252C%2522shellId%2522%253A%2522clr7tdsm9000regh7c8jc9myw%2522%257D%252C%257B%2522type%2522%253A%2522TASK_LOG%2522%252C%2522taskId%2522%253A%2522dev%2522%252C%2522id%2522%253A%2522clr7tdtpz00573j81amrzj8un%2522%252C%2522mode%2522%253A%2522permanent%2522%257D%252C%257B%2522type%2522%253A%2522TASK_LOG%2522%252C%2522taskId%2522%253A%2522CSB_RUN_OUTSIDE_CONTAINER%253D1%2520devcontainer%2520templates%2520apply%2520--template-id%2520%255C%2522ghcr.io%252Fdevcontainers%252Ftemplates%252Ftypescript-node%255C%2522%2520--template-args%2520%27%257B%257D%27%2520--features%2520%27%255B%255D%27%2522%252C%2522id%2522%253A%2522clr7tevyo00883j81dvi3l6tm%2522%252C%2522mode%2522%253A%2522permanent%2522%257D%255D%257D%257D%252C%2522showDevtools%2522%253Atrue%252C%2522showShells%2522%253Atrue%252C%2522showSidebar%2522%253Atrue%252C%2522sidebarPanelSize%2522%253A15%257D

以下は概要です。
- ドキュメントのコード例は`input`を内包したカスタムフィールドですが、より汎用的に利用したいため、`<slot/>`に置き換えました。
- `<ValidationProvider/>`の呼び出し元では`v-slot`で`handlers`を取り出して`<input/>`のイベントを購読しています。
  - `<Field/>`を使う場合と同様に値の変更は`<ValidationProvider/>`で行います。
  - `<input/>`に`v-model`で渡してしまうと、値を変更した際に二重で`password`が更新されてしまいます。`:value`を渡してください。
- `vid`, `name`はV3との互換性を維持しています。
- ValidationProviderのrefからは`useField()`の返り値を取得できるようにしています。
- 各modeのバリデーションイベントのハンドリング処理は記事のものを一部変更して、blurイベントもバリデーションの対象にしています。
- `<ValidationProvider/>`内の`useField`のオプションで`syncVModel: true`を指定することで、自動で変更をemitしてくれます。`defineEmits`も忘れずに。
  - mode=`validateOnUpdate`は`<input/>`以外から値を変更する場合に必要になります。本来は`useField`の`validateOnValueUpdate: true`とするだけで動作するはずなのですが、動作しないのでwatchを追加しています。
- ValidationProviderのref、v-slotからFieldの情報にアクセスできるようにするためにuseFieldの返り値をすべて`defineExpose()`, `<slot />`に渡しています。
- ページの下部にバリデーションエラーの項目をまとめて表示するために`Pinia`でvalidかどうかや日本語名を保持するようにしています。これによって`<ValidationObserver/>`の`fields`の代替の役割を果たします。

#### Cleave.jsとの併用
Cleave.jsを使っている場合は、`<ValidationProvider/>`で値を変更するように変わっているため、結構手が込んだ対応が必要となります。

ポイントとしては、
- `<ValidationProvider/>`でCleave.jsのインスタンスを持ち、Cleave.jsの`onValueChange`を購読して生の値で更新・バリデーションします。
- `<ValidationProvider/>`からv-slotでCleave.jsのインスタンス登録用の関数をエクスポートし、Cleave.jsインスタンスを呼び出します。

また時間があればサンプルを更新したい・・・

## 引用・参考
https://tech.andpad.co.jp/entry/2022/12/05/100000
