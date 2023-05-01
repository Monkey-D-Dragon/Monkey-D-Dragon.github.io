---
layout: post
read_time: true
show_date: true
title:  NR PUSCH Repetition
date:   2023-03-27 16:32:20 -0600
description: 翻译学习总结下NR PUSCH的Repetition相关协议内容
img: posts/20230327/datu_wesuijixvlie.jpg 
tags: [PUSCH, NR, Repetition]
author: Geoffrey Hou
category: NR
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

翻译学习总结下NR PUSCH的Repetition相关协议内容。

## PUSCH repetition type A
PUSCH repetition type A意思是UE在连续的多个slot上发送同一个TB，连续发送的slot个数用$K$表示，并且在这$K$个slot上时域资源分配完全相同。
相当于，PUSCH的repetition配置结束后就可以$K$次重传，HARQ则需要收到NACK才会重传。
正常的PUSCH传输可以认为是$K=1$的PUSCH repetition type A。
### 什么时候使用PUSCH repetition type A/B
- 通过DCI format 0_1调度的PUSCH，如果pusch-RepTypeIndicatorDCI-0-1配置为’pusch-RepTypeB’，UE采用PUSCH repetition Type B。
- 通过DCI format 0_2调度的PUSCH，如果pusch-RepTypeIndicatorDCI-0-2配置为’pusch-RepTypeB’，UE采用PUSCH repetition Type B。
- 如果不满足上面两个条件，UE采用PUSCH repetition Type A。

意思是如果DCI format 0_1或0_2中指定了才会使用PUSCH repetition Type B，其他一般情况下都是PUSCH repetition Type A。
### PUSCH repetition type A的调度方式
- 通过DCI format 0_1或0_2调度，并且使用C-RNTI，MCS-C-RNTI或CS-RNTI加扰，NDI=1。
- 通过DCI format 0_0调度，并使用TC-RNTI加扰。
- 通过RAR UL grant调度。


### PUSCH repetition type A的重复次数
R15中最大重复次数是8，R16中最大重复次数是16，R17中最大重复次数是32。R16和R17的IE见下图
![image](./assets/img/posts/20230327/chongfucishu.png)
可以看到，R16中numberOfRepetitions可以取：1/2/3/4/7/8/12/16，R17中numberOfRepetitions可以取：1/2/3/4/7/8/12/16/20/24/28/32。
实际上，$K$的取值是这样配置的：如果资源分配表格中存在*numberOfRepetitions*，那么$K=$*numberOfRepetitions*；如果资源分配表格中没有*numberOfRepetitions*，那么如果配置了*pusch-AggregationFactor*，$K=$*AggregationFactor*；否则$K=1$。

### PUSCH repetition type A相关的其他配置
- 会影响冗余版本。
- 只支持层数为1.
- 映射类型可以是PUSCH mapping type A和PUSCH mapping type B。
- 如果需要，可以放弃repetition在该slot上的传输，具体场景见38213/11.1。

### PUSCH repetition type A跳频相关
#### 跳频开关
- 通过DCI format 0_2调度时，*pusch-Config*中的*frequencyHoppingDCI-0-2*配置。
- 通过除了DCI format 0_2的其他format调度时，*pusch-Config*中的*frequencyHopping*配置。
- 对于configured PUSCH transmission，*configuredGrantConfig*中的*frequencyHopping*配置。
- RAR UL grant或者TC-RNTI加扰的DCI format 0_0，有对应的跳频信息字段。

#### 支持的跳频类型
- Intra-slot frequency hopping。适用于单slot和多slot的configured PUSCH transmission，DCI format 0_1或0_2调度的多slotPUSCH传输（如果DCI中配置了*pusch-TimeDomainAllocationListForMultiPUSCH*，那么多个PUSCH传输都通过这一个DCI调度；如果提供了*cg-nrofSlots*和*cg-nrofPUSCH-InSlot*，那么多个configured grant PUSCH transmissions也是使用一个配置）
- Inter-slot frequency hopping。适用于多slot PUSCH。

注意：频域资源分配type 1时，才可能跳频。
#### 跳频偏移的配置
可以分为两种情况，一种是配置一个值*frequencyHoppingOffset*，一种是配置一个偏移值列表*frequencyHoppingOffsetLists*，不过最后都是选择一个偏移值使用。
偏移值列表需要说明：
- 如果激活的BWP大小小于50个PRB，那么可以配置2个偏移值，并选择一个使用。
- 如果激活的BWP大小大于等于50个PRB，那么可以配置4个偏移值，并选择一个使用。

在R17的DCI中有这样一段说明
![image](./assets/img/posts/20230327/DCI.png)

也就是如果*frequencyHoppingOffsetLists*包含两个偏移值，$N_{UL_hop}=1$；如果*frequencyHoppingOffsetLists*包含四个偏移值，$N_{UL_hop}=2$。$N_{UL_hop}$配合下面这个表使用
![image](./assets/img/posts/20230327/Table8.3-1.png)

#### 跳频的方法
对于intra-slot frequency hopping，计算如下：
$$
\mathrm{RB}_{\text {start }}=\left\{\begin{array}{cc}
\mathrm{RB}_{\text {start }} & i=0 \\
\left(\mathrm{RB}_{\text {start}}+\mathrm{RB}_{\text {offset}}\right) \bmod N_{BWP}^{\text {size}} & i=1
\end{array}\right.
$$
$i=0$和$i=1$分别表示第一跳和第二跳。按照上面的跳频偏移值的选择就可以得到两跳的RB位置（注意，这里的RB位置只是起始位置，后面的没说明，但是默认不会超过BWP）。
两跳的符号划分是这样的，第一跳的符号数是$\left\lfloor N_{symb}^{PUSCH,s} / 2\right\rfloor$，第二跳的符号数是$N_{symb}^{PUSCH,s}-\left\lfloor N_{symb}^{PUSCH,s} / 2\right\rfloor$，这里的$N_{symb}^{PUSCH,s}$表示一个slot上分配给PUSCH传输的符号数。

对于inter-slot frequency hopping，如果*PUSCH-DMRS-Bundling*不使能，或者，对于RAR UL grant或TC-RNTI加扰的DCI format 0_0调度的inter-slot frequency hopping，在slot $n_s^\mu$上的起始RB计算为：
$$
\mathrm{RB}_{\text {start }}\left(n_s^\mu\right)=\left\{\begin{array}{cc}
\mathrm{RB}_{\text {start }} & n_s^\mu \bmod 2=0 \\
\left(\mathrm{RB}_{\text {start }}+\mathrm{RB}_{\text {offset }}\right) \bmod N_{B W P}^{s i z e} & n_s^\mu \bmod 2=1
\end{array}\right.
$$
inter-slot frequency hopping就没有符号的划分了，因为是整个slot上的。

对于inter-slot frequency hopping，如果*PUSCH-DMRS-Bundling*使能，并且不满足（RAR UL grant或TC-RNTI加扰的DCI format 0_0调度），在slot $n_s^\mu$上的起始RB计算为：
$$
\mathrm{RB}_{\text {start }}\left(n_s^\mu\right)=\left\{\begin{array}{cc}
\mathrm{RB}_{\text {start }} & \left\lfloor\frac{n_S^\mu}{N_{F H}}\right\rfloor \bmod 2=0 \\
\left(\mathrm{RB}_{\text {start }}+\mathrm{RB}_{\text {offset }}\right) \bmod N_{B W P}^{s i z e} & \left\lfloor\frac{n_S^\mu}{N_{F H}}\right\rfloor \bmod 2=1
\end{array}\right.
$$
$F_{FH}$是高层参数*PUSCH-Frequencyhopping-Interval*的值。

## 时域资源分配
首先有协议上一长串关于时域资源分配的描述。下面内容都是协议的章节编号和翻译。
#### 6.1.2.1 Resource allocation in time domain
当UE被一个DCI或RAR UL grant或fallbackRAR UL grant调度发送一个TB（并且没有CSI上报），或者UE被一个DCI调度发送一个TB（并且有CSI上报）时，该DCI中的字段'*Time domain resource assignment*'的值$m$ 或者 该RAR UL grant或fallbackRAR UL grant中的字段*PUSCH time resource allocation*的值$m$，提供了一个资源分配表的行索引$m+1$（关于为什么是$m+1$？因为DCI中这个字段占用4bit，可以表示0至15，但是协议中定义的行索引是1至16，所以需要加1，不知道为什么直接定义行索引0至15）。确定使用哪个资源分配表的方法见协议38214/6.1.2.1.1。

这个索引行定义了PUSCH传输使用的一些参数，这些参数包括：slot偏移$K_2$，符号的起始和长度指示*SLIV*或直接给出起始符号*S*和符号长度*L*，PUSCH的映射类型，用于TBS determination的slot个数（如果资源分配表中存在参数*numberOfSlotsTBoMS*），和重复次数（如果资源分配表中存在参数*numberOfRepetitions*）。

当UE被一个DCI的'*CSI request*'字段调度发送一个有CSI上报没有TB的PUSCH时，该DCI的'*Time domain resource assignment*'字段值$m$，提供了一个资源分配表的行索引$m+1$。
这个索引行定义了：符号的起始和长度指示*SLIV*或直接给出起始符号*S*和符号长度*L*，PUSCH的映射类型，$K_2$的值通过下面公式得到：
$$
K_2=\max _j Y_j(m+1)
$$
这里的$Y_j, j=0, \ldots, N_{\text {Rep }}-1$是$N_{\text {Rep }}$触发的CSI上报设置，表示下面高层参数的相应列表实例：
- *reportSlotOffsetListDCI-0-2*或者*reportSlotOffsetListDCI-0-2-r17*（如果通过DCI format 0_2调度PUSCH，并且该配置了这个参数）；
- *reportSlotOffsetListDCI-0-1*或者*reportSlotOffsetListDCI-0-1-r17*（如果通过DCI format 0_1调度PUSCH，并且该配置了这个参数）；
- 如果上面两条都不满足，那么就是*reportSlotOffsetList*或者*reportSlotOffsetList-r17*。
上面的参数都在IE *CSI-ReportConfig*中。$Y_j(m+1)$是$Y_j$的第$m+1$个实例。

UE发送PUSCH的slot $K_s$和$K_2$有关，并且通过下面公式计算：
如果至少有一个调度小区的UE配置了参数*ca-SlotOffset*，计算如下：
$$
K_s=\left\lfloor n \right\rfloor
$$
$$
K_s=\left\lfloor n \cdot \frac{2^{\mu P U S C H}}{2^{\mu_{P D C C H}}}\right\rfloor+K_2+\left\lfloor\left(\frac{N_{\text {slot, offset, }, P D C C H}^{C A}}{2^{\mu_0 f f s e t, P D C C H}}-\frac{N_{\text {slot, offset, }, P U S C H}^{C A}}{2^{\mu_0 \text { offet }, \text { PUSCH }}}\right) \cdot 2^{\mu_{P U S C H}}\right\rfloor
$$
否则，计算如下：
$$
K_s=\left\lfloor n \cdot \frac{2^{\mu P U S C H}}{2^{\mu P D C C H}}\right\rfloor+K_2+K_{\text {offset }} \cdot \frac{2^{\mu \text { PUSCH }}}{2^{\mu_{\text {off set }}}}
$$
这里的$K_{\text {offset }}$见38213/4.2，$\mu_{K_{\text {offset }}}$是$K_{\text {offset }}$的子载波间隔配置，对FR1，取0。$n$是调度DCI的slot，$K_2$和PUSCH的参数集有关，$\mu_{\text {PUSCH}}$和$\mu_{\text {PDCCH}}$是PUSCH和PDCCH的子载波间隔配置，并且要求调度的DCI不是由TC-RNTI CRC加扰的DCI format 0_0。

$N_{\text {slot, offset, PDCCH }}^{\mathrm{CA}}$和$\mu_{\text {offset,PDCCH}}$是小区接收到的PDCCH的$N_{\text {slot, offset}}^{\mathrm{CA}}$和$\mu_{\text {offset}}$，由*ca-SlotOffset*配置。对应的PUSCH的参数由发送PUSCH的小区的参数*ca-SlotOffset*配置，见38211/4.5。

在确定时域资源分配时
- 对于DCI format 0_1调度的PUSCH，如果*pusch-RepTypeIndicatorDCI-0-1*配置为'pusch-RepTypeB'，UE采用PUSCH repetition Type B过程。
- 对于DCI format 0_2调度的PUSCH，如果*pusch-RepTypeIndicatorDCI-0-2*配置为'pusch-RepTypeB'，UE采用PUSCH repetition Type B过程。
- 否则，对于PDCCH或RAR UL grant或fallbackRAR UL grant调度的PUSCH，UE采用PUSCH repetition Type A过程。

在确定时域资源分配时
对于DCI format 0_1或DCI format 0_2调度的PUSCH，如果参数*numberOfSlotsTBoMS*存在并且取值大于1，那么UE采用TB processing over multiple slots过程。

对于PUSCH repetition Type A和TB processing over multiple slots，起始符号*S*相对于slot的起始位置，连续符号数*L*从*S*开始计数，*S* *L*和*SLIV*之间的关系为：
if $(L-1) \leq 7$，then
$$
SLIV=14 \cdot(L-1)+S
$$
else
$$
SLIV=14 \cdot(14-L+1)+(14-1-S)
$$
并且$0<L \leq 14-S$。

对于PUSCH repetition Type B，起始符号*S*相对于slot起始位置，连续符号数*L*从*S*开始技术，并且这两个参数由资源分配表中的*startSymbol*和*length*确定（也就是对PUSCH repetition Type B，不能通过*SLIV*的方式配置）。

对于PUSCH repetition Type A和TB processing over multiple slots，PUSCH映射类型可以配置为Type A或Type B，见38211/6.4.1.1.3，由索引行确定。
对于PUSCH repetition Type B，PUSCH映射类型只能是Type B。

这是协议定义合法的起始符号和符号长度组合。
![image](./assets/img/posts/20230327/fuhaochangdu.png)

对于TB processing over multiple slots，当使用C-RNTI，MCS-C-RNTI或CS-RNTI加扰CRC（NDI=1）的DCI format 0_1或0_2调度发送PUSCH时，有以下要求：
- 用于TBS determination的slot个数$N$由*numberOfSlotsTBoMS*指示。
- 如果资源分配表格中存在*numberOfRepetitions*，那么重复次数*K*等于*numberOfRepetitions*；否则$K=1$。
- 当UE支持TB processing over multiple slots的重复时，UE不希望$N \cdot K$超过32。

当UE配置$\mu=5$或6时，UE不希望通过一个或多个DCI在一个slot上调度超过一个PUSCH。

对于PUSCH repetition Type A，当使用C-RNTI，MCS-C-RNTI或CS-RNTI加扰CRC（NDI=1）的DCI format 0_1或0_2调度发送PUSCH时，有以下要求：
- 如果资源分配表格中存在*numberOfRepetitions*，那么重复次数$K$等于*numberOfRepetitions*。
- 或者如果UE配置了*pusch-AggregationFactor*，那么重复次数$K$等于*pusch-AggregationFactor*。
- 否则，$K=1$.
- 用于TBS determination的slot个数$N=1$。

对于PUSCH repetition type A，RAR UL grant调度发送PUSCH时，MCS信息字段的2 MSBs提供了一个codepoint，用来确定重复次数$K$，配合下面的表格使用（用于TBS determination的slot个数$N=1$）。
对于PUSCH repetition type A，TC RNTI加扰CRC的DCI format 0_0调度发送PUSCH时，MCS信息字段的2 MSBs提供了一个codepoint，用来确定重复次数$K$，配合下面的表格使用（用于TBS determination的slot个数$N=1$）。
![image](./assets/img/posts/20230327/Table6.1.2.1-1A.png)

如果配置了*pusch-TimeDomainAllocationListForMultiPUSCH*，UE不希望配置*pusch-AggregationFactor*。

如果UE在pusch *TimeDomainAllocationListForMultiPUSCH*中配置有*extendedK2*，其中一个或多个行包含用于服务小区的UL BWP上的pusch的多个*SLIV*，则UE不应用pusch *AggregationFactor*（如果配置），在服务小区的UL BWP上转换为DCI格式0_1，并且UE不期望在pusch *TimeDomainAllocationListForMultiPUSCH*中配置*numberOfRepetitions*。

如果UE在pusch *TimeDomainAllocationListForMultiPUSCH*中配置有*extendedK2*，其中一个或多个行包含用于服务小区的UL BWP上的pusch的多个*SLIV*，当任意两个UL DCI在同一符号中结束并且DCI中的至少一个调度多个pusch时，其中与DCI相关联的跨度被定义为从第一个调度的PUSCH的开始到最后一个调度的PUSCH的结束。

对于unpaired spectrum（TDD）：
- 当*AvailableSlotCounting*使能的时候，如果$K>1$，那么UE基于基于*tdd-UL-DL-ConfigurationCommon*，*tdd-UL-DL-ConfigurationDedicated*，和*ssb-PositionsInBurst*以及DCI format 0_1或0_2中的TDRA信息字段值，确定$N \cdot K$个slot用于PUSCH传输。
-- 如果使用的资源分配表格的行索引指示的符号，至少有一个符号和该slot上的一个DL符号（由*tdd-UL-DL-ConfigurationCommon*或*tdd-UL-DL-ConfigurationDedicated*指示或由*ssb-PositionsInBurst*指示）重叠，那么这个slot不计算在这$N \cdot K$个slot内。
- 否则，UE基于DCI format 0_1或0_2的TDRA字段确定PUSCH repetition type A的PUSCH传输的$N \cdot K$个连续的slot（注意这里是连续的）。
- UE基于*tdd-UL-DL-ConfigurationCommon*，*tdd-UL-DL-ConfigurationDedicated*和*ssb-PositionsInBurst*以及DCI格式0_1或0_2中的TDRA信息字段值，来确定用于在由DCI格式0_1或0_2调度的TB processing over multiple slots处理的PUSCH的$N \cdot K$个slot。
-- 如果使用的资源分配表格的行索引指示的符号，至少有一个符号和该slot上的一个DL符号（由*tdd-UL-DL-ConfigurationCommon*或*tdd-UL-DL-ConfigurationDedicated*指示或由*ssb-PositionsInBurst*指示）重叠，那么这个slot不计算在这$N \cdot K$个slot内。
- 剩下的还有RAR UL grant和DCI format 0_0 with CRC scrambled by TC-RNTI调度的PUSCH repetition Type A的类似的描述，都是说和DL符号有重叠的时候该slot不在$N \cdot K$个slot计数。
对于paired spectrum（FDD）和SUL band：
- 对于DCI format 0_1或0_2调度的TB processing over multiple slots的PUSCH，UE基于DCI format 0_1或0_2的TDRA字段确定$N \cdot K$个连续的slot。
- 一段对半双工radcap UE的描述，暂时省去。
- 对RAR UL grant和DCI format 0_0 with CRC scrambled by TC-RNTI调度的PUSCH repetition Type A，也是UE基于DCI format 0_1或0_2的TDRA字段确定$N \cdot K$个连续的slot。

If a UE would transmit a PUSCH of PUSCH repetition Type A when AvailableSlotCounting is enabled and K>1 or a TB processing over multiple slots over  slots, and the UE does not transmit the PUSCH of a TB processing over multiple slots or the PUSCH repetition Type A in a slot from the  slots, according to Clause 9, Clause 11.1, Clause 11.2A, Clause 15 and Clause 17.2 of [6, TS 38.213], the UE counts the slots in the number of  slots.

对于PUSCH repetition Type A，如果$K>1$：
- 如果PUSCH通过DCI format 0_1或0_2调度：
-- 如果*AvailableSlotCounting*使能，$N \cdot K$个的slot上全部使用相同的符号分配，并且PUSCH被限制为1层。UE应该在$N \cdot K$个的slot上重复这个TB，并且每个slot上使用相同的符号分配。（也就是这时候，多个slot可能不连续）
-- 否则，$N \cdot K$个连续的slot上全部使用相同的符号分配，并且PUSCH被限制为1层。UE应该在$N \cdot K$个连续的slot上重复这个TB，并且每个slot上使用相同的符号分配。
- 或者如果通过RAR UL grant或DCI format 0_0 with CRC scrambled by TC-RNTI调度PUSCH时，$N \cdot K$个的slot上全部使用相同的符号分配，并且PUSCH被限制为1层。UE应该在$N \cdot K$个的slot上重复这个TB，并且每个slot上使用相同的符号分配。（也就是这时候，多个slot可能不连续）

对于TB processing over multiple slots：
- 和上面类似，只是paired spectrum or supplementary uplink band时连续的slot，其他可能不连续。


对于DCI format 0_1或0_2或0_0 with CRC scrambled by TC-RNTI调度的PUSCH，该TB的第$n$次传输时刻使用的冗余版本由下表确定，其中$\mathrm{n}=0,1, \ldots N \cdot K-1$。
对于RAR UL grant调度的PUSCH，该TB的第$n$次传输时刻使用的冗余版本由下表的第一行确定，其中$\mathrm{n}=0,1, \ldots N \cdot K-1$。
![image](./assets/img/posts/20230327/Table6.1.2.1-2.png)

当在non-initial UL BWP上发送MsgA PUSCH时，如果配置了*startSymbolAndLengthMsgA-PO*，那么UE根据*startSymbolAndLengthMsgA-PO*确定*S*和*L*。
发送MsgA PUSCH时，如果没有配置*startSymbolAndLengthMsgA-PO*，并且如果*PUSCH-ConfigCommon*提供了TDRA列表*PUSCH-TimeDomainResourceAllocationList*，那么UE应该使用*msgA-PUSCH-TimeDomainAllocation*只是使用列表中的哪个值。如果没有提供*PUSCH-TimeDomainResourceAllocationList*，俺么UE应该使用表格6.1.2.1.1-2和6.1.2.1.1-3中的*S*和*L*（*msgA-PUSCH-TimeDomainAllocation*指示使用列表中的哪个值），关于PUSCH传输的时间偏移见38213。

对于PUSCH repetition Type A和TB processing over multiple slots，在multi-slot PUSCH transmission中的一个slot上的PUSCH会被丢掉，见38213/9/11.1/11.2A/15/17.2。




对于PUSCH repetition Type B，除了没有TB数据的CSI上报PUSCH，名义重复个数由*numberOfRepetitions*给出。对于第$n$次名义重复，$n=0, \ldots, numberOfRepetitions-1$, 
- 名义重复开始的slot等于$K_s+\left\lfloor\frac{S+n \cdot L}{N_{s y m b}^{s l o t}}\right\rfloor$，并且相对于该slot起始的起始符号位置等于$\bmod \left(S+n \cdot L, N_{s y m b}^{s l o t}\right)$。
- 名义重复结束的slot等于$K_s+\left\lfloor\frac{S+(n+1) \cdot L-1}{N_{s y m b}^{s l o t}}\right\rfloor$，并且相对于该slot起始的结束符号位置等于$\bmod \left(S+(n+1) \cdot L-1, N_{s y m b}^{s l o t}\right)$。

这里的$K_s$是PUSCH传输开始的slot，$N_{s y m b}^{s l o t}$是每个slot上的符号数。

接下来协议给出了很多PUSCH repetition Type B中，不可用的符号位置场景，例如S slot上的DL符号位置，这里不翻译。

对于PUSCH repetition Type B，在确定$K$个名义重复中所有不可用的符号后，剩下的符号认为是潜在的合法符号位置。如果对于一个名义重复，潜在的合法符号个数大于0，那么该名义重复包含一个或者多个实际重复，每个实际重复包含一个所有潜在的合法符号的连续的集合（在一个slot内的符号）。只有一个符号的实际重复会被丢掉，除非$L=1$。一个实际重复可能会按照协议丢掉，见38213/9/11.1/11.2A/15/17.2。
UE应该在实际重复上重复该TB。第$n$次实际重复使用的冗余版本，根据表格6.1.2.1-2（这里的计数包括被丢掉的实际重复）确定，$N=1$。


对于PUSCH repetition Type B，当UE接收到调度非周期CSI上报的DCI或者激活PUSCH上的半持续CSI上报（没有TB）时，名义重复个数认为是1，不管配置的*numberOfRepetitions*是多少。并且在这个情况下，希望第一个名义重复和第一个实际重复相同。
对于携带半持续CSI上报没有PDCCH的PUSCH repetition Type B，如果第一个名义重复和第一个十几重复不一样，那么第一个名义重复会被丢掉；否则第一个名义重复按照协议丢掉。见38213/9/11.1/11.2A/15/17.2。

对于PUSCH repetition Type B，当UE被调度发送一个TB和非周期CSI上报，那么CSI上报只在第一次实际重复上复用，也就是后面的重复只发送TB。UE不希望第一个实际重复只占用一个符号。

如果*pusch-Config*中的*pusch-TimeDomainAllocationListForMultiPUSCH*包含的行指示了2个到8个连续的PUSCH的资源分配信息，*k2-r16*配置的$K_2$指示了多个PUSCH中第一个PUSCH发送的slot。每个PUSCH有各自的*SLIV*和映射类型。调度的PUSCH个数由DCI format 0_1指示的*pusch-TimeDomainAllocationListForMultiPUSCH*的行中的合法的*SLIV*个数确定。

对于*pusch-Config*中的*pusch-TimeDomainAllocationListForMultiPUSCH*，每个PUSCH有各自的*SLIV*和映射类型。调度的PUSCH个数由DCI format 0_1指示的*pusch-TimeDomainAllocationListForMultiPUSCH*的行中的合法的*SLIV*个数确定。

如果UE配置了*pusch-TimeDomainAllocationListForMultiPUSCH*中的*extendedK2*，其中一个或多个行包含了多个*SLIV*，并且DCI format 0_1指示PUSCH重传，PUSCH对应configured grant Type 1或Type 2，那么UE不希望知识的*SLIV*的个数大于1。



当UE配置了*minimumSchedulingOffsetK2*，那么要是用DCI format 0_1中字段'*Minimum applicable scheduling offset indicator*'指示的最小调度偏移约束。如果UE配置了*minimumSchedulingOffsetK2*，但是没有收到带有'*Minimum applicable scheduling offset indicator*'的DCI，那么UE应该使用'Minimum applicable scheduling offset indicator'值为0对应的约束。
使用最小调度偏移约束时，UE不希望被DCI调度在slot $n$发送C-RNTI，CS_RNTI，MCS-RNTI或SP-CSI-RNTI调度的PUSCH，$K_2$小于$\left\lceil K_{2 \min } \cdot \frac{2^{\mu^{\prime}}}{2^\mu}\right\rceil$，这里的$K_{2 \min }$是使用的最小调度偏移约束。
使用RAR UL grant或fallbackRAR UL grant调度时，或者PUSCH使用TC-RNTI调度时，不使用最小调度偏移约束。最小调度偏移限制变更的应用延迟见5.3.1。




对于PUSCH repetition Type A：
当*srs-ResourceSetToAddModList*或*srs-ResourceSetToAddModListDCI-0-2*配置了两个SRS资源集合时，并且*SRS-ResourceSet*中的*usage*配置为'codebook'或'noncodebook'，如果$K>1$，$K$个连续的slot使用相同的符号分配，并且限制为1层。每个slot中第一个和第二个SRS资源集合的时机按照下面确定：
- 如果DCI format 0_1或DCI format 0_2为*SRS resource set indicator*指示了codepoint "00"，那么第一个SRS资源集合和所有$K$个连续的slot相关。
- 如果DCI format 0_1或DCI format 0_2为*SRS resource set indicator*指示了codepoint "01"，那么第一个SRS资源集合和所有$K$个连续的slot相关。
- 如果DCI format 0_1或DCI format 0_2为*SRS resource set indicator*指示了codepoint "10"，有下面规则：
-- 当$K=2$时，第一个和第二个SRS资源集合分别被用于第一个和第二个2个连续的slot。
-- 当$K>2$时，如果*PUSCH-Config*中的*cyclicMapping*使能，那么第一个和第二个SRS资源集合分别被用于$K$个连续slot的第一个和第二个2个连续的slot，剩下的slot这样重复。
-- 当$K>2$时，如果*PUSCH-Config*中的*sequentialMapping*使能，那么第一个SRS资源集合用于第一和第二个slot，第二个SRS资源集合用于第三和第四个slot，剩下的slot这样重复。
- 如果DCI format 0_1或DCI format 0_2为*SRS resource set indicator*指示了codepoint "11"，有下面规则：
-- 当$K=2$时，第一个和第二个SRS资源集合分别被用于第一个和第二个2个连续的slot。
-- 当$K>2$时，如果*PUSCH-Config*中的*cyclicMapping*使能，那么第一个和第二个SRS资源集合分别被用于$K$个连续slot的第一个和第二个2个连续的slot，剩下的slot这样重复。
-- 当$K>2$时，如果*PUSCH-Config*中的*sequentialMapping*使能，那么第一个SRS资源集合用于第一和第二个slot，第二个SRS资源集合用于第三和第四个slot，剩下的slot这样重复。

对于PUSCH repetition Type B：
当*srs-ResourceSetToAddModList*或*srs-ResourceSetToAddModListDCI-0-2*配置了两个SRS资源集合时，并且*SRS-ResourceSet*中的*usage*配置为'codebook'或'noncodebook'，SRS资源集合和名义PUSCH重复相关，和上一段类型，只是把上面的slot换成名义重复。




对于PUSCH repetition Type A和PUSCH repetition Type B，当DCI format 0_1或DCI format 0_2为*SRS resource set indicator*指示了codepoint "10"或"11"时，第$n$次传输时机（PUSCH repetition Type A）使用的冗余版本$\mathrm{n}=0,1, \ldots K-1$，或者第$n$次实际重复（PUSCH repetition Type B，包含被丢掉的实际传输）根据表格6.1.2.1-2和6.1.2.1-3确定。
For all PUSCH repetitions associated with the SRS resource set of the first transmission occasion or actual repetition, the redundancy version to be applied is derived according to Table 6.1.2.1-2, where n is counted only considering PUSCH transmission occasions or actual repetitions associated with the same SRS resource set as the first transmission occasion or actual repetition.
The redundancy version for PUSCH transmission occasions or actual repetitions that are associated with an SRS resource set other than the SRS resource set of the first transmission occasion or actual repetition is derived according to Table 6.1.2.1-3, where additional shifting operation for each redundancy version is configured by higher layer parameter sequenceOffsetforRV in PUSCH-Config and  is counted only considering PUSCH transmission occasions or actual repetitions that are not associated with the SRS resource set of the first transmission occasion or actual repetition.
![image](./assets/img/posts/20230327/Table6.1.2.1-3.png)

