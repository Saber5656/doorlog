# Review resolution contract

Repository: Saber5656/doorlog; PR #1

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTNkIE86QHPu4

### Bind the sample compose port to a trusted interface

**Normative resolution**

sample composeは既定で127.0.0.1等のtrusted interfaceへbindし、LAN公開が必要な場合だけ利用者が明示的にLAN address/firewallを設定する。no-auth UI/APIを0.0.0.0やpublic IPv6へ暗黙公開しない。

**Focused verification before resolving this thread:**

clean hostでcomposeを起動し、listen socketがloopback既定になり外部interfaceから到達できないことを確認する。明示LAN binding時はdocumented firewall/Host guardが有効であることを検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPu5

### Roll up digests from the previous send time

**Normative resolution**

digest windowはcalendar day 00:00ではなく、直前のsuccessful send/cutoff timestampから現在cutoffまでとする。送信失敗・late-evening eventのcarry-forwardと重複防止のcursorを永続化する。

**Focused verification before resolving this thread:**

21:00送信後と送信前後のevent、送信失敗、翌日retryをfake clockで作り、各eventが一度だけ適切なdigestへ入ることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPu7

### Grant the non-root container read access to host logs

**Normative resolution**

non-root containerでhost logを読むため、compose/Dockerfileとinstall docsに最小権限のgroup_add/ACLまたは明示的なread-only staging方式を定義する。権限がない場合はsecretを露出せず、ingest不能をclear errorにする。

**Focused verification before resolving this thread:**

UID 65532相当でauth.log/fail2ban mountを読み、supported host setupでは成功、未設定ではsilent emptyではなく診断可能なpermission errorになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPu8

### Do not publish a shared ntfy topic in the sample compose

**Normative resolution**

sampleのntfyはdisabledまたはunique topic placeholderとし、public shared topicを既定値にしない。enable時はsetupで利用者が生成したsecret/unique topicを要求し、例示値を実運用へ送信できないようにする。

**Focused verification before resolving this thread:**

未変更sample composeを起動して外部ntfyへpublishしないこと、enableした場合にunique topic未設定がfailすること、設定済みtopicだけが使われることを検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvA

### Ignore empty fingerprints when matching devices

**Normative resolution**

fingerprint matchはknownとeventの双方がnon-emptyで完全一致する場合だけ成立させる。空fingerprintはuser+ip等の別matcherへfall throughし、unknown login alertを抑制しない。

**Focused verification before resolving this thread:**

known/eventのfingerprintが空・一致・不一致の組み合わせをfixture化し、空同士がknown扱いにならず、non-empty exact matchだけが即時alert抑止になることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvD

### Reject device matcher subsets that can never match

**Normative resolution**

device matcher schemaは実際にresolverが扱えるfingerprint、またはuserとipの完全組合せだけを受理し、user-only/ip-onlyの保存を拒否する。将来subsetを許可する場合は対応matcherを同時に定義する。

**Focused verification before resolving this thread:**

API validationで各組合せを送信し、許可組合せだけが保存され、許可後のlogin eventに対してresolverが必ず判定できることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvE

### Treat IPv6 ULA addresses as private/tailnet

**Normative resolution**

private/tailnet classifierへRFC4193 fc00::/7 ULAを追加し、loopback/link-localと同じ安全なlocal分類へ入れる。global IPv6は引き続きpublicとして扱う。

**Focused verification before resolving this thread:**

fd00::/8、fc00::/7、fe80::/10、::1、global IPv6を分類し、severity、geo wording、digest scopeがそれぞれ規定どおりになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvF

### Keep backfilled buckets separate from live attempts

**Normative resolution**

backfilled=1 bucketをlive failureのIP-only keyと共有しない。backfill bucketを閉じるまでlive専用key/namespaceを使い、digest exclusion属性がlive attemptへ継承されないようにする。

**Focused verification before resolving this thread:**

同一IPでbackfill行を作成後にlive failureを投入し、別bucketとして集計・digestされること、backfill属性がliveへ移らないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvG

### Scope the banned {@html} grep away from docs

**Normative resolution**

banned {@html} checkはSvelte/source/compiled UIなど実行対象へscopeし、docs内のliteral explanationを検査対象から除外する。またはdocs exampleを安全にescapeする。CIの目的と対象pathを固定する。

**Focused verification before resolving this thread:**

docsにliteral tokenを残したfixtureとsourceに実際の{@html}を置いたfixtureをscanし、前者は許可、後者はfailすることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvI

### Use a cursor that matches the timeline sort

**Normative resolution**

timeline paginationは(ts,id) composite cursorでevent time DESC/id tie-break DESCを一貫して使うか、ULIDがevent time順であることを強制する。backfilled/delayed eventがskip/duplicateにならない順序をAPI契約にする。

**Focused verification before resolving this thread:**

同timestamp、遅延timestamp、backfill IDの順序をfixture化し、page boundaryを跨いでも全eventが一度だけ返り、cursorの後続が前ページへ戻らないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvJ

### Persist UI settings in the configuration precedence

**Normative resolution**

boot precedenceへpersisted settings tableを明示的に追加し、env/yaml/defaultとの優先順位・validationを固定する。UI変更がrestart後もlocale、digest time、notification toggleへ反映される。

**Focused verification before resolving this thread:**

UIでsettingsを更新して再起動し、precedenceどおりの値がeffective configになること、env overrideの優先順位と不正stored valueのfail/default動作を確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvK

### Parse failed publickey lines with key details

**Normative resolution**

sshd failed publickey parserはssh2以降のkey type/fingerprint等のsuffixを許容し、Invalid user行の有無に依存せずattemptを記録する。既存形式との互換性を保つ。

**Focused verification before resolving this thread:**

suffixなし、RSA SHA256 suffix、ssh2以外のfailure、valid root accountのfixtureを解析し、各publickey attemptが正しくcountされ、無関係なlineを誤認しないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvL

### Include the journald PID in synthetic syslog lines

**Normative resolution**

journald adapterは_PIDを取得してsshd[pid]:またはsshd-session[pid]:形式のsynthetic lineを生成する。PID欠損時のplaceholder/skip policyも固定し、既存parser契約へ適合させる。

**Focused verification before resolving this thread:**

PID有り・無しとsshd/sshd-sessionをfixture化し、synthetic lineがparserで同じeventへ変換されること、malformed identifierがsilently countedされないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvM

### Allow empty optional paths during config validation

**Normative resolution**

feature disableを意味するoptional pathは空文字を許可し、non-emptyの場合だけabsolute/containment validationを行う。required pathとoptional pathのschemaを分離して、空値を誤ってfilesystem root扱いしない。

**Focused verification before resolving this thread:**

fail2ban_path/geo.mmdb_pathを空、relative、absolute、外部pathで検証し、空はdisabled、relative/unsafeは拒否、valid absoluteだけが有効になることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvN

### Honor digest.skip_empty for quiet days

**Normative resolution**

digest.skip_empty=trueではzero-activity dayのpushをskipする。recordを作るかどうか、retry/cutoffをどう進めるかを明示し、falseではheartbeat digest.noneを従来どおり送る。

**Focused verification before resolving this thread:**

activity有り/無しとskip_empty true/falseをfake clockで実行し、push件数、stored digest、next cutoffが契約どおりになり、quiet dayのeventが次windowから欠落しないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIE86QHPvO

### Point fuzz CI at the package containing the target

**Normative resolution**

nightly fuzz jobは対象parser packageを明示し、-fuzz targetがexactly one packageへ適用されるコマンドにする。fuzz failure/timeoutをCI successへ変換しない。

**Focused verification before resolving this thread:**

repo rootからworkflow commandを実行し、target packageのfuzz testが実際に開始されること、存在しない/複数package指定が早期失敗することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
