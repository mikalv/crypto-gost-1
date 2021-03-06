# crypto-gost

A thin Clojure warpper for Bouncycastle library (https://bouncycastle.org) to work with GOST algorithms - Russian cryptographic standards.


This library provides functions to work with: 
* encryption, mac (imito) using GOST 28147-89, GOST 3412-2015
* digest, hmac  using GOST3411-94/2012 256 and 512 bits, 
* signature with GOST3410-2001, GOST3410-2012
* key generation: secret, public/private, password based


## Usage

Add dependencies to your project:

```clojure
:dependencies [[org.bouncycastle/bcprov-jdk15on "1.59"]
               [crypto-gost "0.2.2"]]
```

Note: if you use Oracle JDK then  Bouncycastle jar should be in class path as separate library, not in uberjar, cause its content is signed.
In uberjar Oracle JDK throws java.security.NoSuchProviderException: JCE cannot authenticate the provider BC.
Use Open JDK to avoid problems with uberjar or as separate library in class path with Oracle JDK. 

Add necessary namespaces to ns form:

```clojure
(:require [crypto-gost.common]
            [crypto-gost.encrypt]
            [crypto-gost.digest]
            [crypto-gost.sign])
(:import java.security.SecureRandom)
```

Not all fn's are demonstrated in following examples. Look inside code to get more info. 

### Key generation

To generate secret key for GOST 28147-89 using Java SecureRandom use crypto-gost.encrypt/gen-secret-key.
gen-secret-key returns generated key as bytes array 32 bytes length:

```clojure
  ;; secret key generation
  (let [secret-key (crypto-gost.encrypt/gen-secret-key)
        key-hex (crypto-gost.common/bytes-to-hex secret-key)]
    key-hex)
  ;; => "f60e38bc57b61c8201c7dd70f3859fc6eb6db2c65be7518ea2caef6ae2de35f2"
```

To generate secret key from ^String password use crypto-gost.encrypt/gen-secret-key-from-pwd.
gen-secret-key-from-pwd returns 32 bytes length secret key.

```clojure
 ;; pbe secret key generation
  (let [secret-key (crypto-gost.encrypt/gen-secret-key-from-pwd "MyPassword12")
        key-hex (crypto-gost.common/bytes-to-hex secret-key)]
    key-hex)
;; => "340fb3b14f0d554e683a85c35ae9d67d7002f8c716eac6652f5bfc5f1e764536"
```
Note: calling gen-secret-key-from-pwd with same password always returns same secret key value.

 To generate private and public keypair for GOST3410-2001 use crypto-gost.sign/gen-keypair.

```clojure
 ;;generate GOST 3410-2001 KeyPair, save it to a files "gost-keypair.priv" / "gost-keypair.pub"
 ;;and then load it from files.
  (let [key-pair (crypto-gost.sign/gen-keypair)
        _ (crypto-gost.sign/save-key-pair key-pair "gost-keypair" "MyPassword12")
        key-pair2 (crypto-gost.sign/load-key-pair "gost-keypair" "MyPassword12")]
    key-pair2)
```

To generate private and public keypair for GOST3410-2012 use crypto-gost.sign/gen-keypair-2012.

```clojure
 ;;generate GOST 3410-2012 KeyPair, save it to a files "gost-keypair.priv" / "gost-keypair.pub"
 ;;and then load it from files.
(let [key-pair  (crypto-gost.sign/gen-keypair-2012)
      _         (crypto-gost.sign/save-key-pair key-pair "gost-keypair" "MyPassword12")
      key-pair2 (crypto-gost.sign/load-key-pair-2012 "gost-keypair" "MyPassword12")]
  key-pair2) 
```

### Encryption

This library provides GOST 28147-89 & GOST 3412-2015 CFB encryption mode.
Here is example for GOST 28147-89 encrypt/decrypt operations.

```clojure

  ;; define plain text
  (def m3 "Suppose the original message has length = 50 bytes")

  ;; generate secert key from password
  (def k (crypto-gost.encrypt/gen-secret-key-from-pwd "MyPassword12"))

  ;; this fn generates random iv 
  ;; iv must be random during every encryption of plain text
  ;; encrypted iv should be passed with encrypted text
  (defn gen-iv
    "generate random iv for GOST 28147-89"
    []
    (let [iv-array (byte-array 8)]
      (.nextBytes (SecureRandom.) iv-array)
      iv-array))

  (crypto-gost.common/bytes-to-hex (gen-iv))
  ;; => "933c5e712fa250b6"

  ;; for this example define iv as constant.
  ;; never do it in prod. iv must be random 
  ;; convert hex to bytes array
  (def iv (crypto-gost.common/hex-to-bytes "933c5e712fa250b6"))

  ;;encrypt and decrypt data
  (let [plain-text (.getBytes m3)
        enc-data (crypto-gost.encrypt/encrypt-cfb k iv plain-text)
        dec-data (crypto-gost.encrypt/decrypt-cfb k iv enc-data)]
    (= m3 (String. dec-data)))
    
```

Here some examples for GOST 3412-2015.

```clojure

(def k-3412 (crypto-gost.common/hex-to-bytes "8899aabbccddeeff0011223344556677fedcba98765432100123456789abcdef")) ;;32 bytes
(def iv-3412 (crypto-gost.common/hex-to-bytes "00112233445566778899aabbccddeeff")) ;; 16 bytes

(def m4 "1122334455667700ffeeddccbbaa9988")

(let [plain-text (crypto-gost.common/hex-to-bytes m4)
        enc-data   (crypto-gost.encrypt/encrypt-3412-cfb k-3412 iv-3412 plain-text)
        dec-data   (crypto-gost.encrypt/decrypt-3412-cfb k-3412 iv-3412 enc-data)]
    (assert (= m4 (crypto-gost.common/bytes-to-hex dec-data))))
```

Also you can encrypt/decrypt streaming input and output (File, Socket, URL etc.)

```clojure
;; suppose m3.txt file has a message m3
(let [plain-file "m3.txt"
        enc-file "m3.enc"
        dec-file "m3.dec"]
    (crypto-gost.encrypt/encrypt-stream-cfb k iv plain-file enc-file)
    (crypto-gost.encrypt/decrypt-stream-cfb k iv enc-file dec-file)
    (= (slurp plain-file) (slurp dec-file)))
```

GOST has mac defense against encrypted text modification. Mac is calculated for plain text.
Just send encrypted mac 4 bytes vector with encrypted text and verify mac with decrypted plain text.
If macs are not the same then encrypted text was tampered.
Here is example of mac generation:

```clojure

(let [plain-text (.getBytes m3)
      mac (crypto-gost.encrypt/mac-gen k plain-text)]
    mac)

(let [plain-text (crypto-gost.common/hex-to-bytes m4)
        mac        ((crypto-gost.encrypt/mac-3412-gen k-3412 plain-text)]
    (assert (= "51aa8ebefe937200c21e2518bd4a2edb" ((crypto-gost.common/bytes-to-hex mac))))
```

GOST 3412-2015 produces mac 16 bytes length.

### Digest/HMAC

Now GOST has 3 message digest algorithms: 3411-94, 3411-2012 256 bit, 3411-2012 512 bit.
All of them are available. Here is example how to generate hash (message digest) from message.
While crypto-gost.digest/digest-str accepts ^String data and return ^String digest,
crypto-gost.digest/digest works with binary data. 

```clojure
(let [d1 (crypto-gost.digest/digest-str :3411-94 m3)
      d2 (crypto-gost.digest/digest-str :3411-2012-256 m3)
      d3 (crypto-gost.digest/digest-str :3411-2012-512 m3)]
    [d1 d2 d3])
  ;;  ["c3730c5cbccacf915ac292676f21e8bd4ef75331d9405e5f1a61dc3130a65011"
  ;;   "a3ed85322e1a1479b605a752b1d487fd138863aa1ea67a91e157aa53fce796f3"
  ;;   "275557a47dcfcb8235ff029b74837f0441efe41aed12c207313e83b27abd1e6a9d892713bc30a16bf947d46a59bbbb3a33ee33385391c73675e7d0c360213540"]
```

Also, you can protect data with HMAC which is use seed during digest generation. 
So, if you don't know seed you'll never got correct HMAC value.
Here is example of using HMAC. 

```clojure
  (def hmac-seed "012346789abcdef")
  (let [hmac-hex (crypto-gost.digest/hmac-str :3411-94 m3 hmac-seed)]
    hmac-hex)
  ;; => "3a05abb54a68dc8d22bb4b7d19999ebc10f2fbbcfe0167ceb0ba8fbb80e0250f"
```

### Sign/Verify

This library supports GOST3410-2001. GOST3410-2012 Elliptic curve Russian signature algorithm with sign of 512 bits length.
Here is example of sign generation and verification.

```clojure
(let [kp (crypto-gost.sign/gen-keypair)
        filename "keys-3410"
        pwd "pwd123"
        _ (crypto-gost.sign/save-key-pair kp filename pwd)
        kp2 (crypto-gost.sign/load-key-pair filename pwd)
        hash (crypto-gost.digest/digest :3411-94 (.getBytes m3))
        sign (crypto-gost.sign/sign hash (.getPrivate kp))
        v1 (crypto-gost.sign/verify hash (.getPublic kp) sign)
        v2 (crypto-gost.sign/verify hash (.getPublic kp2) sign)
        sign2 (crypto-gost.sign/sign hash (.getPrivate kp2))
        v3 (crypto-gost.sign/verify hash (.getPublic kp) sign2)]
    (assert (= true v1))
    (assert (= true v2))
    (assert (= true v3)))
    
;; sign and verify with GOST3410-2012

(let [kp   (crypto-gost.sign/gen-keypair-2012) ;;default private key 512 bits length
        hash (crypto-gost.digest/digest :3411-2012-512 (.getBytes m3))
        sign (crypto-gost.sign/sign-2012 hash (.getPrivate kp))
        v1   (crypto-gost.sign/verify-2012 hash (.getPublic kp) sign)]
    (assert (= true v1)))

;; generate, save, load, sign with GOST3410-2012

(let [kp       (sut/gen-keypair-2012)
        filename "keys-3410-2012"
        pwd      "pwd123"
        _        (sut/save-key-pair kp filename pwd)
        kp2      (sut/load-key-pair-2012 filename pwd)
        hash     (crypto-gost.digest/digest :3411-2012-512 (.getBytes m3))
        sign     (sut/sign-2012 hash (.getPrivate kp))
        v1       (sut/verify-2012 hash (.getPublic kp) sign)
        v2       (sut/verify-2012 hash (.getPublic kp2) sign)
        sign2    (sut/sign-2012 hash (.getPrivate kp2))
        v3       (sut/verify-2012 hash (.getPublic kp) sign2)]
    (assert (= true v1))
    (assert (= true v2))
    (assert (= true v3)))
```


## License

Copyright © 2017-2018 Mike Ananev

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
