---
layout: post
read_time: true
show_date: true
title:  Recovery Procedure for an 802.11ax Packet
date:   2023-04-16 16:32:20 -0600
description: Recovery Procedure for an 802.11ax Packet
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
matlab的Recovery Procedure for an 802.11ax Packet页面翻译。


# Recovery Procedure for an 802.11ax Packet

https://ww2.mathworks.cn/help/wlan/ug/recovery-procedure-for-an-802-11ax-packet.html#HESignalRecoveryExample-15
此示例显示如何从接收到的IEEE®802.11ax波形中检测数据包并解码payload比特。接收机从前导字段中恢复数据包格式参数，以对数据字段和MAC帧进行解码。

## Introduction

在802.11ax数据包中，使用L-SIG、HE-SIG-A和HE-SIG-B前导字段将传输参数用信号通知给接收机：

- L-SIG field：包含让接收机确定数据包的传输时间的信息。

- HE-SIG-A field：包含用于HE-MU用户的公共传输参数以及用于HE-SU和HE-EXT-SU数据包的所有传输参数。
- HE-SIG-B field：包含用于HE-MU数据包中的用户的资源单元（RU）分配信息和传输参数。
- L-SIG field中的长度信息、调制类型和HE-SIG-A字段中的OFDM符号数量的组合确定了HE数据包格式。

在这个例子中，我们检测并解码生成的波形中的HE-MU数据包。该示例还可以恢复HE-SU和HE-EXT-SU数据包。除了信道带宽之外的所有传输参数被假定为未知的，并且因此从解码的L-SIG、HE-SIG-A和HE-SIG-B前导字段中恢复传输参数。所恢复的传输参数用于对HE数据字段进行解码。此外，还进行了以下分析：

- 恢复并显示检测到的数据包的波形。
- 恢复并显示检测到的数据包的频谱。
- 显示所有空间流的均衡数据符号的星座图。
- 测量每个字段的误差矢量幅值（EVM）。
- 检测A-MPDU，并为恢复的MAC帧确定帧校验序列（FCS）。
- 显示在子载波上平均的每个数据符号和空间流的EVM。
- 显示在符号上平均的每个数据子载波和空间流的EVM。

## Waveform Transmission

本例使用的是合成的802.11ax HE-MU波形。另外也可以使用其他方式获取的波形。

合成的波形考虑到以下影响：

- 2x2 TGax室内衰落信道

- 加性高斯白噪声

- 载波频率偏移

为了生成HE-MU波形，我们配置了HE-MU格式配置对象 [wlanHEMUConfig](matlab:edit('wlanHEMUConfig.m'))。请注意，[wlanHEMUConfig](matlab:edit('wlanHEMUConfig.m'))配置对象仅用于发送端。接收端将制定一个HE恢复配置对象[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))。对L-SIG、HE-SIG-A和HE-SIG-B字段中的信息位进行解码后，设置HE恢复配置对象的参数。辅助函数[heSigRecGenerateWaveform](matlab:edit('heSigRecGenerateWaveform.m'))生成受损波形。执行以下处理步骤：

- 为MAC帧创建MSDU的随机payload，MAC帧被编码到HE-MU数据包中。
- 该波形通过TGax室内衰落信道模型。
- 载波频率偏移（CFO）和加性高斯白噪声（AWGN）被添加到波形中。

```matlab
% A mixed OFDMA and MU-MIMO configuration is defined for an HE-MU packet.
% The allocation index 17 defines two 52-tone RUs, with one user in each
% RU, and one 106-tone RU. The 106-tone RU has two users in a MU-MIMO
% configuration.
cfgMU = wlanHEMUConfig(17);
cfgMU.NumTransmitAntennas = 2;

% Configure RU 1 and user 1
cfgMU.RU{1}.SpatialMapping = 'Direct';
cfgMU.User{1}.STAID = 1;
cfgMU.User{1}.APEPLength = 1e3;
cfgMU.User{1}.MCS = 5;
cfgMU.User{1}.NumSpaceTimeStreams = 2;
cfgMU.User{1}.ChannelCoding = 'LDPC';

% Configure RU 2 and user 2
cfgMU.RU{2}.SpatialMapping = 'Fourier';
cfgMU.User{2}.STAID = 2;
cfgMU.User{2}.APEPLength = 500;
cfgMU.User{2}.MCS = 4;
cfgMU.User{2}.NumSpaceTimeStreams = 1;
cfgMU.User{2}.ChannelCoding = 'BCC';

% Configure RU 3 and user 1
cfgMU.RU{3}.SpatialMapping = 'Fourier';
cfgMU.User{3}.STAID = 3;
cfgMU.User{3}.APEPLength = 100;
cfgMU.User{3}.MCS = 2;
cfgMU.User{3}.NumSpaceTimeStreams = 1;
cfgMU.User{3}.ChannelCoding = 'BCC';

% Configure RU 3 and user 2
cfgMU.User{4}.STAID = 4;
cfgMU.User{4}.APEPLength = 500;
cfgMU.User{4}.MCS = 3;
cfgMU.User{4}.NumSpaceTimeStreams = 1;
cfgMU.User{4}.ChannelCoding = 'LDPC';

% Specify propagation channel
numRx = 2; % Number of receive antennas
delayProfile = 'Model-D'; % TGax channel delay profile

% Specify impairments
noisePower = -40; % Noise power to apply in dBW
cfo = 62e3; % Carrier frequency offset Hz

% Generate waveform
rx = heSigRecGenerateWaveform(cfgMU,numRx,delayProfile,noisePower,cfo);
```

## Packet Recovery

要处理的信号存储在变量rx中。恢复数据包的处理步骤包括：

- 将创建一个HE恢复配置对象[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))。对象参数随着前导字段的解码而更新。
- 数据包被检测到并同步。
- 对L-LTF进行提取和解调。如[2]第21.3.7.5节所述，解调的L-LTF符号不包括每个20MHz段的tone rotation。
- 解调的L-LTF符号用于信道和噪声估计。
- 使用包含等效于紧接在L-LTF之后的四个OFDM符号的样本的时域信号来确定HE数据包格式。数据包格式在[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))对象中更新。
- 对L-LTF进行解调。解调的L-LTF符号包括每个20MHz段的tone rotation，如[2]第21.3.7.5节所述。L-LTF信道估计（具有tone rotation）用于对pre-HE-LTF进行解码。
- 提取L-SIG和RL-SIG字段。在L-SIG和RL-SIG字段中的每个子信道额外的四个子载波上估计信道。L-LTF信道估计被更新以包括额外子载波上的信道估计。
- L-SIG字段中的信息位被恢复以确定以微秒为单位的数据包长度。
- 在HE-SIG-A解码之后，利用HE-MU数据包的公共传输参数以及HE-SU和HE-EXT-SU数据包的所有传输参数来更新恢复配置对象。
- 对于HE-MU数据包格式，对HE-SIG-B字段进行解码。对于非压缩SIGB波形，首先解码HE-SIG-B公共字段，然后解码HE-SIG-B用户字段。对于压缩的SIGB波形，仅对HE-SIG-B用户字段进行解码。
- 对于没有SIGB压缩的HE-MU格式，从HE-SIG-B字段恢复RU分配和用户传输参数。对于压缩的SIGB波形，从HE-SIG-a字段推断RU分配信息，并且从HE-SIG-B用户字段比特确定用户传输参数。
- [wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))对象是使用HE-SIG-B解码后恢复的每个用户的传输参数创建的。
- 提取并解调HE-LTF字段。解调后的符号用于分配给感兴趣的用户的子载波的信道估计。MIMO信道估计用于对HE数据字段进行解码。
- 提取HE数据字段，并使用每个用户的[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))对象恢复PSDU位。
- 检测恢复的PSDU中的A-MPDU，并检查FCS中恢复的MAC帧。

## Setup Waveform Recovery Parameters

在这个例子中，除了信道带宽之外的所有传输参数都被假设为未知的，并且将被恢复。创建恢复配置对象[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))，以将恢复的信息存储在L-SIG、HE-SIG-A和HE-SIG-B前导字段中。[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))中的传输参数在后续解码前导字段后更新。以下代码配置要处理的对象和变量。

```matlab
chanBW = cfgMU.ChannelBandwidth; % Assume channel bandwidth is known
sr = wlanSampleRate(cfgMU); % Sample rate

% Specify pilot tracking method for recovering the data field. This can be:
%  'Joint' - use joint common phase error and sample rate offset tracking
%  'CPE' - use only common phase error tracking
% When recovering 26-tone RUs only CPE tracking is used as the joint
% tracking algorithm is susceptible to noise.
pilotTracking = 'Joint';

% Create an HE recovery configuration object and set the channel bandwidth
cfgRx = wlanHERecoveryConfig;
cfgRx.ChannelBandwidth = chanBW;

% The recovery configuration object is used to get the start and end
% indices of the pre-HE-SIG-B field.
ind = wlanFieldIndices(cfgRx);

% Setup plots for the example
[spectrumAnalyzer,timeScope,ConstellationDiagram,EVMPerSubcarrier,EVMPerSymbol] = heSigRecSetupPlots(sr);

% Minimum packet length is 10 OFDM symbols
lstfLength = double(ind.LSTF(2));
minPktLen = lstfLength*5; % Number of samples in L-STF

rxWaveLen = size(rx,1);
```

## Front-End Processing

前端处理包括数据包检测、粗载波频率偏移校正、定时同步和细载波频率偏移纠正。while循环用于检测和同步接收到的波形中的数据包。采样点偏移量searchOffset用于索引到rx中以检测数据包。检测并处理rx中的第一个数据包。如果检测到的数据包同步失败，则样本索引偏移量searchOffset将递增，以移动到rx中已处理的数据包之外。重复此操作，直到成功检测到数据包并对其进行同步。

```matlab
searchOffset = 0; % Offset from start of waveform in samples
while (searchOffset + minPktLen) <= rxWaveLen
    % Packet detection
    pktOffset = wlanPacketDetect(rx,chanBW,searchOffset);

    % Adjust packet offset
    pktOffset = searchOffset + pktOffset;
    if isempty(pktOffset) || (pktOffset + ind.LSIG(2) > rxWaveLen)
        error('** No packet detected **');
    end

    % Coarse frequency offset estimation and correction using L-STF
    rxLSTF = rx(pktOffset+(ind.LSTF(1):ind.LSTF(2)), :);
    coarseFreqOffset = wlanCoarseCFOEstimate(rxLSTF,chanBW);
    rx = frequencyOffset(rx,sr,-coarseFreqOffset);

    % Symbol timing synchronization
    searchBufferLLTF = rx(pktOffset+(ind.LSTF(1):ind.LSIG(2)),:);
    pktOffset = pktOffset+wlanSymbolTimingEstimate(searchBufferLLTF,chanBW);

    % Fine frequency offset estimation and correction using L-STF
    rxLLTF = rx(pktOffset+(ind.LLTF(1):ind.LLTF(2)),:);
    fineFreqOffset = wlanFineCFOEstimate(rxLLTF,chanBW);
    rx = frequencyOffset(rx,sr,-fineFreqOffset);

    % Timing synchronization complete: packet detected
    fprintf('Packet detected at index %d\n',pktOffset + 1);

    % Display estimated carrier frequency offset
    cfoCorrection = coarseFreqOffset + fineFreqOffset; % Total CFO
    fprintf('Estimated CFO: %5.1f Hz\n\n',cfoCorrection);

    break; % Front-end processing complete, stop searching for a packet
end

% Scale the waveform based on L-STF power (AGC)
gain = 1./(sqrt(mean(rxLSTF.*conj(rxLSTF))));
rx = rx.*gain;
```

## Packet Format Detection

等效于L-LTF之后的四个OFDM符号的时域样本用于确定HE数据包格式[1图27-63]。对L-LTF进行提取和解调。对于数据包检测，如[2]第21.3.7.5节所述，解调的L-LTF符号不得包括每个20MHz段的tone rotation。解调后的L-LTF用于信道和噪声估计。L-LTF信道（没有tone rotation）和噪声功率估计用于检测数据包格式。

```matlab
rxLLTF = rx(pktOffset+(ind.LLTF(1):ind.LLTF(2)),:);
lltfDemod = wlanLLTFDemodulate(rxLLTF,chanBW);
lltfChanEst = wlanLLTFChannelEstimate(lltfDemod,chanBW);
noiseVar = wlanLLTFNoiseEstimate(lltfDemod);

disp('Detect packet format...');
rxSIGA = rx(pktOffset+(ind.LSIG(1):ind.HESIGA(2)),:);
pktFormat = wlanFormatDetect(rxSIGA,lltfChanEst,noiseVar,chanBW);
fprintf('  %s packet detected\n\n',pktFormat);

% Set the packet format in the recovery object and update the field indices
cfgRx.PacketFormat = pktFormat;
ind = wlanFieldIndices(cfgRx);
```

输出为：
![image-20230416143342477](https://user-images.githubusercontent.com/115327603/232277596-368e730b-a944-4c4e-bfce-20135902cabc.png)


## L-LTF Channel Estimate

对L-LTF进行解调并执行信道估计。解调的L-LTF符号包括每个20MHz段的tone rotation，如[2]第21.3.7.5节所述。L-LTF信道估计（具有tone rotation）用于均衡和解码前HE LTF字段。

```matlab
lltfDemod = wlanHEDemodulate(rxLLTF,'L-LTF',chanBW);
lltfChanEst = wlanLLTFChannelEstimate(lltfDemod,chanBW);
```

## L-SIG and RL-SIG Decoding

L-SIG字段用于确定数据包的接收时间或RXTIME。使用L-SIG payload的长度比特来计算RXTIME。L-SIG和RL-SIG字段被恢复以对L-SIG和RL-SIG字段中的额外子载波执行信道估计。lltfChanEst信道估计被更新以包括L-SIG和RL-SIG字段中的额外子载波上的信道估计。使用从L-LTF字段获得的信道和噪声功率的估计来解码L-SIG payload。wlanHERecoveryConfig中的L-SIG长度属性在L-SIG解码后更新。

```matlab
disp('Decoding L-SIG... ');
% Extract L-SIG and RL-SIG fields
rxLSIG = rx(pktOffset+(ind.LSIG(1):ind.RLSIG(2)),:);

% OFDM demodulate
helsigDemod = wlanHEDemodulate(rxLSIG,'L-SIG',chanBW);

% Estimate CPE and phase correct symbols
helsigDemod = wlanHETrackPilotError(helsigDemod,lltfChanEst,cfgRx,'L-SIG');

% Estimate channel on extra 4 subcarriers per subchannel and create full
% channel estimate
preheInfo = wlanHEOFDMInfo('L-SIG',chanBW);
preHEChanEst = wlanPreHEChannelEstimate(helsigDemod,lltfChanEst,chanBW);

% Average L-SIG and RL-SIG before equalization
helsigDemod = mean(helsigDemod,2);

% Equalize data carrying subcarriers, merging 20 MHz subchannels
[eqLSIGSym,csi] = preHESymbolEqualize(helsigDemod(preheInfo.DataIndices,:,:), ...
    preHEChanEst(preheInfo.DataIndices,:,:),noiseVar,preheInfo.NumSubchannels);

% Decode L-SIG field
[~,failCheck,lsigInfo] = wlanLSIGBitRecover(eqLSIGSym,noiseVar,csi);

if failCheck
    disp(' ** L-SIG check fail **');
else
    disp(' L-SIG check pass');
end
% Get the length information from the recovered L-SIG bits and update the
% L-SIG length property of the recovery configuration object
lsigLength = lsigInfo.Length;
cfgRx.LSIGLength = lsigLength;

% Measure EVM of L-SIG symbols
EVM = comm.EVM;
EVM.ReferenceSignalSource = 'Estimated from reference constellation';
EVM.Normalization = 'Average constellation power';
EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK');
rmsEVM = EVM(eqLSIGSym);
fprintf(' L-SIG EVM: %2.2fdB\n\n',20*log10(rmsEVM/100));

% Calculate the receive time and corresponding number of samples in the
% packet
RXTime = ceil((lsigLength + 3)/3) * 4 + 20; % In microseconds
numRxSamples = round(RXTime * 1e-6 * sr);   % Number of samples in time

fprintf(' RXTIME: %dus\n',RXTime);
fprintf(' Number of samples in the packet: %d\n\n',numRxSamples);
```

输出为：
![image-20230416143440542](https://user-images.githubusercontent.com/115327603/232277625-9437cf07-2c8d-449d-bca6-c850549b1c2f.png)


在给定计算的RXTIME和相应的样本数量的情况下，显示rx内检测到的数据包的波形和频谱。

```matlab
sampleOffset = max((-lstfLength + pktOffset),1); % First index to plot
sampleSpan = numRxSamples + 2*lstfLength; % Number samples to plot
% Plot as much of the packet (and extra samples) as we can
plotIdx = sampleOffset:min(sampleOffset + sampleSpan,rxWaveLen);

% Configure timeScope to display the packet
timeScope.TimeSpan = sampleSpan/sr;
timeScope.TimeDisplayOffset = sampleOffset/sr;
timeScope.YLimits = [0 max(abs(rx(:)))];
timeScope(abs(rx(plotIdx,:)));
release(timeScope);

% Display the spectrum of the detected packet
spectrumAnalyzer(rx(pktOffset + (1:numRxSamples),:));
release(spectrumAnalyzer);
```

输出为：
![image-20230416143844229](https://user-images.githubusercontent.com/115327603/232277640-85f36a2e-6852-4ef6-ab17-4ec09f0eb707.png)
![image-20230416143836176](https://user-images.githubusercontent.com/115327603/232277643-14e48c95-e55a-4fe2-a688-607856b4df00.png)


## HE-SIG-A Decoding

HE-SIG-A字段包含HE数据包的传输配置。需要从L-LTF获得的信道和噪声功率的估计来解码HE-SIG-A字段。

```matlab
disp('Decoding HE-SIG-A...')
rxSIGA = rx(pktOffset+(ind.HESIGA(1):ind.HESIGA(2)),:);
sigaDemod = wlanHEDemodulate(rxSIGA,'HE-SIG-A',chanBW);
hesigaDemod = wlanHETrackPilotError(sigaDemod,preHEChanEst,cfgRx,'HE-SIG-A');

% Equalize data carrying subcarriers, merging 20 MHz subchannels
preheInfo = wlanHEOFDMInfo('HE-SIG-A',chanBW);
[eqSIGASym,csi] = preHESymbolEqualize(hesigaDemod(preheInfo.DataIndices,:,:), ...
                                      preHEChanEst(preheInfo.DataIndices,:,:), ...
                                      noiseVar,preheInfo.NumSubchannels);
% Recover HE-SIG-A bits
[sigaBits,failCRC] = wlanHESIGABitRecover(eqSIGASym,noiseVar,csi);

% Perform the CRC on HE-SIG-A bits
if failCRC
    disp(' ** HE-SIG-A CRC fail **');
else
    disp(' HE-SIG-A CRC pass');
end

% Measure EVM of HE-SIG-A symbols
release(EVM);
if strcmp(pktFormat,'HE-EXT-SU')
    % The second symbol of an HE-SIG-A field for an HE-EXT-SU packet is
    % QBPSK.
    EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK',[0 pi/2 0 0]);
    % Account for scaling of L-LTF for an HE-EXT-SU packet
    rmsEVM = EVM(eqSIGASym*sqrt(2));
else
    EVM.ReferenceConstellation = wlanReferenceSymbols('BPSK');
    rmsEVM = EVM(eqSIGASym);
end
fprintf(' HE-SIG-A EVM: %2.2fdB\n\n',20*log10(mean(rmsEVM)/100));
```

输出为：
![image-20230416143534587](https://user-images.githubusercontent.com/115327603/232277658-efd187bd-ef3c-41ac-9158-5786520b72a5.png)


## Interpret Recovered HE-SIG-A bits

[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))对象在解释恢复的HE-SIG-A位之后进行更新。

```matlab
cfgRx = interpretHESIGABits(cfgRx,sigaBits);
ind = wlanFieldIndices(cfgRx); % Update field indices
```

显示从HE-MU数据包的HE-SIG-A字段获得的通用传输配置。用-1表示的参数未知或未定义。成功解码HE-SIG-B字段后，将更新未知用户相关的参数。

```matlab
disp(cfgRx)
```

输出为：
![image-20230416143603111](https://user-images.githubusercontent.com/115327603/232277665-395b0bb4-8e14-484a-b134-5f098b81ffd7.png)


## HE-SIG-B Decoding

对于HE-MU数据包，HE-SIG-B字段包含：

- 非压缩SIGB波形的RU分配信息是从HE-SIG-B公共字段推断出来的[1表27-24]。对于压缩的SIGB波形，从恢复的HE-SIG-a比特推断RU分配信息。
- 对于非压缩SIGB波形，在wlanHERecoveryConfig对象中更新HE-SIG-B符号的数量。只有在HE-SIG-A字段中指示的HE-SIG-B符号的数量被设置为15并且所有内容信道都通过CRC的情况下，才更新符号。如果任何HE-SIG-B内容信道未通过CRC，则不更新在HE-SIG-A字段中指示的HE-SIG-B符号的数量。
- SIGB压缩波形和非压缩波形的用户传输参数都是从HE-SIG-B用户字段推断出来的[1表27-27，27-28]。

需要从L-LTF获得的信道和噪声功率的估计来解码HE-SIG-B字段。

```matlab
if strcmp(pktFormat,'HE-MU')
    fprintf('Decoding HE-SIG-B...\n');
    if ~cfgRx.SIGBCompression
        fprintf(' Decoding HE-SIG-B common field...\n');
        s = getSIGBLength(cfgRx);
        % Get common field symbols. The start of HE-SIG-B field is known
        rxSym = rx(pktOffset+(double(ind.HESIGA(2))+(1:s.NumSIGBCommonFieldSamples)),:);
        % Decode HE-SIG-B common field
        [status,cfgRx] = heSIGBCommonFieldDecode(rxSym,preHEChanEst,noiseVar,cfgRx);

        % CRC on HE-SIG-B content channels
        if strcmp(status,'Success')
            fprintf('  HE-SIG-B (common field) CRC pass\n');
        elseif strcmp(status,'ContentChannel1CRCFail')
            fprintf('  ** HE-SIG-B CRC fail for content channel-1\n **');
        elseif strcmp(status,'ContentChannel2CRCFail')
            fprintf('  ** HE-SIG-B CRC fail for content channel-2\n **');
        elseif any(strcmp(status,{'UnknownNumUsersContentChannel1','UnknownNumUsersContentChannel2'}))
            error('  ** Unknown packet length, discard packet\n **');
        else
            % Discard the packet if all HE-SIG-B content channels fail
            error('  ** HE-SIG-B CRC fail **');
        end
        % Update field indices as the number of HE-SIG-B symbols are
        % updated
        ind = wlanFieldIndices(cfgRx);
    end

    % Get complete HE-SIG-B field samples
    rxSIGB = rx(pktOffset+(ind.HESIGB(1):ind.HESIGB(2)),:);
    fprintf(' Decoding HE-SIG-B user field... \n');
    % Decode HE-SIG-B user field
    [failCRC,cfgUsers] = heSIGBUserFieldDecode(rxSIGB,preHEChanEst,noiseVar,cfgRx);

    % CRC on HE-SIG-B users
    if ~all(failCRC)
        fprintf('  HE-SIG-B (user field) CRC pass\n\n');
        numUsers = numel(cfgUsers);
    elseif all(failCRC)
        % Discard the packet if all users fail the CRC
        error('  ** HE-SIG-B CRC fail for all users **');
    else
        fprintf('  ** HE-SIG-B CRC fail for at least one user\n **');
        % Only process users with valid CRC
        numUsers = numel(cfgUsers);
    end

else % HE-SU, HE-EXT-SU
    cfgUsers = {cfgRx};
    numUsers = 1;
end
```

输出为：
![image-20230416143624148](https://user-images.githubusercontent.com/115327603/232277685-983a37e9-3e6b-4071-b26c-25d6c0233c49.png)


## HE-Data Decoding

然后，每个用户更新的[wlanHERecoveryConfig](matlab:edit('wlanHERecoveryConfig.m'))对象可以用于恢复HE数据字段中每个用户的PSDU位。

```matlab
cfgDataRec = trackingRecoveryConfig;
cfgDataRec.PilotTracking = pilotTracking;

fprintf('Decoding HE-Data...\n');
for iu = 1:numUsers
    % Get recovery configuration object for each user
    user = cfgUsers{iu};
    if strcmp(pktFormat,'HE-MU')
        fprintf(' Decoding User:%d, STAID:%d, RUSize:%d\n',iu,user.STAID,user.RUSize);
    else
        fprintf(' Decoding RUSize:%d\n',user.RUSize);
    end

    heInfo = wlanHEOFDMInfo('HE-Data',chanBW,user.GuardInterval,[user.RUSize user.RUIndex]);

    % HE-LTF demodulation and channel estimation
    rxHELTF = rx(pktOffset+(ind.HELTF(1):ind.HELTF(2)),:);
    heltfDemod = wlanHEDemodulate(rxHELTF,'HE-LTF',chanBW,user.GuardInterval, ...
        user.HELTFType,[user.RUSize user.RUIndex]);
    [chanEst,pilotEst] = wlanHELTFChannelEstimate(heltfDemod,user);

    % Number of expected data OFDM symbols
    symLen = heInfo.FFTLength+heInfo.CPLength;
    numOFDMSym = double((ind.HEData(2)-ind.HEData(1)+1))/symLen;

    % HE-Data demodulation with pilot phase and timing tracking
    % Account for extra samples when extracting data field from the packet
    % for sample rate offset tracking. Extra samples may be required if the
    % receiver clock is significantly faster than the transmitter.
    maxSRO = 120; % Parts per million
    Ne = ceil(numRxSamples*maxSRO*1e-6); % Number of extra samples
    Ne = min(Ne,rxWaveLen-numRxSamples); % Limited to length of waveform
    numRxSamplesProcess = numRxSamples+Ne;
    rxData = rx(pktOffset+(ind.HEData(1):numRxSamplesProcess),:);
    if user.RUSize==26
        % Force CPE only tracking for 26-tone RU as algorithm susceptible
        % to noise
        cfgDataRec.PilotTracking = 'CPE';
    else
        cfgDataRec.PilotTracking = pilotTracking;
    end
    [demodSym,cpe,peg] = heTrackingOFDMDemodulate(rxData,chanEst,numOFDMSym,user,cfgDataRec);

    % Estimate noise power in HE fields
    demodPilotSym = demodSym(heInfo.PilotIndices,:,:);
    nVarEst = heNoiseEstimate(demodPilotSym,pilotEst,user);

    % Equalize
    [eqSym,csi] = heEqualizeCombine(demodSym,chanEst,nVarEst,user);

    % Discard pilot subcarriers
    eqSymUser = eqSym(heInfo.DataIndices,:,:);
    csiData = csi(heInfo.DataIndices,:);

    % Demap and decode bits
    rxPSDU = wlanHEDataBitRecover(eqSymUser,nVarEst,csiData,user,'LDPCDecodingMethod','norm-min-sum');

    % Deaggregate the A-MPDU
    [mpduList,~,status] = wlanAMPDUDeaggregate(rxPSDU,wlanHESUConfig);
    if strcmp(status,'Success')
        fprintf('  A-MPDU deaggregation successful \n');
    else
        fprintf('  A-MPDU deaggregation unsuccessful \n');
    end

    % Decode the list of MPDUs and check the FCS for each MPDU
    for i = 1:numel(mpduList)
        [~,~,status] = wlanMPDUDecode(mpduList{i},wlanHESUConfig,'DataFormat','octets');
        if strcmp(status,'Success')
            fprintf('  FCS pass for MPDU:%d\n',i);
        else
            fprintf('  FCS fail for MPDU:%d\n',i);
        end
    end

    % Plot equalized constellation of the recovered HE data symbols for all
    % spatial streams per user
    hePlotEQConstellation(eqSymUser,user,ConstellationDiagram,iu,numUsers);

    % Measure EVM of HE-Data symbols
    release(EVM);
    EVM.ReferenceConstellation = wlanReferenceSymbols(user);
    rmsEVM = EVM(eqSymUser(:));
    fprintf('  HE-Data EVM:%2.2fdB\n\n',20*log10(rmsEVM/100));

    % Plot EVM per symbol of the recovered HE data symbols
    hePlotEVMPerSymbol(eqSymUser,user,EVMPerSymbol,iu,numUsers);

    % Plot EVM per subcarrier of the recovered HE data symbols
    hePlotEVMPerSubcarrier(eqSymUser,user,EVMPerSubcarrier,iu,numUsers);
end
```

输出为：
![image-20230416143640737](https://user-images.githubusercontent.com/115327603/232277726-9e542e5d-d2fd-49c0-a628-a3549e2e1afd.png)
![image-20230416143806314](https://user-images.githubusercontent.com/115327603/232277733-e9234521-f1df-4ae3-82eb-001a92590801.png)
![image-20230416143754680](https://user-images.githubusercontent.com/115327603/232277736-3b5edaa0-989c-46fc-85a4-915df965bcd5.png)
![image-20230416143731556](https://user-images.githubusercontent.com/115327603/232277743-bc3bb5c2-7023-4ba5-8455-a1d4dd204e5c.png)


## Selected Bibliography

1. IEEE Std 802.11ax™-2021. IEEE Standard for Information Technology - Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks - Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications - Amendment 1: Enhancements for High-Efficiency WLAN.
2. IEEE Std 802.11™-2020 Standard for Information Technology - Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks - Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications.
