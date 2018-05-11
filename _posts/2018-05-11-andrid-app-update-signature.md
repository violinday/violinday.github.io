---
layout: post
title: Android App（更新时）签名(证书)校验过程源码分析
categories: [Android]
description: Android Install/Update Signature Check
keywords: Signature, Update
---



## Android App（更新时）签名(证书)校验过程源码分析

还是看到知乎上的一个【[问题](https://www.zhihu.com/question/266891554/answer/388809643)】，然后翻了翻Android源代码，给出了一个答案，在这里好好整理一下代码流程。

>对于非系统APP来说，更新包的时候，需要检查更新包和被更新目标是否是相同的签发者所签发，即更新包和被更新包是由相同的签名密钥签发，此时认为更新包和原应用是由相同实体（发布者）所生成，并建立更新包和已经存在应用的信任关系。这个规则叫做【同源策略TOFU Trust On First Use】——摘自《Android安全架构探究》


APP在升级的时候，会验证新的APK包的完整性，确保没被修改，验证通过，证书信息最终会保存到PackageParser.Package.mSignatures字段中。然后判断新旧证书是否是同一个，接下来进行升级或者拒绝。

接下来感兴趣的话看代码吧~
PackageManagerService

``` Java
@Override
public int checkSignatures(String pkg1, String pkg2) {
    synchronized (mPackages) {
        final PackageParser.Package p1 = mPackages.get(pkg1);
        final PackageParser.Package p2 = mPackages.get(pkg2);
        if (p1 == null || p1.mExtras == null
                || p2 == null || p2.mExtras == null) {
            return PackageManager.SIGNATURE_UNKNOWN_PACKAGE;
        }
        final int callingUid = Binder.getCallingUid();
        final int callingUserId = UserHandle.getUserId(callingUid);
        final PackageSetting ps1 = (PackageSetting) p1.mExtras;
        final PackageSetting ps2 = (PackageSetting) p2.mExtras;
        if (filterAppAccessLPr(ps1, callingUid, callingUserId)
                || filterAppAccessLPr(ps2, callingUid, callingUserId)) {
            return PackageManager.SIGNATURE_UNKNOWN_PACKAGE;
        }
        return compareSignatures(p1.mSignatures, p2.mSignatures);
    }
}

```

然后：

``` Java

/**
 * Compares two sets of signatures. Returns:
 * <br />
 * {@link PackageManager#SIGNATURE_NEITHER_SIGNED}: if both signature sets are null,
 * <br />
 * {@link PackageManager#SIGNATURE_FIRST_NOT_SIGNED}: if the first signature set is null,
 * <br />
 * {@link PackageManager#SIGNATURE_SECOND_NOT_SIGNED}: if the second signature set is null,
 * <br />
 * {@link PackageManager#SIGNATURE_MATCH}: if the two signature sets are identical,
 * <br />
 * {@link PackageManager#SIGNATURE_NO_MATCH}: if the two signature sets differ.
 */
static int compareSignatures(Signature[] s1, Signature[] s2) {
    if (s1 == null) {
        return s2 == null
                ? PackageManager.SIGNATURE_NEITHER_SIGNED
                : PackageManager.SIGNATURE_FIRST_NOT_SIGNED;
    }

    if (s2 == null) {
        return PackageManager.SIGNATURE_SECOND_NOT_SIGNED;
    }

    if (s1.length != s2.length) {
        return PackageManager.SIGNATURE_NO_MATCH;
    }

    // Since both signature sets are of size 1, we can compare without HashSets.
    if (s1.length == 1) {
        return s1[0].equals(s2[0]) ?
                PackageManager.SIGNATURE_MATCH :
                PackageManager.SIGNATURE_NO_MATCH;
    }

    ArraySet<Signature> set1 = new ArraySet<Signature>();
    for (Signature sig : s1) {
        set1.add(sig);
    }
    ArraySet<Signature> set2 = new ArraySet<Signature>();
    for (Signature sig : s2) {
        set2.add(sig);
    }
    // Make sure s2 contains all signatures in s1.
    if (set1.equals(set2)) {
        return PackageManager.SIGNATURE_MATCH;
    }
    return PackageManager.SIGNATURE_NO_MATCH;
}
```

咱们来看看Signature类，看看什么时候认为两个包是相同实体发布的，重点关注equals、hashCode方法：

``` Java
/*
 * Copyright (C) 2008 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.content.pm;

import android.os.Parcel;
import android.os.Parcelable;

import com.android.internal.util.ArrayUtils;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.lang.ref.SoftReference;
import java.security.PublicKey;
import java.security.cert.Certificate;
import java.security.cert.CertificateEncodingException;
import java.security.cert.CertificateException;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.util.Arrays;

/**
 * Opaque, immutable representation of a signing certificate associated with an
 * application package.
 * <p>
 * This class name is slightly misleading, since it's not actually a signature.
 */
public class Signature implements Parcelable {
    private final byte[] mSignature;
    private int mHashCode;
    private boolean mHaveHashCode;
    private SoftReference<String> mStringRef;
    private Certificate[] mCertificateChain;

    /**
     * Create Signature from an existing raw byte array.
     */
    public Signature(byte[] signature) {
        mSignature = signature.clone();
        mCertificateChain = null;
    }

    /**
     * Create signature from a certificate chain. Used for backward
     * compatibility.
     *
     * @throws CertificateEncodingException
     * @hide
     */
    public Signature(Certificate[] certificateChain) throws CertificateEncodingException {
        mSignature = certificateChain[0].getEncoded();
        if (certificateChain.length > 1) {
            mCertificateChain = Arrays.copyOfRange(certificateChain, 1, certificateChain.length);
        }
    }

    private static final int parseHexDigit(int nibble) {
        if ('0' <= nibble && nibble <= '9') {
            return nibble - '0';
        } else if ('a' <= nibble && nibble <= 'f') {
            return nibble - 'a' + 10;
        } else if ('A' <= nibble && nibble <= 'F') {
            return nibble - 'A' + 10;
        } else {
            throw new IllegalArgumentException("Invalid character " + nibble + " in hex string");
        }
    }

    /**
     * Create Signature from a text representation previously returned by
     * {@link #toChars} or {@link #toCharsString()}. Signatures are expected to
     * be a hex-encoded ASCII string.
     *
     * @param text hex-encoded string representing the signature
     * @throws IllegalArgumentException when signature is odd-length
     */
    public Signature(String text) {
        final byte[] input = text.getBytes();
        final int N = input.length;

        if (N % 2 != 0) {
            throw new IllegalArgumentException("text size " + N + " is not even");
        }

        final byte[] sig = new byte[N / 2];
        int sigIndex = 0;

        for (int i = 0; i < N;) {
            final int hi = parseHexDigit(input[i++]);
            final int lo = parseHexDigit(input[i++]);
            sig[sigIndex++] = (byte) ((hi << 4) | lo);
        }

        mSignature = sig;
    }

	......
    
    /**
     * Returns the public key for this signature.
     *
     * @throws CertificateException when Signature isn't a valid X.509
     *             certificate; shouldn't happen.
     * @hide
     */
    public PublicKey getPublicKey() throws CertificateException {
        final CertificateFactory certFactory = CertificateFactory.getInstance("X.509"); // DER 编码的X.509格式密钥
        final ByteArrayInputStream bais = new ByteArrayInputStream(mSignature);
        final Certificate cert = certFactory.generateCertificate(bais);
        return cert.getPublicKey();
    }

    ......

    @Override
    public boolean equals(Object obj) {
        try {
            if (obj != null) {
                Signature other = (Signature)obj;
                return this == other || Arrays.equals(mSignature, other.mSignature);
            }
        } catch (ClassCastException e) {
        }
        return false;
    }

    @Override
    public int hashCode() {
        if (mHaveHashCode) {
            return mHashCode;
        }
        mHashCode = Arrays.hashCode(mSignature);
        mHaveHashCode = true;
        return mHashCode;
    }
    ......
}

```
查看源代码可知，
> Opaque, immutable representation of a signing certificate associated with an application package.

它是一个“不透明的、不可变的且与应用程序包关联的签名证书表示类（签名整数对象）”。
即，它是一个与某个APK文件关联的DER编码格式签名证书的封装。

查看APK安装过程源码，找到证书数据加载的位置，在PackageManagerService.java类中:

``` Java
private PackageParser.Package scanPackageInternalLI(PackageParser.Package pkg, File scanFile, int policyFlags, int scanFlags, long currentTime, @Nullable UserHandle user) {
	......

	//注意变量是否APP更新
	final boolean isUpdatedPkg = updatedPkg != null; 
    final boolean isUpdatedSystemPkg = isUpdatedPkg && (policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0;
    
    ......

    //如果APP更新，设置policyFlags
    if (isUpdatedPkg) {
        // An updated system app will not have the PARSE_IS_SYSTEM flag set
        // initially
        policyFlags |= PackageParser.PARSE_IS_SYSTEM;

        // An updated privileged app will not have the PARSE_IS_PRIVILEGED
        // flag set initially
        if ((updatedPkg.pkgPrivateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0) {
            policyFlags |= PackageParser.PARSE_IS_PRIVILEGED;
        }
    }

    // Verify certificates against what was last scanned
    collectCertificatesLI(ps, pkg, scanFile, policyFlags);

}
```          

经过
collectCertificatesLI，collectCertificatesInternal，
层层跳转，进入 PackageParse.java

``` Java 
private static void collectCertificates(Package pkg, File apkFile, int parseFlags)
            throws PackageParserException {

    ......

    //省略了V2签名，直接看V1
    StrictJarFile jarFile = null;
    try {
        // Ignore signature stripping protections when verifying APKs from system partition.
        // For those APKs we only care about extracting signer certificates, and don't care
        // about verifying integrity.
        boolean signatureSchemeRollbackProtectionsEnforced =
                (parseFlags & PARSE_IS_SYSTEM_DIR) == 0;
        //初始化 jarFile
        jarFile = new StrictJarFile(
                apkPath,
                !verified, // whether to verify JAR signature
                signatureSchemeRollbackProtectionsEnforced);
        
        ......

        // Verify that entries are signed consistently with the first entry
        // we encountered. Note that for splits, certificates may have
        // already been populated during an earlier parse of a base APK.
        for (ZipEntry entry : toVerify) {
        	//紧接着，调用loadCertificates方法：
            final Certificate[][] entryCerts = loadCertificates(jarFile, entry);
            if (ArrayUtils.isEmpty(entryCerts)) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
                        "Package " + apkPath + " has no certificates at entry "
                        + entry.getName());
            }
            final Signature[] entrySignatures = convertToSignatures(entryCerts);

            //仅赋值一次，不为空则判断每次获取到的Certificate是否相同，不同则抛出PackageParserException异常
            if (pkg.mCertificates == null) {
                pkg.mCertificates = entryCerts;
                pkg.mSignatures = entrySignatures;
                pkg.mSigningKeys = new ArraySet<PublicKey>();
                for (int i=0; i < entryCerts.length; i++) {
                    pkg.mSigningKeys.add(entryCerts[i][0].getPublicKey());
                }
            } else {
                if (!Signature.areExactMatch(pkg.mSignatures, entrySignatures)) {
                    throw new PackageParserException(
                            INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
                                    + " has mismatched certificates at entry "
                                    + entry.getName());
                }
            }
        }
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    } catch (GeneralSecurityException e) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_CERTIFICATE_ENCODING,
                "Failed to collect certificates from " + apkPath, e);
    } catch (IOException | RuntimeException e) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
                "Failed to collect certificates from " + apkPath, e);
    } finally {
        closeQuietly(jarFile);
    }
}
```

快到地方了，不要着急，咱们接着看loadCertificates方法：

``` Java
private static Certificate[][] loadCertificates(StrictJarFile jarFile, ZipEntry entry)
        throws PackageParserException {
    InputStream is = null;
    try {
        // We must read the stream for the JarEntry to retrieve
        // its certificates.
        
        //在getInputStream方法中初始化certificates数据
        is = jarFile.getInputStream(entry);
        
        //在其中调用is[JarFileInputStream类型]的read方法
        //read方法调用verify方法，读取密钥到verifiedEntries中
        readFullyIgnoringContents(is);
        return jarFile.getCertificateChains(entry);
    } catch (IOException | RuntimeException e) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                "Failed reading " + entry.getName() + " in " + jarFile, e);
    } finally {
        IoUtils.closeQuietly(is);
    }
}
```
OK,终于看到出处了，StrictJarFile类中：

``` Java 
public InputStream getInputStream(ZipEntry ze) {
    final InputStream is = getZipInputStream(ze);
    if (isSigned) {
    	//init 方法，创建VerifierEntry
        StrictJarVerifier.VerifierEntry entry = verifier.initEntry(ze.getName());
        if (entry == null) {
            return is;
        }

        return new JarFileInputStream(is, ze.getSize(), entry);
    }

    return is;
} 

public Certificate[][] getCertificateChains(ZipEntry ze) {
    if (isSigned) {
    	//verifier 从哪里来呢，接下来往下看。
        return verifier.getCertificateChains(ze.getName());
    }
    return null;
}
```

verifier类型为StrictJarVerifier.java类查看源代码可知，verifiedEntries的赋值是在其内部类VerifierEntry的verify()方法中。
``` Java
//存放密钥的变量
private final Hashtable<String, Certificate[][]> verifiedEntries =
            new Hashtable<String, Certificate[][]>();

......

Certificate[][] getCertificateChains(String name) {
    return verifiedEntries.get(name);
}
```

又因为verifiedEntries数据从外部传进来，所以还需要继续找。。。源头，在
 jarFile = new StrictJarFile(apkPath, !verified, signatureSchemeRollbackProtectionsEnforced);
这句代码中，

``` Java
private StrictJarFile(String name,
        FileDescriptor fd,
        boolean verify,
        boolean signatureSchemeRollbackProtectionsEnforced)
                throws IOException, SecurityException {
    this.nativeHandle = nativeOpenJarFile(name, fd.getInt$());
    this.fd = fd;

    try {
        // Read the MANIFEST and signature files up front and try to
        // parse them. We never want to accept a JAR File with broken signatures
        // or manifests, so it's best to throw as early as possible.
        if (verify) {
            HashMap<String, byte[]> metaEntries = getMetaEntries();
            this.manifest = new StrictJarManifest(metaEntries.get(JarFile.MANIFEST_NAME), true);
            this.verifier =
                    new StrictJarVerifier(
                            name,
                            manifest,
                            metaEntries,
                            signatureSchemeRollbackProtectionsEnforced);
            Set<String> files = manifest.getEntries().keySet();
            for (String file : files) {
                if (findEntry(file) == null) {
                    throw new SecurityException("File " + file + " in manifest does not exist");
                }
            }
            //此处readCertificates为最终数据初始化的地方
            isSigned = verifier.readCertificates() && verifier.isSignedJar();
        } else {
            isSigned = false;
            this.manifest = null;
            this.verifier = null;
        }
    } catch (IOException | SecurityException e) {
        nativeClose(this.nativeHandle);
        IoUtils.closeQuietly(fd);
        closed = true;
        throw e;
    }

    guard.open("close");
}
```
还是要到StrictJarVerifier.java类中

``` Java
/**
 * If the associated JAR file is signed, check on the validity of all of the
 * known signatures.
 *
 * @return {@code true} if the associated JAR is signed and an internal
 *         check verifies the validity of the signature(s). {@code false} if
 *         the associated JAR file has no entries at all in its {@code
 *         META-INF} directory. This situation is indicative of an invalid
 *         JAR file.
 *         <p>
 *         Will also return {@code true} if the JAR file is <i>not</i>
 *         signed.
 * @throws SecurityException
 *             if the JAR file is signed and it is determined that a
 *             signature block file contains an invalid signature for the
 *             corresponding signature file.
 */
synchronized boolean readCertificates() {
    if (metaEntries.isEmpty()) {
        return false;
    }

    Iterator<String> it = metaEntries.keySet().iterator();
    while (it.hasNext()) {
        String key = it.next();
        //只解析这三种类型的密钥
        if (key.endsWith(".DSA") || key.endsWith(".RSA") || key.endsWith(".EC")) {
            verifyCertificate(key);
            it.remove();
        }
    }
    return true;
}


/**
 * @param certFile
 */
private void verifyCertificate(String certFile) {
    ...... //省略了使用证书（如RSA这行书）中的 .SF属性块对整个MF文件验证的过程

    //验证通过，signature在StrictJarVerifier类中的initEntry方法中变成二维数组，
    //放进StrictJarVerifier 的verifiedEntries变量中
    signatures.put(signatureFile, entries);
}
```

至此分析结束


参考： 
- [Android App签名(证书)校验过程源码分析](https://www.2cto.com/kf/201609/551023.html)
- [Android签名机制之—签名过程详解](http://www.520monkey.com/archives/571)
- [Android签名机制之—签名验证过程详解](http://www.520monkey.com/archives/573)