---
title: Springboot集成腾讯云对象存储COS
date: 2017-12-12 12:43:07
tags:
---
引入SDK
---
我在这个Springboot项目中是使用Maven对项目进行管理。集成COS是通过腾讯云提供的Java SDK来实现。
引入SDK只需在pom.xml中添加依赖(现在调用的是基于XML API的SDK,一个项目中只能选择基于XML或者JSON二者其中之一的SDK)。
```
<dependency>
            <groupId>com.qcloud</groupId>
            <artifactId>cos_api</artifactId>
            <version>5.2.2</version>
</dependency>
```
目前基于XML API的SDK未支持对bucker中目录的操作；而JSON则支持，但版本稍旧，且腾讯云建议使用基于XML API的SDK，所以这里选择使用基于XML API的SDK。<br>
具体实现
---
<!-- more -->
新建一个cos的工具类，这个工具类我是参照SDK的[DEMO](https://github.com/tencentyun/cos-java-sdk-v5/blob/master/src/main/java/com/qcloud/cos/demo/Demo.java)写的
```java
import com.qcloud.cos.COSClient;
import com.qcloud.cos.ClientConfig;
import com.qcloud.cos.auth.BasicCOSCredentials;
import com.qcloud.cos.auth.COSCredentials;
import com.qcloud.cos.auth.COSSigner;
import com.qcloud.cos.exception.CosClientException;
import com.qcloud.cos.exception.CosServiceException;
import com.qcloud.cos.http.HttpMethodName;
import com.qcloud.cos.model.ObjectMetadata;
import com.qcloud.cos.model.PutObjectRequest;
import com.qcloud.cos.model.PutObjectResult;
import com.qcloud.cos.model.UploadResult;
import com.qcloud.cos.region.Region;
import com.qcloud.cos.transfer.TransferManager;
import com.qcloud.cos.transfer.Upload;
import com.qcloud.cos.utils.UrlEncoderUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import java.io.File;
import java.io.InputStream;
import java.sql.Time;
import java.util.Map;
import java.util.Timer;
import java.util.TimerTask;

/**
 * 腾讯云对象存储工具类
 */
@Service
public class CosUtils {
    private Logger logger = LoggerFactory.getLogger(CosUtils.class);
    @Autowired
    private SecretkeyProperties secretkeyProperties;	//这个类是用于获取配置文件中的appid、secretId、secretKey，需要自己写好

    //生成COS客户端的公共方法
    public COSClient commonSetting(){
        String appId = secretkeyProperties.getAppid();
        String secretId = secretkeyProperties.getSecretId();
        String secretKey = secretkeyProperties.getSecretKey();
        // 1 初始化身份信息(appid, secretId, secretKey)
        COSCredentials cred = new BasicCOSCredentials(appId, secretId, secretKey);
        // 2 设置 bucket 的区域, COS 地域的简称请参照 https://cloud.tencent.com/document/product/436/6224
        ClientConfig clientConfig = new ClientConfig(new Region("ap-guangzhou"));
        // 3 生成 cos 客户端
        COSClient cosClient = new COSClient(cred, clientConfig);
        return cosClient;
    }

    /**
     * Local File Upload
     * @param bucketName    COS 存储桶名
     * @param localFilePath 要上传的本地文件路径
     * @param cosFilePath   上传到COS的路径
     * @return
     */
    public UniformResult localFileUploda(String bucketName, String localFilePath, String cosFilePath){
        UniformResult uniformResult = new UniformResult();	//这是自己写的统一返回格式对象，下同
        uniformResult.setTarget("LocalFileUpload");
        try {
            COSClient cosClient = commonSetting();  //获取Cos客户端
            File localFile = new File(localFilePath);  //本地文件
            PutObjectRequest putObjectRequest = new PutObjectRequest(bucketName, cosFilePath, localFile);
            PutObjectResult putObjectResult = cosClient.putObject(putObjectRequest);
            uniformResult.setState("00");
            uniformResult.setMsg(putObjectResult.getETag());
        }catch (Exception e){
            uniformResult.setState("01");
            uniformResult.setMsg("LocalFileUploadError");
        }
        return uniformResult;
    }

    /**
     * InputStream Upload    自动文件流上传(自动分块)
     * @param bucketName    COS 存储桶名
     * @param file          要上传的文件
     * @param cosFilePath   上传到COS的路径
     * @return
     */
    public UniformResult inputStreamUpload(String bucketName, MultipartFile file, String cosFilePath){
        UniformResult uniformResult = new UniformResult();
        uniformResult.setTarget("InputStreamUpload");
        try{
            InputStream inputStream = file.getInputStream();
            COSClient cosClient = commonSetting();  //获取Cos客户端
            TransferManager transferManager = new TransferManager(cosClient);   // 生成 TransferManager
            ObjectMetadata objectMetadata = new ObjectMetadata();
            objectMetadata.setContentLength(file.getSize());
            PutObjectRequest putObjectRequest = new PutObjectRequest(bucketName, cosFilePath, inputStream, objectMetadata);
            Upload upload = transferManager.upload(putObjectRequest);// 上传文件输入流
            // 等待传输结束（如果想同步的等待上传结束，则调用 waitForCompletion）
            UploadResult uploadResult = upload.waitForUploadResult();
            transferManager.shutdownNow();
            inputStream.close();
            uniformResult.setState("00");
            uniformResult.setMsg(uploadResult.getETag());
        }catch (Exception e){
            uniformResult.setState("01");
            uniformResult.setMsg("InputStreamUploadError");
        }
        return uniformResult;
    }

    /**
     * InputStream Simple Upload    简单文件流上传
     * @param bucketName           COS 存储桶名
     * @param file                 要上传的文件
     * @param cosFilePath          上传到COS的路径
     * @return
     */
    public UniformResult inputStreamSimpleUpload(String bucketName, MultipartFile file, String cosFilePath){
        UniformResult uniformResult = new UniformResult();
        uniformResult.setTarget("InputStreamSimpleUpload");
        try{
            InputStream inputStream = file.getInputStream();
            COSClient cosClient = commonSetting();  //获取Cos客户端
            ObjectMetadata objectMetadata = new ObjectMetadata();
            objectMetadata.setContentLength(file.getSize());
            PutObjectResult putObjectResult = cosClient.putObject(bucketName, cosFilePath, inputStream, objectMetadata);
            inputStream.close();
            uniformResult.setState("00");
            uniformResult.setMsg(putObjectResult.getETag());
        }catch (Exception e){
            uniformResult.setState("01");
            uniformResult.setMsg("InputStreamSimpleUploadError");
        }
        return uniformResult;
    }
}
```
为了安全起见，我把腾讯云的密钥信息放在了application-xxx.yml这个配置文件里面了，关于如何读取配置文件以后我有机会可以再补充介绍。
目前上传文件有个还没实现的比较重要的功能就是实时的上传进度，等实现了再在这篇文章里面更新。