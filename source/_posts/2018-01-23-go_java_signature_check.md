---
layout: post
title: "openpgp签名与验签"
date: "2018-01-23 00:00:00 +0800"
categories: "algorithms"
tags: "algorithms"
---

#### 存在的问题  

目前每次程序版本发布需要手动gpg签名并打包发布，存在的问题：  
 
- 每次都需要人工手动使用gpg签名  
- 如果如果需要签名发布的app变多也会增加很多重复劳动  
- 其他项目或应用文件签名下发需要一个统一的接口    

<!--more-->

其实也考虑过写脚本进行签名，但是生产机器还需要装gpg比较麻烦。其实主要还是想学习一下如何代码进行gpg加密,哈哈...

#### golang openpgp签名验签  

其实公司需要的是java版，但是最近一直在做go的项目，并且以后go的项目说不定也会用到。虽然语言不通，但也作为摸索签名加密的一种途径。

```
package main

import (
	"bytes"
	"golang.org/x/crypto/openpgp"
	"os"
	"io/ioutil"
)

func main()  {
  ArmoredDetachSign("d:/test.txt")
  CheckArmoredDetachedSignature("d:/test.txt.asc","d:/test.txt")
}

//CheckArmoredDetachedSignature openpgp验签
func CheckArmoredDetachedSignature(signatureFile, verificationFile string) (*openpgp.Entity, error) {

	pubring, err := ioutil.ReadFile("path to pubring.gpg")
	keyRingReader := bytes.NewBuffer(pubring)

	signature, err := os.Open(signatureFile)
	if err != nil {
		return nil, err
	}
	defer signature.Close()

	verification, err := os.Open(verificationFile)
	if err != nil {
		return nil, err
	}
	defer verification.Close()

	keyring, err := openpgp.ReadKeyRing(keyRingReader)
	if err != nil {
		return nil, err
	}
	entity, err := openpgp.CheckArmoredDetachedSignature(keyring, verification, signature)
	if err != nil {
		return nil, err
	}
	return entity, nil
}

//ArmoredDetachSign openpgp签名
func ArmoredDetachSign(signFile string) error {

	var entity *openpgp.Entity
	var entityList openpgp.EntityList

	pubring, err := ioutil.ReadFile("path to secring.gpg")
	if err!=nil{
		return err
	}
	keyringFileBuffer := bytes.NewBuffer(pubring)

	entityList, err = openpgp.ReadKeyRing(keyringFileBuffer)
	if err != nil {
		return err
	}
	entity = entityList[0]

	passPhraseByte := []byte("your private key password")
	entity.PrivateKey.Decrypt(passPhraseByte)
	for _, subKey := range entity.Subkeys {
		subKey.PrivateKey.Decrypt(passPhraseByte)
	}

	sign, err := os.Open(signFile)
	if err != nil {
		return err
	}
	defer sign.Close()

	asc, err := os.Create(signFile + ".asc")
	if err != nil {
		return err
	}
	defer asc.Close()

	err = openpgp.ArmoredDetachSign(asc, entity, sign, nil)
	if err != nil {
		return err
	}
	return nil
}

```

#### java openpgp签名验签  

maven引入依赖   
```
<dependency>
     <groupId>org.bouncycastle</groupId>
     <artifactId>bcprov-jdk15on</artifactId>
     <version>1.59</version>
 </dependency>
 <dependency>
     <groupId>org.bouncycastle</groupId>
     <artifactId>bcpg-jdk15on</artifactId>
     <version>1.59</version>
 </dependency>
```
记住一定要使用最新的1.59版本，之前的版本都是jdk1.5、1.6版本编译的，我们现在项目都是1.8版本差异太大无法编译。


```
import org.apache.commons.io.*;
import org.bouncycastle.bcpg.*;
import org.bouncycastle.jce.provider.*;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.jcajce.*;
import org.bouncycastle.openpgp.operator.jcajce.*;

import java.io.*;
import java.security.*;
import java.util.*;

/**
 * OpenGPG签名工具类
 *
 * @author SunJun
 */
public class OpenPgpService {

    public static void main(String[] args) throws Exception {

        Security.addProvider(new BouncyCastleProvider());
        createSignature("d:/test.txt", "d:/test.txt.asc", true);
        verifySignature("d:/test.txt","d:/test.txt.asc");
    }

    /**
     * verifySignature 签名验证
     *
     * @param fileName      待验签文件名
     * @param inputFileName asc文件
     * @author SunJun
     */
    public static boolean verifySignature(String fileName, String inputFileName) throws IOException,
            PGPException {

        InputStream in = null;
        InputStream keyIn = null;
        boolean result;

        try {

            in = new BufferedInputStream(new FileInputStream(inputFileName));
            keyIn = new BufferedInputStream(new FileInputStream(new File("pubring.asc")));
            result = verifySignature(fileName, in, keyIn);

        } finally {

            IOUtils.closeQuietly(keyIn);
            IOUtils.closeQuietly(in);
        }
        return result;
    }

    /**
     * verifySignature
     *
     * @param fileName 待验签文件名
     * @param in       asc文件流
     * @param keyIn    公钥文件
     * @author SunJun
     */
    private static boolean verifySignature(String fileName, InputStream in, InputStream keyIn) throws IOException,
            PGPException {

        PGPSignature sig;
        InputStream dIn = null;
        PGPSignatureList p3;
        in = PGPUtil.getDecoderStream(in);
        JcaPGPObjectFactory pgpFact = new JcaPGPObjectFactory(in);

        Object o = pgpFact.nextObject();
        if (o instanceof PGPCompressedData) {

            PGPCompressedData c1 = (PGPCompressedData) o;
            pgpFact = new JcaPGPObjectFactory(c1.getDataStream());
            p3 = (PGPSignatureList) pgpFact.nextObject();

        } else {

            p3 = (PGPSignatureList) o;

        }

        PGPPublicKeyRingCollection pgpPubRingCollection =
                new PGPPublicKeyRingCollection(PGPUtil.getDecoderStream(keyIn), new JcaKeyFingerprintCalculator());

        try {

            dIn = new BufferedInputStream(new FileInputStream(fileName));
            sig = p3.get(0);

            PGPPublicKey key = pgpPubRingCollection.getPublicKey(sig.getKeyID());
            sig.init(new JcaPGPContentVerifierBuilderProvider().setProvider("BC"), key);

            int ch;
            while ((ch = dIn.read()) >= 0) {
                sig.update((byte) ch);
            }

        } finally {
            IOUtils.closeQuietly(dIn);
        }

        return sig.verify();
    }

    /**
     * createSignature 文件签名
     *
     * @param inputFileName  待签名文件
     * @param outputFileName 签名asc文件
     * @param armor          是否生成ASSCII码形式签名文件
     * @author SunJun
     */
    public static void createSignature(String inputFileName, String outputFileName, boolean armor) throws
            IOException, PGPException {

        InputStream keyIn = null;
        OutputStream out = null;

        try {

            keyIn = new BufferedInputStream(new FileInputStream(new File("secring.asc")));
            out = new BufferedOutputStream(new FileOutputStream(outputFileName));

            createSignature(inputFileName, keyIn, out, "private key password".toCharArray(), armor);

        } finally {

            IOUtils.closeQuietly(keyIn);
            IOUtils.closeQuietly(out);
        }
    }

    /**
     * createSignature 文件签名
     *
     * @param fileName     待签名文件名
     * @param keyIn        私钥文件流
     * @param outputStream 签名asc文件
     * @param pass         私钥密码
     * @param armor        是否生成ASSCII码形式签名文件
     * @author SunJun
     */
    private static void createSignature(String fileName, InputStream keyIn, OutputStream outputStream, char[] pass, boolean armor) throws
            IOException, PGPException {

        if (armor) {
            outputStream = new ArmoredOutputStream(outputStream);
        }

        PGPSecretKey pgpSec = readSecretKey(keyIn);

        PGPPrivateKey pgpPriKey = pgpSec.extractPrivateKey(new JcePBESecretKeyDecryptorBuilder().setProvider("BC").build(pass));
        PGPSignatureGenerator sGen = new PGPSignatureGenerator(
                new JcaPGPContentSignerBuilder(pgpSec.getPublicKey().getAlgorithm(), PGPUtil.SHA1).setProvider("BC"));

        sGen.init(PGPSignature.BINARY_DOCUMENT, pgpPriKey);

        BCPGOutputStream bOut = new BCPGOutputStream(outputStream);
        InputStream fIn = null;

        try {

            int ch;
            fIn = new BufferedInputStream(new FileInputStream(fileName));

            while ((ch = fIn.read()) >= 0) {
                sGen.update((byte) ch);
            }
            sGen.generate().encode(bOut);

        } finally {

            IOUtils.closeQuietly(fIn);
            IOUtils.closeQuietly(bOut);
        }
    }

    /**
     * readSecretKey
     *
     * @param input 私钥文件流
     * @return 私钥
     */
    private static PGPSecretKey readSecretKey(InputStream input) throws IOException, PGPException {

        PGPSecretKeyRingCollection pgpSec = new PGPSecretKeyRingCollection(
                PGPUtil.getDecoderStream(input), new JcaKeyFingerprintCalculator());
        Iterator keyRingIterator = pgpSec.getKeyRings();

        while (keyRingIterator.hasNext()) {

            PGPSecretKeyRing keyRing = (PGPSecretKeyRing) keyRingIterator.next();
            Iterator keyIterator = keyRing.getSecretKeys();

            while (keyIterator.hasNext()) {

                PGPSecretKey key = (PGPSecretKey) keyIterator.next();
                if (key.isSigningKey()) {
                    return key;
                }
            }
        }

        throw new IllegalArgumentException("Can't find signing key in key ring.");
    }
}

```