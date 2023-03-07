# Tapyrus DID Method仕様書
ドラフト版

初版：2023/02/22

# Abstract
TapyrusをVerifiable Data Registryとして用いる際の、didの採番方法についてまとめた仕様書である。なお、didメソッド名は did:tprsとする。
また、今現在このドキュメントはドラフトであり、いつでも更新される可能性がある。did:tprsを利用する場合は、この仕様がまだ策定中であり、更新される可能性があることを十分に理解して利用しなければならない。

# 1. Introduction

　Tapyrus DID Method(did:tprs)はTapyrus Blockchainを用いてDIDを表現する。これにより、エンタープライズ向けに提供される各種サービスにおいて、そのサービスの対象者に固有のIDを付与することが可能となる。
 did:tprsの第２の目的として、任意のsubjectに対してIDを付与することも可能とする。これにより、いわゆるNFTや任意のドキュメント、画像データなど人格を有しないものに対しても固有のIDが付与でき、Verifiable Credentialの対象とすることが可能となる。
 did:btcrの仕様を基本的には踏襲するが、一部Tapyrusで拡張されている項目については適宜改変する。この仕様書では改変項目を漏れなく定義し、またdid:btcrの項目から改変のない部分についても再定義する。 
 did:tprsはTxref形式を用いて表現する。この制限は、DIDが必ずオンチェーンに記録されなければいけないという多少の不便さを強要することにはなるが、その代わりに、DIDが必ず存在していることが保証されるという大きなメリットが得られる。

# 2. Terminology

   TBD

# 3. Basic Concepts

## 3.1. Txref
**TxRef Piece**

　did:btcrと同様にdid:tprsではBIP-0136, [Bech32 Encoded Transaction Position References.](https://github.com/bitcoin/bips/blob/master/bip-0136.mediawiki) エンコーディングを用いて識別子を組み立てる。
ただし、Tapyrusでは複数のネットワークが存在し、それらはNetwork IDで識別されるためTxrefの対象のパラメータを以下の表3-1に示すものに拡張する。
なお、Reserved BitはVersion以降のbit列がdid:btcrと同じになるように調整するためのものである。将来的な利用については十分な検討が必要。


* 表3-1. Txref対象のパラメータ

| | description | possible type | # ob Bits used | values |
|----|----|----|----|----| 
| Network ID | Tapyrusのネットワークを識別するID。 | Uint32 | 32 | 0 to 4294967296 <br/> ex)Tapyrus API Network = 1 |
| Reserved Bit | bit列の調整のため | Uint8 | 3 | 0 固定 |
| Version | 将来的に利用 | Uint8 | 1 | 0 固定 |
| Block Height | Txを含むBlockの高さ | Uint32 | 24 | Block 0 to Block 16777215 |
| Transaction Index | DIDを含むTxのブロックの中の位置 | Uint16 | 15 | Tx 0 to Tx 32767 |
| Outpoint Index | DIDとなるTxのoutpoint位置 | Uint16 | 15 | Tx 0 to Tx 32767 |



**Encoding Table**

　BIP-0350, Bech32mと同じエンコーディング表を用いて、5bitデータを文字列に変換する。マッピングする32文字は以下の通り。

* 表3-2. 文字列への変換表

| |0|1|2|3|4|5|6|7|
|---|---|---|---|---|---|---|---|---|
|+0|q|p|z|r|y|9|x|8|
|+8|g|f|2|t|v|d|w|0|
|+16|s|3|j|n|5|4|k|h|
|+24|c|e|6|m|u|a|7|l|


**Encoding Algorism**

　表3-1に示すピースについてTxRef文字列に変換するアルゴリズムを示す。

　最初に以下の順序で5bit数値の配列を生成する。

1. Network ID + Reserved Bitの35bitを下位5bitずつ切り出す
2. Block Height + Versionの25bitを下位5bitずつ切り出す
3. Transaction Indexの15bitを下位5bitずつ切り出す
4. Outpoint Indexが0でなければ、15bitを下位5bitずつ切り出す

最後に上記5bit数値配列のchecksumを計算したのち、表3-2のchar表に沿ってそれぞれの値をcharに変換する。

以下のリスト3-1にサンプルコードを示す。

* リスト3-1. 5bit数値の配列生成のサンプルコード

```python
def to_uint5_array(val, length):
   res = []
   for i in range(0, length):
   res.append(val & 0x1f)
   val = val >> 5
   return res

def bech32_encoding(network_id, block_height, tx_index, out_index):
   res = []
   res += to_uint5_array(network_id << 3, 7)
   res += to_uint5_array(block_height << 1, 5)
   res += to_uint5_array(tx_index, 3)
   if out_index > 0:
   res += to_uint5_array(out_index, 3)
   return res
```

**Encoding Examples**

　Network ID #1、Transaction Index #1234、Block Height #456789、Outpoint Index #0の場合のTxRefの例を以下に示す。



| | Decimal Value | Binary Value | # of Bits Used |  Bit Index and Value |
|---|---|---|---|---|
| Network ID | 1 | 00000000<br/>00000000<br/>00000000<br/>00000001<br/> | 32 | (ni31, ni30, ni29, ni28, ni27) = (0, 0, 0, 0, 0)<br/>(ni26, ni25, ni24, ni23, ni22) = (0, 0, 0, 0, 0)<br/>(ni21, ni20, ni19, ni18, ni17) = (0, 0, 0, 0, 0)<br/>(ni16, ni15, ni14, ni13, ni12) = (0, 0, 0, 0, 0)<br/>(ni11, ni10, ni09, ni08, ni07) = (0, 0, 0, 0, 0)<br/>(ni06, ni05, ni04, ni03, ni02) = (0, 0, 0, 0, 0)<br/>(ni01, ni00)                           = (0, 1) |
| Reserved Bit | 0 | 00000000 | 3 | (rb02, rb01, rb00) = (0, 0, 0) |
| Version | 0 | 00000000 | 1 | (vr00) = (0) |
| Block height | 456789 | 00000110<br/>11111000<br/>01010101<br/> | 24 | (bh23, bh22, bh21, bh20)           = (0, 0, 0, 0)<br/>(bh19, bh18, bh17, bh16, bh15) = (0, 1, 1, 0, 1)<br/>(bh14, bh13, bh12, bh11, bh10) = (1, 1, 1, 1, 0)<br/>(bh09, bh08, bh07, bh06, bh05) = (0. 0. 0. 1. 0)<br/>(bh04, bh03, bh02, bh01, bh00) = (1, 0, 1, 0, 1)<br/> |
| Transaction index | 1234 | 00000100<br/>11010010 | 15 | (ti14, ti13, ti12, ti11, ti10) = (0, 0, 0, 0, 1)<br/>(ti09, ti08, ti07, ti06, ti05) = (0, 0, 1, 1, 0)<br/>(ti04, ti03, ti02, ti01, ti00) = (1, 0, 0, 1, 0)<br/> |
| Outpoint index | 0 | 省略 | 省略 | 省略 |

| | ni01 | ni00 | rb02 | rb01 | rb00 | decimal value | encoding char |
|---|---|---|---|---|---|---|---|
| data[0] | 0 | 1 | 0 | 0 | 0 | 8 | g |

| | ni06 | ni05 | ni04 | ni03 | ni02 | decimal value | encoding char |
|---|---|---|---|---|---|---|---|
| data[1] | 0 | 0 | 0 | 0 | 0 | 0 | q |

|  | ni11 | ni10 | ni09 | ni08 | ni07 | decimal value | encoding char |
|---|---|---|---|---|---|---|---|
| data[2] |0 | 0 | 0 | 0 | 0 | 0 | **q** |

| | ni16 | ni15 | ni14 | ni13 | ni12 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[3] | 0 | 0 | 0 | 0 | 0 | 0 | **q** |

| | ni21 | ni20 | ni19 | ni18 | ni17 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[4] | 0 | 0 | 0 | 0 | 0 | 0 | **q** |

| | ni26 | ni25 | ni24 | ni23 | ni22 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[5] | 0 | 0 | 0 | 0 | 0 | 0 | **q** |

| | ni31 | ni30 | ni29 | ni28 | ni27 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[6] | 0 | 1 | 0 | 0 | 0 | 0 | **q** |

| | bh03 | bh02 | bh01 | bh00 | vr00 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[7] | 0 | 1 | 0 | 1 | 0 | 10 | **2** |

|  | bh08 | bh07 | bh06 | bh05 | bh04 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[8] | 0 | 0 | 1 | 0 | 1 | 5 | **9** |

| | bh13 | bh12 | bh11 | bh10 | bh09 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[9] | 1 | 1 | 1 | 0 | 0 | 28 | **u** |

| | bh18 | bh17 | bh16 | bh15 | bh14 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[10] | 1 | 1 | 0 | 1 | 1 | 27 | **m** |

| | bh23 | bh22 | bh21 | bh20 | bh19 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[11] | 0 | 0 | 0 | 0 | 0 | 0 | **q** |

| | ti04 | ti03 | ti02 | ti01 | ti00 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[12] | 1 | 0 | 0 | 1 | 0 | 18 | **j** |

| | ti09 | ti08 | ti07 | ti06 | ti05 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[13] | 0 | 0 | 1 | 1 | 0 | 6 | **x** |

| | ti14 | ti13 | ti12 | ti11 | ti10 | decimal value | encoding char
|---|---|---|---|---|---|---|---|
| data[14] | 0 | 0 | 0 | 0 | 1 | 1 | **p** |

返還後は以下の文字列が得られる。

| | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| decimal | 8 | 0 | 0 | 0 | 0 | 0 | 0 | 10 | 5 | 28 | 27 | 0 | 18 | 6 | 1 |
| char | g | q | q | q | q | q | q | 2 | 9 | u | m | q | j | x | p |

**Checksum**

最後にBIP-0350, Bech32m形式のChecksumを付与する。Checksumとして次の6文字が得られる。
check_sum=['p', 'k', '2', 'f', 'u', 'x']

最終的なTxRefは次の文字列となる。

```
gqqqqqq29umqjxppk2fux
```

**Prefixの排除**

BTCR DIDと同様にTPRS DIDでもBIP-0173で定義されているPrefixを除外する。

Outpoint Index=0の時の表現のあいまいさの排除
BTCR DIDと同様にTPRS DIDでもOutpoint Indexが0の場合は、省略表現のみを用いる。以下に例を挙げる。

* Network ID=1, Block Height=120739, Transaction Index=10, Outpoint Index=0の場合

```
例）Outpoint Indexなし

VALID:
gqqqqqqx6t8q2qqllpff-y

INVALID:
gqqqqqqx6t8q2qqqqqaajauw
```

* Network ID=1, Block Height=120739, Transaction Index=10, Outpoint Index=2の場合

```
例）Outpoint Indexあり

gqqqqqqx6t8q2qqzqqeeq68h
```

DID Resolverは無効なformatを検出した場合はDIDを拒否しなければならない。

## 3.2. TPRS DID Format

TPRS DIDの形式は次の通り。

```
tprs-did  = "did:tprs:" tprs-identifier
             [ ";" did-service ] [ "/" did-path ]
             [ "?" did-query ] [ "#" did-fragment ]

tprs-identifier = TxRef encoded transaction id
```

例）Network ID=1のTapyrus chain上のtxid 67c0ee676221d9e0e08b98a55a8bf8add9cba854f13dda393e38ffa1b982b833の場合。

```
block height 1201739, transaction position 2, tx outpoint index 1なので、

gqqqqqqkytfzzqqpqqzyj5es
```

## 3.3. TPRS DID construction

　TPRS DID トランザクションは、OP_RETURN フィールドを使用する場合と使用しない場合の 2 つの方法で構築できる。トランザクションの OP_RETURN フィールドの有無に関係なく、トランザクション自体から構築されたデフォルト機能が付与され、DIDドキュメントが生成される。

　OP_RETURNフィールドを使用する場合、OP_RETURNにはDID対象となる任意のデジタルデータのHash値を記録する。OP_RETURNフィールドを使用する場合は、そのOutpointはDIDトランザクション Outpoint のindex+1でなければならない。

　TPRS DID トランザクションはUTXO状態のみ有効と見なされる。SPENT（使用済み）状態のTPRS DID トランザクションは無効としなければならない。

## 3.4. Default Capabilities
UNSPENT TPRS DID のトランザクションに対して、DID リゾルバーはトランザクション自体から DID ドキュメントを生成する必要がある。具体的には、トランザクション署名キーに次の機能を付与する。
認証(authentication)
Verifiable Credentialへの署名(assertionMethod)

* 例） DETAILED EXAMPLE OF DEFAULT CAPABILITIES
```
{
   "@context": ["https://www.w3.org/ns/did/v1", "https://w3id.org/security/suites/jws-2020/v1"],
   "id": "did:tprs:gqqq-qqqx-6t8q-2qql-lpff-y",
   "verificationMethod": [
      {
         "id": "did:tprs:gqqq-qqqx-6t8q-2qql-lpff-y#address-0",
         "controller": "did:tprs:gqqq-qqqx-6t8q-2qql-lpff-y",
         "publicKeyJwk": {
            "kty": "EC",
            "crv": "secp256-k1",
            "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
            "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4"
         }
      }
   ],
   "authentication": [
      "did:tprs:gqqq-qqqx-6t8q-2qql-lpff-y#address-0"
   ],
   "assertionMethod": [
      "did:tprs:gqqq-qqqx-6t8q-2qql-lpff-y#address-0"
   ]
}
```

　BTCR DIDと同様にTPRS DIDではトランザクションが未使用の場合に、ドキュメントが生成される。TPRS DIDのトランザクションが使用済み(SPENT)の場合は、トランザクションチェーンをたどり、未使用のtip txを探す必要がある。

## 3.5. Continuation DID Documents
TPRS DIDではBTCR DIDで定義されている、DDOは保有しない。TPRS DIDはトランザクションのみからDID Documentを生成する。以下の点を考慮して、このような拡張性の無い仕様を採用した。
トランザクションチェーンを辿ってtipトランザクションを検索する行為はコストが高く、攻撃ベクターになりやすい。
トランザクションの送信先を固定できない以上、トランザクションチェーンによってDIDが更新可能になると、DIDを奪う攻撃が可能となってしまう。

TPRS DIDではより広範な利用者を想定しているため、セキュリティリスクに重きを置いた設計としている。

# 4. Operations

## 4.1. Creating a DID

　TPRS DIDはこのセクションで説明する通り、Tapyrusトランザクションを作成することで作成される。任意のUTXOがTPRS DIDとして利用可能である。

**略語:**

- Bi = ビットコインアドレス i
- Pi = 公開鍵 i
- Si = 署名鍵 i (または秘密鍵 i)

**最初の TPRS DID の作成:**

[WIP]

1. キーセットの作成 (B0/P0/S0)
2. キーセットの作成 (B1/P1/S1)
3. Tapyrusランザクションを作成します。 
4. 出力: 変更アドレス B1 
5. オプションの出力: OP_RETURN(デジタルデータに対して発行する場合)
6. 署名鍵は S0 であり、トランザクションで公開鍵 P0 が明らかになります 
7. TX0 を発行し、Blockのconfirmationを待ちます。 
8. 確認済みトランザクションTX0のTxRef エンコーディングを取得します。

上記が完了すると、did:tprs:TxRef(TX0) という形式の DID が得られる。

## 4.2. Reading a DID

[WIP]

TPRS DIDからトランザクション参照（did:tprs:TxRef(TX0)のTxRef(TX0)部分）を抽出する。
トランザクションを検索する。outpointはUTXOか？

YES: 有効なDIDである。リゾルバーはDID Documentを生成して返さなければならない。(3.4.を参照)

NO: UTXOを持つtip txを探索する。その後、tip txの公開鍵がVerificationMethodとなる。

## 4.3. Updating a DID

[WIP]


# 5. 検討事項
## 5-1. OP_RETURNフィールドの取り扱い
- OP_RETURNフィールドにデジタルデータのsha256 hash値を刻むことで、デジタルデータのIDをTRPS DIDで表現したい。
- OP_RETURNフィールドの取り扱いはTPRS DIDの範囲外するか？ -> アプリケーションSpecific
-- この場合、リゾルバの挙動が実装依存になる。

## 5-2. マーカー方法
- 通常の支払いに使われないP2PKを用いる。
  - outputがそのままDIDとして使える。
  - 探索性能が高い
  - プライバシーがやや弱い(DIDを逆検索可能)
- P2PKHを用いる。
  - インプットの公開鍵＝DIDに現在マッピングされる公開鍵
  - アウトプットのP2PKH＝↑のDIDのUpdate keyかつ次の公開鍵のハッシュ
  - プライバシーが堅牢(DIDが逆検索不可)
  - 探索が困難。何かしらの追加マーカーが必要かも？