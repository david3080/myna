# マイナンバーカードを使った認証

本書では、マイナンバーカードを使ってクライアント証明書認証を行うスマホアプリを開発するため、スマートカードの扱いを順をおって説明していきます。

## 前提条件
- 利用PCはMacBook Air M1を想定
- スマホはAndroidとiPhoneを対象とするためFlutterでの開発を想定
- マイナンバーカードを読み込めるスマートカードリーダがPCとUSB接続して認識されていること。安価でMacBookでの操作実績のあるおすすめは「Identiv uTrust 2700 R USB」です。[ここ](https://support.identiv.com/2700r/)からダウンロードしてドライバのインストールが必要です。

## マイナンバーカードについて
- マイナンバーカードには、公的個人認証AP、券面事項補助入力AP、券面事項確認AP、住基APの4機能が入っています。
- 公的個人認証APには、署名用証明書セット（署名証明書、署名用CA、署名用鍵）と認証証明書セット（認証証明書、認証用CA、認証用鍵）が入っています。
- 本書ではマイナンバーカードを使った認証を扱いますので、認証証明書セットを扱います。

## 1. コマンド操作

### ① OpenSC操作
PCにスマートカードリーダを取り付けて、openscコマンドを使った操作を行います。

1. OpenSCのインストール

    下記サイトから最新のdmgインストーラをダウンロードしてインストールします。ここでは0.23.0-rc1をインストールしました。
    https://github.com/OpenSC/OpenSC/releases

    "/usr/local/bin"配下にopensc-xxコマンドがインストールされるのでパスを通し、下記コマンドでバージョンが表示されればインストール完了です。

    ```
    % opensc-tool --version
    ```

2. カード内の全PKCS#15オブジェクトのダンプ

    カード内のすべての PKCS#15 オブジェクトをダンプします。署名用証明書セット(Digital Signature)と認証証明書セット(User Authentication)が出力されます。「...」には具体的なオブジェクト値が入ります。
    ```
    % pkcs15-tool -D
    Using reader with a card: Identiv uTrust 2700 R Smart Card Reader

    PKCS#15 Card [JPKI]:
    ...
    PIN [User Authentication PIN]
    ...
    PIN [Digital Signature PIN]
    ...
    Private RSA Key [User Authentication Key]
    ...
    Private RSA Key [Digital Signature Key]
    ...
    Public RSA Key [User Authentication Public Key]
    ...
    Public RSA Key [Digital Signature Public Key]
    ...
    X.509 Certificate [User Authentication Certificate]
    ...
    X.509 Certificate [Digital Signature Certificate]
    ...
    X.509 Certificate [User Authentication Certificate CA]
    ...
    X.509 Certificate [Digital Signature Certificate CA]
    ...
    ```
3. PINのロックまでの残回数確認
    署名証明書と認証証明書にはPINがかかっており、何回か間違えるとロックされて再設定するまで情報を取り出せなくなります。あと何回間違えることができるかの残回数確認を確認します。
    ```
    % pkcs15-tool --list-pins
    Using reader with a card: Identiv uTrust 2700 R Smart Card Reader
    
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

4. 認証証明書の取得

    pkcs15-toolコマンドで認証証明書のID「1」を指定すると認証証明書をPEM形式で取得できます。
    ```
    % pkcs15-tool --read-certificat 1
    Using reader with a card: Identiv uTrust 2700 R Smart Card Reader
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
            ...
        X509v3 extensions:
        ...
    Signature Algorithm: sha256WithRSAEncryption
    ...
    ```

5. 署名用証明書の取得
    pkcs15-toolコマンドで署名用証明書のID「2」を指定し、PINのID「02」を指定すると、署名用証明書のパスワードの入力が求められ(大文字もしくは数字での組み合わせのパスワード入力が必要です)、正しく入力すると署名用証明書のをPEM形式で取得できます。

    ```
    % pkcs15-tool --read-certificate 2 --verify-pin --auth-id 02
    ```

    ここで取得する証明書はX509v3 Subject Alternative Nameに個人情報が記載されていますが、UTF-8で記載されているためopensslコマンドで確認しても該当部分は<unsupported>と表記されます。

### ② OpenSC+SSH
マイナンバーカードのユーザ認証用証明書を使って同一PC上でsshログインを行います。

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

<span style="color: red; ">作成中</span>

### 4. keycloakへのクライアント証明書認証
スマホアプリからマイナンバーカードのユーザ認証用証明書を読み取り、keycloakにクライアント証明書認証してアクセストークンを取得します。

<span style="color: red; ">作成中</span>
