# BIOS設定

ECDSA Attestationでは、BIOSから設定する項目がある。  
現在これに対応しているのは、第三世代Intel Xeonのみである。  

# Flexible Launch Control (FLC)
BIOSの設定項目は、CPUのFlexible Launch Control (FLC)に関するものである。  
従来のEPID Attestationでは、Enclaveの起動可否をハードウェア製のLaunch Enclaveで行っていた。FLCは、サードパーティが独自のLaunch Enclaveを利用できるようにしたもので、これにより独自のEnclave起動制御が可能になる。ただし、この利用にはCPUだけでなくBIOSもFLCをサポートする必要がある。さらに、各設定項目の有無は、各コンピュータのBIOS設定マニュアル等で確認する必要がある。

現在、一部のDell PowerEdgeで利用できることを確認している。

# 設定項目

## SGX LE Public Key Hash(0~3)
起動を許可するLaunch Enclaveの公開鍵のハッシュ値(== Launch EnclaveのMRSIGNER)を登録できる。その他のEnclaveの起動制御は、ここに登録したLaunch Enclaveで行う。

ただし、最新のSGXドライバーでは、任意のEnclaveの起動時にこの値をそのEnclaveのMRSIGNERに書き換えることで、任意のEnclaveが起動できるようになっている。その書き換え可否はEnable writes to SGXLEPUBKEYHASH[3:0] from OS/SWで設定できる。

## Enable writes to SGXLEPUBKEYHASH[3:0] from OS/SW
SGX LE Public Key Hash(0~3)の値を、ソフトウェアが編集する事を許可するか否かを決定する。例えば、SGXドライバーが値を書き換える事がある。

# 設定例

## 任意のEnclaveを利用できるようにする。
Enable writes to SGXLEPUBKEYHASH[3:0] from OS/SWをEnable(Yes)にする。(SGX LE Public Key Hash(0~3)は設定不要)

## 独自のLEで起動制御を行う。
Enable writes to SGXLEPUBKEYHASH[3:0] from OS/SWをDisable(No)にする。SGX LE Public Key Hash(0~3)に独自LEのMRSIGNERを書き込む。
