# Find backdoor on unifi u6 lr 
> **Researchers:** [Mi0r4](https://github.com/miora-sora)  & [Mehrdoost](https://github.com/Mehrdoost)

firmware format device version:BZ.MT7622_6.6.55+15189.231127.1104
type:unifi u6 lr

in binary blebrd we can see lot of parameter handlers and after get some search we can see the parameter def-password,def-username after some search we can see the controlled params for default credential use hardcoded password and user name to get access to device

in these part for def-username we can see the param_2 defined and after that the def-username use the param_2 offset to get data these is also for def-password  like these .

the blebrd is the web server of the unifi device

so we can login into all devices from ble login  and get access to them:)
with json or like these for example ``` def-password=D_DpOT0_EUlDpOT_E_& def-username=D_DpOT0_EUlDpOT_E_```

```
        00458140    cbz        param_1 ,LAB_00458178
        00458144    adrp       param_2 ,s_D_DpOT0_EUlDpOT_E__00565f58+168       = "D_DpOT0_EUlDpOT_E_"
        00458148     add        param_2 =>s_def-username_00566902 ,param_2 ,#0x902 = "def-username"
        0045814c     mov        param_1 ,x21
```
![so much fun](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/prof_xkmc.png)
