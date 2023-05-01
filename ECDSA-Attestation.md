# 概要

ECDSA Attestationは、従来のEPID Attestationとは異なるSGXのRemote Attestationです。サードパーティがプロビジョニングとアテステーションプロセスを実行可能であり、Intelに権限が集中しないよう設計されているのが特徴です。元々は、以下のようなEPID Attestationが不都合なユースケースを想定して開発されました。

1. ネットワークの大部分を、インターネットベースのサービスにアクセスできない環境(大規模イントラネット)で運用している事業者。 

2. 第三者に信頼性の判断を委託するリスクを嫌う事業者。 

3. 非常に分散して動作する特定のアプリケーションモデル（例：Peer-to-Peerネットワーク）。単一の検証ポイントを使用することは、このモデルにとって最適とは言えない。 

4. EPIDが提供するプライバシー特性と相反する要求がある環境。


しかし、最新のSGXではEPIDが廃止されており、全てのケースでECDSA Attestationを使うことになります。

# ハードウェア鍵とソフトウェア鍵
SGXではeFuseに焼き付けられた2種類のハードウェア鍵が利用され、これらがRoot of Trustになる。これらは主に派生鍵を生成するために利用され、eFuse外には出ない。

- Root Provisioning Key (RPK)  
Intel Key Generation Facility (iKGF)と呼ばれるIntelが所有する安全なオフライン環境下で生成される共通鍵である．この鍵はCPU出荷時に埋め込まれ，Intelは生成された全てのRPKを把握している．また，RPKをセキュリティバージョンナンバー(SVN)の更新回数だけ疑似ランダム関数(PRF)でループさせることにより，TCBと紐づいた中間値を生成することができる．

- Root Sealing Key (RSK)  
CPU内でランダムに生成される共通鍵である．Intelを含む誰もRSKの値は知らないため，この鍵から派生した鍵は模倣が非常に困難になる．

## 派生鍵
ソフトウェア鍵は、ハードウェア鍵から派生して生成される。生成はEGETKEY命令で行い、同一の環境下(同じEnclave内、同じSVN、同じプラットフォーム)であれば何度呼び出しても同一の鍵を得られる．

- **Provisioning Certification Key (PCK)**  
RPKから派生する．この鍵はCPUとTCBを構成する各セキュリティコンポーネントのSVNに固有である．PCEのみがこの鍵を扱うことができ，ECDSA AttestationにおいてAttestation Keyの証明書の役割を持つ構造体を発行するために利用される．

- Report Key  
RPKから派生する．この鍵はReportを生成及び検証する際のCMACに利用され，Local Attestationの要になる．

- Seal Key  
RSKから派生する．この鍵は生成するEnclaveのMRENCLAVEまたはMRSIGNERに固有の値になり，Enclaveのシーリングに利用される．

- EINIT Token  
RPKから派生する．この鍵を扱えるのはLE及びref-LEのみであり，他のEnclaveの起動に必要なLaunch Tokenを生成及び取得するために利用される．ただし現在，Launch Tokenはデフォルトで廃止されており，代替として0埋めされた構造体が渡されている．

## Attestation Key
ECDSA Attestationで利用されるAttestation KeyはECDSA秘密鍵である．これはQE3によって生成及び利用され，PCEによってこの鍵に対する証明書の役割を持つ構造体が発行される．

# 特殊なEnclave
ECDSA Attestationには、特定の用途で使用される特殊なEnclaveがある。

- Provisioning Certification Enclave (PCE)  
ECDSA Attestationのプロビジョニングで利用されるEnclaveで，内部にプラットフォームとTCBに固有の公開鍵であるProvisioning Certification Key (PCK)を持つ．中間認証局と似た役割を持ち，PCKを利用してECDSA AttestationのAttestation Keyに対して証明書の役割を持つ構造体を発行する．PCK証明書を発行できるのはIntelのみであり，これによりCPUが本物でTCBが適切かどうかを確認できる．

- Quoting Enclave (QE3)  
ECDSA AttestationのQuoteを生成するためのEnclaveであり，Attestation Keyを利用できる．Architectural EnclaveのQEと役割は同じだが，それとは異なるEnclaveである．こちらはオープンソースであることから，QEと区別してQE3と呼ばれる．QE3のアイデンティティ情報は，Intelが運営するサービスであるIntel Provisioning Certification Service (PCS)から取得できる．

- Quote Verification Enclave (QvE)  
ECDSA Attestationにおいて，Quoteを検証するために利用できるEnclaveである．QvEのアイデンティティ情報はPCSから取得できる．

- Reference Launch Enclave (ref-LE)  
サードパーティが定めたホワイトリストやルールに基づいてEnclaveの起動可否を決めるためのEnclaveである．ref-LEはIntel以外のサードパーティが署名して利用する事ができ，その内容も変更する事ができる．

# プロビジョニング

# アテステーションプロセス
