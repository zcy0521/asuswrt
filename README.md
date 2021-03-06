# Asuswrt-Merlin

## DDNS

https://github.com/RMerl/asuswrt-merlin.ng/wiki/Custom-DDNS

### 新建`ddns-start`脚本

```shell
vi /jffs/scripts/ddns-start
chmod +x /jffs/scripts/ddns-start
```

### 编辑`ddns-start`脚本

- aliddns

```shell
#!/bin/sh

ACCESS_KEY_ID=""
ACCESS_KEY_SECRET=""
DOMAIN_NAME=""

uuid() {
    echo -n $(cat /proc/sys/kernel/random/uuid)
}

current_time() {
    # -u UTC time
    echo -n $(date -u "+%Y-%m-%dT%H%%3A%M%%3A%SZ")
}

urlencode() {
    local encode_str=""

    while read -n1 c
    do
        case $c in
            [a-zA-Z0-9.~_-]) encode_str="$encode_str$c" ;;
            *) encode_str="$encode_str`printf '%%%02X' "'$c"`" ;;
        esac
    done

    echo -n $encode_str
}

send_request() {
    # Sign https://help.aliyun.com/document_detail/29747.html
    local string_to_sign="GET&%2F&$(echo -n $1 | urlencode)"
    local sign="$(echo -n $string_to_sign | openssl dgst -sha1 -hmac "$ACCESS_KEY_SECRET&" -binary | openssl base64)"

    curl -s "http://alidns.aliyuncs.com/?$1&Signature=$(echo -n $sign | urlencode)"
}

add_domain_record() {
    # Common Args https://help.aliyun.com/document_detail/29745.html
    # AddDomainRecord https://help.aliyun.com/document_detail/29772.html
    # Arg Order: AccessKeyId Action DomainName Format RR SignatureMethod SignatureNonce SignatureVersion Timestamp Type Value Version
    send_request "AccessKeyId=$ACCESS_KEY_ID&Action=AddDomainRecord&DomainName=$1&Format=JSON&RR=$2&SignatureMethod=HMAC-SHA1&SignatureNonce=$(uuid)&SignatureVersion=1.0&Timestamp=$(current_time)&Type=$3&Value=$4&Version=2015-01-09"
}

delete_sub_domain_records() {
    # Common Args https://help.aliyun.com/document_detail/29745.html
    # DeleteSubDomainRecords https://help.aliyun.com/document_detail/29779.html
    # Arg Order: AccessKeyId Action DomainName Format RR SignatureMethod SignatureNonce SignatureVersion Timestamp Version
    send_request "AccessKeyId=$ACCESS_KEY_ID&Action=DeleteSubDomainRecords&DomainName=$1&Format=JSON&RR=$2&SignatureMethod=HMAC-SHA1&SignatureNonce=$(uuid)&SignatureVersion=1.0&Timestamp=$(current_time)&Version=2015-01-09"
}

update_dns_record() {
    local RR="%40"
    local type="A"
    delete_sub_domain_records $1 $RR
    add_domain_record $1 $RR $type $2
}

WAN_IP=${1}
if update_dns_record $DOMAIN_NAME $WAN_IP | grep -q "RecordId"; then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
```

### 在WebUI应用DDNS

- 外部网络(WAN) - DDNS
    - Method to retrieve WAN IP: Internal
    - 服务器: "Custom"
    - 主机名称: "域名"
