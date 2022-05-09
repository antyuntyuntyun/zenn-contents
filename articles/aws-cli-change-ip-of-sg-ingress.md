---
title: 'aws-cliから特定のsgのインバウンドルールに登録しているIPを変更'
emoji: '🔥'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'iam', 'cli', 'シェル', 'shell']
published: true
---

## はじめに

VPN 接続設定用の EC2 セキュリティグループが用意されていて、インバウンドルールに接続者の IP をホワイトリストとして登録して運用している環境によく巡り合う。
永遠に在宅勤務で IP 不変であればよいが、よく移動したりとかそもそも IP の変更が定期的に走る環境の場合、都度登録 IP の変更が必要になる。
マネコンからポチポチでも十分楽に設定できるが iam が発行されているとシェルでやりたくなるので、シェルでやってみた。

## .sh

Description が識別子になっていることが基本だと思うので、それに沿って設計・実装。
aws-cli のクレデンシャル設定を済ませておき、ソース冒頭の以下の変数の値を適宜変更することで実行可能。

- SgMyDescription: セキュリティグループに登録されている(登録する予定)の Description
- SgGroupID: 対象のセキュリティグループの ID
- protocols, port: 許可する通信プロトコルとポートの組み合わせ

```bash:aws-change-ip-of-sg-ingress.sh
# !/bin/sh
echo "[ec2 change ingress-ip of security group]"
echo

echo ----------------- SG ------------------
SgMyDescription="antyuntyun"
SgGroupID="sg-xxxxxxxxxx"
# 上下の組み合わせでタプルとして扱う
protocols=("udp" "udp")
ports=(500 4500)

echo Description: $SgMyDescription
echo SgGroupID: $SgGroupID
echo GroupName: $(aws ec2 describe-security-groups --group-ids ${SgGroupID} --output json | jq '.SecurityGroups[].GroupName')
echo Description: $(aws ec2 describe-security-groups --group-ids ${SgGroupID} --output json | jq '.SecurityGroups[].Description')

echo ----------- IP/protcol/port -----------
DescribeSG=$(aws ec2 describe-security-groups --group-id ${SgGroupID} --output json)
# ディスクリプションが一致する既存設定IPがある場合それを取得
IFS=$'\n'; for item in $(echo $DescribeSG | jq -c '.SecurityGroups[].IpPermissions[].IpRanges[]'); do
  if [ $(echo $item | jq -r '.Description') = $SgMyDescription ]; then
    OldMyIp=$(echo $item | jq -r '.CidrIp')
  fi
done
NewMyIp=$(curl -s httpbin.org/ip | jq -r .origin)/32

[ -n "$OldMyIp" ] && echo OldMyIp: $OldMyIp
echo NewMyIp: $NewMyIp;
for ix in ${!protocols[@]}
do
    echo "(protocol:${protocols[ix]} port:${ports[ix]})"
done
echo ---------------------------------------
echo

echo START PROCESS
echo

for ix in ${!protocols[@]}
do
    protocol=${protocols[ix]}
    port=${ports[ix]}

    echo "target:(protocol:${protocols[ix]} port:${ports[ix]})---------"
    # 設定済みIPがある場合は閉鎖
    if [ -n "$OldMyIp" ]; then
        echo "there is old ip: $OldMyIp"
        IpPermissions="IpProtocol=${protocol},FromPort=${port},ToPort=${port},IpRanges=[{CidrIp=${OldMyIp}}]"
        echo "try to revoke old-ip (protocol:${protocol} port:${port}) ..."
        result=$(aws ec2 revoke-security-group-ingress --group-id ${SgGroupID} --ip-permissions ${IpPermissions})
        result_status=$(echo $result | jq -r '.Return')
        if [ result_status="true" ]; then
            echo "\033[34mrevoke ${OldMyIp} udp port:500 success\033[m"
        else
            echo "\033[31mrevoke ${OldMyIp} udp port:500 failed\033[m"
        fi
        echo
    fi
    # 新規IPを許可
    IpPermissions="IpProtocol=${protocol},FromPort=${port},ToPort=${port},IpRanges=[{CidrIp=${NewMyIp},Description=${SgMyDescription}}]"
    echo "try to authorize new-ip (protocol:${protocol} port:${port}) ..."
    result=$(aws ec2 authorize-security-group-ingress --group-id ${SgGroupID} --ip-permissions ${IpPermissions})
    result_status=$(echo $result | jq -r '.Return')
    if [ result_status="true" ]; then
        echo "\033[34mauthorize ${NewMyIp} udp port:500 success\033[m"
    else
        echo "\033[31mauthorize ${NewMyIp} udp port:500 failed\033[m"
    fi
    echo ---------------------------------------
    echo
done

echo END PROCESS
```

## おわりに

aws コマンドの Exception を補足したかったが、try catch しようとすると面倒だったので本日はここまで。
例えば以下の場合にエラーが起こる

- 同一 IP/Protocol/Port で別ディスクリプション名で既に設定がある
- プロトコルとポートの組み合わせについて、欠損や不正な値が含まれる

シェル書くモチベが湧くのは cli でサクッと自動化したいものがあるときだと思うが、上記のようなエラーハンドリングを真面目にやりだすと書くのに時間がかかってしまう。そもそもエラー出たらターミナル見てエラーメッセージ見たらそこから解決はできるし、シェルのエラーハンドリングはそこまで真面目によらなくてよいはず！ということにします。
