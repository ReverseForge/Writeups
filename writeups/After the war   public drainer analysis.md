# After the war   public drainer analysis 
> **Researchers:** [Mi0r4](https://github.com/miora-sora)  & [Mehrdoost](https://github.com/Mehrdoost)


One of the reasons why, instead of working on vulnerabilities during my day job, I spend my free time on malware analysis is that I’m looking for the vulnerabilities being used in the wild. The good thing about this process is that the exploits and their exploitation structure are fully prepared — it’s just that the behavior might get detected; otherwise, everything we need is right there for us.

There are plenty of active malware samples, and contrary to what we used to think, they’re not doing anything so extremely sophisticated that would make them impossible to analyze as malware. They’re often files where, just by looking at them, you’d immediately think, “This is definitely malware.” But they do use interesting techniques, and one of those might be the use of 1-day or even 0-day vulnerabilities.

Everything started when I received a forwarded message from someone, containing a link that might have been using browser exploits, for me to investigate.


![yooo](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_184209_x2l9.png)

Did we reach the treasure chest?

For now, when we try to open the link, the site doesn’t load anymore. But previously, it had a page for connecting a wallet.


The site is no longer accessible, but just so you know, after checking it out, I realized it’s not just a simple drainer.

It tries to steal during the sale and transfer process.

![yooo](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_185001_3iin.png)


As you can see, it’s heavily obfuscated


![yooooooo with more ooooo](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_191817_9g59.png)

Sooo in here we have some debug traps for diffrent part of the code to try protect the code from debugging on browser


![and as you can see we have the part of the wallet address](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_193202_10j6.png)


```
list_data.indexOf("0xbbAD2bf2FA3af37a3b6bc1")
585 

as hex -> 249 search for 249 usage for array :)
```


![this is the main part we need to find the rest of attackers wallet](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_193851_seky.png)


in line 1 we have the global ``` const _0x4642f0 = _0x216e;``` so after analysis we see the structure the something like the function to call the _0x216e and get the return value we make it more clear


```
//this is the  more cleaner named const _0x4642f0 = _0x216e

const function_arraycaller = array_indexer_func; 
 
function define_and_return_array() {
    const key_array = ['owAfw', ';\x20samesite=Strict;\x20path......
    define_and_return_array = function() {
        return key_array;
    };
```

and after that so its just get index and pass the value to the function in array 0x194 to get it 

```
function array_indexer_func(_0x5aad83, _0x1ce30c) {
    const array_we_have = define_and_return_array();
    array_indexer_func = function(index, _0x15ed39) {
        index = index - 0x194;
        let _0x575abf = array_we_have[index];
        return _0x575abf;
    };
    return array_indexer_func(_0x5aad83, _0x1ce30c);
}
```



oh
```
list_data[194]
'{}.constructor("return t' 
```

i guess this dude is dumb 


```
array_indexer_func(0x2f1)
"BLgMd" 
```
insted of writing the value of hex 21f he wrote the 2f1 wtf broooooo????!!!!!!!


```
after the patch the anti debug i saw this true 
```


After some simple analysis, we’re able to find the wallet address value. But we realize that the attacker made a mistake when trying to concatenate and join two separate parts of his own wallet address into an array — he used the wrong indexes


```
what the attacker wants:
class wallet_part_of_it {
    static[function_arraycaller(0x3db)] = function_arraycaller(0x35d);
    static['contractAddress'] = function_arraycaller(0x2f1) + 'F20748948EBD875A76';
    static['contractAbi'] = [function_arraycaller(0x2c3) + function_arraycaller(0x357)];
    static['rpcUrl'] = 'https://bsc-dataseed1.ni' + function_arraycaller(0x334);


wallet = 0xbbAD2bf2FA3af37a3b6bc1F20748948EBD875A76




but what he got = BLgMdF20748948EBD875A76
```

in the last month i ask to my self why bro dont have any transactions and i saw this 


![i dont know](https://raw.githubusercontent.com/ReverseForge/Writeups/writeup-draft/assets/screenshot_2025-07-11_202058_6en0.png)


sequenceDiagram
    Malware->>Wallet: approve(attacker_address)
    Wallet->>Malware: Signature
    Malware->>C2: Exfil signature
    Malware->>Blockchain: transferFrom(victim, attacker)


more information:

 https://bb628184.nicoin.io  
https://bsc-dataseed1.ni


script:/7a6fd93a/index.js	Injects malicious logic


ABI : approve(), transferFrom(), setApprovalForAll().
