# Review resolution contract

Repository: Saber5656/pastecheck; PR #28

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTN3-aM6PBMfq

### Extract only the Cargo version before comparing tags

**Normative resolution**

release workflowはcargo metadata --no-deps --format-version 1等の機械可読値、またはcargo pkgidの最後の@以降だけをversionとして抽出し、v prefixを一度だけ付ける。package path/nameをversion比較へ混入させない。

**Focused verification before resolving this thread:**

Cargo package名、path+file URL、version 0.1.0を含むfixtureでtag v0.1.0との比較を行い、正しいtagだけが通過し、不一致tagとpackage名混入値が拒否されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMfu

### Preserve gate stderr for scan-impossible prompts

**Normative resolution**

gate exit 3のstderrをhookが保持・表示し、paste/abort等の選択前にscan不能理由をユーザーへ渡す。exit 1のfindingとexit 3のoperational failureを別扱いにし、stderrを無条件に捨てない。

**Focused verification before resolving this thread:**

oversized input、config fallback、binary missingでgate exit 3を返すfixtureを作り、hook表示にstderrが含まれ、理由なしにpasteが進まないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMfz

### Rebuild sanitize output from raw bytes

**Normative resolution**

sanitizerは入力raw bytesをsource of truthとして走査し、control keepではinvalid UTF-8をlossy decode/re-encodeしない。remove/replace対象だけをbyte/rune境界に従って変更し、keep bytesを完全保持する。

**Focused verification before resolving this thread:**

invalid UTF-8とPC220 bytesを含む入力でcontrol keep/remove/replaceを比較し、keep出力のbyte列が入力と一致すること、対象以外のbytesが変化しないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf1

### Don't let set -e abort the release smoke check

**Normative resolution**

release smoke checkはpastecheckの期待するexit 1を一時captureして判定し、shellのset -eで後続assertを飛ばさない。unexpected exit codeとstderrは失敗として扱う。

**Focused verification before resolving this thread:**

exit 1の正常finding、exit 0のunexpected pass、exit 2/3のerrorをrunnerで実行し、各々が意図した判定とログになり、expected exit 1だけがrelease gateを通ることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf3

### Keep gate output templates ASCII-only

**Normative resolution**

gateが生成する機械向けtemplateはASCII文字だけで構成し、乗算記号はASCIIのx<count>等へ置換する。入力由来Unicodeの表示経路とtemplate固定値を分離する。

**Focused verification before resolving this thread:**

全finding count、0件、複数件のtemplateをbyte検査し、0x00–0x7fだけであることとcountが正しく表示されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf4

### Choose one JSON shape for rules output

**Normative resolution**

rules --format jsonのtop-level schemaをobjectに固定し、rules配列とunicode_data metadataを明示する。件数契約はrules.length==26へ統一し、array lengthを前提にしたconsumerとdocumentationを同時更新する。

**Focused verification before resolving this thread:**

JSON schema testでtop-level keys/typesを検査し、rules.length=26とunicode_data versionが存在すること、jq/CLI consumerが同じshapeだけを受理し、旧array/sibling混在を拒否することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf6

### Honor fail-closed when the zsh binary is missing

**Normative resolution**

pastecheck binaryがない場合、PASTECHECK_GATE_FAIL=closedではpass-through widgetを定義せずpasteをdiscard/abortする。open policyの場合だけ明示したwarning付きfallbackを許可し、環境変数の既定値をfail-openにしない。

**Focused verification before resolving this thread:**

binaryをPATHから外し、closed/open各設定でbracketed pasteを試す。closedはshellへ未検査内容を挿入せず、openは警告と契約どおりのfallbackになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf9

### Preserve sanitize's exit status when using the sentinel

**Normative resolution**

sentinelはsanitize commandのexit statusを上書きしない構造にする。command substitutionとprintfのstatusを分離し、sanitize error/config error/binary missingはhookへそのまま伝播させる。

**Focused verification before resolving this thread:**

sanitizeが0、1、2、3を返すfixtureでhookを実行し、各statusが保持されること、失敗時に空/partial textを挿入しないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMf-

### Drain oversized bash pastes before returning

**Normative resolution**

bash intercept modeは保存するpayloadを5 MiB等でcapしつつ、ESC[201~終端までTTYをdrainしてからreturnする。超過分やsuffix commandをreadlineへ残さず、gate failure policyに従ってdiscardする。

**Focused verification before resolving this thread:**

上限直前・超過・終端後に追加commandを含むbracketed pasteを送り、超過pasteが挿入されず、終端後のcommandが次のreadlineへ混入しないことをpty testで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgB

### Don't assert colorized output has no ESC

**Normative resolution**

SEC-3のinput-derived output safety testは--color neverで実施するか、rendererが生成する許可済みSGRだけを明示的にallowlistする。ユーザー入力由来のESCはどのcolor modeでも拒否/escapeする。

**Focused verification before resolving this thread:**

color neverのbyte allowlist testとcolor alwaysのSGR-only testを分離し、入力へESC/CSIを混ぜてもraw controlが漏れず、trusted renderer SGRだけが許可されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgD

### Use a byte-capable encoding for corpus inputs

**Normative resolution**

attack corpus formatはinvalid bytesをhex/base64等で、Unicode scalar/IVSを\\UXXXXXXXX相当のscalar表現で表せるようにする。loaderはsurrogate/out-of-rangeを拒否し、byteとtext入力を区別する。

**Focused verification before resolving this thread:**

invalid UTF-8、PC220、U+E0041/U+E0100を含むcorpusをloadして期待bytes/codepointsが再現されること、欠損escape・surrogate・範囲外値がfail closedすることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgE

### Use a non-ASCII-only PC502 example

**Normative resolution**

PC502 fixtureは全codepointがnon-ASCIIのconfusable wordにする。末尾ASCII lのようなmixed inputはPC501 fixtureへ分離し、severityとrule ownershipを取り違えない。

**Focused verification before resolving this thread:**

PC502の各文字がU+007F超であることをfixture lintで確認し、all-non-ASCII入力がPC502、ASCII混在入力がPC501になることをassertする。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgK

### Exclude PC104 line breaks from residual control findings

**Normative resolution**

PC104が所有するVT/FF/NEL等をresidual control scanの入力から除外し、PC210/PC206へ二重計上しない。newline detectorの分類とseverityをsingle-owner registryで共有する。

**Focused verification before resolving this thread:**

VT、FF、NEL、通常改行、その他のC0/C1 controlを各々入力し、PC104だけがline-breakとして報告され、residual ruleが重複critical findingを出さないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgN

### Merge downloaded artifacts before checksumming

**Normative resolution**

download-artifactはmerge-multiple=trueで一つのchecksum directoryへ展開するか、nested artifact pathsをglobして全4 tarballを対象にする。checksum前にexpected artifact countとfilenameを検査する。

**Focused verification before resolving this thread:**

4 artifactをdownloadしたpublish fixtureで全tarballがchecksum対象になり、欠損・重複・unexpected filenameがreleaseをfailさせることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTN3-aM6PBMgN

### Reject every non-SHA action reference

**Normative resolution**

SEC-12 scannerはworkflow中の全uses: referenceを解析し、40文字hex SHA以外（tag、branch、短縮SHA、未指定）を一件でも検出したらfailする。valid pinned actionが別に存在することを成功条件にしない。

**Focused verification before resolving this thread:**

valid SHAのみのworkflowが通過し、@main、@v4、短縮SHA、混在workflowが全てfailすることをfixture scanで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
