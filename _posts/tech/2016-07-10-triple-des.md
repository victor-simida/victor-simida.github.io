---
layout: post
title: 3DES解密，golang实现
category: 技术
tags: Go
description: 工作中遇到的3DES解密，这里整理一个函数
---
3des是一种性能很挫的加密算法，这个算法需要一个key和一个iv（偏移矢量），key用于加密，iv用于将加密后的密文做异或比较
下面是3DES加密中的CBC实现。

```go
package main

import (
	"crypto/des"
	"crypto/cipher"
	"fmt"
	"bytes"
	"encoding/base64"
)

/*******************************************
DES加密方式
CBC方式时，第一个块使用默认密钥，后面的块使用前面块的加密数据做密钥
ECB方式时，使用统一的密钥

*时间：2016/6/28 20:34
*******************************************/

func main() {

	key := []byte("gotestgotestgotestgotest")
	iv := []byte("testtest")            //CBC方式解码时，如果此iv错误，可以发现结果的前面一部分解成了错误的码，后面的部分解码是正确的。这是由于CBC
	// 使用前面部分加密的结果作为自己的向量。这样就导致了前面部分会出错，后面的部分一定会正确
	originStr := `CBC方式解码时，如果此iv错误，可以发现结果的前面一部分解成了错误的码，后面的部分解码是正确的。`+
`这是由于CBC使用前面部分加密的结果作为偏移矢量。这样就导致了在相同的密码下，iv错误，则只有第一块解密会失败，后面的部分一定会正确`

	output, err := TripleDesEncrypt([]byte(originStr), key, iv)
	if err != nil {
		fmt.Printf("TripleDesEncrypt(%v) Error:%s", []byte("test"), err.Error())
		return
	}
	str := string(output)

	input, err := base64.StdEncoding.DecodeString(str)

	/*下面这个iv如果错误，则解密出来的第一块一定错误，后面则正确*/
	origData, err := TripleDesDecrypt(input, key, iv)
	if err != nil {
		fmt.Printf("TripleDesDecrypt Error:%s", err.Error())
		return
	}

	fmt.Println(string(origData))

}

// 3DES加密
func TripleDesEncrypt(origData, key, iv []byte) ([]byte, error) {
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		return nil, err
	}
	origData = PKCS5Padding(origData, block.BlockSize())
	// origData = ZeroPadding(origData, block.BlockSize())
	blockMode := cipher.NewCBCEncrypter(block, iv)
	crypted := make([]byte, len(origData))
	blockMode.CryptBlocks(crypted, origData)

	/*添加base64加密*/
	output := base64.StdEncoding.EncodeToString(crypted)
	return []byte(output), nil
}

// 3DES解密
func TripleDesDecrypt(crypted, key, iv []byte) ([]byte, error) {
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		return nil, err
	}
	blockMode := cipher.NewCBCDecrypter(block, iv)
	origData := make([]byte, len(crypted))
	// origData := crypted
	blockMode.CryptBlocks(origData, crypted)
	origData = PKCS5UnPadding(origData)
	// origData = ZeroUnPadding(origData)

	return origData, nil
}

func PKCS5Padding(ciphertext []byte, blockSize int) []byte {
	padding := blockSize - len(ciphertext) % blockSize
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}

func PKCS5UnPadding(origData []byte) []byte {
	length := len(origData)
	// 去掉最后一个字节 unpadding 次
	unpadding := int(origData[length - 1])
	return origData[:(length - unpadding)]
}
```

如上所述，需要注意的是如果iv错误，则cbc方式的情况下，解密出来的明文只有头部一小块数据错误，如使用"12345678"
作为iv进行解密后，明文如下

   _������解码时，如果此iv错误，可以发现结果的前面一部分解成了错误的码，后面的部
    分解码是正确的。这是由于CBC使用前面部分加密的结果作为偏移矢量。这样就导致了在相
    同的密码下，iv错误，则只有第一块解密会失败，后面的部分一定会正确_
