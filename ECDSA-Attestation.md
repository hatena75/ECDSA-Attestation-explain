# 概要
ECDSA Attestationは、従来のEPID Attestationとは異なるSGXのRemote Attestationである。サードパーティがプロビジョニングとアテステーションプロセスを実行可能であり、Intelに権限が集中しないよう設計されているのが特徴である。元々は、以下のようなEPID Attestationが不都合なユースケースを想定して開発された。

1. ネットワークの大部分を、インターネットベースのサービスにアクセスできない環境(大規模イントラネット)で運用している事業者。 

2. 第三者に信頼性の判断を委託するリスクを嫌う事業者。 

3. 非常に分散して動作する特定のアプリケーションモデル（例：Peer-to-Peerネットワーク）。単一の検証ポイントを使用することは、このモデルにとって最適とは言えない。 

4. EPIDが提供するプライバシー特性と相反する要求がある環境。

しかし、最新のSGXではEPIDが廃止されているため、これら以外のケースでもECDSA Attestationを使うことになる。

# Enclave Measurement
Enclaveの識別には2つのハッシュ値が用いられる。これらは鍵の生成やQuoteの検証項目として利用される。  

## MRENCLAVE
Enclaveのアイデンティティを示す256bitハッシュ値である。Enclaveが初期化されてメモリ上に読み込まれると、SGXは以下を含む暗号化されたログを生成する。
- 内容：コード、データ、スタック、ヒープ
- Enclave内の各ページの配置
- 使用中のセキュリティフラグ

これらのログの値に対する256bitハッシュ値がMRENCLAVEである。

## MRSIGNER
Enclaveの署名に対応した公開鍵の256bitハッシュ値である。主にEnclaveの作成者を示す目的で使われる。

# ハードウェア鍵とソフトウェア鍵
SGXではeFuseに焼き付けられた2種類のハードウェア鍵が利用され、これらがRoot of Trustになる。これらは主に派生鍵を生成するために利用され、eFuse外には出ない。

- Root Provisioning Key (RPK)  
Intel Key Generation Facility (iKGF)と呼ばれるIntelが所有する安全なオフライン環境下で生成される共通鍵である。この鍵はCPU出荷時に埋め込まれ、Intelは生成された全てのRPKを把握している。また、RPKをセキュリティバージョンナンバー(SVN)の更新回数だけ疑似ランダム関数(PRF)でループさせることにより、TCBと紐づいた中間値を生成することができる。

- Root Sealing Key (RSK)  
CPU内でランダムに生成される共通鍵である。Intelを含む誰もRSKの値は知らないため、この鍵から派生した鍵はそのプラットフォーム以外での模倣は非常に困難になる。

## 派生鍵
ここでは、ハードウェア鍵から派生して生成される鍵を紹介する。生成はSGX用の特殊命令であるEGETKEYで行う。全てEnclave、SVN、CPUに固有であり、同じ環境下であれば、何度呼び出しても同一の鍵を得られる。

- Provisioning Certification Key (PCK)  
RPKから派生する公開鍵。Provisioning Certification Enclaveのみがこの鍵を扱うことができ、ECDSA AttestationにおいてAttestation Keyの証明書の役割を持つ構造体を発行するために利用される。

- Report Key  
RPKから派生する共通鍵。この鍵はReportを生成及び検証する際のCMACに利用され、Local Attestationの要になる。

- Seal Key  
RSKから派生する共通鍵。この鍵は生成するEnclaveのMRENCLAVEまたはMRSIGNERに固有の値になり、Enclaveのシーリング(二次記憶装置への保存)に利用される。

- EINIT Token  
RPKから派生する共通鍵。この鍵を扱えるのはECDSA AttestationではReference Launch Enclaveのみであり、他のEnclaveの起動に必要なLaunch Tokenを生成及び取得するために利用される。ただし現在、Launch Tokenはデフォルトで廃止されており、代替として0埋めされた構造体が渡されている。

## Attestation Key
Attestation Keyは、Quoteの署名に使うECDSA秘密鍵である。Quoteを生成するQuoting Enclaveによって生成及び利用される。

# Local Attestation
Local Attestationは、2つのEnclave間で特定の通信を行うことで、それらが同一プラットフォーム上にある事を証明する仕組みである。

この証明には、Reportと呼ばれる構造体を生成するEREPORT命令を利用する。この命令では、自身以外のMRENCLAVEを入力できる。Reportには、命令を呼び出したEnclaveのMRENCLAVE及びMRSIGNER、SGXのセキュリティバージョン情報、任意のユーザーデータ、入力したMRENCLAVEのReport KeyによるCMACが含まれている。

Local Attestationの流れを以下に示す。ここでは、Enclave1がLocal Attestationを要求し、Enclave2がそれに応答する。

1. Enclave1はEnclave2に対してLocal Attestationを要求する。

2. Enclave2はEnclave1に対して自身のMRENCLAVEを送信する。

3. Enclave1はEnclave2のMRENCLAVEを入力として、EREPORT命令でReport構造体を生成する。その後、ReportはEnclave2に送信される。

4. Enclave2は渡されたReportのCMACを検証するため、EGETKEY命令でReport Keyを生成する。Report KeyはプラットフォームとEnclave毎に固有であるため、この検証によってEnclave2はEnclave1が同一プラットフォーム上にあることを確認できる。その後Enclave2は、Reportに含まれたEnclave1のMRENCLAVEを入力として、EREPORT命令で新たなReportを生成し、これをEnclave1に送信する。

5. Enclave1は受信したReportのCMACを検証するため、EGETKEY命令でReport Keyを生成する。これにより、Enclave1もEnclave2が同じプラットフォーム上にあることを確認し、Local Attestationが完了する。

# 特殊なEnclave
ECDSA Attestationには、特定の用途で使用される特殊なEnclaveがある。

- Provisioning Certification Enclave (PCE)  
ECDSA Attestationのプロビジョニングで利用されるEnclaveである。、内部にプラットフォームとTCBに固有の公開鍵であるPCKを持つ。中間認証局と似た役割を持ち、PCKを利用してECDSA AttestationのAttestation Keyに対して証明書の役割を持つ構造体を発行する。PCK証明書を発行できるのはIntelのみであり、これによりCPUが本物でTCBが適切かどうかを確認できる。

- Quoting Enclave (QE3)  
Quoteを生成するためのEnclaveであり、Attestation Keyを利用できる。EPID AttestationのQEとは異なり、こちらはオープンソースであることから、QEと区別してQE3と呼ばれる。QE3のアイデンティティ情報は、Intelが運営するサービスであるIntel Provisioning Certification Service (PCS)から取得できる。

- Quote Verification Enclave (QvE)  
ECDSA Attestationにおいて、Quoteを検証するために利用できるEnclaveである。QvEのアイデンティティ情報はPCSから取得できる。

- Reference Launch Enclave (ref-LE)  
サードパーティが定めたホワイトリストやルールに基づいてEnclaveの起動可否を決めるためのEnclaveである。ref-LEはIntel以外のサードパーティが署名して利用する事ができ、その内容も変更する事ができる。

# プロビジョニング
プロビジョニングでは、アテステーションに使う鍵と証明書、つまりトラストチェーンの一部を構築する。これにはPCEとQE3を利用する。プロビジョニングフローを以下に示す。

<img src ="https://github.com/hatena75/ECDSA-Attestation-explain/blob/master/img/Provisioning.svg" width="400px">

1. QE3はAttestation Keyを生成する。
2. QE3はPCEとLocal Attestationを行った後、PCEのMRENCLAVEを入力としてReportを生成し、それとAttestation Keyを合わせてPCEに送信する。
3. PCEは受信したReportを検証した後、それらにPCKで署名する。この作成された構造体はAttestation Key Certと呼ばれ、Attestation Keyの証明書の役割を果たす。
4. PCEはAttestation Key CertをQE3に返送する。QE3はこれをEnclave内で保持する。

このフローにより、Attestation Key CertからPCK証明書、Intel CAのルート証明書までのトラストチェーンが確立され、実行時にはQuoteに対するAttestation Keyの署名からIntel CAまでのトラストチェーンを検証できるようになる。これはプラットフォーム内で完結しており、Intelの専用サービスとの通信を行うEPID Attestationのプロビジョニングとは異なる。

# アテステーションプロセス



