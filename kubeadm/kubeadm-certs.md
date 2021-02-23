
##  主要是cert证书的流程Phase 添加

```initRunner.AppendPhase(phases.NewCertsPhase()) ```


## 实例化一个workflowPhase, 主要调用的Phase ==>>  newCertSubPhases()

```// NewCertsPhase returns the phase for the certs 
func NewCertsPhase() workflow.Phase {
   return workflow.Phase{
      Name:   "certs",
      Short:  "Certificate generation",
      Phases: newCertSubPhases(),
      Run:    runCerts,
      Long:   cmdutil.MacroCommandLongDescription,
   }
}
```
## newCertSubPhases() 子流程的具体过程
* newCertSubPhases() 主要是基于certsphase.GetDefaultCertList() 获取默认证书列表，然后根据证书是否包含CA的name来分别调用runCAPhase()，runCertPhase() 两个不同的接口生成证书。
* runCAPhase(cert) 主要是用于生成一些根证书，这些证书的年限是10年
* runCertPhase(cert) 主要是生成一些server证书，这些证书的年限是1年
```
// newCertSubPhases returns sub phases for certs phase
func newCertSubPhases() []workflow.Phase {
   subPhases := []workflow.Phase{}

   // All subphase
   allPhase := workflow.Phase{
      Name:           "all",
      Short:          "Generate all certificates",
      InheritFlags:   getCertPhaseFlags("all"),
      RunAllSiblings: true,
   }

   subPhases = append(subPhases, allPhase)

   // This loop assumes that GetDefaultCertList() always returns a list of
   // certificate that is preceded by the CAs that sign them.
   var lastCACert *certsphase.KubeadmCert
  // GetDefaultCertList() 获取需要创建的Default的证书列表， 然后依次的创建该证书。
   for _, cert := range certsphase.GetDefaultCertList() {
      var phase workflow.Phase
      if cert.CAName == "" {
         phase = newCertSubPhase(cert, runCAPhase(cert))  ## runCAPhase(cert) 主要是用于生成一些根证书，这些证书的年限是10年
         lastCACert = cert
      } else {
         phase = newCertSubPhase(cert, runCertPhase(cert, lastCACert)) //  runCertPhase(cert) 主要是生成一些server证书，这些证书的年限是1年
         phase.LocalFlags = localFlags()
      }
      subPhases = append(subPhases, phase)
   }

   // SA creates the private/public key pair, which doesn't use x509 at all
   saPhase := workflow.Phase{
      Name:         "sa",
      Short:        "Generate a private key for signing service account tokens along with its public key",
      Long:         saKeyLongDesc,
      Run:          runCertsSa,
      InheritFlags: []string{options.CertificatesDir},
   }

   subPhases = append(subPhases, saPhase)

   return subPhases
}

```
## 默认证书列表：默认证书包括： CA根证书， APIServer 证书， FrontProxy 证书， Etcd 证书。
```
func GetDefaultCertList() Certificates {
   return Certificates{
      &KubeadmCertRootCA,
      &KubeadmCertAPIServer,
      &KubeadmCertKubeletClient,
      // Front Proxy certs
      &KubeadmCertFrontProxyCA,
      &KubeadmCertFrontProxyClient,
      // etcd certs
      &KubeadmCertEtcdCA,
      &KubeadmCertEtcdServer,
      &KubeadmCertEtcdPeer,
      &KubeadmCertEtcdHealthcheck,
      &KubeadmCertEtcdAPIClient,
   }
}
```
## runCAPhase() 生成根证书的主要流程如下:
```
func runCAPhase(ca *certsphase.KubeadmCert) func(c workflow.RunData) error {
   return func(c workflow.RunData) error {
      data, ok := c.(InitData)
      if !ok {
         return errors.New("certs phase invoked with an invalid data struct")
      }  
      //  通过kubeadm.conf 获取信息，判断ca.Name是不是etcd-ca,如果是就直接跳过
      // if using external etcd, skips etcd certificate authority generation
      if data.Cfg().Etcd.External != nil && ca.Name == "etcd-ca" {
         fmt.Printf("[certs] External etcd mode: Skipping %s certificate authority generation\n", ca.BaseName)
         return nil
      }
      // 直接从节点Load一次证书，并且判断证书是否过期，合法。
      // 如果有证书文件，再继续判断私钥是否存在，并且校验合法性。
      // 即： （1）有证书且合法，不管有没有Key 都是直接返回,不在生成新的证书与Key。
      //     （2）没有证书，或证书不合法，就需要生成新的证书，往下走
      if _, err := pkiutil.TryLoadCertFromDisk(data.CertificateDir(), ca.BaseName); err == nil {
         if _, err := pkiutil.TryLoadKeyFromDisk(data.CertificateDir(), ca.BaseName); err == nil {
            fmt.Printf("[certs] Using existing %s certificate authority\n", ca.BaseName)
            return nil
         }
         fmt.Printf("[certs] Using existing %s keyless certificate authority\n", ca.BaseName)
         return nil
      }


      // if dryrunning, write certificates authority to a temporary folder (and defer restore to the path originally specified by the user)
      cfg := data.Cfg()
      cfg.CertificatesDir = data.CertificateWriteDir()
      defer func() { cfg.CertificatesDir = data.CertificateDir() }()

      // 创建新的证书与Key
      // create the new certificate authority (or use existing)
      return certsphase.CreateCACertAndKeyFiles(ca, cfg)
   }
}
// certsphase.CreateCACertAndKeyFiles() 主要就是创建CA证书与Key
```
## CreateCACertAndKeyFiles 创建CA证书与Key.
```
// The certSpec should be one of the variables from this package.
func CreateCACertAndKeyFiles(certSpec *KubeadmCert, cfg *kubeadmapi.InitConfiguration) error {
   if certSpec.CAName != "" {
      return errors.Errorf("this function should only be used for CAs, but cert %s has CA %s", certSpec.Name, certSpec.CAName)
   }
   klog.V(1).Infof("creating a new certificate authority for %s", certSpec.Name)

   certConfig, err := certSpec.GetConfig(cfg)
   if err != nil {
      return err
   }

   // NewCertificateAuthority() 就是生成caCert证书 + caKey 
   caCert, caKey, err := pkiutil.NewCertificateAuthority(certConfig)
   if err != nil {
      return err
   }

   return writeCertificateAuthorityFilesIfNotExist(
      cfg.CertificatesDir,
      certSpec.BaseName,
      caCert,
      caKey,
   )
}
```
 ## NewCertificateAuthority
（1）调用NewPrivateKey() 生成x509的一个私钥Key
（2）调用NewSelfSignedCACert（）生成x509的一个根证书
```
 // NewCertificateAuthority creates new certificate and private key for the certificate authority
func NewCertificateAuthority(config *CertConfig) (*x509.Certificate, crypto.Signer, error) {
   key, err := NewPrivateKey(config.PublicKeyAlgorithm)
   if err != nil {
      return nil, nil, errors.Wrap(err, "unable to create private key while generating CA certificate")
   }

   cert, err := certutil.NewSelfSignedCACert(config.Config, key)
   if err != nil {
      return nil, nil, errors.Wrap(err, "unable to create self-signed CA certificate")
   }


   return cert, key, nil
}
 ```
 ## NewSelfSignedCACert（）
（1）生成证书的包含一个 duration365d * 10 的期限：即10年的期限。
```
 // NewSelfSignedCACert creates a CA certificate
func NewSelfSignedCACert(cfg Config, key crypto.Signer) (*x509.Certificate, error) {
   now := time.Now()
   tmpl := x509.Certificate{
      SerialNumber: new(big.Int).SetInt64(0),
      Subject: pkix.Name{
         CommonName:   cfg.CommonName,
         Organization: cfg.Organization,
      },
      NotBefore:             now.UTC(),
      NotAfter:              now.Add(duration365d * 10).UTC(),
      KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
      BasicConstraintsValid: true,
      IsCA:                  true,
   }

   certDERBytes, err := x509.CreateCertificate(cryptorand.Reader, &tmpl, &tmpl, key.Public(), key)
   if err != nil {
      return nil, err
   }
   return x509.ParseCertificate(certDERBytes)
}
```
## runCertPhase() 生成server证书的流程与上面基本一致，就是证书的期限不是10年，而是1年。具体代码如下：
```
 // NewSignedCert creates a signed certificate using the given CA certificate and key
func NewSignedCert(cfg *CertConfig, key crypto.Signer, caCert *x509.Certificate, caKey crypto.Signer) (*x509.Certificate, error) {
   serial, err := cryptorand.Int(cryptorand.Reader, new(big.Int).SetInt64(math.MaxInt64))
   if err != nil {
      return nil, err
   }
   if len(cfg.CommonName) == 0 {
      return nil, errors.New("must specify a CommonName")
   }
   if len(cfg.Usages) == 0 {
      return nil, errors.New("must specify at least one ExtKeyUsage")
   }

   RemoveDuplicateAltNames(&cfg.AltNames)

   certTmpl := x509.Certificate{
      Subject: pkix.Name{
         CommonName:   cfg.CommonName,
         Organization: cfg.Organization,
      },
      DNSNames:     cfg.AltNames.DNSNames,
      IPAddresses:  cfg.AltNames.IPs,
      SerialNumber: serial,
      NotBefore:    caCert.NotBefore,
	// CertificateValidity defines the validity for all the signed certificates generated by kubeadm
	// 因为 CertificateValidity = time.Hour * 24 * 365， 所以是356 day 
      NotAfter:     time.Now().Add(kubeadmconstants.CertificateValidity).UTC(),  
      KeyUsage:     x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
      ExtKeyUsage:  cfg.Usages,
   }
   certDERBytes, err := x509.CreateCertificate(cryptorand.Reader, &certTmpl, caCert, key.Public(), caKey)
   if err != nil {
      return nil, err
   }
   return x509.ParseCertificate(certDERBytes)
}

