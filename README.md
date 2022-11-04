# マイナンバーカードを使った認証

本書では、マイナンバーカードを使ってクライアント証明書認証を行うスマホアプリを開発するため、スマートカードの扱いを順をおって説明していきます。

## 前提条件
- 利用PCはMacBook Air M1を想定
- スマホはAndroidとiPhoneを対象とするためFlutterでの開発を想定
- マイナンバーカードを読み込めるスマートカードリーダがPCとUSB接続して認識されていること。安価でMacBookでの操作実績のあるおすすめは「Identiv uTrust 2700 R USB」です。[ここ](https://support.identiv.com/2700r/)からダウンロードしてドライバのインストールが必要です。

## マイナンバーカードについて
- マイナンバーカードには、公的個人認証AP、券面事項補助入力AP、券面事項確認AP、住基APの4機能が入っています。
- 公的個人認証APには、認証用証明書セット（認証用証明書、認証用CA証明書、認証用鍵）と署名用証明書セット（署名証明書、署名用CA証明書、署名用鍵）が入っています。
- 本書ではマイナンバーカードを使った認証を扱いますので、認証用証明書セットを扱います。

## 1. コマンド操作

### ① OpenSC操作
PCにスマートカードリーダを取り付けて、openscコマンドを使った操作を行います。

1. OpenSSLのインストール
    ```
    % brew install openssl
    ```

2. OpenSCのインストール

    下記サイトから最新のdmgインストーラをダウンロードしてインストールします。ここでは0.23.0-rc1をインストールしました。
    https://github.com/OpenSC/OpenSC/releases

    "/usr/local/bin"配下にopensc-xxコマンドがインストールされるのでパスを通し、下記コマンドでバージョンが表示されればインストール完了です。

    ```
    % opensc-tool --version
    ```

2. カード内の全PKCS#15オブジェクトのダンプ

    カード内のすべての PKCS#15 オブジェクトをダンプします。認証用証明書セット(User Authentication)と署名用証明書セット(Digital Signature)が出力されます。「...」には具体的なオブジェクト値が入ります。
    ```
    % pkcs15-tool -D
    Using reader with a card: Identiv uTrust 2700 R Smart Card Reader

    PKCS#15 Card [JPKI]:
        Version        : 0
	    Serial number  : 00000000
	    Manufacturer ID: JPKI
	    Flags          : 
        PIN [User Authentication PIN]

    PIN [User Authentication PIN]
	    Object Flags   : [0x12], modifiable
	    ID             : 01
        ...
    PIN [Digital Signature PIN]
        Object Flags   : [0x12], modifiable
	    ID             : 02
        ...
    Private RSA Key [User Authentication Key]
        ...
        Auth ID        : 01
	    ID             : 01
        ...
    Private RSA Key [Digital Signature Key]
        ...
        Auth ID        : 02
        ID             : 02
        ...
    Public RSA Key [User Authentication Public Key]
        ...
        ID             : 01

    Public RSA Key [Digital Signature Public Key]
        ...
        ID             : 02

    X.509 Certificate [User Authentication Certificate]
        ...
	    ID             : 01
        ...

    X.509 Certificate [Digital Signature Certificate]
        ...
	    ID             : 02
        
    X.509 Certificate [User Authentication Certificate CA]
        ...
	    ID             : 03
        ...

    X.509 Certificate [Digital Signature Certificate CA]
        ...
	    ID             : 04
        ...
    ```
    ここで出力される「User Authentication Certificate」は「認証用証明書」、「Digital Signature Certificate」は「署名用証明書」を表します。

3. PINのロックまでの残回数確認
    認証用証明書と署名用証明書にはPINがかかっており、何回か間違えるとロックされて再設定するまで情報を取り出せなくなります。あと何回間違えることができるかの残回数確認を確認します。
    ```
    % pkcs15-tool --list-pins
    
    PIN [User Authentication PIN]
        Object Flags   : [0x12], modifiable
        ID             : 01
        Flags          : [0x12], local, initialized
        Length         : min_len:4, max_len:4, stored_len:0
        Pad char       : 0x00
        Reference      : 1 (0x01)
        Type           : ascii-numeric
        Tries left     : 3

    PIN [Digital Signature PIN]
        Object Flags   : [0x12], modifiable
        ID             : 02
        Flags          : [0x12], local, initialized
        Length         : min_len:6, max_len:16, stored_len:0
        Pad char       : 0x00
        Reference      : 2 (0x02)
        Type           : ascii-numeric
        Tries left     : 5
    ```
    「Tries left」の後に表示される数字が残回数になります。

4. 認証用証明書の取得

    pkcs15-toolコマンドで認証用証明書(User Authentication Certificate)のID「1」を指定すると認証用証明書をPEM形式で取得できます。
    ```
    % pkcs15-tool --read-certificat 1
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
    ```

    PEM出力をパイプして、「openssl x509 -text -noout」コマンドに通すことによってX509の形式に変換できます。
    ```
    % pkcs15-tool --read-certificat 1 | openssl x509 -text -noout

    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number: ...
        Signature Algorithm: sha256WithRSAEncryption
            Issuer: C=JP, O=JPKI, OU=JPKI for user authentication, OU=Japan Agency for Local Authority Information Systems
            Subject: C=JP, CN=...
            ...
        X509v3 extensions:
        ...
    Signature Algorithm: sha256WithRSAEncryption
    ...
    ```

5. 署名用証明書の取得
    pkcs15-toolコマンドで署名用証明書(Digital Signature Certificate)のID「2」を指定し、PINのID「2」を指定すると、署名用証明書のパスワードの入力が求められ(大文字もしくは数字での組み合わせのパスワード入力が必要です)、正しく入力すると署名用証明書のをPEM形式で取得できます。

    ```
    % pkcs15-tool --read-certificate 2 --verify-pin --auth-id 2
    ```

    ここで取得する証明書はX509v3 Subject Alternative Nameに個人情報が記載されていますが、UTF-8で記載されているためopensslコマンドで確認しても該当部分は<unsupported>と表記されます。

6. 認証用CA証明書の取得

    pkcs15-toolコマンドで認証用CA証明書(User Authentication Certificate CA)のID「3」を指定すると認証用CA証明書をPEM形式で取得できます。
    ```
    % pkcs15-tool --read-certificat 3
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
    ```

    PEM出力をパイプして、「openssl x509 -text -noout」コマンドに通すことによってX509の形式に変換できます。
    ```
    % pkcs15-tool --read-certificat 3 | openssl x509 -text -noout

    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number: ...
        Signature Algorithm: sha256WithRSAEncryption
            Issuer: C=JP, O=JPKI, OU=JPKI for user authentication, OU=Japan Agency for Local Authority Information Systems
            ...
            Subject: Subject: C=JP, O=JPKI, OU=JPKI for user authentication, OU=Japan Agency for Local Authority Information Systems
            ...
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE
        ...
    Signature Algorithm: sha256WithRSAEncryption
    ...
    ```
7. 署名用CA証明書の取得

    pkcs15-toolコマンドで署名用CA証明書(Digital Signature Certificate CA)のID「4」を指定すると署名用CA証明書をPEM形式で取得できます。
    ```
    % pkcs15-tool --read-certificat 4
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
    ```

    PEM出力をパイプして、「openssl x509 -text -noout」コマンドに通すことによってX509の形式に変換できます。
    ```
    % pkcs15-tool --read-certificat 4 | openssl x509 -text -noout

    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number: ...
        Signature Algorithm: sha256WithRSAEncryption
            Issuer: C=JP, O=JPKI, OU=JPKI for digital signature, OU=Japan Agency for Local Authority Information Systems
            ...
            Subject: C=JP, O=JPKI, OU=JPKI for digital signature, OU=Japan Agency for Local Authority Information Systems
            ...
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE
        ...
    Signature Algorithm: sha256WithRSAEncryption
    ...
    ```

8. 証明書の種類と識別子と発行者

    以上のとおり、公的個人認証APに含まれる証明書は下記の通りまとめることができます。

    |ID|名称|英名|発行者<br>(Issuer)|識別子<br>(Subject)|
    | :- | :--- | :- | :- | :- |
    |1|認証用<br>証明書|User<br>Authentication<br>Certificate|C=JP, O=JPKI,<br>OU=JPKI for user authentication,<br>OU=Japan Agency for Local Authority Information Systems|C=JP, CN=...|
    |2|署名用<br>証明書|Digital<br>Signature<br>Certificate|C=JP, O=JPKI,<br>OU=JPKI for digital signature,<br>OU=Japan Agency for Local Authority Information Systems|C=JP, L=XX-ken, L=XX-shi,<br>CN=YYYYMMDDhhmmss(発行要求作成日時)xxxxx(シーケンス番号)XXXXXXXXX(受付窓口識別記号)|
    |3|認証用<br>CA証明書|User<br>Authentication<br>Certificate<br>CA|C=JP, O=JPKI,<br>OU=JPKI for user authentication,<br>OU=Japan Agency for Local Authority Information Systems|C=JP, O=JPKI,<br>OU=JPKI for user authentication,<br>OU=Japan Agency for Local Authority Information Systems|
    |4|署名用<br>CA証明書|Digital<br>Signature<br>Certificate<br>CA|C=JP, O=JPKI,<br>OU=JPKI for digital signature,<br>OU=Japan Agency for Local Authority Information Systems|C=JP, O=JPKI,<br>OU=JPKI for digital signature,<br>OU=Japan Agency for Local Authority Information Systems|

### ② OpenSC+SSH
マイナンバーカードの認証用証明書を使って同一PC上でsshログインを行います。

1. Macでsshdを有効にし、sshログインを行い、ホームディレクトリに「.ssh」フォルダを作成します。

    https://pc-karuma.net/mac-ssh-login/

2. マイナンバーカードからssh-rsaの「User Authentication Public Key」を取得します。

    下記コマンドを実行し、一行目をコピーして、ログイン先ユーザのホームディレクトリ配下の「~/.ssh/authorized_keys」に追記します。なければ作成して600のパーミッションで保存します。

    ```
    % ssh-keygen -D /Library/OpenSC/lib/opensc-pkcs11.so

    ssh-rsa ... User Authentication Public Key
    ssh-rsa ... User Authentication Certificate CA
    ssh-rsa ... Digital Signature Certificate CA
    ```

3. 下記コマンドを実行してsshログインを行います。認証用証明書のPIN入力が求められるので入力するとログインが成功します。

    ```
    % ssh -I /Library/OpenSC/lib/opensc-pkcs11.so [ユーザ名]@localhost

    Enter PIN for 'JPKI (User Authentication PIN)': 
    Last login: ...
    [ユーザ名]@[ホスト名] ~ % 
    ```

### ③ nginxへのクライアント証明書認証
スマホアプリからマイナンバーカードのユーザ認証用証明書を読み取り、nginxにクライアント証明書認証します。

作成中

### 4. keycloakへのクライアント証明書認証
スマホアプリからマイナンバーカードのユーザ認証用証明書を読み取り、keycloakにクライアント証明書認証してアクセストークンを取得します。

作成中

## 参考資料

1. PKCSについて
https://qiita.com/kunichiko/items/7796ecfb88a62ce26b36
2. 