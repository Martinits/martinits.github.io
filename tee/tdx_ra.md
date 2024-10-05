## TDX Remote Attestation

1. MRTD
   - SHA384
   - `TDH.MNG.INIT`开始
   - `TDH.MEM.PAGE.ADD`添加内存时，把GPA加入度量中
   - Control Structures不度量，包括TDR、TDCX、TDVPR、Secure EPT
   - `TDH.MR.EXTEND`加入初态度量的内存，包括GPA和内容，以256B为单位
     - 对于TD会擦除并重新初始化的页，可以不用加入度量
   - `TDH.MR.FINALIZE`形成初态度量，此后不能再进行`TDH.MEM.PAGE.ADD` `TDH.MR.EXTEND`等操作
2. RTMR
   - 用于Guest做动态度量，典型用例就是在Guest启动过程中充当vTPM，从而让VBIOS实现可信启动
   - SHA384
   - 每个guest有4个
   - 初态为0
   - 使用`TDG.MR.RTMR.EXTEND`更新
   - 更新类似TPM的PCR：`RTMR[i] = SHA384(RTMR[i] || x);`
3. Measurement Report
   - Guest调用`TDG.MR.REPORT`形成report，传递一个自定义REPORTDATA，返回一个TDREPORT_STRUCT，包含：
     - TDREPORT_STRUCT包含：CPU的SVN、TDX Module的SVN和度量值、TD的信息（MRTD，XFAM/ATTRIBUTES，版本和owner（MRCONFIID，MROWNER，MROWNERCONFIG），RTMR、SERVTD的HASH）、REPORTDATA以及REPORTMACSTRUCT
     - 其中REPORTMACSTRUCT是真正去验证的东西，其包括上面几个东西的值或者hash，会被CPU使用CPU独有的HMAC key做HMAC
     - TDREPORT_STRUCT这块内存虽然是Guest分配的，但是设计上只能给CPU使用（HMAC保护），软件只能通过`TDG.MR.VERIFYREPORT`和SGX的`ENCLU(EVERIFYREPORT2)`验证这个Report
   - Local Attestation
     - 任何TD可以对同一个机器上的TD做Local Report Verification（`TDG.MR.VERIFYREPORT`），TDX Module会使用SEAMOPS的SEAMVERIFYREPORT leaf验证REPORTMACSTRUCT
     - SGX Enclave可以使用`EVERIFYREPORT2`指令对同一个机器上的TD做Local Report Verification
     - Report Verification失败的可能原因有：TDX Module做了热升级、CPU microcode更新过、或者TD被迁移过，这些TD都是无感的。解决方案就是让被verify的TD重新生成Report
   - Remote Attestation
     - 被验证的TD把Report发送给同一台机器的Quoting Enclave，QE对Report做Local Attestation，然后用ECDSA 384 key对Report签名形成Quote，接入SGX Attestation Infrastructure
4. Host Measured Boot
   - **TODO**
   - ACM
   - Intel TXT
