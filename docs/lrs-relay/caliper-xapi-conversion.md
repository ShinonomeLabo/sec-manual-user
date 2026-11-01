# Caliper から xAPI への変換

このページは、らくらくLRS転送サービスが LMS から受け取った Caliper のデータを、どのような xAPI にして CampusLA へ送るかを説明します。
LMS を設定する方、送られたデータを CampusLA で確かめる方、受け取りを拒まれたり保留になったりした理由を調べる方に向けたページです。
接続の作り方や画面の操作は [らくらくLRS転送サービス](lrs-relay/lrs-relay.md) にあります。

> このページに書いた対応は、らくらくLRS転送サービスの作りに基づきます。
> CampusLA から xAPI の形の指定があった場合は、それに合わせて変わることがあります。

## 変換の流れ

LMS が送る 1 回分のデータ（Caliper の Envelope）は、次の順に処理されます。

1. Envelope 全体を確かめ、形が正しくなければ Envelope ごと受け付けません
2. Envelope の中のイベントを 1 件ずつ確かめ、外れたイベントだけを「拒否」にします
3. 受け付けたイベントを xAPI の statement に変換します
4. 誰のデータかを決めて、CampusLA が学習者を見分けるためのメールアドレスを作ります
5. メールアドレスを作れなかったイベントと、送る前の検査で外れたイベントは「保留」、送らない役割のイベントは「対象外」にします
6. 残ったイベントを、受け取った順に CampusLA へ送ります

1 件のイベントが拒否になっても、同じ Envelope のほかのイベントは受け付けます。

拒否、保留、対象外になったイベントも、受け取ったデータを 7 日間保管します。
設定を直したあと、再送で送り直せます（[らくらくLRS転送サービス](lrs-relay/lrs-relay.md) の「8. 再送する」）。

## 受け付ける Caliper

Caliper 1.1 と Caliper 1.2 を受け付けます。
版は、Envelope の `dataVersion` で決まります。

| `dataVersion` | 扱う版 |
| --- | --- |
| `http://purl.imsglobal.org/ctx/caliper/v1p1` | Caliper 1.1 |
| `http://purl.imsglobal.org/ctx/caliper/v1p2` | Caliper 1.2 |

Envelope は次の条件を満たす必要があります。
1 つでも外れると、Envelope の中のイベントは 1 件も受け付けません。

- `Content-Type` に `/json` か `+json` が含まれること（例：`application/json`、`application/ld+json`）。大文字と小文字は区別しません
- 1 回に送るデータが 5,242,880 バイト（5 MiB）以下であること
- `sensor`、`sendTime`、`dataVersion`、`data` の 4 つの項目があること
- `data` が配列であること。「ゆるく確かめる」の接続では、オブジェクトも受け付けます（下の「「厳しく確かめる」と「ゆるく確かめる」」）
- `dataVersion` が上の表のどちらかであること
- `data` の要素が 5000 個以下であること

本文の先頭に BOM（バイト順マーク）が 1 つ付いていても、それを除いて読みます。

`data` の要素のうち、`type` が `Event` で終わる文字列のオブジェクトをイベントとして扱います。
`type` を IRI で書いた要素も、`Event` で終わればイベントです。
それ以外のオブジェクト（`Person` など）は、イベントから参照される情報として扱い、それだけでは送りません。
オブジェクトでない要素（数、文字列、`null`、配列）と空のオブジェクト `{}` は読み飛ばし、拒否にも数えません。

イベントの `actor`、`object`、`group`、`target`、`generated`、`edApp`、`membership`、`session` には、情報を直接書くかわりに、同じ Envelope の別の要素の `id` を文字列で書けます。
この参照は、同じ Envelope の中でだけ探します。
見つからなければ、`id` だけを持つ情報として扱います。
参照をたどるのはこの 8 つの項目の 1 段だけで、たとえば `object` の中の `assignee` を参照で書いた場合はたどりません。

`id`、`action`、`eventTime`、`name` のように文字で書く項目に整数が書かれていたときは、その数字を文字として読みます（`12345` なら `"12345"`）。
`1.0` や `1e3` のように整数と同じ値の数も、整数として読みます（`"1"`、`"1000"`）。
整数でない数（`1.5` など）、9007199254740991 を超える整数、`true` などの値は、書かれていないものとして扱います。
値の前後にある空白は取り除きません。
たとえば `id` の前後に空白があると、UUID の形ではないものとして扱います。

### 受け付けるイベントの種類と動作

検証の厳しさを「厳しく確かめる」にした接続（初期設定）では、イベントの種類（`type`）ごとに、その版の仕様が認める動作（`action`）だけを受け付けます。
表に無い種類のイベントは、動作にかかわらず拒否します。

| イベントの種類 | Caliper 1.1 で受け付ける動作 | Caliper 1.2 で受け付ける動作 |
| --- | --- | --- |
| AnnotationEvent | Attached、Bookmarked、Classified、Commented、Described、Disliked、Highlighted、Identified、Liked、Linked、Questioned、Ranked、Recommended、Shared、Subscribed、Tagged | Bookmarked、Highlighted、Shared、Tagged |
| AssessmentEvent | Paused、Reset、Restarted、Resumed、Started、Submitted | Paused、Reset、Restarted、Resumed、Started、Submitted |
| AssessmentItemEvent | Completed、Reviewed、Skipped、Started、Viewed | Completed、Skipped、Started |
| AssignableEvent | Abandoned、Activated、Completed、Deactivated、Hid、Reviewed、Showed、Started、Submitted | Activated、Completed、Deactivated、Reviewed、Started、Submitted |
| FeedbackEvent | Commented、Ranked（拡張プロファイル） | Commented、Ranked |
| ForumEvent | Subscribed、Unsubscribed | Subscribed、Unsubscribed |
| GradeEvent | Graded | Graded |
| MediaEvent | ChangedResolution、ChangedSize、ChangedSpeed、ChangedVolume、ClosedPopout、DisabledCloseCaptioning、DisabledClosedCaptioning、EnabledCloseCaptioning、EnabledClosedCaptioning、Ended、EnteredFullScreen、ExitedFullScreen、ForwardedTo、JumpedTo、Muted、OpenedPopout、Paused、Restarted、Resumed、Rewound、Started、Unmuted | ChangedResolution、ChangedSize、ChangedSpeed、ChangedVolume、ClosedPopout、DisabledCloseCaptioning、DisabledClosedCaptioning、EnabledCloseCaptioning、EnabledClosedCaptioning、Ended、EnteredFullScreen、ExitedFullScreen、ForwardedTo、JumpedTo、Muted、OpenedPopout、Paused、Restarted、Resumed、Started、Unmuted |
| MessageEvent | MarkedAsRead、MarkedAsUnRead、MarkedAsUnread、Posted | MarkedAsRead、MarkedAsUnRead、MarkedAsUnread、Posted |
| NavigationEvent | NavigatedTo | NavigatedTo |
| OutcomeEvent | Graded | （受け付けない） |
| QuestionnaireEvent | Started、Submitted（拡張プロファイル） | Started、Submitted |
| QuestionnaireItemEvent | Completed、Skipped、Started（拡張プロファイル） | Completed、Skipped、Started |
| ReadingEvent | NavigatedTo、Searched、Viewed | （受け付けない） |
| ResourceManagementEvent | Archived、Copied、Created、Deleted、Described、Downloaded、Modified、Printed、Published、Restored、Retrieved、Saved、Unpublished、Uploaded（拡張プロファイル） | Archived、Copied、Created、Deleted、Described、Downloaded、Modified、Printed、Published、Restored、Retrieved、Saved、Unpublished、Uploaded |
| SearchEvent | Searched（拡張プロファイル） | Searched |
| SessionEvent | LoggedIn、LoggedOut、TimedOut | LoggedIn、LoggedOut、TimedOut |
| SurveyEvent | OptedIn、OptedOut（拡張プロファイル） | OptedIn、OptedOut |
| SurveyInvitationEvent | Accepted、Declined、Sent（拡張プロファイル） | Accepted、Declined、Sent |
| ThreadEvent | MarkedAsRead、MarkedAsUnRead、MarkedAsUnread | MarkedAsRead、MarkedAsUnRead、MarkedAsUnread |
| ToolLaunchEvent | Launched、Returned（拡張プロファイル） | Launched、Returned |
| ToolUseEvent | Used | Used |
| ViewEvent | Viewed | Viewed |

Caliper 1.1 の列に「（拡張プロファイル）」とある組は、Caliper 1.1 の仕様の本体ではなく、Caliper 1.1 の公式の拡張プロファイル（Feedback、Resource Management、Search、Survey、Tool Launch）が定める組です。

種類を特定しない `Event` は、その版の動作の語彙をすべて受け付けます。
ただし、Caliper 1.1 の `Event` では、拡張プロファイルだけが定める動作（`Launched`、`Uploaded` など）は受け付けません。

イベントの種類は、書かれた綴りのまま表と比べます。
`http://purl.imsglobal.org/caliper/GradeEvent` のように IRI で書いた種類は、表のどの行にも当たりません。
成績のイベント（下の「学習者の決め方」）としても扱いません。
`MarkedAsUnRead`、`DisabledCloseCaptioning`、`EnabledCloseCaptioning` は、仕様書の表や JSON-LD のコンテキストにある綴りの揺れで、それぞれ `MarkedAsUnread`、`DisabledClosedCaptioning`、`EnabledClosedCaptioning` と同じものとして受け付けます。
受け付けた揺れは、動詞を決めるときに正しい綴りに揃えます（下の「動作と動詞の対応」）。

## 拒否するイベント

イベントごとの検査で外れたイベントは「拒否」になり、CampusLA へは送りません。
理由は、接続の「送れなかったデータ」の画面に次の文言で表示されます。

| 画面に表示される理由 | 拒否する場合 |
| --- | --- |
| データに ID がありませんでした。 | イベントに `id` が無いか、空の文字列か、文字としても整数としても読めない値（小数、`true`、オブジェクトなど） |
| データの ID が UUID の形ではありませんでした。 | 「厳しく確かめる」の接続で、`id` が `urn:uuid:` で始まる UUID ではない（大文字の `URN:UUID:` で始まるものは UUID の形として受け付けます。前後に空白がある値や、整数で書いた `id` は、UUID の形ではないのでここに当たります） |
| その種類では扱わない操作でした。 | `action` が無いか空の文字列。または「厳しく確かめる」の接続で、種類と動作の組み合わせが上の表に無い（整数で書いた `action` は表に無いので当たります） |
| 誰の操作か (actor) が書かれていませんでした。 | `actor` が、項目を 1 つ以上持つオブジェクトか、空でない文字列で書かれていない（配列、数値、`{}`、空の文字列を含む） |
| 何に対する操作か (object) が書かれていませんでした。 | `object` が、項目を 1 つ以上持つオブジェクトか、空でない文字列で書かれていない（配列、数値、`{}`、空の文字列を含む） |
| 操作の日時の形が正しくありませんでした。 | `eventTime` が `2026-11-01T10:15:00Z` や `2026-11-01T19:15:00.000+09:00` の形（秒と時差まで書いた日時）ではない。時差の無いもの、`+0900` のようにコロンの無い時差、`24:00:00`、`:60` の秒、2 月 30 日のように無い日付も当たります |
| データが大きすぎて受け付けられませんでした。 | イベント 1 件が 262,144 バイト（256 KiB）を超える |
| 送る形に変換できませんでした。 | 変換した statement が xAPI の必須の形にならない。`object` の `id` が `https:` や `urn:` のような書き出しで始まらない場合などに起きる |

`actor` と `object` は、情報を直接書いても、別の要素への参照（文字列）で書いてもかまいません。

## 「厳しく確かめる」と「ゆるく確かめる」

検証の厳しさは、接続ごとに選べます。
初期設定は「厳しく確かめる」です。
「ゆるく確かめる」にしたときに変わるのは、次の 3 点だけです。

| 項目 | 厳しく確かめる | ゆるく確かめる |
| --- | --- | --- |
| イベントの `id` | `urn:uuid:` で始まる UUID でなければ拒否します | UUID でなければ、元の `id` から決まった UUID を作り、それを statement の `id` にします。同じ `id` からは毎回同じ UUID ができるので、LMS が同じイベントを送り直しても別の statement になりません |
| 種類と動作の組み合わせ | 上の表に無い組み合わせは拒否します | 確かめません。`type` が `Event` で終わり、`action` が空でなければ受け付けます |
| Envelope の `data` | 配列でなければ、Envelope ごと受け付けません | オブジェクトも受け付けます。キーが `"0"`、`"1"`、… と 0 から続く番号だけのオブジェクトは、番号の順に並べた要素として読みます。それ以外のオブジェクトは、それを 1 件の要素として読みます |

`id` が無いイベント、`actor` や `object` が無いイベント、日時の形が正しくないイベント、大きすぎるイベントは、どちらの設定でも拒否します。

設定を「ゆるく確かめる」に変えても、すでに拒否したイベントはそのままでは送られません。
拒否したイベントを送るには、設定を変えたあとに再送します。
再送のときは、受け取ったときと同じ Caliper の版の決まりと今の設定で検証し直し、通ったものを送ります。
検証は通っても、メールアドレスを作れないなどで送れないイベントは、拒否のまま残ります。
そのときの理由は、今の設定で受け取り直したときの理由に書き換わります。
メールアドレスを作れないときや、送らない役割のときは、「再送で見直したところ形は正しくなりましたが、…」の形の文言になります。
statement が xAPI の形にならないときは、「送る形に変換できませんでした。」になります。

## xAPI の組み立て

受け付けたイベント 1 件から、xAPI の statement を 1 件作ります。
statement の各項目は、次のように作ります。

| statement の項目 | 元にする Caliper の項目 | 作り方 |
| --- | --- | --- |
| `id` | `id` | `urn:uuid:` を除いた UUID です（大文字の `URN:UUID:` も除きます）。「ゆるく確かめる」で UUID を作った場合は、作った UUID です |
| `actor` | `actor`（成績のイベントは `object.assignee`） | 学習者のメールアドレスで表します。作り方は下の「学習者の決め方」にあります |
| `verb` | `action` | 下の「動作と動詞の対応」の IRI です。`display` の `en-US` には、Caliper の動作の名前（例：`NavigatedTo`）をそのまま入れます |
| `object` | `object` | `objectType` は常に `Activity` です。`id` は Caliper の `object.id`、`definition.type` は下の「種類と活動の種類の対応」の IRI です。`object.name` が空でない文字列なら `definition.name` の `en-US` に入れます |
| `result.score` | `generated` | 下の「得点」の規則で作ります |
| `result.completion` | `action` | 動作が `Completed`、`Ended`、`Submitted` のときだけ `true` を入れます |
| `result.duration` | `generated` か `object` | 下の「かかった時間」の規則で作ります |
| `context.contextActivities.grouping` | `group` | 下の「科目」の規則で作ります |
| `context.extensions` | `group` | 科目の ID と名前を入れます。下の「科目」と「extensions に残るもの」をご覧ください |
| `context.platform` | `edApp` | `edApp` の `id`（空でない文字列のとき）を入れます。`edApp` が文字列なら、その文字列を入れます |
| `timestamp` | `eventTime` | 受け取った値をそのまま入れます。時差（`+09:00` など）も書き換えません |

次の項目は、statement に入れません。

- `result.success`（合否）。Caliper のデータからは合格の基準が分からないためです
- `result.response`（解答の中身）、`result.extensions`、`result.score.min`
- `context.registration`、`context.contextActivities.parent`、`context.instructor`、`context.team`
- イベントの `target`、`membership`、`session`、`federatedSession`、`referrer`、`extensions`、`sensor`
- `stored`、`authority`、`version`、`attachments`

`membership` は、送るかどうかを決めるための役割を読むだけに使います（下の「学習者の決め方」）。

### 動作と動詞の対応

次の 30 の動作は、xAPI でよく使われる動詞に置き換えます。

| Caliper の動作 | xAPI の動詞 | 意味 |
| --- | --- | --- |
| Started | `http://adlnet.gov/expapi/verbs/initialized` | 始めた |
| Ended | `http://adlnet.gov/expapi/verbs/terminated` | 終えた |
| Paused | `http://adlnet.gov/expapi/verbs/suspended` | 中断した |
| Resumed | `http://adlnet.gov/expapi/verbs/resumed` | 再開した |
| Restarted | `http://adlnet.gov/expapi/verbs/resumed` | 再開した |
| Abandoned | `https://w3id.org/xapi/adl/verbs/abandoned` | 途中でやめた |
| TimedOut | `https://w3id.org/xapi/adl/verbs/abandoned` | 途中でやめた（時間切れ） |
| Completed | `http://adlnet.gov/expapi/verbs/completed` | 完了した |
| Submitted | `http://adlnet.gov/expapi/verbs/completed` | 完了した（提出した） |
| Graded | `http://adlnet.gov/expapi/verbs/scored` | 採点された |
| Viewed | `http://id.tincanapi.com/verb/viewed` | 見た |
| Showed | `http://id.tincanapi.com/verb/viewed` | 見た |
| NavigatedTo | `http://id.tincanapi.com/verb/viewed` | 見た（ページを開いた） |
| Launched | `http://adlnet.gov/expapi/verbs/launched` | 起動した |
| Reviewed | `http://id.tincanapi.com/verb/reviewed` | 見直した |
| Created | `http://activitystrea.ms/schema/1.0/create` | 作った |
| Added | `http://activitystrea.ms/schema/1.0/create` | 作った（追加した） |
| Modified | `http://activitystrea.ms/schema/1.0/update` | 変更した |
| Deleted | `http://activitystrea.ms/schema/1.0/delete` | 削除した |
| Removed | `http://activitystrea.ms/schema/1.0/delete` | 削除した（取り除いた） |
| Posted | `http://activitystrea.ms/schema/1.0/post` | 投稿した |
| Commented | `http://adlnet.gov/expapi/verbs/commented` | コメントした |
| Shared | `http://adlnet.gov/expapi/verbs/shared` | 共有した |
| LoggedIn | `https://w3id.org/xapi/adl/verbs/logged-in` | ログインした |
| LoggedOut | `https://w3id.org/xapi/adl/verbs/logged-out` | ログアウトした |
| Subscribed | `http://activitystrea.ms/schema/1.0/follow` | 購読した |
| Unsubscribed | `http://activitystrea.ms/schema/1.0/unfollow` | 購読をやめた |
| Uploaded | `http://activitystrea.ms/schema/1.0/share` | 共有した（アップロードした） |
| Downloaded | `http://id.tincanapi.com/verb/downloaded` | ダウンロードした |
| Searched | `http://activitystrea.ms/schema/1.0/search` | 検索した |

これ以外の動作は、Caliper の動作の IRI のまま送ります。
IRI は `http://purl.imsglobal.org/vocab/caliper/action#` の後ろに動作の名前を付けたものです（例：`Bookmarked` なら `http://purl.imsglobal.org/vocab/caliper/action#Bookmarked`）。
意味の近い xAPI の動詞へ推測で置き換えることはしません。
次の 3 つの綴りは、右の綴りに揃えてから動詞を決めます。
`display` の `en-US` にも、揃えた綴りを入れます。

- `MarkedAsUnRead` は `MarkedAsUnread`
- `DisabledCloseCaptioning` は `DisabledClosedCaptioning`
- `EnabledCloseCaptioning` は `EnabledClosedCaptioning`

たとえば `MarkedAsUnRead` は、`http://purl.imsglobal.org/vocab/caliper/action#MarkedAsUnread` になります。

### 種類と活動の種類の対応

`object` の種類（`type`）は、次の 12 種類を xAPI の活動の種類に置き換えます。

| Caliper の種類 | xAPI の活動の種類 | 意味 |
| --- | --- | --- |
| Assessment | `http://adlnet.gov/expapi/activities/assessment` | テスト |
| AssessmentItem | `http://adlnet.gov/expapi/activities/question` | 問題 |
| Question | `http://adlnet.gov/expapi/activities/question` | 問題 |
| Attempt | `http://adlnet.gov/expapi/activities/attempt` | 受験の 1 回分 |
| DigitalResource | `http://adlnet.gov/expapi/activities/module` | 教材 |
| Document | `http://adlnet.gov/expapi/activities/module` | 教材 |
| Page | `http://adlnet.gov/expapi/activities/module` | 教材 |
| WebPage | `http://adlnet.gov/expapi/activities/module` | 教材 |
| Chapter | `http://adlnet.gov/expapi/activities/module` | 教材 |
| Course | `http://adlnet.gov/expapi/activities/course` | 科目 |
| CourseOffering | `http://adlnet.gov/expapi/activities/course` | 科目 |
| CourseSection | `http://adlnet.gov/expapi/activities/course` | 科目 |

種類は、書き方によって次のように決めます。

| 種類の書き方 | 活動の種類 |
| --- | --- |
| 上の表の名前（例：`WebPage`） | 上の表の IRI |
| `http://purl.imsglobal.org/caliper/` の後ろに上の表の名前を付けた IRI（例：`http://purl.imsglobal.org/caliper/WebPage`） | 上の表の IRI（名前だけで書いたときと同じ） |
| それ以外の、`http://`、`https://`、`urn:` で始まる IRI（例：`https://lms.example.ac.jp/types/Quiz`） | その IRI をそのまま使います |
| 上の表に無い名前（例：`VideoObject`） | `http://purl.imsglobal.org/caliper/` の後ろに種類の名前を付けた IRI にします（`http://purl.imsglobal.org/caliper/VideoObject`）。大文字と小文字は直しません |
| それ以外で `:` を含む形（例：`ex:Quiz`。`HTTP://` のように大文字で始まる IRI も含む） | 入れません |
| 種類が書かれていないか、文字列ではない | 入れません |

参照が同じ Envelope の中で見つからなかった `object` は、種類が分からないので、活動の種類を入れません。
活動の種類も名前も無いときは、`definition` ごと入れません。
科目（`group`）の種類も、同じ規則で決めます。

### 得点

得点は、イベントの `generated`（`Score` など）から作ります。

| statement の項目 | 作り方 |
| --- | --- |
| `result.score.raw` | `scoreGiven` の値です。無ければ `resultScore` の値です |
| `result.score.max` | `maxScore` の値です。無ければ `maxResultScore` の値です。満点が 0 以下のときと、得点が満点を超えるときは入れません |
| `result.score.scaled` | `raw` と `max` の両方を入れたときだけ、`raw ÷ max` を入れます。-1 より小さければ -1 にします |

得点も満点も無ければ、`result.score` は入れません。
満点だけがあり、それが 0 以下のときも、`result.score` は入れません。
値が文字列のときは、先頭の数値の部分を読みます（`"7.5pts"` は 7.5、`" 7"` は 7）。
数値として読めない文字列は 0 になります。
大きすぎて数として扱えない値（`1e999` など）は、無いものとして扱います。

得点が満点を超えるときと、満点が 0 以下のときは、`raw` だけを送ります。
CampusLA が使う SQL LRS は、得点が満点を超える statement を受け付けず、同じときに送ったほかの statement もまとめて拒むためです。
満点が分からないときに `maxScore` を 0 にする LMS があり、その場合もこれに当たります。

### かかった時間

`result.duration` には、学習者が費やした時間だけを入れます。
`generated` の種類が `Attempt`（受験の 1 回分）か、`Response`（解答）か、解答の下位の種類で、`duration` が空でない文字列なら、その値を入れます。
解答の下位の種類は、`DateTimeResponse`、`FillinBlankResponse`、`MultipleChoiceResponse`、`MultipleResponseResponse`、`MultiselectResponse`、`OpenEndedResponse`、`RatingScaleResponse`、`SelectTextResponse`、`TrueFalseResponse` の 9 つです。
種類は名前だけで比べます（IRI で書いた種類は当たりません）。
`generated` から取れなければ、`object` について同じことを確かめます。
動画や音声（`VideoObject`、`AudioObject` など）の `duration` は、教材そのものの長さなので入れません。

### 科目

イベントの `group` に `id` があれば、科目として statement に入れます。

`context.contextActivities.grouping` には、`group` を活動として 1 件入れます。
活動の種類は「種類と活動の種類の対応」の表で決めるので、`CourseSection` などは科目（`course`）になります。
`group` に種類が書かれていなければ、活動の種類は入れません。
活動の名前は、`group.name` が文字列で書かれていればその値を使い、空の文字列なら名前を付けません。
`group.name` が書かれていないか文字列でなければ、`group.courseNumber` の値（空でない文字列のとき）を使います。

`context.extensions` には、`https://sec.shinonome.io/xapi/extensions/course-id` に `group.id` を入れます。
活動の名前があれば、`https://sec.shinonome.io/xapi/extensions/course-name` に同じ名前を入れます。

科目は `grouping` にだけ入れ、`parent` には入れません。

### extensions に残るもの

statement の `context.extensions` に入るのは、上の科目の 2 項目だけです。
イベントや `Person` の `extensions`（Blackboard の `bb:user.externalId` など）は、statement には入りません。
学習者のメールアドレスを作る元の値としては使えます（下の「学習者の決め方」）。

## 学習者の決め方

CampusLA は、学習者をメールアドレス（statement の `actor.mbox`）で見分けます。
そこで、らくらくLRS転送サービスは、イベントの役割を読み、接続の設定に従ってメールアドレスを作ります。
作れないときや、作れても正しいか分からないときは送りません。
誤ったメールアドレスで送ると、CampusLA で別の人の学習データになってしまうためです。

決め方は次の順です。

1. イベントの役割を読みます
2. 役割が、接続の「CampusLA に送る役割」に入っていなければ「対象外」にします
3. 役割が書かれていなければ、接続の「役割が書かれていないデータの扱い」に従います
4. 誰のデータかを決めます
5. 役割に合ったメールアドレスの作り方で、メールアドレスを作ります

### 役割の読み方

役割は、イベントの `membership.roles` から読みます。
`roles` は配列でも、1 つの文字列でもかまいません。

| `roles` に書かれた値 | 扱う役割 |
| --- | --- |
| `Learner`、`Learner#NonCreditLearner`、`http://purl.imsglobal.org/vocab/lis/v2/membership#Learner`、`…/membership/Learner#NonCreditLearner` など | 学習者 |
| `Instructor`、`Instructor#Grader`、`…/membership#Instructor`、`…/membership/Instructor#Grader` など（ティーチングアシスタントを除く） | 教員 |
| `Instructor#TeachingAssistant`、`…/membership/Instructor#TeachingAssistant` など | ティーチングアシスタント |
| 上のどれでもない値（`Mentor`、`Administrator` など） | その他 |

読み方の規則は次のとおりです。

1. `http` で始まる値は、最後の `/` より後ろを取り出し、それが `membership#` で始まれば、その `membership#` を除きます（`…/membership#Learner` は `Learner`、`…/membership/Learner#NonCreditLearner` は `Learner#NonCreditLearner`）
2. 1 で得た値（`http` で始まらない値はそのまま）が `#TeachingAssistant` で終われば、ティーチングアシスタントです
3. そうでなければ、最初の `#` より前を役割の名前とします。`Learner` は学習者、`Instructor` は教員、`TeachingAssistant` はティーチングアシスタント、それ以外の名前は「その他」です

大文字と小文字は区別します。
`learner` は学習者ではなく「その他」になります。
`urn:lti:role:ims/lis/Learner` のような、`http` で始まらない IRI の値も「その他」になります。

1 つのイベントに複数の役割があるときは、教員、ティーチングアシスタント、その他、学習者の順に、先に当たる役割を使います。
学習者用のメールアドレスの規則を教員に当てて、別の人のデータにしないためです。

成績のイベント（`GradeEvent` と `OutcomeEvent`）は、役割にかかわらず学習者として扱います。
成績のイベントの `actor` は採点したシステムや教員で、成績を受けた学習者は `object.assignee` に書かれるためです。

`membership` が無いイベントと、`roles` に役割が 1 つも書かれていないイベントは、役割が書かれていないものとして扱います。
`membership` を別の要素への参照（文字列）で書いたときは、同じ Envelope の中で探し、見つからなければ役割が書かれていないものとして扱います。

### 役割ごとの扱い

| 役割 | 扱い |
| --- | --- |
| 「CampusLA に送る役割」に入っていない役割 | 「対象外」にします |
| 学習者 | 「学習者用」のメールアドレスの作り方を使います |
| 教員、ティーチングアシスタント、その他 | 「教職員用」のメールアドレスの作り方を使います。教職員用の作り方が空なら「保留」にします |
| 役割が書かれていない | 「役割が書かれていないデータの扱い」が「保留にする」なら「保留」、「送らない」なら「対象外」、「学習者として送る」なら「学習者用」の作り方を使います。ただし、「学習者として送る」でも、「CampusLA に送る役割」に学習者が入っていなければ「対象外」にします |

### 誰のデータか

メールアドレスを作る相手は、イベントの `actor` です。
成績のイベント（`GradeEvent` と `OutcomeEvent`）だけは `object.assignee` を使います。

次の場合は、誰のデータか特定できないものとして「保留」にします。

- 成績のイベントに、情報か `id` の文字列を書いた `object.assignee` が無い。採点者のメールアドレスで成績を送らないためです
- 成績のイベントで、`object.assignee` を `id` だけで書いている（`id` の文字列か、`{"id": "…"}` のように `id` しか持たないオブジェクト）。`assignee` の参照はたどらないので、同じ Envelope の中にその `id` の情報があっても保留になります
- `actor` を `{"id": "…"}` のように `id` しか持たないオブジェクトで書いている
- `actor` を `id` の文字列だけで書いていて、同じ Envelope の中にその `id` の情報が無い
- 相手の `id` が `http://purl.imsglobal.org/caliper/Person`（名前を明かさない利用者を表す値）。ほかに何を持っていても保留にします

2 つ目から 4 つ目の場合は、メールアドレスの元の値に「ID の末尾」を選んでいても保留になります。
相手の情報が `id` と `type` だけでも、相手は特定できたものとして、下の「メールアドレスの作り方」に当てます。
たとえば、元の値が「ID の末尾」ならメールアドレスを作れます。
元の値が「メールアドレス (otherIdentifiers の EmailAddress)」なら、元の値が無いので保留になります。

### メールアドレスの作り方

メールアドレスは、接続の設定の「元の値」と「組み立て方」から作ります。
元の値は、次の 3 つから選びます。

| 画面の選択肢 | 読む値 |
| --- | --- |
| メールアドレス (otherIdentifiers の EmailAddress) | 相手の `otherIdentifiers` のうち、`identifierType` が `EmailAddress` で、`identifier` が空でない最初のもの（`identifier` が `"0"` のものは空とみなして飛ばします） |
| ID の末尾 (Person の id の、最後の / より後ろ) | 相手の `id` の、最後の `/` より後ろ。`/` が無ければ `id` 全体 |
| extensions の中の項目 (学籍番号など) | 相手の `extensions` の中の、指定した名前の項目（例：`bb:user.externalId`） |

元の値は、文字列か数値でなければ使いません。
「メールアドレス」と「ID の末尾」では、数値は整数のときだけ使います（`12345` なら `"12345"`）。
「extensions の中の項目」では、小数も使います（`1.5` なら `"1.5"`）。
読んだ値は、次の順に処理します。

1. 元の値が無いか空なら、保留にします
2. 元の値が 256 文字を超えていたら、保留にします
3. 「前後の空白を除く」をオンにしていれば、前後の空白を除きます。除いたあとに何も残らなければ、保留にします
4. 「一部だけ取り出す正規表現」を入れていれば、その正規表現の最初の括弧 `( )` に当たった部分だけを取り出します。当たらないか、取り出した部分が空なら、保留にします
5. 「小文字にする」をオンにしていれば、小文字にします
6. 「組み立て方」の `{value}` を、ここまでの値に置き換えます
7. できた値がメールアドレスの形でなければ、保留にします

7 のメールアドレスの形は、次をすべて満たすものです。

- 全体が 254 文字以内
- `@` の前が 1〜64 文字で、半角の英数字と `` !#$%&'*+/=?^_`{|}~.- `` だけでできている
- `@` の後ろが、半角の英数字とハイフンからなる部分を `.` で 2 つ以上つないだもの

日本語や空白、`( ) , : ; < > [ ] \` と `"` を含む値は、メールアドレスにしません。
CampusLA が使う SQL LRS は、これらを含むメールアドレスの statement を受け付けず、同じときに送ったほかの statement もまとめて拒むためです。

正規表現は、大文字と小文字を区別して当てます（「小文字にする」は、取り出したあとに行います）。
たとえば、学籍番号 `S2026001` から数字だけを使うなら、正規表現を `S(\d+)` にします。

### 保留の理由

保留にしたイベントは、「送れなかったデータ」の画面に次の理由で表示されます。

| 画面に表示される理由 | 保留にした場合 |
| --- | --- |
| 役割が書かれていないため、保留にしています。 | 役割が書かれておらず、扱いが「保留にする」 |
| 誰のデータか特定できないため、保留にしています。 | 上の「誰のデータか」で特定できなかった |
| メールアドレスの元にする値がデータに無いため、保留にしています。 | 元の値が無いか空。前後の空白を除いて空になった場合も含みます |
| メールアドレスの元にする値が長すぎるため、保留にしています。 | 元の値が 256 文字を超えている |
| 元の値から取り出す部分が見つからないため、保留にしています。 | 正規表現が当たらなかったか、取り出した部分が空だった |
| メールアドレスの形になっていないため、保留にしています。 | できた値がメールアドレスの形ではない |
| 教職員用のメールアドレスの作り方が未設定のため、保留にしています。 | 教員などのイベントで、教職員用の作り方が空 |
| 送り先の決まりに合わない値 (時間の書き方など) があるため、送らずに保留にしています。 | 下の「送る前の検査」で外れた |

対象外にしたイベントの理由は、役割ごとに「学習者のデータで、送る役割に入っていません。」のように表示されます。
役割が書かれていないイベントを「送らない」にした場合は「役割が書かれていないデータを、送らない設定にしています。」と表示されます。
役割が書かれていないイベントを「学習者として送る」にしていて、送る役割に学習者が入っていない場合は「学習者のデータで、送る役割に入っていません。」と表示されます。

### 送る前の検査

送ると決めたイベントは、送る前に、作った statement が xAPI の形になっているかを確かめます。
外れた statement は送らずに保留にします。
1 件の形の誤りで、同じときに送るほかの statement まで CampusLA に拒まれないようにするためです。
たとえば、次のときに外れます。

- `result.duration` が `PT25M` のような期間の形になっていない（`"120"` など）
- 作ったメールアドレスが xAPI のメールアドレスとして正しくない（`@` の前が `.` で始まるなど）
- `eventTime` の時差を `-00:00` と書いている

### statement の actor の形

作ったメールアドレスは、`mailto:` を付けて `actor.mbox` に入れます。
相手に文字列の `name` があれば、`actor.name` にも入れます。

```json
"actor": { "objectType": "Agent", "mbox": "mailto:s2026001@st.example.ac.jp", "name": "テスト学生1" }
```

`actor` には `account` を入れません。
CampusLA がメールアドレスで学習者を見分けるためです。

### メールアドレスの作り方を試す

接続の編集画面の「直近のデータで試す」で、入力中の作り方を、直近に受け取ったデータに当てて試せます。
試す対象と結果の見方は、[らくらくLRS転送サービス](lrs-relay/lrs-relay.md) の「5. メールアドレスの作り方を試す」にあります。

## 変換の例

ここに挙げるデータは、説明のために作った架空のものです。
どちらの例も、接続の設定は次のとおりとします。

- 送る役割：学習者だけ
- 学習者用の元の値：extensions の中の項目（`bb:user.externalId`）
- 組み立て方：`{value}@st.example.ac.jp`
- 前後の空白を除く、小文字にする：オン

### 例 1：成績のイベント（Caliper 1.1）

受け取った Envelope です。
`actor` は採点したシステムで、成績を受けた学習者は `object.assignee` に書かれています。

```json
{
  "sensor": "https://lms.example.ac.jp/sensor",
  "sendTime": "2026-11-01T01:20:05.000Z",
  "dataVersion": "http://purl.imsglobal.org/ctx/caliper/v1p1",
  "data": [
    {
      "id": "urn:uuid:5b2f3c1e-8a4d-4f6b-9c2e-1d7a0e3b4c5f",
      "type": "GradeEvent",
      "action": "Graded",
      "actor": { "id": "https://lms.example.ac.jp", "type": "SoftwareApplication" },
      "object": {
        "id": "https://lms.example.ac.jp/courses/301/quizzes/1/attempts/1",
        "type": "Attempt",
        "assignee": {
          "id": "https://lms.example.ac.jp/users/1001",
          "type": "Person",
          "name": "テスト学生1",
          "extensions": { "bb:user.externalId": "S2026001" }
        }
      },
      "generated": { "type": "Score", "scoreGiven": 8, "maxScore": 10 },
      "group": { "id": "https://lms.example.ac.jp/courses/301", "type": "CourseSection", "name": "テスト科目1" },
      "edApp": { "id": "https://lms.example.ac.jp", "type": "SoftwareApplication" },
      "eventTime": "2026-11-01T01:20:00.000Z"
    }
  ]
}
```

CampusLA へ送る statement です。

```json
{
  "verb": { "id": "http://adlnet.gov/expapi/verbs/scored", "display": { "en-US": "Graded" } },
  "id": "5b2f3c1e-8a4d-4f6b-9c2e-1d7a0e3b4c5f",
  "actor": { "objectType": "Agent", "mbox": "mailto:s2026001@st.example.ac.jp", "name": "テスト学生1" },
  "object": {
    "objectType": "Activity",
    "id": "https://lms.example.ac.jp/courses/301/quizzes/1/attempts/1",
    "definition": { "type": "http://adlnet.gov/expapi/activities/attempt" }
  },
  "result": { "score": { "raw": 8, "max": 10, "scaled": 0.8 } },
  "context": {
    "contextActivities": {
      "grouping": [
        {
          "objectType": "Activity",
          "id": "https://lms.example.ac.jp/courses/301",
          "definition": { "type": "http://adlnet.gov/expapi/activities/course", "name": { "en-US": "テスト科目1" } }
        }
      ]
    },
    "extensions": {
      "https://sec.shinonome.io/xapi/extensions/course-id": "https://lms.example.ac.jp/courses/301",
      "https://sec.shinonome.io/xapi/extensions/course-name": "テスト科目1"
    },
    "platform": "https://lms.example.ac.jp"
  },
  "timestamp": "2026-11-01T01:20:00.000Z"
}
```

成績のイベントなので、`actor` は採点したシステムではなく `object.assignee` の学習者から作っています。
`membership` がありませんが、成績のイベントは学習者として扱うので保留になりません。

### 例 2：参照を使った提出のイベント（Caliper 1.2）

受け取った Envelope です。
`actor` には学習者の `id` だけを書き、学習者の情報は `data` の別の要素として送っています。

```json
{
  "sensor": "https://lms.example.ac.jp/sensor",
  "sendTime": "2026-11-01T01:15:05.000Z",
  "dataVersion": "http://purl.imsglobal.org/ctx/caliper/v1p2",
  "data": [
    {
      "id": "https://lms.example.ac.jp/users/1001",
      "type": "Person",
      "name": "テスト学生1",
      "extensions": { "bb:user.externalId": "S2026001" }
    },
    {
      "id": "urn:uuid:0c9e7d2a-3b1f-4e5d-8a6c-2f4b9d1e7a30",
      "type": "AssessmentEvent",
      "action": "Submitted",
      "actor": "https://lms.example.ac.jp/users/1001",
      "object": { "id": "https://lms.example.ac.jp/courses/301/quizzes/1", "type": "Assessment", "name": "小テスト1" },
      "generated": { "id": "https://lms.example.ac.jp/courses/301/quizzes/1/attempts/1", "type": "Attempt", "duration": "PT25M" },
      "membership": { "roles": ["Learner"] },
      "group": { "id": "https://lms.example.ac.jp/courses/301", "type": "CourseSection", "name": "テスト科目1" },
      "eventTime": "2026-11-01T01:15:00.000Z"
    }
  ]
}
```

CampusLA へ送る statement です。

```json
{
  "verb": { "id": "http://adlnet.gov/expapi/verbs/completed", "display": { "en-US": "Submitted" } },
  "id": "0c9e7d2a-3b1f-4e5d-8a6c-2f4b9d1e7a30",
  "actor": { "objectType": "Agent", "mbox": "mailto:s2026001@st.example.ac.jp", "name": "テスト学生1" },
  "object": {
    "objectType": "Activity",
    "id": "https://lms.example.ac.jp/courses/301/quizzes/1",
    "definition": { "type": "http://adlnet.gov/expapi/activities/assessment", "name": { "en-US": "小テスト1" } }
  },
  "result": { "duration": "PT25M", "completion": true },
  "context": {
    "contextActivities": {
      "grouping": [
        {
          "objectType": "Activity",
          "id": "https://lms.example.ac.jp/courses/301",
          "definition": { "type": "http://adlnet.gov/expapi/activities/course", "name": { "en-US": "テスト科目1" } }
        }
      ]
    },
    "extensions": {
      "https://sec.shinonome.io/xapi/extensions/course-id": "https://lms.example.ac.jp/courses/301",
      "https://sec.shinonome.io/xapi/extensions/course-name": "テスト科目1"
    }
  },
  "timestamp": "2026-11-01T01:15:00.000Z"
}
```

`actor` の参照は、同じ Envelope の `Person` の要素で解決しています。
`Submitted` なので `result.completion` が `true` になり、`generated` の `Attempt` の `duration` が `result.duration` に入っています。
`edApp` が無いので、`context.platform` はありません。
