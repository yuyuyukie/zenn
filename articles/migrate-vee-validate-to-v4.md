---
title: "VeeValidateをV4へアップグレードする際のあれこれ"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue"]
published: false
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
// TODO validatedの仕様を把握
`dirty`は値が変更されると`true`になります。`validated

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

今回は代替案として、ValidationProviderでField作成時にVuexストアにメタ情報を保存するような方策を取りました。詳細は[ValidationObserverのfieldsの代替](#validationobserverのfieldsの代替)に記載しています。
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
ドキュメントのコード例は`input`を内包したコンポーネントですが、既存コードを流用したいため、`<slot/>`に置き換えてみましょう。
```vue
// app.vue
<script>
const providerRef = ref<InstanceType<typeof ValidationProvider>>();
const password = ref("");
</script>
<template>
  <ValidationProvider 
    ref="providerRef"
    mode="eager"
    v-model="password" 
    v-slot="{ handlers }"
    vid="password"
    name="パスワード"
  >
    <input :value="password" v-on="handlers" />
  </ValidationProvider>
  <ValidationProvider 
    ref="providerRef"
    mode="eager"
    v-model="password" 
    v-slot="{ handlers }"
    vid="password"
    name="パスワード"
  >
    <input type="checkbox" :checked="password" v-on="handlers" />
  </ValidationProvider>
</template>
```
- `v-slot`で`handlers`を取り出して`<input/>`のイベントを購読しています。
  - `<Field/>`を使う場合と同様に値の変更は`<ValidationProvider/>`で行います。
  - `<input/>`に`v-model`で渡してしまうと、値を変更した際に二重で`password`が更新されてしまいます。`:value`を渡してください。
- `vid`, `name`はV3との互換性を維持しています。
- ValidationProviderのrefからは`useField()`の返り値を取得できるようにしています。
```typescript
// interactionModes.ts
import { FieldContext } from 'vee-validate';

type InteractionEventGetter = (ctx: FieldContext) => string[];

// Validates on submit only
const passive: InteractionEventGetter = () => [];
const lazy: InteractionEventGetter = () => ["change"];
const aggressive: InteractionEventGetter = () => ["input", "blur"];
const eager: InteractionEventGetter = (errorMessage) =>
        errorMessage ? ["input"] : ["change", "blur"];

export const modes = {
  passive,
  lazy,
  aggressive,
  eager,
};
```
```vue
// ValidationProvider.vue
<script setup lang="ts">
import { computed, toRef } from 'vue';
import { useField } from 'vee-validate';
import { modes } from '../interactionModes';

const props = withDefaults(
  defineProps<{
    modelValue: string;
    vid: string;
    name: string;
    mode?: Mode;
    type?: InputType
  }>(),
  {
    mode: 'aggressive',
  }
);

const emit = defineEmits<{
  (e: "update:modelValue", value: string): void
}>();

// use `toRef` to create reactive references to `name` prop which is passed to `useField`
// this is important because vee-validte needs to know if the field name changes
// https://vee-validate.logaretm.com/v4/guide/composition-api/caveats
const { meta, value, errorMessage, handleChange, handleBlur } = useField(
  toRef(props, 'vid'),
  null,
  {
    label: props.name,
    initialValue: props.modelValue,
    type: props.type,
    validateOnValueUpdate: props.mode === "validateOnUpdate" || props.type === "checkbox",
    syncVModel: true,
  }
);

// generates the listeners
const handlers = computed(() => {
  const on = {
    blur: handleBlur,
    // default input event to sync the value
    // the `false` here prevents validation
    input: [(e) => handleChange(e, false)],
  };

  // Get list of validation events based on the current mode
  const triggers = modes[props.mode]({
    errorMessage,
    meta,
  });

  // add them to the "on" handlers object
  triggers.forEach((t) => {
    if (Array.isArray(on[t])) {
      on[t].push(handleChange);
    } else {
      on[t] = handleChange;
    }
  });

  return on;
});
</script>
<template>
  <slot
    :handlers="handlers"
  />
</template>
```

- ValidationProviderでv-modelを定義し、ValidationProviderから上に値を更新する。
  - このとき、useFieldのオプションで`syncVModel: true`を指定すると、自動でemitしてくれます。 

#### ValidationObserverのfieldsの代替
#### Cleave.jsとの併用
今回のプロジェクトでは、ページの下部にバリデーションエラーの項目をまとめて表示する必要があり、そのためのカスタムコンポーネントがあります。
#### interactionModeの再現

## 引用・参考
https://tech.andpad.co.jp/entry/2022/12/05/100000
