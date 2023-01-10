---
title: 【逆向】鲁大师Android跑分结果还原
date: 2023/1/11 00:56:25
tag: [逆向, 教程]
category: [工具使用]
excerpt: 由于鲁大师Android将跑分结果加密后保存在SharedPreferences中，不方便保存数据和批量分析，本文用简单的逆向将加密结果还原
---

> 前提：测试机需root，能导出/data/data/com.ludashi.benchmark/shared_prefs/benchScore.xml
> APP版本：com.ludashi.benchmark: 10.7.3
> 工具：jadx: 1.4.5

## 事因
最近用鲁大师对机器性能进行批量跑分测试，每次都需要将结果截图后对比，十分不方便。通过搜索发现，鲁大师APP将跑分结果加密后保存在/data/data/com.ludashi.benchmark/shared_prefs/benchScore.xml中，但进行了加密，不方便保存数据和批量分析。
``` xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="key_total_score_v4" value="710975" />
    <string name="key_score_v4">D0Io8ZEyB6fvNwRJ1vBjhdDpcrCOLCvfznn3cK2VgCBAVZzLMlnBCP7GCc0AQ2sLnQoKfPw7MbeI9rRw1TW4dwdTiQ0ie7diuONavKMQedwiKkk3TmW9l9sVPDlkXDJdtRbWTVdgIiob2E8l17f5nU5tvLzNPHs3uzCU7sccVwGPQ1VE8/CawM+D4zUoJuzx+KRX5R1t0KFifFOfosAcmrLt6XcC75sI128OcQVyJlL9pch+zhrSCFOfVacqbdpHIVHa32OUqLCF4ZoiIFTmlBWlxUYhGx6rMZYJdXAH+vyG6JPSsa1CNhJduef7Oql+K401LjsFvIsghPQyYJHwG4yktnl8c73t2+Wt3j8q2DXQBLXeqFAGCzaVas3iKS3+cCB8fw+ihadhAw==</string>
    <string name="key_data_v4">D0Iry4MVAJH2PUgcxfNtkMSpL+3+ORXYnDaZJunCwCAIBcKeVAuZQP6MZ55FGhYaiQhMEbhhfOGe4KUsqDumcT1Snkp2LLFwu/5M/fNELZN0eCxiBzzAkcgUMygzXDJftxraVEYhZTcr2FcY7bb80AY/q7fSLi9l4XPE58sHTAmAVU5YtOiRx8ON4iomafPA84Bk+R92gP52f1mAsJlf1q/L6lxSsM9ZujBccBV3aC6i+5k43gzBQwvBJvI8cc9ON0zemDzBvbqF/o51eQisgwiY/0YiWEDDdcpOcWI/zujX8JDQvq9aOEJdst7/K7JvK7wHKDkmqKdw2uM5bJfuCtfvolJmPuqAm7LygHhgh2mPWPbKvkNCXHLYJPPkIQLgdzBkcgqiiKB4XM0avlt7uMWfApVw61iGeaQ00JWZcmuSvGup8ijKsIz+Tq2MtpiAIqd/o9wyiMzk/UR8YKU=</string>
</map>
```
看数据结构像是Base64加密，但用Base64解密后是乱码，估计是进行了位移操作，之前分析某地图SDK的log也用了类似的加密，加密和解密函数通常是同一个。

## 反编译
将APK拖进jadx中发现没加固，简单搜索关键字发现在com.ludashi.benchmark.c.f.c中读取了KEY
``` java
    private c() {
        int k2 = com.ludashi.framework.sp.a.k(f17904i /*key_total_score_v4*/, 0, f17901f);
        this.f17906d = k2;
        h(b(this.f17906d, com.ludashi.framework.sp.a.s(f17903h, "", f17901f)), b(k2, com.ludashi.framework.sp.a.s(f17902g, "", f17901f)));
    }
```
其中b函数就是解密函数，一切来的太简单了, 准备好的frida都没用上。  
解密函数将总分作为密钥，所以总分是明文，加密是先位移再Base64，解密是先Base64再位移，用的都是同一个函数。用ChatGPT搜这个函数是这样解释的：  
> 这段代码看起来是一个类似 LFSR（线性反馈移位寄存器）的加密算法。它通过一个特定的位运算来生成一个伪随机序列，然后用这个序列与原数组进行异或运算来加密原数组。`
``` java
    // com.ludashi.benchmark.c.f.c
    private String b(int i2, String str) {
        return f.f(str, i2, false);
    }

    // com.ludashi.framework.utils.f
    public static String f(String str, int i2, boolean z) {
        byte[] decode;
        if (str == null || TextUtils.isEmpty(str)) {
            return str;
        }
        try {
            if (z) {
                byte[] bytes = str.getBytes("UTF-8");
                g(bytes, i2);
                decode = Base64.encode(bytes, 2);
            } else {
                decode = Base64.decode(str, 2);
                g(decode, i2);
            }
            return new String(decode);
        } catch (Throwable th) {
            com.ludashi.framework.utils.log.d.V(a, "", th);
            return null;
        }
    }

    public static void g(byte[] bArr, int i2) {
        if (bArr == null || bArr.length == 0) {
            return;
        }
        int length = bArr.length;
        for (int i3 = 0; i3 < 64; i3++) {
            i2 <<= 1;
            if (1 == (((((i2 >> 3) & 1) ^ ((i2 >> 15) & 1)) ^ ((i2 >> 23) & 1)) ^ ((i2 >> 7) & 1))) {
                i2 |= 1;
            }
        }
        for (int i4 = 0; i4 < length; i4++) {
            byte b2 = 0;
            for (int i5 = 0; i5 < 8; i5++) {
                if (1 == ((i2 >> 31) & 1)) {
                    b2 = (byte) (b2 | 1);
                }
                b2 = (byte) (b2 << 1);
                i2 <<= 1;
                if (1 == (((((i2 >> 3) & 1) ^ ((i2 >> 15) & 1)) ^ ((i2 >> 23) & 1)) ^ ((i2 >> 7) & 1))) {
                    i2 |= 1;
                }
            }
            bArr[i4] = (byte) (bArr[i4] ^ b2);
        }
    }
```

## 解密 
解密就比较简单了，将上面两段代码复制出来就行，解密结果：
``` json
TotalScore = 710975
{"p_cpu_mono":99638,"p_cpu_dual":101479,"p_cpu_cul":63934,"p_gpu_scene1":142581,"p_gpu_scene2":134853,"p_ram_score":30977,"p_ram_size":69733,"p_db_score":9304,"rom_read_speed":31022,"rom_write_speed":13640,"rom_read_speed_rd":9691,"rom_write_speed_rd":4123,"p_rom_score":58476,"total_point":710975}
{"seqWrite":197,"seqRead":1216,"randRead":105,"randWrite":3,"insert":2709,"select":29,"update":3184,"memoryPerf":751,"memorySizePerf":11592,"monoFloat":693,"monoInt":552,"dualFloat":1973,"dualInt":1701,"cipherPerf":77,"avgFps":4634,"offscreenAvgFps":5187,"width":2340,"height":1080,"avgFps2":4040,"offscreenAvgFps2":5545,"width2":2340,"height2":1080}
```
完整代码在：https://gist.github.com/hqw700/27fac24e7a0a335435a1d3080fb436d3
<script src="https://gist.github.com/hqw700/27fac24e7a0a335435a1d3080fb436d3.js"></script>

