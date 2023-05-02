---
layout: post
read_time: true
show_date: true
title:  wlanHESUConfig
date:   2023-04-18 16:32:20 -0600
description: wlanHESUConfig
img: posts/20230418/datu.jpg 
tags: [matlab, WIFI]
author: Geoffrey Hou
category: matlab
mathjax: yes
toc: yes # leave empty or erase for no TOC
---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

matlab wlanHESUConfig页面翻译，熟悉wifi吧。

# wlanHESUConfig
https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021
## Description

wlanHESUConfig 对象是 WLAN HE 单用户 (HE SU) 和 HE 扩展范围单用户 (HE ER SU) 数据包格式的配置对象。

## Creation

`cfgHESU = wlanHESUConfig` creates a configuration object that initializes parameters for an IEEE® 802.11™ HE SU [PPDU](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_9b35d668-a229-40ce-8ddd-a87365a591dc). For a detailed description of the HE WLAN formats, see [[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021).

`cfgHESU = wlanHESUConfig(Name,Value)` sets properties using one or more name-value pairs. Enclose each property name in quotation marks. For example, `wlanHESUConfig('GuardInterval',1.6)` specifies a 1.6 microsecond guard interval (cyclic prefix) duration.

At runtime, the calling function validates object settings for properties relevant to the operation of the function.

## Properties

- `ChannelBandwidth` ：PPDU 传输的信道带宽，指定为以下值之一：
	- 'CBW20' – 20 MHz 的信道带宽
	- 'CBW40' – 40 MHz 的信道带宽
	- 'CBW80' – 80 MHz 的信道带宽
	- 'CBW160' – 160 MHz 的信道带宽

- `ExtendedRange`：启用 HE ER SU 格式，指定为数字或逻辑值 1 (true) 或 0 (false)。 要创建 HE ER SU 格式配置对象，请将此属性设置为 1 (true)。

	- 约束：**此属性仅在您将 ChannelBandwidth 属性设置为“CBW20”时适用。**

- `Upper106ToneRU`：启用更高频率的 106-tone RU，指定为数字或逻辑值 1 (true) 或 0 (false)。 要在 HE ER SU 传输的主要 20 MHz 信道带宽内仅使用更高频率的 106-tone RU，请将此属性设置为 1（true）。

	- 约束：**仅当您将 ChannelBandwidth 属性设置为“CBW20”并将 ExtendedRange 属性设置为 1 (true) 时，此属性才适用。**

- `InactiveSubchannels`：指示 HE sounding null data packet (NDP) 中的非活动 20 MHz 子信道，指定为数字或逻辑 0（false）或至少一个元素设置为 0（true）的逻辑向量。 当指定向量时，元素以绝对频率的升序对应于子信道。 每个元素指示相应的 20 MHz 子信道是否处于非活动状态。 要指示不活动的 20 MHz 子信道，请将相应的元素设置为 1（true）。 如果将此属性设置为 0 (false)，则 wlanHESUConfig 对象将该值应用于所有 20 MHz 子通道，表示所有子通道均处于活动状态。

	- 例如：[0 0 0 1]表示HE sounding null data packet (NDP)中的绝对频率最高的子信道是没有激活的。
	- 约束：**要启用此属性，请将 ChannelBandwidth 属性设置为“CBW80”或“CBW160”，并将 APEPLength 属性设置为 0。**

- `NumTransmitAntennas`：发射天线数，指定为正整数。

- `PreHECyclicShifts`：波形的pre-HE fields的附加发射天线的循环移位值（以纳秒为单位）。 前八个天线使用 [[1\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211-2020)的表 21-10 中指定的循环移位值。 其余 L 个天线使用您在此属性中指定的值，其中$L=$ NumTransmitAntennas -8。将此属性指定为以下值之一：

	- 区间 [–200, 0] 中的整数 – wlanHESUConfig 对象将此循环移位值用于$L$个附加天线中的每一个。
	- 区间 [–200, 0] 中长度为$L$的整数行向量 – wlanHESUConfig 对象使用第$k$个元素作为第$k+8$个发射天线的循环移位值。
	- 约束：**要启用此属性，请将 NumTransmitAntennas 属性设置为大于 8 的值。**

- `NumSpaceTimeStreams`：传输中的时空流数，指定为区间 [1, 8] 中的整数。

- `SpatialMapping`：空间映射方案，指定为“Direct”、“Hadamard”、“Fourier”或“Custom”。

	- 约束：**默认值“Direct”仅在您将 NumTransmitAntennas 和 NumSpaceTimeStreams 属性设置为相同值时适用。**

- `SpatialMappingMatrix`：空间映射矩阵，指定为下列值之一：

	- 复值标量。 该值适用于所有子载波。

	- 大小为$N_{\mathrm{STS}} \times N_{\mathrm{T}}$的复值矩阵，其中：$N_{\mathrm{STS}}$是时空流数，$N_{\mathrm{T}}$是发送天线数。在这种情况下，空间映射矩阵适用于所有子载波。

	- 大小为$N_{\mathrm{ST}} \times N_{\mathrm{STS}} \times N_{\mathrm{T}}$的复数值 3-D 数组，其中$N_{\mathrm{ST}}$是占用的子载波数。 ChannelBandwidth 属性决定了$N_{\mathrm{ST}}$的值。 在这种情况下，每个占用的子载波都有自己的空间映射矩阵。此表显示了 ChannelBandwidth 设置和相应的$N_{\mathrm{ST}}$​：

		| ChannelBandwidth | $N_{\mathrm{ST}}$ |
		| ---------------- | ----------------- |
		| 'CBW20'          | 242               |
		| 'CBW40'          | 484               |
		| 'CBW80'          | 996               |
		| 'CBW160'         | 1992              |

	使用此属性旋转和缩放星座映射器的输出向量。 空间映射矩阵用于在发射天线上进行波束成形和混合时空流。 调用函数对每个子载波的空间映射矩阵进行归一化。

	- 约束：**仅当您将 SpatialMapping 属性设置为“Custom”时，此属性才适用。**

- `Beamforming`：使用波束成形启用传输信号，指定为数字或逻辑 1 (true) 或 0 (false)。 要应用波束成形控制矩阵，请将此属性设置为 1 (true)。 SpatialMappingMatrix 属性指定波束形成控制矩阵。

	- 约束：**仅当您将 SpatialMapping 属性设置为“Custom”时，此属性才适用。**

- `PreHESpatialMapping`：启用 PPDU 的 pre-HE-short-training-field (pre-HE-STF) 部分的空间映射，指定为数字或逻辑值 1 (true) 或 0 (false)。 要以与每个音调上 HE-LTF 的第一个符号相同的方式在空间上映射 PPDU 的前 HE-STF 部分，请将此属性设置为 1（真）。 要不对 PPDU 的 HE-STF 之前部分应用空间映射，请将此属性设置为 0（假）。

- `STBC`：启用 PPDU 数据字段的空时块编码 (STBC)，指定为数字或逻辑值 1 (true) 或 0 (false)。 STBC 通过分配的天线传输数据流的多个副本。

	- 当您将此属性设置为 0 (false) 时，STBC 不会应用于数据字段。 时空流的数量等于空间流的数量。
	- 当您将此属性设置为 1 (true) 时，STBC 将应用于数据字段。 时空流的数量是空间流数量的两倍。
	- 约束：**此属性仅在 NumSpaceTimeStreams 属性为 2 且 DCM 属性为 0 (false) 时适用。**

- `MCS`：用于传输当前数据包的调制和编码方案 (MCS)，指定为区间 [0, 11] 中的非负整数。 下表显示了每个 MCS 有效值的调制类型和编码率：![image-20230418164542325](https://user-images.githubusercontent.com/115327603/232736636-103c3d9c-14e0-4ee3-97fa-bc8272573bd7.png)

	- 约束：**当您将 ExtendedRange 设置为 1 (true) 时，您只能将此属性设置为 0、1 或 2。**
	- 约束：**当您将 Upper106ToneRU 设置为 1 (true) 时，您只能将此属性设置为 0。**

- `DCM`：双载波调制 (DCM) 指示符，指定为数字或逻辑值 1 (true) 或 0 (false)。 要指示 DCM 用于 HE-Data 字段，请将此属性设置为 1 (true)。

	- 约束：**当满足所有这些条件时，您只能将此属性设置为 1 (true)：**
		- MCS 属性为 0、1、3 或 4。
		- STBC 属性为 0（false）。
		- NumSpaceTimeStreams 属性小于或等于 2。

- `ChannelCoding`：HE-Data 字段的前向纠错 (FEC) 编码类型，指定为低密度奇偶校验 (LDPC) 编码的“LDPC”或二进制卷积编码 (BCC) 的“BCC”。
	- 约束：**当满足所有这些条件时，您只能将此属性设置为“BCC”：**
		- MCS 属性不是 10 或 11。
		- 任何 RU 的大小小于或等于 242。使用 ruInfo 对象函数获取 RU 大小。
		- NumSpaceTimeStreams 属性小于或等于 4。

- `APEPLength`：聚合 MPDU (A-MPDU) pre-end-of-frame (EOF 前) padding (APEP) 长度，以字节为单位，指定为区间 [0, 6451631] 中的整数。 将此属性设置为 0 指定传输 HE NDP。该对象使用此属性来确定数据字段中 OFDM 符号的数量。 有关详细信息，请参阅[[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021)。

- `GuardInterval`：数据包内数据字段的保护间隔（循环前缀）持续时间，以微秒为单位，指定为 3.2、1.6 或 0.8。

- `HELTFType`：HE-LTF HE PPDU的压缩模式，指定为4、2或1。该属性表示HE-LTF的类型，其中值4、2或1分别对应四倍、两倍或一倍HE -LTF持续时间压缩模式，分别。 [[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021) 的表 27-1 将 HE-LTF 类型列举为：

	- 1 × HE-LTF – 持续时间为 3.2 μs，保护间隔持续时间为 0.8 μs 或 1.6 μs
	- 2 × HE-LTF – 持续时间为 6.4 μs，保护间隔持续时间为 0.8 μs 或 1.6 μs
	- 4 × HE-LTF – 持续时间为 12.8 μs，保护间隔持续时间为 0.8 μs 或 3.2 μs

	有关 HE-LTF 的更多信息，请参阅[[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021)的第 27.3.11.10 节。

- `UplinkIndication`：上行链路传输指示符，指定为数字或逻辑值 1 (true) 或 0 (false)。 要指示 PPDU 是在下行链路传输上发送的，请将此属性设置为 0 (false)。 要指示 PPDU 是在上行链路传输上发送的，请将此属性设置为 1 (true)。

- `BSSColor`：基本服务集 (BSS) 颜色标识符，指定为区间 [0, 63] 中的整数。
- `SpatialReuse`：空间重用指标，指定为区间 [0, 15] 内的整数。
- `TXOPDuration`：传输机会 (TXOP) 保护的持续时间信息，指定为区间 [0, 127] 中的整数。 HE-SIG-A字段的TXOP子字段除了第一位指定TXOP长度粒度外，每一位都等于TXOPDuration。 因此，必须根据[[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021)的表 27-18 中列出的过程转换以微秒为单位的持续时间。
- `HighDoppler`：高多普勒模式指示符，指定为数字或逻辑值 1 (true) 或 0 (false)。 要在 HE-SIG-A 字段中指示高多普勒模式，请将此属性设置为 1 (true)。
	- 约束：**仅当 NumSpaceTimeStreams 属性小于或等于任何 RU 的 4 时，此属性的 1（true）值才有效。**
- `MidamblePeriodicity`：HE-Data 字段的中Midamble周期，以 OFDM 符号数表示，指定为 10 或 20。
	- 约束：**此属性仅在 HighDoppler 属性为 1（true）时适用。**
- `NominalPacketPadding`：Nominal packet padding，以微秒为单位，指定为 0、8 或 16。wlanHESUConfig 对象使用此属性和前向纠错 (pre-FEC) 填充因子来计算数据包的持续时间 TPE 扩展 (PE) 字段。 有关数据包扩展字段的更多信息，请参阅[[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021) 的第 27.3.12 节。此表显示了此属性和 $a$的不同值的$T_{\mathrm{PE}}$的可能值，它由 [[2\]](https://ww2.mathworks.cn/help/wlan/ref/wlanhesuconfig.html#mw_12ce61d2-4b46-4827-9281-e65b2413e9ba_sep_mw_biblio_80211ax-2021)的等式 (27-83) 或 (27-84) 定义。![image-20230418170055045](https://user-images.githubusercontent.com/115327603/232736757-edb9048b-383d-44d0-9834-b7ae8d8c8eeb.png)
	- 约束：**要启用此属性，请将 APEPLength 属性设置为区间 [1, 6,500,531] 中的整数。 NDP 的 PE 字段的持续时间，无论Nominal packet padding如何，都是 4 微秒。**
- `PostFECPaddingSource`：wlanWaveformGenerator 函数使用的后 FEC 填充位源，指定为这些值之一：
	- 'mt19937ar with seed' - 使用 mt19937ar 算法生成正态分布的随机位，种子在 PostFECPaddingSeed 属性中指定。
	- 'Global stream' - 使用当前全局随机数流生成正态分布的随机位。
	- 'User-defined' - 使用 PostFECPaddingBits 属性中指定的位作为后 FEC 填充位。
- `PostFECPaddingSeed`：mt19937ar 算法的后 FEC 填充位种子，指定为非负整数。
	- 约束：**要启用此属性，请将 PostFECPaddingSource 属性设置为“mt19937ar with seed”。**
- `PostFECPaddingBits`：FEC 后填充位，指定为二进制值标量或列向量。要生成波形，wlanWaveformGenerator 函数需要 n 位，其中 n 取决于指定的配置。 要计算 n，请使用 getNumPostFECPaddingBits 对象函数，将指定的配置对象作为输入参数，并将此属性指定为长度为 n 的向量。 或者，将此输入指定为二进制值标量或任意长度的列向量。 如果此属性的长度小于 n，则波形发生器循环矢量以创建长度为 n 的矢量。 如果此属性的长度大于 n，则该函数仅使用前 n 个条目作为后 FEC 填充位。
