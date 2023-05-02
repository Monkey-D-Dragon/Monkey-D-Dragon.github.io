---
layout: post
read_time: true
show_date: true
title:  Recovery Procedure for an 802.11ac Packet
date:   2023-04-16 16:32:20 -0600
description: Recovery Procedure for an 802.11ac Packet
img: posts/20230416/datu.jpg 
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

matlab的Recovery Procedure for an 802.11ac Packet页面翻译。

# Recovery Procedure for an 802.11ac Packet

https://ww2.mathworks.cn/help/wlan/ug/recovery-procedure-for-an-802-11ac-packet.html

此示例显示对收到的IEEE®802.11ac VHT波形，如何检测数据包并解码payload。接收机从前导码字段中恢复数据包格式参数以对数据进行解码。

## Introduction

在单用户802.11ac数据包中，使用L-SIG和VHT-SIG-a前导字段[^1]将传输参数用信号通知给接收机：

- L-SIG field：包含允许接收机确定分组的传输时间的信息。
- VHT-SIG-A field：包含传输参数，包括调制和编码方案、空时流的数量和信道编码。

在这个例子中，我们检测并解码生成的波形中的数据包，该波形包含具有帧检查序列（FCS）的有效MAC帧。除了信道带宽之外的所有传输参数被假定为未知的，并且因此从每个数据包中的解码的L-SIG和VHT-SIG-A前导字段中检索。检索到的传输配置用于对VHT-SIG-B和VHT数据字段进行解码。此外，还进行了以下分析：

- 显示检测到的数据包的波形。
- 显示检测到的数据包的频谱。
- 显示每个空间流的均衡数据符号的星座。
- 测量每个字段的误差矢量幅度（EVM）。

## Waveform Transmission

在该示例中，本地生成802.11ac VHT单用户波形，但是可以使用捕获的波形。MATLAB®可用于使用仪器控制工具箱从各种仪器获取I/Q数据™ 以及软件定义的无线电平台。

本地生成的波形受到以下影响：

- 3x3 TGAc衰落信道

- 加性高斯白噪声

- 载波频率偏移

为了在本地生成波形，我们配置了VHT数据包格式配置对象。注意，VHT数据包配置对象仅在发送端使用。当数据包被解码时，接收端将动态地制定另一个VHT配置对象。辅助函数[vhtSigRecGenerateWaveform](matlab:edit('vhtSigRecGenerateWaveform.m'))在本地生成受损波形。helper函数中的处理步骤包括：

- 生成有效的MAC帧并将其编码为VHT波形。
- 该波形通过TGac衰落信道模型。
- 载波频率偏移被添加到波形中。
- 将加性高斯白噪声添加到波形中。

```matlab
% VHT link parameters
cfgVHTTx = wlanVHTConfig( ...
    'ChannelBandwidth',    'CBW80', ...
    'NumTransmitAntennas', 3, ...
    'NumSpaceTimeStreams', 2, ...
    'SpatialMapping',      'Hadamard', ...
    'STBC',                true, ...
    'MCS',                 5, ...
    'GuardInterval',       'Long', ...
    'APEPLength',          1052);

% Propagation channel
numRx = 3;                  % Number of receive antennas
delayProfile = 'Model-C';   % TGac channel delay profile

% Impairments
noisePower = -30;  % Noise power to apply in dBW
cfo = 62e3;        % Carrier frequency offset (Hz)

% Generated waveform parameters
numTxPkt = 1;      % Number of transmitted packets
idleTime = 20e-6;  % Idle time before and after each packet

% Generate waveform
rx = vhtSigRecGenerateWaveform(cfgVHTTx, numRx, ...
    delayProfile, noisePower, cfo, numTxPkt, idleTime);
```

## Packet Recovery

要处理的信号存储在变量rx中。恢复数据包的处理步骤包括：

- 数据包被检测到并同步。
- 检测到数据包的格式。
- 提取L-SIG字段并恢复其信息位，以确定以微秒为单位的数据包长度。
- VHT-SIG-A字段被提取并且其信息比特被恢复。
- 从解码的L-SIG和VHT-SIG-A比特中检索数据包格式参数。
- 提取VHT-LTF字段以执行MIMO信道估计，用于解码VHT-SIG-B和VHT数据字段。
- VHT-SIG-B字段被提取并且其信息比特被恢复。
- 提取VHT数据字段，并使用检索到的分组参数来恢复PSDU和VHT-SIG-B CRC比特。

一些前导字段的开始和结束索引取决于信道带宽，但与所有其他传输参数无关。使用具有已知带宽的默认传输配置对象来计算这些索引。

```matlab
cfgVHTRx = wlanVHTConfig('ChannelBandwidth', cfgVHTTx.ChannelBandwidth);
idxLSTF = wlanFieldIndices(cfgVHTRx, 'L-STF');
idxLLTF = wlanFieldIndices(cfgVHTRx, 'L-LTF');
idxLSIG = wlanFieldIndices(cfgVHTRx, 'L-SIG');
idxSIGA = wlanFieldIndices(cfgVHTRx, 'VHT-SIG-A');
```

以下代码配置要处理的对象和变量。

```matlab
chanBW = cfgVHTTx.ChannelBandwidth;
sr = wlanSampleRate(cfgVHTTx);

% Setup plots for example
[spectrumAnalyzer, timeScope, constellationDiagram] = vhtSigRecSetupPlots(sr);

% Minimum packet length is 10 OFDM symbols
lstfLen = double(idxLSTF(2)); % Number of samples in L-STF
minPktLen = lstfLen*5;

rxWaveLen = size(rx, 1);
```

## Front-End Processing

前端处理包括数据包检测、粗载波频率偏移校正、定时同步和细载波频率偏移纠正。while循环用于检测和同步接收到的波形中的数据包。样本偏移量searchOffset用于对数组rx内的元素进行索引，以检测数据包。检测并处理rx中的第一个数据包。如果检测到的数据包同步失败，则样本索引偏移量searchOffset将递增，以移动到rx中已处理的数据包之外。重复此操作，直到成功检测到数据包并对其进行同步。

```matlab
searchOffset = 0; % Offset from start of waveform in samples
while (searchOffset + minPktLen) <= rxWaveLen
    % Packet detection
    pktOffset = wlanPacketDetect(rx, chanBW, searchOffset);

    % Adjust packet offset
    pktOffset = searchOffset + pktOffset;
    if isempty(pktOffset) || (pktOffset + idxLSIG(2) > rxWaveLen)
        error('** No packet detected **');
    end

    % Coarse frequency offset estimation using L-STF
    LSTF = rx(pktOffset + (idxLSTF(1):idxLSTF(2)), :);
    coarseFreqOffset = wlanCoarseCFOEstimate(LSTF, chanBW);

    % Coarse frequency offset compensation
    rx = frequencyOffset(rx,sr,-coarseFreqOffset);

    % Symbol timing synchronization
    LLTFSearchBuffer = rx(pktOffset+(idxLSTF(1):idxLSIG(2)),:);
    pktOffset = pktOffset+wlanSymbolTimingEstimate(LLTFSearchBuffer,chanBW);
    if (pktOffset + minPktLen) > rxWaveLen
        fprintf('** Not enough samples to recover packet **\n\n');
        break;
    end

    % Timing synchronization complete: packet detected
    fprintf('Packet detected at index %d\n\n', pktOffset + 1);

    % Fine frequency offset estimation using L-LTF
    LLTF = rx(pktOffset + (idxLLTF(1):idxLLTF(2)), :);
    fineFreqOffset = wlanFineCFOEstimate(LLTF, chanBW);

    % Fine frequency offset compensation
    rx = frequencyOffset(rx, sr, -fineFreqOffset);

    % Display estimated carrier frequency offset
    cfoCorrection = coarseFreqOffset + fineFreqOffset; % Total CFO
    fprintf('Estimated CFO: %5.1f Hz\n\n', cfoCorrection);

    break; % Front-end processing complete, stop searching for a packet
end
```

输出为：

![image-20230416150856672](https://user-images.githubusercontent.com/115327603/232280075-a7236d24-89fe-43ae-9556-2f6b9931a5ce.png)

## Format Detection

使用紧跟在L-LTF之后的三个OFDM符号来检测数据包的格式。需要对从L-LTF获得的信道和噪声功率进行估计。

```matlab
% Channel estimation using L-LTF
LLTF = rx(pktOffset + (idxLLTF(1):idxLLTF(2)), :);
demodLLTF = wlanLLTFDemodulate(LLTF, chanBW);
chanEstLLTF = wlanLLTFChannelEstimate(demodLLTF, chanBW);

% Estimate noise power in non-HT fields
noiseVarNonHT = wlanLLTFNoiseEstimate(demodLLTF);


% Detect the format of the packet
fmt = wlanFormatDetect(rx(pktOffset + (idxLSIG(1):idxSIGA(2)), :), ...
    chanEstLLTF, noiseVarNonHT, chanBW);
disp([fmt ' format detected']);
if ~strcmp(fmt,'VHT')
    error('** A format other than VHT has been detected **');
end
```

输出为：

![image-20230416150953451](https://user-images.githubusercontent.com/115327603/232280092-be7f82ca-52d7-49ae-97ff-63f8c5d107a2.png)

## L-SIG Decoding

在VHT传输中，L-SIG字段用于确定分组的接收时间或RXTIME。RXTIME是使用L-SIG有效载荷的字段比特来计算的[1等式22-105]。然后可以计算在rx内包含分组的样本的数量。使用从L-LTF获得的信道和噪声功率的估计来解码L-SIG有效载荷。

```matlab
% Recover L-SIG field bits
disp('Decoding L-SIG... ');
[rxLSIGBits, failCheck, eqLSIGSym] = wlanLSIGRecover(rx(pktOffset + (idxLSIG(1):idxLSIG(2)), :), ...
    chanEstLLTF, noiseVarNonHT, chanBW);

if failCheck % Skip L-STF length of samples and continue searching
    disp('** L-SIG check fail **');
else
    disp('L-SIG check pass');
end

% Measure EVM of L-SIG symbol
EVM = comm.EVM;
EVM.ReferenceSignalSource = 'Estimated from reference constellation';
EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK');
rmsEVM = EVM(eqLSIGSym);
fprintf('L-SIG EVM: %2.2f%% RMS\n', rmsEVM);

% Calculate the receive time and corresponding number of samples in the
% packet
lengthBits = rxLSIGBits(6:17);
RXTime = ceil((bit2int(double(lengthBits),12,false) + 3)/3) * 4 + 20; % us
numRxSamples = RXTime * 1e-6 * sr; % Number of samples in receive time

fprintf('RXTIME: %dus\n', RXTime);
fprintf('Number of samples in packet: %d\n\n', numRxSamples);
```

![image-20230416151037642](https://user-images.githubusercontent.com/115327603/232280125-bffec2a6-09a7-4c98-9a3d-94be29b44ef0.png)

对于计算的RXTIME和相应数量的样本，显示rx内检测到的数据包的波形和频谱。

```matlab
sampleOffset = max((-lstfLen + pktOffset), 1); % First index to plot
sampleSpan = numRxSamples + 2*lstfLen;         % Number of samples to plot
% Plot as much of the packet (and extra samples) as we can
plotIdx = sampleOffset:min(sampleOffset + sampleSpan, rxWaveLen);

% Configure timeScope to display the packet
timeScope.TimeSpan = sampleSpan/sr;
timeScope.TimeDisplayOffset = sampleOffset/sr;
timeScope.YLimits = [0 max(abs(rx(:)))];
timeScope(abs(rx(plotIdx ,:)));

% Display the spectrum of the detected packet
spectrumAnalyzer(rx(pktOffset + (1:numRxSamples), :));
```

![image-20230416151117314](https://user-images.githubusercontent.com/115327603/232280138-b118ad82-b5e7-4c37-96db-94249a6329e7.png)

![image-20230416151127278](https://user-images.githubusercontent.com/115327603/232280149-d2220011-e81a-423b-baf6-532f40f7fdf9.png)

## VHT-SIG-A Decoding

VHT-SIG-A字段包含数据包的传输配置。使用从L-LTF获得的信道和噪声功率估计来恢复VHT-SIG-A比特。

```matlab
% Recover VHT-SIG-A field bits
disp('Decoding VHT-SIG-A... ');
[rxSIGABits, failCRC, eqSIGASym] = wlanVHTSIGARecover(rx(pktOffset + (idxSIGA(1):idxSIGA(2)), :), ...
    chanEstLLTF, noiseVarNonHT, chanBW);

if failCRC
    disp('** VHT-SIG-A CRC fail **');
else
    disp('VHT-SIG-A CRC pass');
end

% Measure EVM of VHT-SIG-A symbols for BPSK and QBPSK modulation schemes
release(EVM);
EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK');
rmsEVMSym1 = EVM(eqSIGASym(:,1));
release(EVM);
EVM.ReferenceConstellation = wlanReferenceSymbols('QBPSK');
rmsEVMSym2 = EVM(eqSIGASym(:,2));
fprintf('VHT-SIG-A EVM: %2.2f%% RMS\n', mean([rmsEVMSym1 rmsEVMSym2]));
```

输出为：

![image-20230416151220277](https://user-images.githubusercontent.com/115327603/232280161-57c2a4cd-6cb8-41de-a938-d31b689fd474.png)

helper函数[helperVHTConfigRecover](matlab:edit('helperVHTConfigRecover.m'))基于恢复的VHT-SIG-a和L-SIG位返回VHT格式配置对象cfgVHTRx。解码波形不需要的财产被设置为[`wlanVHTConfig`](https://ww2.mathworks.cn/help/wlan/ref/wlanvhtconfig.html)对象的默认值，因此可能与cfgVHTTx中的值不同。此类财产的示例包括NumTransmitAntennas和SpatialMapping。

```matlab
% Create a VHT format configuration object by retrieving packet parameters
% from the decoded L-SIG and VHT-SIG-A bits
cfgVHTRx = helperVHTConfigRecover(rxLSIGBits, rxSIGABits);

% Display the transmission configuration obtained from VHT-SIG-A
vhtSigRecDisplaySIGAInfo(cfgVHTRx);
```

输出为：

![image-20230416151320696](https://user-images.githubusercontent.com/115327603/232280167-d1903f38-7dce-419f-a49d-cc1af682aa21.png)

VHT-SIG-A提供的信息允许计算接收波形内的后续字段的位置。

```matlab
% Obtain starting and ending indices for VHT-LTF and VHT-Data fields
% using retrieved packet parameters
idxVHTLTF  = wlanFieldIndices(cfgVHTRx, 'VHT-LTF');
idxVHTSIGB = wlanFieldIndices(cfgVHTRx, 'VHT-SIG-B');
idxVHTData = wlanFieldIndices(cfgVHTRx, 'VHT-Data');

% Warn if waveform does not contain whole packet
if (pktOffset + double(idxVHTData(2))) > rxWaveLen
    fprintf('** Not enough samples to recover entire packet **\n\n');
end
```

## VHT-SIG-B Decoding

VHT-SIG-B的主要用途是在多用户数据包中用信号通知用户信息。在单个用户分组中，VHT-SIG-B携带数据包的长度，该长度也可以使用L-SIG和VHT-SIG-a来计算（在上面的部分中示出）。尽管不需要对单个用户数据包进行解码，但VHT-SIG-B在下面被恢复并且比特被解释。使用从VHT-LTF获得的MIMO信道估计来解调VHT-SIG-B符号。注意，VHT-SIG-B的CRC是在VHT数据字段中携带的。

```matlab
% Estimate MIMO channel using VHT-LTF and retrieved packet parameters
demodVHTLTF = wlanVHTLTFDemodulate(rx(pktOffset + (idxVHTLTF(1):idxVHTLTF(2)), :), cfgVHTRx);
[chanEstVHTLTF, chanEstSSPilots] = wlanVHTLTFChannelEstimate(demodVHTLTF, cfgVHTRx);

% The L-LTF OFDM demodulator normalizes the output by the number of
% subcarriers. The VHT-SIG-B OFDM demodulator normalizes the output by the
% number of subcarriers and space-time streams. Therefore, estimate the
% noise power in VHT-SIG-B by scaling the L-LTF noise estimate by the ratio
% of the number of subcarriers in both fields, and the number of space-time
% streams.

numSTSTotal = sum(cfgVHTRx.NumSpaceTimeStreams, 1);
scalingFactor = (height(demodVHTLTF)/size(demodLLTF, 1))*numSTSTotal;
noiseVarVHT = noiseVarNonHT*scalingFactor;

% VHT-SIG-B Recover
disp('Decoding VHT-SIG-B...');
[rxSIGBBits, eqSIGBSym] = wlanVHTSIGBRecover(rx(pktOffset + (idxVHTSIGB(1):idxVHTSIGB(2)),:), ...
    chanEstVHTLTF, noiseVarVHT, chanBW);

% Measure EVM of VHT-SIG-B symbol
release(EVM);
EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK');
rmsEVM = EVM(eqSIGBSym);
fprintf('VHT-SIG-B EVM: %2.2f%% RMS\n', rmsEVM);

% Interpret VHT-SIG-B bits to recover the APEP length (rounded up to a
% multiple of four bytes) and generate reference CRC bits
[refSIGBCRC, sigbAPEPLength] = helperInterpretSIGB(rxSIGBBits, chanBW, true);
disp('Decoded VHT-SIG-B contents: ');
fprintf('  APEP Length (rounded up to 4 byte multiple): %d bytes\n\n', sigbAPEPLength);
```

输出为：

![image-20230416151500030](https://user-images.githubusercontent.com/115327603/232280181-fc3c93f4-61df-4638-b49c-6d0a706e8a4d.png)

## VHT Data Decoding

然后可以使用重建的VHT配置对象来恢复VHT数据字段。这包括VHT-SIG-B CRC比特和PSDU。

然后可以根据需要对恢复的VHT数据符号进行分析。在该示例中，显示每个空间流的恢复的VHT数据符号的均衡星座。

```matlab
% Extract VHT Data samples from the waveform
vhtdata = rx(pktOffset + (idxVHTData(1):idxVHTData(2)), :);

% Estimate the noise power in VHT data field
noiseVarVHT = vhtNoiseEstimate(vhtdata, chanEstSSPilots, cfgVHTRx);

% Recover PSDU bits using retrieved packet parameters and channel
% estimates from VHT-LTF
disp('Decoding VHT Data field...');
[rxPSDU, rxSIGBCRC, eqDataSym] = wlanVHTDataRecover(vhtdata, chanEstVHTLTF, noiseVarVHT, cfgVHTRx, ...
    'LDPCDecodingMethod', 'norm-min-sum');

% Plot equalized constellation for each spatial stream
refConst = wlanReferenceSymbols(cfgVHTRx);
[Nsd, Nsym, Nss] = size(eqDataSym);
eqDataSymPerSS = reshape(eqDataSym, Nsd*Nsym, Nss);
for iss = 1:Nss
    constellationDiagram{iss}.ReferenceConstellation = refConst;
    constellationDiagram{iss}(eqDataSymPerSS(:, iss));
end

% Measure EVM of VHT-Data symbols
release(EVM);
EVM.ReferenceConstellation = refConst;
rmsEVM = EVM(eqDataSym(:));
fprintf('VHT-Data EVM: %2.2f%% RMS\n', rmsEVM);
```

输出为：

![image-20230416151557024](https://user-images.githubusercontent.com/115327603/232280191-3fcfad8f-b7e9-4086-bd19-dd5c82e4b902.png)

然后将在VHT数据中恢复的VHT-SIG-B的CRC比特与本地生成的参考进行比较，以确定VHT-SIGB和VHT数据服务比特是否已经成功恢复。

```matlab
% Test VHT-SIG-B CRC from service bits within VHT Data against
% reference calculated with VHT-SIG-B bits
if ~isequal(refSIGBCRC, rxSIGBCRC)
    disp('** VHT-SIG-B CRC fail **');
else
    disp('VHT-SIG-B CRC pass');
end
```

MAC帧中的FCS可以使用[`wlanMPDUDecode`](https://ww2.mathworks.cn/help/wlan/ref/wlanmpdudecode.html)进行验证。当VHT格式帧被恢复时，PSDU包含a-MPDU。MPDU是使用[`wlanAMPDUDeaggregate`](https://ww2.mathworks.cn/help/wlan/ref/wlanampdudeaggregate.html)从A-MPDU中提取的。

```matlab
mpduList = wlanAMPDUDeaggregate(rxPSDU, cfgVHTRx);
fprintf('Number of MPDUs present in the A-MPDU: %d\n', numel(mpduList));
```

mpduList包含MPDU的取消聚合列表。列表中的每个MPDU被传递给wlanMPDUDecode，后者验证FCS并解码MPDU。

```matlab
for i = 1:numel(mpduList)
    [macCfg, payload, decodeStatus] = wlanMPDUDecode(mpduList{i}, cfgVHTRx, ...
                                                    'DataFormat', 'octets');
    if strcmp(decodeStatus, 'FCSFailed')
        fprintf('** FCS failed for MPDU-%d **\n', i);
    else
        fprintf('FCS passed for MPDU-%d\n', i);
    end
end
```

## Selected Bibliography

1. IEEE Std 802.11™-2020. IEEE Standard for Information Technology - Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks - Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications.
