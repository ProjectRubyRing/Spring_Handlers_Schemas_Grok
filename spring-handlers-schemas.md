# spring.handlers と spring.schemas

調査日: 2026-10-22

この文書は、Spring の XML 設定で使う二つのファイルが何か、どこに置くと Spring が見つけるか、WAR の外側の `META-INF` に置く意味があるか、独自タグは認識されるか、Spring Framework 6.2.9 で必要か、`mvc.xml` が無いと壊れるか、版によって何が変わるかを説明します。

最初の章は、小学校の図書館にたとえた短い話です。その後の章は、Spring Framework `6.2.9`（2025-07-17 公開）のソースと、その後の `7.0.9`（2026-08-20 公開）まで照らした詳しい説明です。

この作業フォルダには、対象の WAR や `mvc.xml` の実物はありません。説明は、Spring 本体の決まりごとについてのものです。

---

## 1. 先に、質問への答え

| 質問 | 答え |
| --- | --- |
| 二つのファイルは何か | `spring.handlers` は「この名前空間のタグは、どの Java クラスが解釈するか」の当番表。`spring.schemas` は「この XSD の住所は、インターネットではなく JAR の中のどのファイルか」の本棚地図。 |
| Spring が見る場所 | クラスパス上の `META-INF/spring.handlers` と `META-INF/spring.schemas`。クラスローダの `getResources` で、同名ファイルを全部集める。 |
| 正しい置き場所 | 公式分は、各 OSS の JAR の中。自作分は `WEB-INF/classes/META-INF/`（Maven / Gradle なら `src/main/resources/META-INF/`）、または `WEB-INF/lib` の自作 JAR の中。 |
| WAR 直下の `META-INF` に置く意味 | XML 名前空間の仕組みとしては無い。サーブレットのクラスローダはそこを見ない。独自タグも、そこへ置いただけでは認識されない。 |
| 独自タグは作れるか | 作れる。ただし置く場所はクラスパス上の `META-INF`。2006 年の Spring 2.0 からある、ライブラリ作者向けの拡張口。 |
| Spring 6.2.9 で必要か | 仕組み自体は 6.2.9 でも生きている。普通の `mvc.xml` なら、spring-webmvc などの JAR の中に既にあるので、アプリが自分でファイルを足す必要はない。 |
| `mvc.xml` を使っていて、アプリ側にファイルが無いと壊れるか | 壊れない。壊れるのは、JAR の中のファイルが無い、JAR 自体がクラスパスに無い、uber-JAR 化でファイルが上書きされた、名前空間 URI を `https` にしてしまった、といった別のとき。 |
| 版で変わるか | ファイルの形式と探す場所は 2.0 から 7.0.9 まで同じ。変わるのは、どの JAR がどの名前空間を持つか、XSD の版番号の付け方、`https` の schemaLocation の扱い、7.0 で `<mvc:*>` が非推奨警告になったこと。 |

6.2 系の OSS サポートは 2026-06-30 に終了しています。2026-10-22 時点の 6.2 系の最終公開版は 6.2.19（2026-06-08）です。現在の開発本線は 7.0 系で、公開されている新しい版は 7.0.9 です。`spring.handlers` の読み方は 6.2.9 と 6.2.19 と 7.0.9 で同じです。7.0 で加わった差は、`<mvc:*>` を使ったときに警告ログが出ることです。タグ自体はまだ動きます。

---

## 2. 小学校の図書館でいうと

給食当番の表を、コンピュータの設定ファイルだと思ってください。

`mvc.xml` は、給食の献立です。「普通のカレーを出す」「MVC の準備スタンプを押す」と書いてあります。

普通のカレーは `<bean>` です。調理場の人は、最初からこの言葉を知っています。当番表を調べなくても作れます。

`<mvc:annotation-driven>` や `<context:component-scan>` は、特別なスタンプです。調理場の人は、スタンプの絵を見ただけでは何をすればよいか分かりません。職員室の当番表を開きます。

`spring.handlers` が、その当番表です。

「MVC というスタンプが来たら、`MvcNamespaceHandler` さんを呼ぶ」

と書いてあります。呼ばれた人が、スタンプを本物の調理手順（Spring の Bean 定義）に翻訳します。

`spring.schemas` は、別の紙です。こちらはルールブックの置き場所です。

献立の欄外に「ルールブックは `https://www.springframework.org/schema/mvc/spring-mvc.xsd` にあります」と書いてあっても、学校の調理場はインターネットの向こうの図書館へ取りに行きません。本棚地図を見て、自分たちの本箱の中の `spring-mvc.xsd` を開きます。ネットが止まっていても、本の版が JAR とずれていても、同じルールブックを使えます。

本は JAR です。出版社（Spring の開発チーム）は、本の奥付に当番表と本棚地図を刷って出荷します。だから、普通の給食では、学校が当番表を書き足す必要はありません。MVC の本（`spring-webmvc`）を本棚に入れてあれば足ります。

WAR は、学校に届く荷物かばんです。

かばんの外側のポケットが、WAR 直下の `META-INF` です。ここに当番表を入れても、調理場の人は見ません。見るのは、かばんの中の決まった棚だけです。

- `WEB-INF/classes`
- `WEB-INF/lib` の中の JAR

Maven で `src/main/resources/META-INF/spring.handlers` に置くと、出来上がった WAR では `WEB-INF/classes/META-INF/spring.handlers` になります。この棚は見えます。

`src/main/webapp/META-INF/spring.handlers` に置くと、WAR の外側のポケットに入ります。この棚は見えません。独自のスタンプをそこに入れても、Spring は「担当者がいません」と言います。

自分たちの特別スタンプを作りたいときだけ、当番表に一行足します。足す場所は、見える棚です。外側のポケットではありません。

---

## 3. 二つのファイルは、別の仕事をする

Spring が XML を読むとき、仕事は二つあります。

1. 文法が合っているかを調べる（XSD 検証）
2. タグの意味を Bean 定義に翻訳する（名前空間ハンドラ）

| | spring.schemas | spring.handlers |
| --- | --- | --- |
| 仕事 | 1 の文法書を、JAR の中から出す | 2 の翻訳者を見つける |
| 読むクラス（6.2.9） | `PluggableSchemaResolver` | `DefaultNamespaceHandlerResolver` |
| 探すパス | `META-INF/spring.schemas` | `META-INF/spring.handlers` |
| Javadoc の `@since` | 2.0 | 2.0 |
| 1 行の意味 | `スキーマの URL = JAR の中の XSD のパス` | `名前空間の URI = ハンドラクラスの完全名` |
| 無いと、まず何が起きるか | 検証がオンのとき、XML パーサが URL を取りに行く。届かなければ起動に失敗する | `<mvc:*>` のようなタグを翻訳できず、起動に失敗する |
| `<bean>` だけなら | `schemaLocation` があれば検証で使う | 使わない。`<bean>` はプログラムに直接書いてある |
| 検証をオフにしたら | ほぼ使われない | それでも必要。翻訳は検証の有無と別 |

形は、どちらも Java のプロパティファイルです。URL の `:` は、プロパティの区切り文字とぶつかるので、`\:` と書きます。区切りは `=` です。

```properties
http\://www.springframework.org/schema/mvc=org.springframework.web.servlet.config.MvcNamespaceHandler
```

```properties
https\://www.springframework.org/schema/mvc/spring-mvc.xsd=org/springframework/web/servlet/config/spring-mvc.xsd
```

左辺が XML に書く文字列、右辺がクラスパスの中の実体です。

読み込みは `PropertiesLoaderUtils.loadAllProperties` です。6.2.9 の Javadoc には、次のことが書いてあります。

- 文字コードは ISO-8859-1
- クラスパス上に同名ファイルが何個あっても、全部を一つにマージする
- 使うクラスローダを省略すると、スレッドのコンテキストクラスローダを使う

行の中身は ASCII で書きます。日本語のコメントを UTF-8 のまま入れると、この読み方では文字化けします。マッピング行自体は ASCII なので動きやすいですが、コメントも ASCII にしておくのが安全です。

同じキーが複数のファイルにあると、`getResources` が返した順番の後ろの値が残ります。どの JAR が後ろになるかは、クラスローダの実装に依存し、Servlet 仕様は固定していません。公式の行をアプリ側で書き換えて上書きしようとしないでください。足すなら、世界で自分だけが使う URI だけを書きます。

---

## 4. なぜ生まれたか

### 4.1 Spring 1 の時代

Spring 1（2004 年）の XML は、DTD という古い文法書を使う `<bean>` の繰り返しでした。新しいタグを足すには、Spring 本体の読み取りプログラムを変える必要がありました。他の会社が「自分たちの短いタグ」を安全に足す口がありませんでした。

文法書は `spring-beans.dtd` です。この時代に `spring.handlers` も `spring.schemas` もありません。

### 4.2 2005 年末から 2006 年、Spring 2.0

Spring 2.0 の「What's new」には、2005 年 12 月の Spring Experience で 2.0 が発表されたと書いてあります。そこで XML Schema に基づく短いタグと、Extensible XML authoring（拡張できる XML）が目玉になりました。

狙いは二つです。

- 利用者が `<bean class="とても長いクラス名">` を何十行も書かずに、`<tx:annotation-driven/>` のように意図を一行で書けるようにする
- その短いタグを、Spring 本体だけでなく、他のライブラリも追加できるようにする

追加する人が、Spring 本体の一つのファイルを取り合うと、JAR を結合した瞬間に誰かの行が消えます。そこで Spring は「同じ名前のファイルを、各 JAR が自分の分だけ持つ。起動時に全部集める」にしました。

集めるファイルが二つに分かれた理由も、仕事が違うからです。

- 翻訳者の登録は `spring.handlers`
- 文法書のありかは `spring.schemas`

`spring.schemas` が必要になった直接の理由は、本番サーバがインターネットに出られないことです。XSD の `schemaLocation` は URL です。何もしないと XML パーサはその URL を取りに行きます。取りに行けない、遅い、途中の HTML エラーページを XSD と勘違いする、公開サイトの XSD が今使っている JAR より新しい、という事故を避けるために、URL を JAR 内のファイルへ差し替えます。

2.0 のドキュメントは、この仕組みの主な作者として、アプリ開発者全員ではなく、サードパーティのフレームワークや製品の作者を想定しています。アプリ開発者の多くは、用意されたタグを使う側です。

### 4.3 2007 年、Spring 2.5

`context` 名前空間が加わりました。`<context:component-scan>` や `<context:annotation-config>` です。登録先は `spring-context` の JAR の中の `spring.handlers` です。アプリの WAR にファイルを足す話ではありません。

### 4.4 2009 年、Spring 3.0 で「ファイルが消える」事故が有名になった

Spring 3.0 で、一つの `spring.jar` が `spring-beans`、`spring-context`、`spring-webmvc` のように分割されました。同じ `META-INF/spring.handlers` が、複数の JAR に存在するようになりました。

クラスパスに JAR が別々に並んでいる通常の WAR では、Spring が全部をマージするので問題ありません。

問題は、全部の JAR を一つの JAR に押し込むツールです。同じパスのファイルは一つしか残せず、最後にコピーされた JAR の `spring.handlers` だけが生き残ります。すると、クラスは全部入っているのに、MVC の当番表だけが無い、という状態になります。

2011 年の SPR-8368（GitHub issue 13015）は、この現象を報告しています。Spring 側の結論は「バグではない。複数ファイルをマージするのがこの設計である」でした。回避は、結合時に中身を連結することです。

Spring 3.0 では、もう一つの道も開きました。`@Configuration` です。Java で Bean を定義する道です。XML の独自タグをアプリが発明する必要性は、ここから下がっていきます。XML の名前空間そのものは、その後も残りました。

`<mvc:*>` も 3.0 です。`MvcNamespaceHandler` の `@since` は 3.0 です。Spring MVC を XML で短く書くために、`spring-webmvc.jar` の中へ当番表が入りました。

### 4.5 2017 年以降、版番号つき XSD を増やさなくなった

6.2.9 の `spring-webmvc` の `spring.schemas` には、次の URL だけが載っています。

- `spring-mvc-3.0.xsd` から `spring-mvc-4.3.xsd` まで
- 版番号の無い `spring-mvc.xsd`
- それぞれ `http` と `https` の両方

`spring-mvc-5.0.xsd` も `spring-mvc-6.2.xsd` も、行がありません。右辺は、どれも同じ現在の `spring-mvc.xsd` を指します。古い番号で書いてある XML も、今の JAR の文法書で検証します。

したがって、`mvc.xml` の `schemaLocation` は版番号なしが適切です。

```xml
http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd
```

`spring-mvc-6.2.xsd` と書くと、6.2.9 の地図に無いので、ローカルの XSD に当たりません。

`spring-beans` の `spring.schemas` も同じ方針です。`spring-beans-2.0.xsd` から `spring-beans-4.3.xsd` までと、版番号なし。5 や 6 の版番号つきファイル名はありません。

### 4.6 2019 年、https の schemaLocation

2019-04 の issue 22739 で、XSD の import が `https` になり、古い `spring.schemas` が `http` しか持たないため IDE が失敗する、という報告がありました。Juergen Hoeller 氏は、実行時は `https` を正規の `http` の項目へ読み替える、と説明しています。

2019-05-07 のコミット「Introduce HTTPS mappings in spring.schemas files」（gh-22903 のバックポート）で、地図そのものにも `https` の行が追加されました。

6.2.9 の `PluggableSchemaResolver` には、両方が残っています。

1. まず、渡された URL を地図から引く
2. 見つからず、URL が `https:` で始まるなら、`http:` に変えてもう一度引く
3. それでも無ければ `null` を返し、XML パーサの標準の取得に任せる

この読み替えは、文法書の URL（`schemaLocation`）だけの話です。名前空間そのものの URI には適用されません。

`spring.handlers` のキーは、6.2.9 でも 7.0.9 でも `http://www.springframework.org/schema/...` です。2019 年のコミットメッセージにも「名前空間ハンドラは、https を使わない正規名で引く」とあります。

XML では、次のように分けます。

- `xmlns:mvc` の値は `http://www.springframework.org/schema/mvc` のまま
- `xsi:schemaLocation` のファイル URL は `https://.../spring-mvc.xsd` でよい

`xmlns:mvc` を `https://www.springframework.org/schema/mvc` にすると、当番表のキーと一致せず、6.2.9 でも起動に失敗します。

### 4.7 2022 年以降、Spring 6 と 7

Spring 6.0（2022-11）は Java 17 と Jakarta EE 9 が基準になりました。パッケージは `javax.*` から `jakarta.*` へ変わりました。XML の名前空間の URI と、二つのファイルの形式は変わっていません。変わったのは、ハンドラの Java コードが参照する API です。

endoflife.date の記載では 6.2.0 は 2024-11-30、6.2.9 は 2025-07-17 です。6.2.9 のリリースノートに、この二つのファイルの仕様変更はありません。6.2.9 の `MvcNamespaceHandler.init()` に非推奨の警告もありません。

Spring 7.0（2025-11-30）で、`<mvc:*>` だけが非推奨になりました。issue 34063 と 7.0 のリリースノートに、次の方針が書いてあります。

- `<mvc:annotation-driven>` や `<mvc:resources>` は、Java 設定と Spring Boot の自動設定に比べて遅れている
- 7.0 で非推奨にする
- 削除の予定は、その時点ではない
- `<bean>` など他の名前空間は非推奨にしない

7.0.9 の `MvcNamespaceHandler.init()` は、初期化のときに次の警告を出してから、従来どおりパーサを登録します。

```text
The XML configuration namespace for Spring MVC is deprecated, please use Java configuration instead.
```

7.0.9 の `spring-beans` の `spring.handlers` は、6.2.9 と同じ 3 行（`c`、`p`、`util`）です。探すクラスも、Javadoc の説明も、7.0.9 の API ドキュメントに残っています。2026-10-16 時点の参照ドキュメント「XML Schema Authoring」も、二つのファイルの書き方を説明したままです。

---

## 5. 起動するとき、Spring は何を見るか

`mvc.xml` は、たいてい `web.xml` の `contextConfigLocation` で `/WEB-INF/mvc.xml` のように指定します。この XML の置き場所は、サーブレットコンテキストの資源です。クラスパスである必要はありません。`/WEB-INF/mvc.xml` は、クラスローダからは見えなくても、Spring の `XmlWebApplicationContext` はサーブレットコンテキスト経由で読めます。

当番表と本棚地図は、別ルートです。XML を開き始めたあと、クラスローダだけを使って探します。`mvc.xml` の隣にファイルを置いても、その隣がクラスパスでなければ無関係です。

処理の順番は、おおむね次のとおりです。

1. XML パーサが文書を読む
2. `xsi:schemaLocation` が出てくると、`PluggableSchemaResolver` が `META-INF/spring.schemas` を全部マージして URL を引く
3. 当たれば、JAR 内の XSD で検証する
4. 当たらなければ `null` を返し、パーサが URL へ取りに行く。取れなければ、検証エラーで起動に失敗する
5. 検証を通過した DOM を、Spring が要素ごとに歩く
6. 名前空間が `http://www.springframework.org/schema/beans` か、名前空間が空なら、`<bean>`、`<import>`、`<alias>` としてプログラムが直接解釈する。`spring.handlers` は見ない
7. それ以外の要素は `parseCustomElement` に入る。`spring.handlers` を全部マージした地図から URI を引く
8. ハンドラが無いと、要素の位置に次のエラーを出す

```text
Unable to locate Spring NamespaceHandler for XML schema namespace [ここにURI]
```

9. ハンドラがあれば、そのクラスを生成して `init()` を一度呼び、要素名に対応する `BeanDefinitionParser` へ渡す
10. クラス名が書いてあるのにクラスが無い場合は、`Could not find NamespaceHandler class [...]` です

属性の場合は、要素と少し違います。`<bean p:name="太郎">` のような装飾は `decorateIfRequired` です。

- ハンドラがあれば、その属性を Bean 定義に反映する
- ハンドラが無く、URI が `http://www.springframework.org/schema/` で始まるなら、上と同じエラー
- ハンドラが無く、URI が Spring 公式の外なら、debug ログだけで無視する。要素として独立している独自タグは、この「無視」には入りません。独立要素は必ずエラーです

`p` と `c` は属性用の名前空間です。要素ではありません。それでも `spring-beans.jar` の `spring.handlers` に登録されています。この JAR の当番表が欠けていると、`<bean>` 本体は動いても `p:` で失敗します。

検証をオフにしても、手順 7 以降は動きます。文法チェックをやめても、翻訳者は必要です。逆に、XSD が正しくても、当番表が無ければ翻訳で失敗します。IDE が緑色でも起動に失敗する、起動は成功しても IDE が赤線を引く、というずれは、この二つの仕事が別だから起きます。

クラスローダは、通常はウェブアプリのクラスローダです。`DefaultNamespaceHandlerResolver` は、明示されなければコンテキストクラスローダを使います。起動処理の途中では、これがウェブアプリのクラスローダになっているのが普通です。別スレッドで XML を読み、そのスレッドのコンテキストクラスローダがコンテナ側のままだと、WAR の中にファイルがあっても見つかりません。これは置き場所の問題ではなく、どのクラスローダで探しているかの問題です。

---

## 6. 正しい置き場所

Servlet 仕様がクラスローダに見せる場所は、次の二つです。

- `/WEB-INF/classes/`
- `/WEB-INF/lib/` の JAR

クラスは `WEB-INF/classes` を先に探します。資源名 `META-INF/spring.handlers` を全部列挙する `getResources` の順番までは、仕様は固定していません。Tomcat の通常設定では、ウェブアプリの中を先に、親の共通クラスローダを後に見ます。

WAR 直下の `META-INF` は、マニフェストや、Tomcat が直接読む `context.xml` のための場所です。クラスローダの資源一覧には入りません。

| ソース上の場所 | WAR の中の場所 | Spring は見るか |
| --- | --- | --- |
| `src/main/resources/META-INF/spring.handlers` | `WEB-INF/classes/META-INF/spring.handlers` | 見る |
| ライブラリ JAR の `META-INF/spring.handlers` | `WEB-INF/lib/そのJAR.jar` の中 | 見る |
| `src/main/webapp/META-INF/spring.handlers` | `WAR/META-INF/spring.handlers` | 見ない |
| `src/main/webapp/WEB-INF/META-INF/spring.handlers` | `WEB-INF/META-INF/spring.handlers` | 見ない |
| `mvc.xml` と同じ `WEB-INF/spring.handlers` | `WEB-INF/spring.handlers` | 見ない |

Gradle も同じです。`processResources` の出力が `WEB-INF/classes` に入り、`src/main/webapp` が WAR のルートに入ります。

Spring Boot の実行可能 WAR / JAR では、見える場所が `BOOT-INF/classes` と `BOOT-INF/lib` の中の JAR に変わります。`LaunchedURLClassLoader` が、内側の各 JAR の `META-INF/spring.handlers` を列挙します。外側の `META-INF` にコピーする必要はありません。Boot は JAR を一つに押し潰さないので、shade のような「最後の1ファイルだけ残る」事故は通常起きません。

IDE（IntelliJ IDEA の Spring プラグインや、古い Spring Tools）も、プロジェクトのクラスパスから同じファイルを読み、補完と検証に使います。`src/main/webapp/META-INF` は、Maven プロジェクトではクラスパスではありません。IDE も実行時も、そこは見ません。

隣にある `META-INF/spring.tooling` は、第三のファイルです。6.2.9 の `spring-beans` にも入っています。内容は名前空間の表示名やアイコンで、IDE 向けです。`DefaultNamespaceHandlerResolver` も `PluggableSchemaResolver` も、このファイルは読みません。実行時の成否には関係しません。`META-INF/spring.factories` や `META-INF/spring/*.imports` は、Spring Boot の自動設定用で、XML 名前空間の当番表ではありません。

---

## 7. WAR 直下の META-INF に置くことには、意味があるか

XML の名前空間については、意味がありません。

公式 JAR の中身を、WAR の外側の `META-INF` へコピーしても、Spring は読みません。独自タグの行をそこに書いても、認識されません。欠けた公式の当番表の代わりにもなりません。

外側の `META-INF` に意味があるのは、別の仕様です。例は次のとおりです。

- `META-INF/MANIFEST.MF`
- Tomcat が配布物から直接読む `META-INF/context.xml`
- 一部のアプリサーバが配布物から直接読むデプロイ記述子

これらはクラスローダの `getResources("META-INF/spring.handlers")` とは別の読み方です。「META-INF はコンテナが見る場所」という記憶で `spring.handlers` を置くと、期待と実装がずれます。

アプリのクラスパスに、自分だけの URI の行を足すことには意味があります。その場所は `WEB-INF/classes/META-INF` か、`WEB-INF/lib` の JAR です。WAR 直下ではありません。

2019 年の issue 22739 で、Juergen Hoeller 氏は「アプリのクラスパスに、足りない `https` の行だけを書いた `spring.schemas` を置けば、実行時も STS もそれを拾う」と説明しています。これは、クラスパスに置く場合の話です。6.2.9 は公式 JAR が `https` の行と、`https` から `http` への読み替えの両方を持っているので、公式スキーマのためだけにアプリへ地図を足す必要はありません。

---

## 8. 独自タグは、いつ認識されるか

認識される条件は、次の五つが同時に揃うことです。

1. XSD がある
2. `NamespaceHandler` があり、要素ごとの `BeanDefinitionParser` を `init()` で登録している
3. `META-INF/spring.handlers` に、名前空間 URI とハンドラクラスの行がある
4. `META-INF/spring.schemas` に、`schemaLocation` の URL と XSD のクラスパス位置の行がある
5. 3 と 4 のファイルが、実行しているクラスローダから見える

ファイルの例です。

`META-INF/spring.handlers`

```properties
http\://example.com/schema/myns=com.example.xml.MyNamespaceHandler
```

`META-INF/spring.schemas`

```properties
http\://example.com/schema/myns/myns.xsd=com/example/xml/myns.xsd
https\://example.com/schema/myns/myns.xsd=com/example/xml/myns.xsd
```

使う側の XML です。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:myns="http://example.com/schema/myns"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
         http://example.com/schema/myns https://example.com/schema/myns/myns.xsd">

    <myns:dateformat id="dateFormat" pattern="yyyy-MM-dd"/>
</beans>
```

`xmlns` は `http` にします。`example.com` の URI は、Spring 公式の `https` 読み替えの対象ではありません。`spring.handlers` は完全一致で引きます。書いた文字列と、ファイルの左辺が、`:` のエスケープを戻したあとで一致している必要があります。

この仕組みの利用シーンは、歴史的に次のものです。

- Spring 自身の短いタグ。`mvc`、`context`、`aop`、`tx`、`util`、`jdbc`、`jms` など。作者はアプリではなく、Spring の各 JAR
- 周辺ライブラリの短いタグ。Spring Security の `security`、Spring Batch の `batch`、Spring Integration などが、自分の JAR に同じ二つのファイルを入れる。アプリは依存関係を足すだけで認識される
- 社内共通ライブラリが、データソースやジョブの定型設定を `<company:datasource/>` のような一行にまとめる。2006 年から 2010 年代前半に多かった
- 一つの JAR に押し込むビルドで、消えた公式の行を、連結した一つの `spring.handlers` として uber-JAR の中へ戻す。これは独自タグではなく、修復

2026 年に、一つの業務アプリの WAR の中だけで新しい XML タグを発明する必要性は、ほとんどありません。同じことは `@Configuration`、`BeanDefinitionRegistryPostProcessor`、`FactoryBean` で書けます。Spring チーム自身が、MVC の XML 名前空間を Java 設定へ寄せています。

それでも独自タグが適切なのは、次のときです。

- すでに XML でしか設定できない長い運用があり、タグを増やす相手も XML の利用者である
- 配布物が JAR ライブラリで、利用者の XML に短いタグを足したい
- 既存の `mvc.xml` を壊さずに、社内基盤の定型 Bean を一行で展開したい

その場合でも、ファイルの置き場所はライブラリ JAR の `META-INF` が第一候補です。アプリの WAR に置くなら `WEB-INF/classes/META-INF` です。

独自のトップレベル要素は、ハンドラが無いと必ず失敗します。Spring 公式以外の URI だからといって、黙って無視されるわけではありません。無視されるのは、Bean に付いた未知の属性のうち、URI が `http://www.springframework.org/schema/` で始まらないものだけです。

---

## 9. Spring 6.2.9 で、mvc.xml にこのファイルは必要か

### 9.1 6.2.9 の公式の当番表

6.2.9 のタグ `v6.2.9` から読んだ、フレームワーク本体の `spring.handlers` です。

| JAR | 名前空間 URI | ハンドラ |
| --- | --- | --- |
| spring-beans | `http://www.springframework.org/schema/c` | `SimpleConstructorNamespaceHandler` |
| spring-beans | `http://www.springframework.org/schema/p` | `SimplePropertyNamespaceHandler` |
| spring-beans | `http://www.springframework.org/schema/util` | `UtilNamespaceHandler` |
| spring-aop | `http://www.springframework.org/schema/aop` | `AopNamespaceHandler` |
| spring-tx | `http://www.springframework.org/schema/tx` | `TxNamespaceHandler` |
| spring-jdbc | `http://www.springframework.org/schema/jdbc` | `JdbcNamespaceHandler` |
| spring-jms | `http://www.springframework.org/schema/jms` | `JmsNamespaceHandler` |
| spring-oxm | `http://www.springframework.org/schema/oxm` | `OxmNamespaceHandler` |
| spring-context | `http://www.springframework.org/schema/context` | `ContextNamespaceHandler` |
| spring-context | `http://www.springframework.org/schema/jee` | `JeeNamespaceHandler` |
| spring-context | `http://www.springframework.org/schema/lang` | `LangNamespaceHandler` |
| spring-context | `http://www.springframework.org/schema/task` | `TaskNamespaceHandler` |
| spring-context | `http://www.springframework.org/schema/cache` | `CacheNamespaceHandler` |
| spring-webmvc | `http://www.springframework.org/schema/mvc` | `MvcNamespaceHandler` |

`http://www.springframework.org/schema/beans` の行は、3.2.18、4.3.30、5.3.39、6.0.23、6.2.9、7.0.9 のいずれの `spring-beans` の `spring.handlers` にもありません。`<bean>` は当番表ではなく、`BeanDefinitionParserDelegate.isDefaultNamespace` が直接扱います。空の名前空間か、beans の URI ならデフォルトです。

`mvc` の 6.2.9 の `init()` が登録する要素は、`annotation-driven`、`default-servlet-handler`、`interceptors`、`resources`、`view-controller`、`redirect-view-controller`、`status-controller`、`view-resolvers`、`freemarker-configurer`、`groovy-configurer`、`script-template-configurer`、`cors` です。

### 9.2 普通の mvc.xml で、アプリ側ファイルが無い場合

次のような `mvc.xml` を考えます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd
         http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <mvc:annotation-driven/>
    <mvc:resources mapping="/static/**" location="/static/"/>
    <context:component-scan base-package="com.example.app"/>
</beans>
```

6.2.9 では、アプリが `spring.handlers` も `spring.schemas` も持っていなくて問題ありません。必要な行は、次の JAR の中にあります。

- `<bean>` の文法書: `spring-beans.jar` の `spring.schemas`
- `<mvc:*>` の当番表と文法書: `spring-webmvc.jar`
- `<context:*>` の当番表と文法書: `spring-context.jar`

`spring-webmvc` を依存関係に入れて、shade で META-INF を潰していなければ、この XML は起動します。6.2.9 では非推奨警告も出ません。

`<mvc:annotation-driven/>` が有効になるのは、ファイルを WAR に置いたからではありません。`spring-webmvc` がクラスパスにあり、その中の当番表が `mvc` を `MvcNamespaceHandler` に結んでいるからです。

### 9.3 ファイルが無くて、本当に壊れる場合

「ファイルが無い」の中身で、結果が変わります。

| 状態 | `<bean>` だけ | `<mvc:*>` や `<context:*>` | 検証 |
| --- | --- | --- | --- |
| 公式 JAR は無事。アプリ側ファイルは無い | 動く | 動く | 版番号なし、または 4.3 までの番号ならローカル XSD で通る |
| `spring-webmvc` が依存関係に無い | 動く | `mvc` で NamespaceHandler エラー | `mvc` の XSD も地図に無い |
| `spring-context` が無い | 動く | `context` で同じエラー | 同様 |
| uber-JAR が `spring.handlers` を上書きし、`mvc` の行が消えた | `<bean>` は動くことがある | クラスがあっても翻訳できない | `spring.schemas` も同様に欠けると検証で外へ取りに行く |
| アプリのファイルだけを WAR 直下 `META-INF` に置いた | 公式 JAR が無事なら動く | 独自タグは認識されない。公式タグは JAR 側で動く | WAR 直下の地図は使われない |
| `xmlns:mvc` を `https` にした | beans の URI が `http` なら `<bean>` は動く | 地図のキーと一致せず失敗 | `schemaLocation` の `https` は、別問題として解決できる |
| `schemaLocation` に `spring-mvc-6.2.xsd` | 翻訳は成功しうる | ハンドラは URI で引くので、XSD のファイル名とは独立 | 6.2.9 の地図に無く、オフラインでは検証失敗になりやすい |
| 検証をオフにし、ハンドラは JAR にある | 動く | 動く | `spring.schemas` を使わない |
| Java Config だけで XML が無い | この仕組みは呼ばれない | 呼ばれない | 呼ばれない |

よく見る失敗メッセージは、次の系統です。

- `Unable to locate Spring NamespaceHandler for XML schema namespace [http://www.springframework.org/schema/mvc]`
  - `mvc` の行が、見えているクラスローダの地図に無い
- `Could not find NamespaceHandler class [org.springframework.web.servlet.config.MvcNamespaceHandler]`
  - 行はあるが、そのクラスが同じクラスローダから見えない
- `Failed to read schema document 'https://www.springframework.org/schema/mvc/spring-mvc.xsd'`
  - 地図に当たらず、ネットワーク取得も失敗した
- `cvc-complex-type` や `cvc-elt.1` のような XSD 検証エラー
  - 古い番号の XSD をネットから取得した、HTML を XSD として読んだ、XML の要素が本当に文法違反、のいずれか

WAR 直下へ公式ファイルをコピーしても、これらのどれも修復できません。修復は、欠けている JAR を `WEB-INF/lib` に戻すか、uber-JAR の中の一つの `META-INF/spring.handlers` と `spring.schemas` に、全 JAR 分を連結して入れることです。

Maven Shade Plugin で連結するときは、`AppendingTransformer` の対象に少なくとも次を入れます。

- `META-INF/spring.handlers`
- `META-INF/spring.schemas`
- IDE も使うなら `META-INF/spring.tooling`

Spring Boot の再パッケージは、通常この連結が不要です。

### 9.4 6.2.9 のまま使い続けることと、この仕組みの関係

6.2.9 に、この仕組みの欠陥があって 6.2.19 で直った、という事実は、今回確認したリリースノートとソースの範囲にはありません。当番表の中身は 6.2 系を通じて同じ設計です。

別の理由はあります。6.2 の OSS サポートは 2026-06-30 に終わっています。6.2.9 は 2025-07-17 のパッチで、6.2 系の最後の公開版は 6.2.19 です。セキュリティ修正を受けるという意味では、6.2.9 に留まる理由はありません。そのアップグレードで、`spring.handlers` の書き方や置き場所を変える必要はありません。

7.0 へ上げると、同じ `mvc.xml` は動いたまま、起動ログに MVC 名前空間の非推奨警告が出ます。`<bean>` の名前空間は、7.0 のリリースノート上、非推奨ではありません。

---

## 10. 版による違いの一覧

| 時期 | 版 | この仕組みで起きたこと | アプリが自分のファイルを足す必要 |
| --- | --- | --- | --- |
| 2004 | 1.x | DTD と `<bean>` だけ。二つのファイルは無い | 無い。拡張口も無い |
| 2005-12 発表、2.0 の文書 | 2.0 | 二つのファイル、XSD、カスタム名前空間が誕生。Javadoc の `@since` は 2.0 | 独自タグを配るライブラリだけが足す |
| 2007 | 2.5 | `context` が `spring-context` の JAR に追加 | アプリは足さない |
| 2009-12 | 3.0 | JAR 分割。マージ前提になる。`<mvc:*>` が `spring-webmvc` に入る。`@Configuration` も本線に | 通常の MVC アプリは足さない。uber-JAR だけ連結が必要 |
| 2011 | 3.1 | `c` 名前空間などが増える。登録方法は同じ | 足さない |
| 2013 から 2016 | 4.0 から 4.3 | 版番号つき XSD を 4.3 まで追加し、版番号なしは「その版の最新」を指す | 足さない |
| 2017-09 | 5.0 | 新しい版番号つき XSD ファイル名を増やさない方針が定着。地図は 4.3 までと版番号なし | `spring-beans-5.0.xsd` のような URL は地図に無い |
| 2019 | 4.3 後期と 5.1 / 5.2 の時期 | `https` の schemaLocation を `http` の項目へ読み替えるコード。続けて地図へ `https` 行を追加 | 公式スキーマのためだけに足す必要は、この対応以降ほぼ無い |
| 2020-10 | 5.3 | 形式は同じ。パッケージは `javax` | 独自ハンドラのコードは `javax` のまま |
| 2022-11 | 6.0 | Jakarta 化。URI とファイル形式は同じ。一部の古いリモート機能は削除され、JNDI は `<jee:jndi-lookup>` が残る、と 6.0 のリリースノートにある | 5.3 用に書いた独自ハンドラは、中の API を Jakarta へ直す。ファイルの書式は直さない |
| 2023-11 | 6.1 | この仕組みの変更なし | なし |
| 2024-11 | 6.2.0 | この仕組みの変更なし | なし |
| 2025-07-17 | 6.2.9 | リリースノートにこの仕組みの変更なし。`MvcNamespaceHandler` に警告なし。本章の表の登録内容 | 普通の `mvc.xml` では足さない |
| 2026-06-08 | 6.2.19 | 6.2 系の最終公開パッチ。OSS サポートは 2026-06-30 で終了 | ファイルの置き方は 6.2.9 と同じ |
| 2025-11-30 | 7.0.0 | `<mvc:*>` が非推奨。警告ログを出して処理は続ける。他の名前空間は対象外 | `mvc.xml` は動く。警告は出る |
| 2026-08-20 | 7.0.9 | 警告文は 7.0 のまま。`spring.handlers` の読み方は 6.2.9 と同じ。参照ドキュメントも作成手順を掲載 | 新しい開発は Java 設定が本線。XML を残すならファイルのルールは従来どおり |

変わらないもののリストです。

- 探す相対パスは `META-INF/spring.handlers` と `META-INF/spring.schemas`
- 複数 JAR をマージする
- 名前空間 URI の正規形は `http`
- `schemaLocation` は `http` でも `https` でも、2019 年以降の版ならローカル XSD に落とせる
- WAR 直下の `META-INF` は、どの版でもクラスパスではない
- `<bean>` は当番表を使わない
- `<mvc:*>` は `spring-webmvc` の当番表を使う

変わるもののリストです。

- どの名前空間がどの JAR に入っているか（モジュール分割と機能追加）
- 地図に載っている XSD の版番号（4.3 で頭打ち、以降は版番号なし）
- `https` をローカル解決できるか（2019 年より前の古いパッチは、読み替えも `https` 行も無いことがある）
- ハンドラの中の API が `javax` か `jakarta` か（6.0）
- `<mvc:*>` を使ったときに警告が出るか（7.0 から出る。6.2.9 では出ない）
- アプリが新しい独自 XML タグを書くべきか（2.0 ではライブラリ作者向けの推奨機能。7.0 では MVC の XML 自体が保守対象から外れつつある）

6.2.9 と 6.2.19 のあいだで、このファイルの「必要性」は変わりません。6.2.9 と 7.0.9 のあいだで変わるのは、MVC 名前空間を使い続けたときの警告と、フレームワークのサポート状況です。ファイルを WAR 直下へ移す、という対応は、どちらのアップグレードでも出てきません。

---

## 11. 判断の順番

現場で「この二つのファイルを WAR に足すべきか」と聞かれたら、上から順に見ます。

1. XML を使っているか。Java Config と Boot の自動設定だけなら、アプリにファイルは不要。JAR の中に眠っているだけで、読まれない。
2. 使っている名前空間は公式か。公式なら、対応する JAR が `WEB-INF/lib` にあるかを見る。アプリのファイルは足さない。
3. `xmlns` は `http://www.springframework.org/schema/...` か。`https` の xmlns は、6.2.9 でも 7.0 でも当番表に一致しない。
4. `schemaLocation` は版番号なしか、地図にある 4.3 までの番号か。`spring-mvc-6.2.xsd` は 6.2.9 の地図に無い。
5. 起動エラーが NamespaceHandler なら、WAR 直下ではなく、クラスローダから見える `META-INF/spring.handlers` を列挙する。`spring-webmvc` の行があるか、shade で消えていないか、別クラスローダで読んでいないか。
6. 起動エラーが schema document なら、`spring.schemas` の地図と、サーバから外への通信を見る。当番表を足しても、このエラーは消えない。
7. 独自タグを本当に配りたいなら、ライブラリ JAR の `src/main/resources/META-INF` に、自分の URI だけを書く。公式の行はコピーしない。
8. どうしても一つの JAR にまとめるなら、`spring.handlers` と `spring.schemas` を連結する。WAR の外側へ逃がしても連結の代わりにならない。

---

## 12. 根拠

次のソースを、2026-10-22 の調査で確認しました。

- Spring Framework タグ `v6.2.9` の `DefaultNamespaceHandlerResolver`、`PluggableSchemaResolver`、`BeanDefinitionParserDelegate`、`PropertiesLoaderUtils`、`MvcNamespaceHandler`
- 同タグの `spring-beans`、`spring-context`、`spring-webmvc`、`spring-aop`、`spring-tx`、`spring-jdbc`、`spring-jms`、`spring-oxm` の `META-INF/spring.handlers`
- 同タグの `spring-beans` と `spring-webmvc` の `META-INF/spring.schemas`、`spring-beans` の `META-INF/spring.tooling`
- タグ `v7.0.9` の `spring-beans` の `spring.handlers` と `MvcNamespaceHandler`（非推奨警告の文言）
- タグ `v3.2.18.RELEASE`、`v4.3.30.RELEASE`、`v5.3.39`、`v6.0.23` の `spring-beans` の `spring.handlers`（beans 名前空間の行が無いことの確認）
- タグ `v4.3.30.RELEASE` の `PluggableSchemaResolver`（`https` を `http` へ読み替える処理）
- Spring Framework 2.0.x reference の "What's new in Spring 2.0" と Extensible XML authoring
- 現行の参照ドキュメント "XML Schema Authoring"（ページは 7.0.9 を最新安定版として案内しつつ、二つのファイルの手順を掲載。2026-10-16 取得）
- GitHub issue 13015（SPR-8368、JAR 分割後の同名ファイル）、issue 22739（https の schemaLocation）、issue 34063（MVC XML の非推奨）
- コミット「Introduce HTTPS mappings in spring.schemas files」（2019-05-07）
- Spring Framework 6.0 / 6.2 / 7.0 のリリースノート、v6.2.9 の GitHub Release（2025-07-17）、Spring Blog "Spring Framework 6.2.9 Available Now"
- endoflife.date の Spring Framework（2026-10-10 更新）。7.0.9 は 2026-08-20、6.2 の OSS サポート終了は 2026-06-30、6.2 の最終は 6.2.19（2026-06-08）
- Jakarta Servlet 仕様の、ウェブアプリクラスローダは `WEB-INF/classes` と `WEB-INF/lib` を見る、という規定

6.2.9 のハンドラ登録と、`https` 読み替えの該当コードは、この版のソースにあります。WAR 直下の `META-INF` を Spring が見ない、という結論は、Spring がクラスローダの `getResources` だけを使い、Servlet 仕様のクラスパスに WAR 直下の `META-INF` が含まれないことから導いています。
