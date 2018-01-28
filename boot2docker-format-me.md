# boot2docker の format-me の話

author
:   Kazuhiro NISHIYAMA

content-source
:   LILO, 東海道らぐ, OpenSUSEMeetUP and 関西Debian 勉強会 LT大会

date
:   2018/01/28

allotted-time
:   10m

theme
:   lightning-simple

# 自己紹介

- Ruby コミッターなど
- Twitter, GitHub: `@znz`

# boot2docker とは?

- Docker 専用軽量ディストリビューション
  - v18.01.0-ce で 45MB の iso
  - https://github.com/boot2docker/boot2docker/releases
- 昔は独自の cli で使用
- 今は Docker Machine が一般的
  - VirtualBox, Hyper-V の時に boot2docker を使用

# 仕組み

- VirtualBox で VM 作成
- ISO から起動
- 仮想 HDD に書き込みが必要なデータを保存
- バージョンアップが ISO の差し替えと再起動だけで可能

# format-me?

- 仮想 HDD を初期化するためのトリック
- https://github.com/boot2docker/boot2docker/blob/master/rootfs/rootfs/etc/rc.d/automount
  - パーティションテーブルがなければ、先頭が `boot2docker, please format-me` なのを確認して HDD をフォーマット

# 作成側 (1) tar を作成

- tar の内容
  - `"boot2docker, please format-me"` というファイル名と内容のファイルが先頭
  - 続いて公開鍵を `".ssh/authorized_keys"` と `".ssh/authorized_keys2"` として追加

# 作成側 (1) 実装箇所
- https://github.com/docker/machine/blob/49dfaa70fdc869c65d9f6c50c355624356ab383b/libmachine/mcnutils/b2d.go#L488
- https://github.com/boot2docker/boot2docker-cli/blob/master/virtualbox/machine.go

# 作成側 (2) VMDK 作成

- `VBoxManage convertfromraw stdin path size --format VMDK`
- 標準入力に tar の内容
- tar そのままをディスクの内容として作成

# 作成側 (2) 実装箇所

- https://github.com/docker/machine/blob/e1a03348ad83d8e8adb19d696bc7bcfb18ccd770/drivers/virtualbox/disk.go
- https://github.com/boot2docker/boot2docker-cli/blob/8a3999640ae7be3493c80a02203eba8c381d2d5c/virtualbox/disk.go

# 気になった点 (1) 疑問点

- tar が偶然 MBR などと誤認識されないか?
  - MBR は先頭 512 バイトの末尾が 55 AA

# 気になった点 (1) tar 調査

- http://www.redout.net/data/tar.html によると tar ヘッダーが 512 バイトで末尾は prefix
- ファイル名は `"boot2docker, please format-me"` なので prefix の末尾2バイトはゼロ

# 気になった点 (1) MBR 調査

- MBR の 55 AA になることはない
- GPT も先頭は MBR 互換なので大丈夫
- パーティションテーブルと誤認識される可能性はなさそう

# 気になった点 (2) 疑問点

- 先頭 4096 バイトを `/userdata.tar` として保存しているのは大丈夫?
  - 内容に関係なく無条件に 4096 バイト

# 気になった点 (2) tar 調査

- http://www.redout.net/data/tar.html によると、末尾にバイナリーゼロ (1024 バイト) がつくらしい
- 末尾を読みすぎるのは大丈夫そう

# 気になった点 (2) 内容

- 公開鍵が大きすぎるとはみ出しそう
  - https://github.com/docker/machine/blob/ab3b7acb9792271dcd349da731150757a9346183/libmachine/ssh/keys.go で自前で 2048 ビット固定の RSA 鍵を作っていたので大丈夫そう
  - カスタマイズして 4096 ビットにするとダメかも (ビルドが面倒そうで試してない)

# 気になった点 (2) 続き

- `authorized_keys2` はもう不要そうなので削っても良さそう
- 楕円曲線暗号の鍵にすれば、小さくできそう

# まとめ

- boot2docker は docker-machine で使われている
- 初回起動時に HDD の先頭をチェックして自動フォーマットしている
